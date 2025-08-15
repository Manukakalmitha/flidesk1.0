# FliDESK Checkout Data Flow System

This document describes the new checkout data flow system that saves onboarding and checkout data to Supabase, then processes it after successful payment to create user subscriptions and send confirmation emails.

## Overview

The system now stores all checkout session data (including wizard/onboarding data) in a Supabase `checkout_sessions` table. After successful payment, edge functions retrieve this data to create user subscriptions and send confirmation emails with generated FliDESK IDs.

## Architecture

```
User Checkout → Save to Supabase → PayHere Payment → Edge Function Processing → User Subscription + Email
```

## Database Schema

### New Table: `checkout_sessions`

```sql
CREATE TABLE public.checkout_sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id TEXT UNIQUE NOT NULL,
  email TEXT NOT NULL,
  business_name TEXT NOT NULL,
  plan_name TEXT NOT NULL,
  plan_id TEXT NOT NULL,
  amount DECIMAL(10,2) NOT NULL,
  phone TEXT,
  service TEXT NOT NULL,
  plan_config JSONB,
  wizard_data JSONB,
  account_info JSONB,
  business_info JSONB,
  service_preferences JSONB,
  budget_info JSONB,
  timeline_info JSONB,
  status TEXT NOT NULL DEFAULT 'pending',
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
  expires_at TIMESTAMP WITH TIME ZONE DEFAULT (now() + interval '24 hours')
);
```

### Existing Table: `user_subscriptions`

The system uses the existing `user_subscriptions` table to store final subscription data after payment processing.

## Data Flow

### 1. Checkout Process

1. **User fills checkout form** with email, business name, phone
2. **Wizard data is collected** from localStorage (if available)
3. **Session ID is generated** using a unique identifier
4. **Data is saved to Supabase** via `saveCheckoutSessionToSupabase()`
5. **User is redirected to PayHere** for payment

### 2. Payment Processing

1. **PayHere processes payment** and redirects back
2. **payment-success edge function** receives the callback
3. **process-subscription edge function** is called with session data
4. **Checkout session is retrieved** from Supabase using session_id
5. **User subscription is created** in `user_subscriptions` table
6. **FliDESK ID is generated** using the database function
7. **Confirmation email is sent** with FliDESK ID
8. **User is redirected to FliDESK** with success parameters

## Key Components

### Frontend

- **supabaseCheckoutManager.ts**: Utility functions for Supabase operations
- **Integration with existing checkout flow**: Seamless integration with current system

### Edge Functions

- **payment-success**: Receives PayHere callback and triggers subscription processing
- **process-subscription**: Creates user subscriptions and sends confirmation emails
- **flidesk-auth**: Handles final redirect to FliDESK application

### Database Functions

- **generate_flidesk_id()**: Generates unique FliDESK IDs
- **cleanup_expired_checkout_sessions()**: Cleans up old session data
- **get_checkout_session()**: Retrieves checkout session data

## Environment Variables

The edge functions require these environment variables:

```bash
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
```

## Usage

### Starting a Checkout

```typescript
import { saveCheckoutSessionToSupabase } from '@/lib/supabaseCheckoutManager';

const checkoutData = {
  sessionId: generateUniqueId(),
  email: 'user@example.com',
  businessName: 'Example Business',
  // ... other data
};

const result = await saveCheckoutSessionToSupabase(checkoutData);
if (result.success) {
  // Redirect to PayHere
  window.location.href = planConfig.payhereUrl;
}
```

### Processing Payment Success

The system automatically processes successful payments through the edge function chain:

1. PayHere redirects to `payment-success` with order_id
2. `payment-success` calls `process-subscription` with session data
3. `process-subscription` creates user subscription and sends email
4. User is redirected to FliDESK with success parameters

## Error Handling

- **Supabase save failures**: Logged as warnings, checkout continues
- **Edge function failures**: Logged, fallback redirects are used
- **Email failures**: Logged as warnings, don't fail the process
- **Session not found**: Returns 404, user can retry checkout

## Security

- **Row Level Security (RLS)** enabled on all tables
- **Service role key** used only in edge functions
- **Session expiration** after 24 hours
- **Status tracking** prevents duplicate processing

## Monitoring

The system logs key events for monitoring:

- Checkout session creation
- Payment success/failure
- Subscription creation
- Email sending status
- Error conditions

## Troubleshooting

### Common Issues

1. **Session not found**: Check if session_id matches between checkout and callback
2. **Supabase connection errors**: Verify environment variables and network access
3. **Email not sent**: Check email service configuration
4. **Duplicate subscriptions**: Verify session status tracking

### Debug Tools

- Browser console logs checkout data storage
- Edge function logs show processing steps
- Database queries can verify data integrity
- Test script provides comprehensive testing

## Future Enhancements

- **Email service integration** (SendGrid, AWS SES)
- **Webhook support** for real-time payment notifications
- **Analytics tracking** for checkout funnel optimization
- **A/B testing** for checkout flow optimization
- **Multi-currency support** beyond LKR
- **Subscription management** dashboard

## Integration with Existing System

The new checkout system is designed to work alongside your existing FliDESK infrastructure:

- **No breaking changes** to existing functionality
- **Seamless integration** with current user management
- **Backward compatibility** maintained
- **Enhanced data persistence** for better user experience

## Testing

Use the provided test script to verify system functionality:

```bash
cd fli-desk
node scripts/test-checkout-system.js
```

This will test:
- Database connectivity
- Table structure
- Edge function accessibility
- Data insertion/retrieval
- FliDESK ID generation
