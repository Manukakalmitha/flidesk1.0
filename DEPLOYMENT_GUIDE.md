# Deployment Guide - FliDESK Checkout Data Flow System

This guide explains how to deploy the new checkout data flow system to the FliDESK web application.

## Prerequisites

- Supabase project with database access
- Supabase CLI installed and configured
- Access to your Supabase project dashboard
- Environment variables configured

## Step 1: Database Migration

### Run the Migration

```bash
# Navigate to your fli-desk project directory
cd fli-desk

# Apply the migration
supabase db push

# Or if you prefer to run the SQL directly:
supabase db reset
```

### Verify Table Creation

```bash
# Check if the table was created
supabase db diff

# Or connect to your database and run:
\dt checkout_sessions
```

## Step 2: Deploy Edge Functions

### Deploy All Functions

```bash
# Deploy all edge functions
supabase functions deploy

# Or deploy individually:
supabase functions deploy payment-success
supabase functions deploy process-subscription
supabase functions deploy flidesk-auth
```

### Verify Deployment

```bash
# List deployed functions
supabase functions list

# Check function status
supabase functions logs payment-success
supabase functions logs process-subscription
supabase functions logs flidesk-auth
```

## Step 3: Environment Configuration

### Set Environment Variables

```bash
# Set Supabase URL
supabase secrets set SUPABASE_URL=https://your-project.supabase.co

# Set service role key (for admin operations)
supabase secrets set SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
```

### Verify Environment Variables

```bash
# List all secrets
supabase secrets list

# Test environment variable access
supabase functions invoke process-subscription --body '{"test": true}'
```

## Step 4: Test the System

### Run the Test Script

```bash
# Install dependencies if needed
npm install @supabase/supabase-js

# Run the test script
node scripts/test-checkout-system.js
```

### Manual Testing

1. **Test Checkout Flow**:
   - Go to your checkout page
   - Fill out the form
   - Check browser console for Supabase save logs
   - Verify data appears in `checkout_sessions` table

2. **Test Edge Functions**:
   - Test `payment-success` endpoint
   - Test `process-subscription` endpoint
   - Test `flidesk-auth` endpoint

3. **Test Database Operations**:
   - Verify checkout session creation
   - Verify user subscription creation
   - Verify FliDESK ID generation

## Step 5: Production Configuration

### Update PayHere Configuration

Ensure your PayHere URLs point to the correct edge function endpoints:

```typescript
// In your plan configurations
{
  payhereUrl: 'https://your-project.supabase.co/functions/v1/payment-success'
}
```

### Update Frontend URLs

Update any hardcoded URLs in your frontend code to match your production Supabase project.

### Configure Email Service

The `process-subscription` function includes a placeholder for email sending. Configure your preferred email service:

```typescript
// Example with SendGrid
import sgMail from '@sendgrid/mail';
sgMail.setApiKey(process.env.SENDGRID_API_KEY);

async function sendConfirmationEmail(checkoutSession, flideskId) {
  const msg = {
    to: checkoutSession.email,
    from: 'noreply@yourdomain.com',
    subject: `Welcome to FliDESK! Your ID: ${flideskId}`,
    html: generateEmailHTML(checkoutSession, flideskId)
  };
  
  return await sgMail.send(msg);
}
```

## Step 6: Monitoring and Maintenance

### Set Up Logging

```bash
# Monitor function logs
supabase functions logs --follow

# Filter by function
supabase functions logs payment-success --follow
```

### Database Maintenance

```sql
-- Clean up expired sessions (run periodically)
SELECT cleanup_expired_checkout_sessions();

-- Check session status
SELECT status, COUNT(*) FROM checkout_sessions GROUP BY status;

-- Monitor subscription creation
SELECT created_at, COUNT(*) FROM user_subscriptions 
WHERE created_at > NOW() - INTERVAL '24 hours' 
GROUP BY DATE(created_at);
```

### Health Checks

Create a simple health check endpoint:

```typescript
// Add to one of your edge functions
if (req.url.includes('/health')) {
  return new Response(JSON.stringify({
    status: 'healthy',
    timestamp: new Date().toISOString(),
    functions: ['payment-success', 'process-subscription', 'flidesk-auth']
  }), {
    status: 200,
    headers: { "Content-Type": "application/json" }
  });
}
```

## Troubleshooting

### Common Issues

1. **Function Not Deployed**:
   ```bash
   supabase functions deploy --project-ref your-project-ref
   ```

2. **Environment Variables Not Set**:
   ```bash
   supabase secrets list
   supabase secrets set KEY=value
   ```

3. **Database Connection Issues**:
   - Verify `SUPABASE_URL` is correct
   - Check if `SUPABASE_SERVICE_ROLE_KEY` has proper permissions
   - Ensure RLS policies allow the operations

4. **CORS Issues**:
   - Verify CORS headers in edge functions
   - Check if your domain is allowed

### Debug Commands

```bash
# Check function status
supabase functions list

# View function logs
supabase functions logs function-name --follow

# Test function locally
supabase functions serve function-name

# Check database connection
supabase db diff
```

## Security Considerations

1. **Service Role Key**: Only use in edge functions, never expose to frontend
2. **RLS Policies**: Ensure proper access control on all tables
3. **Input Validation**: Validate all inputs in edge functions
4. **Rate Limiting**: Consider implementing rate limiting for production
5. **HTTPS Only**: Ensure all endpoints use HTTPS

## Performance Optimization

1. **Database Indexes**: The migration creates necessary indexes
2. **Connection Pooling**: Supabase handles this automatically
3. **Caching**: Consider Redis for frequently accessed data
4. **Async Processing**: Use queues for non-critical operations

## Rollback Plan

If you need to rollback:

```bash
# Revert database changes
supabase db reset --db-only

# Revert edge functions
supabase functions deploy --project-ref old-project-ref

# Or restore from backup
supabase db restore backup-file.sql
```

## Support

For issues or questions:

1. Check Supabase logs and documentation
2. Review the troubleshooting section above
3. Test with the provided test script
4. Check GitHub issues for similar problems

## Next Steps

After successful deployment:

1. **Monitor the system** for the first few days
2. **Set up alerts** for critical failures
3. **Document any customizations** made during deployment
4. **Plan for scaling** as your user base grows
5. **Consider implementing** additional features like webhooks
