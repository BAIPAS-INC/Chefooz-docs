# Payout Provider Setup Guide

## Overview

This document describes the **provider-agnostic payout integration** for automated withdrawals in the Chefooz application. The system supports **Razorpay**, **Cashfree**, and **Stripe** through a unified interface.

## Architecture

### Components

1. **PayoutProvider Interface** - Common contract for all providers
2. **Provider Adapters** - Implementation for each payment gateway
3. **Factory Pattern** - Dynamic provider selection via environment variables
4. **PayoutService** - Business logic for processing withdrawals
5. **PayoutController** - Webhook and admin endpoints
6. **Cron Job** - Automatic processing of approved withdrawals

### Provider Support

| Provider | Status | Notes |
|----------|--------|-------|
| Razorpay | ✅ Implemented | Razorpay X Payouts API |
| Cashfree | ✅ Implemented | Cashfree Payouts API |
| Stripe | ✅ Implemented | Stripe Payouts API |

## Environment Variables

Add these variables to your `.env` file (root of monorepo):

```bash
# ═══════════════════════════════════════════════════════
# PAYOUT PROVIDER CONFIGURATION
# ═══════════════════════════════════════════════════════

# Select provider: 'razorpay', 'cashfree', or 'stripe'
PAYOUT_PROVIDER=razorpay

# Enable automatic payout processing (every 10 minutes)
AUTO_PAYOUT_ENABLED=true

# Webhook signature verification secret
PAYOUT_WEBHOOK_SECRET=your_webhook_secret_here

# ───────────────────────────────────────────────────────
# RAZORPAY CREDENTIALS
# ───────────────────────────────────────────────────────
RAZORPAY_KEY_ID=rzp_test_xxxxxxxxxxxxx
RAZORPAY_KEY_SECRET=your_razorpay_secret_key
RAZORPAY_PAYOUT_ACCOUNT=your_razorpay_account_id

# ───────────────────────────────────────────────────────
# CASHFREE CREDENTIALS
# ───────────────────────────────────────────────────────
CASHFREE_CLIENT_ID=your_cashfree_client_id
CASHFREE_CLIENT_SECRET=your_cashfree_client_secret
CASHFREE_ENV=test  # or 'production'

# ───────────────────────────────────────────────────────
# STRIPE CREDENTIALS
# ───────────────────────────────────────────────────────
STRIPE_SECRET_KEY=sk_test_xxxxxxxxxxxxx
STRIPE_PAYOUT_ACCOUNT=acct_xxxxxxxxxxxxx
```

## Database Schema Updates

The `withdrawal_requests` table has been extended with provider fields:

```sql
ALTER TABLE withdrawal_requests 
ADD COLUMN provider VARCHAR(50),
ADD COLUMN provider_payout_id VARCHAR(255) UNIQUE,
ADD COLUMN utr VARCHAR(255),
ADD COLUMN processed_at TIMESTAMP,
ADD COLUMN failure_reason TEXT;

CREATE INDEX idx_provider_payout_id ON withdrawal_requests(provider_payout_id);
```

## API Endpoints

### 1. Webhook Endpoint

**POST** `/api/v1/payouts/webhook`

Receives status updates from payment providers.

**Headers:**
- `x-webhook-signature` - Generic signature (or provider-specific)
- `x-razorpay-signature` - Razorpay webhook signature
- `x-cashfree-signature` - Cashfree webhook signature
- `stripe-signature` - Stripe webhook signature

**Request Body:** Provider-specific webhook payload

**Response:**
```json
{
  "success": true,
  "message": "Webhook processed successfully"
}
```

**Provider Webhook URLs:**
- Razorpay: `https://your-domain.com/api/v1/payouts/webhook`
- Cashfree: `https://your-domain.com/api/v1/payouts/webhook`
- Stripe: `https://your-domain.com/api/v1/payouts/webhook`

### 2. Manual Processing (Admin)

**POST** `/api/v1/payouts/admin/process`

Manually trigger payout processing for all approved withdrawals.

**Headers:**
- `Authorization: Bearer <admin-jwt-token>`

**Response:**
```json
{
  "success": true,
  "message": "Payout processing completed"
}
```

### 3. Force Process Specific Withdrawal (Admin)

**POST** `/api/v1/payouts/admin/force/:withdrawalId`

Force process a specific approved withdrawal.

**Headers:**
- `Authorization: Bearer <admin-jwt-token>`

**Response:**
```json
{
  "success": true,
  "message": "Withdrawal processed successfully"
}
```

## Webhook Payloads

### Razorpay Webhook

```json
{
  "event": "payout.processed",
  "payload": {
    "payout": {
      "entity": {
        "id": "pout_xxxxxxxxxxxxx",
        "status": "processed",
        "amount": 100000,
        "currency": "INR",
        "utr": "UTR123456789",
        "reference_id": "withdrawal_uuid",
        "processed_at": 1234567890
      }
    }
  }
}
```

**Events:**
- `payout.processed` → Payout successful
- `payout.failed` → Payout failed
- `payout.reversed` → Payout reversed
- `payout.pending` → Payout pending

### Cashfree Webhook

```json
{
  "type": "TRANSFER_SUCCESS",
  "data": {
    "transferId": "withdrawal_uuid",
    "status": "SUCCESS",
    "amount": "1000.00",
    "utr": "UTR123456789",
    "processedOn": "2025-11-10T12:00:00Z"
  }
}
```

**Status Values:**
- `SUCCESS` → Payout successful
- `FAILED` → Payout failed
- `REVERSED` → Payout reversed

### Stripe Webhook

```json
{
  "type": "payout.paid",
  "data": {
    "object": {
      "id": "po_xxxxxxxxxxxxx",
      "status": "paid",
      "amount": 100000,
      "currency": "inr",
      "metadata": {
        "withdrawal_id": "withdrawal_uuid"
      },
      "arrival_date": 1234567890
    }
  }
}
```

**Events:**
- `payout.paid` → Payout successful
- `payout.failed` → Payout failed
- `payout.canceled` → Payout canceled

## Workflow

### 1. User Requests Withdrawal

User submits a withdrawal request via the mobile app.

**Status:** `REQUESTED`

### 2. Admin Approval

Admin reviews and approves the withdrawal.

**Status:** `REQUESTED` → `APPROVED`

### 3. Automatic Payout Processing

Cron job (or manual trigger) processes approved withdrawals:

1. Find all `APPROVED` withdrawals without `providerPayoutId`
2. Create payout via selected provider
3. Save `providerPayoutId` and `provider` name
4. Provider processes the payout asynchronously

**Status:** `APPROVED` (with `providerPayoutId`)

### 4. Webhook Updates Status

Provider sends webhook when payout completes:

- **Success:** Status → `PAID`, save `utr` and `processedAt`
- **Failed:** Status → `REJECTED`, refund coins, save `failureReason`
- **Reversed:** Status → `REJECTED`, refund coins

## Testing

### Local Testing

1. **Set environment variables:**
   ```bash
   PAYOUT_PROVIDER=razorpay
   AUTO_PAYOUT_ENABLED=false  # Disable cron for manual control
   ```

2. **Create and approve a withdrawal:**
   ```bash
   # Use the E2E test or mobile app to create withdrawal
   # Then approve via admin endpoint or database update
   UPDATE withdrawal_requests SET status = 'approved' WHERE id = 'xxx';
   ```

3. **Trigger manual payout:**
   ```bash
   curl -X POST http://localhost:3333/api/v1/payouts/admin/process \
     -H "Authorization: Bearer YOUR_ADMIN_JWT"
   ```

4. **Simulate webhook:**
   ```powershell
   $body = @{
       event = "payout.processed"
       payload = @{
           payout = @{
               entity = @{
                   id = "pout_test123"
                   status = "processed"
                   reference_id = "withdrawal_uuid_here"
                   utr = "UTR123456789"
               }
           }
       }
   } | ConvertTo-Json -Depth 5

   Invoke-WebRequest -Uri "http://localhost:3333/api/v1/payouts/webhook" `
     -Method POST `
     -ContentType "application/json" `
     -Body $body `
     -Headers @{"x-webhook-signature"="test"}
   ```

### Production Testing

1. **Configure provider in test mode:**
   - Razorpay: Use `rzp_test_` key
   - Cashfree: Set `CASHFREE_ENV=test`
   - Stripe: Use `sk_test_` key

2. **Set up webhook URL in provider dashboard:**
   - URL: `https://your-production-domain.com/api/v1/payouts/webhook`
   - Set webhook secret in `.env` as `PAYOUT_WEBHOOK_SECRET`

3. **Enable auto-payout:**
   ```bash
   AUTO_PAYOUT_ENABLED=true
   ```

4. **Monitor logs:**
   ```bash
   # Watch for payout processing
   tail -f /var/log/chefooz-apis.log | grep -i payout
   ```

## Cron Job

The auto-payout cron job runs every 10 minutes:

```typescript
// To enable, install @nestjs/schedule:
// npm install @nestjs/schedule

// Then add @Cron decorator to handleAutoPayouts() in payout.service.ts:
@Cron(CronExpression.EVERY_10_MINUTES)
async handleAutoPayouts(): Promise<void> {
  // ...
}
```

**Manual control:**
- Disable: `AUTO_PAYOUT_ENABLED=false`
- Enable: `AUTO_PAYOUT_ENABLED=true`

## Error Handling

### Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `MISSING_CREDENTIALS` | Provider credentials not set | Check `.env` file |
| `UNSUPPORTED_PROVIDER` | Invalid `PAYOUT_PROVIDER` value | Use: razorpay, cashfree, or stripe |
| `INVALID_SIGNATURE` | Webhook signature mismatch | Verify `PAYOUT_WEBHOOK_SECRET` |
| `Withdrawal not found` | Provider payout ID not in DB | Check webhook payload format |

### Retry Logic

- **Failed payouts** are marked as `REJECTED` and coins are refunded
- **Admin can retry** by manually approving again and triggering process
- **Duplicate prevention** via unique `providerPayoutId` constraint

## Security

### Webhook Verification

All webhooks are verified using HMAC-SHA256 signature:

```typescript
const expectedSignature = crypto
  .createHmac('sha256', PAYOUT_WEBHOOK_SECRET)
  .update(JSON.stringify(payload))
  .digest('hex');

if (signature !== expectedSignature) {
  throw new Error('Invalid signature');
}
```

### Best Practices

1. **Never commit secrets** - Use `.env` and `.gitignore`
2. **Use HTTPS** - Webhooks must use secure URLs
3. **Rotate secrets** - Change `PAYOUT_WEBHOOK_SECRET` periodically
4. **Monitor logs** - Watch for unauthorized access attempts
5. **Rate limiting** - Add rate limits to webhook endpoint (optional)

## Monitoring

### Key Metrics

- **Payout success rate:** Processed / Total attempted
- **Average processing time:** Time from approval to paid
- **Failed payout rate:** Failed / Total attempted
- **Webhook latency:** Time between provider event and DB update

### Logging

Key log entries to monitor:

```
[PayoutService] Payout provider initialized: razorpay
[PayoutService] Running auto-payout cron job...
[PayoutService] Processing 5 approved withdrawals
[PayoutService] Payout created: pout_xxx for withdrawal xxx
[PayoutController] Webhook processed: PAYOUT.PROCESSED for withdrawal xxx
[PayoutService] Withdrawal xxx marked as PAID
```

## Production Deployment Checklist

- [ ] Set production provider credentials
- [ ] Configure webhook URL in provider dashboard
- [ ] Set strong `PAYOUT_WEBHOOK_SECRET`
- [ ] Enable `AUTO_PAYOUT_ENABLED=true`
- [ ] Test webhook with provider test tool
- [ ] Monitor first few payouts manually
- [ ] Set up alerting for failed payouts
- [ ] Document provider account details securely
- [ ] Train admin team on manual processing
- [ ] Test coin refund flow for failed payouts

## Support

For provider-specific documentation:

- **Razorpay:** https://razorpay.com/docs/api/razorpayx/payouts/
- **Cashfree:** https://docs.cashfree.com/reference/payouts-api-overview
- **Stripe:** https://stripe.com/docs/api/payouts

For Chefooz-specific issues, contact the development team.

---

**Last Updated:** November 10, 2025  
**Version:** 1.0.0
