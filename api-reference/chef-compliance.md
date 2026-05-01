# Chef Compliance API Reference

**Last Updated**: 2026-05-01  
**Base path**: `/api/v1`

---

## Chef-facing Endpoints

### `GET /api/v1/chef/compliance/status`

Returns the current compliance checklist state for the authenticated chef.

**Guards**: `JwtAuthGuard`

### `POST /api/v1/chef/compliance/upload-url`

Requests a pre-signed S3 PUT URL for a compliance document upload.

**Guards**: `JwtAuthGuard`

### `POST /api/v1/chef/compliance/identity`

Submits KYC identity documents.

**Guards**: `JwtAuthGuard`

### `POST /api/v1/chef/compliance/bank`

Submits payout bank or UPI details.

**Guards**: `JwtAuthGuard`

### `POST /api/v1/chef/compliance/fssai`

Submits the FSSAI licence and certificate.

**Guards**: `JwtAuthGuard`

### `POST /api/v1/chef/compliance/legal`

Records legal terms acceptance.

**Guards**: `JwtAuthGuard`

---

## Admin Endpoints

### `GET /api/v1/admin/chef-compliance/:chefId`

Returns the full unmasked compliance record for a chef so an admin can review pending KYC items.

**Guards**: `JwtAuthGuard` + `RolesGuard` (`admin`)

#### Response

```json
{
  "success": true,
  "message": "Compliance record retrieved",
  "data": {
    "chefId": "uuid",
    "status": "partial",
    "canReceivePayout": false,
    "chefVerificationStatus": "pending",
    "chefVerified": false,
    "payoutEnabled": false,
    "canVerifyChef": true,
    "canEnablePayout": false,
    "identityApproved": true,
    "bankApproved": false,
    "fssaiApproved": true,
    "adminNotes": "Chef re-submitted identity docs",
    "identityVerified": false,
    "identityData": {
      "documentType": "aadhaar",
      "documentNumber": "123456789012",
      "documentFrontUrl": "https://cdn.chefooz.com/compliance/front.jpg",
      "documentBackUrl": "https://cdn.chefooz.com/compliance/back.jpg",
      "selfieUrl": "https://cdn.chefooz.com/compliance/selfie.jpg",
      "submittedAt": "2026-05-01T08:00:00.000Z",
      "verifiedAt": null,
      "verifiedBy": null,
      "rejectionReason": null
    },
    "bankVerified": true,
    "bankData": {
      "accountHolderName": "Chef One",
      "accountNumber": "1234567890",
      "ifscCode": "SBIN0001234",
      "bankName": "State Bank of India",
      "preferredMode": "bank",
      "submittedAt": "2026-05-01T08:05:00.000Z",
      "verifiedAt": "2026-05-01T09:00:00.000Z",
      "verifiedBy": "admin-42"
    },
    "fssaiSubmitted": false,
    "legalAccepted": true,
    "legalData": {
      "termsVersion": "v2.1",
      "acceptedAt": "2026-05-01T08:15:00.000Z"
    }
  }
}
```

### `PATCH /api/v1/admin/chef-compliance/:chefId/identity`

Verifies or rejects the identity step.

### `PATCH /api/v1/admin/chef-compliance/:chefId/bank`

Verifies or rejects the bank step.

### `PATCH /api/v1/admin/chef-compliance/:chefId/fssai`

Verifies or rejects the FSSAI step.

#### Request Body

```json
{
  "verified": false,
  "rejectionReason": "Document image is blurry",
  "adminNotes": "Asked chef to re-upload front image"
}
```

#### Notes

- `rejectionReason` is required when `verified = false`
- Successful verification clears any prior rejection reason for that step
- `verifiedBy` is set from the authenticated admin JWT

### `PATCH /api/v1/admin/chef-compliance/:chefId/notes`

Updates internal admin notes for the compliance record.

#### Request Body

```json
{
  "notes": "Pending clearer FSSAI certificate scan"
}

### `PATCH /api/v1/admin/chef-compliance/:chefId/chef-status`

Marks the chef verified for operations so they can go online and accept orders.

#### Request Body

```json
{
  "enabled": true,
  "adminNotes": "Identity and FSSAI approved. Chef cleared for operations."
}
```

#### Rules

- Chef verification requires admin-approved identity
- Chef verification requires admin-approved FSSAI
- Legal acceptance is required
- Bank review is not required for chef operations

### `PATCH /api/v1/admin/chef-compliance/:chefId/payout`

Enables payouts after the chef is already verified for operations and bank details are admin-approved.

#### Request Body

```json
{
  "enabled": true,
  "adminNotes": "Bank details reviewed and payout enabled."
}
```

#### Rules

- Payout enablement requires chef operations already verified
- Payout enablement requires admin-approved bank details
- Disabling payout does not revoke chef operational verification
```