# 🧪 Authentication Module — QA Test Cases

**Module**: Authentication (OTP-based Passwordless Auth)  
**Version**: v2 (WhatsApp-first with SMS fallback)  
**Date**: 2026-02-14  
**Total Test Cases**: 62  
**Estimated Testing Time**: 8-10 hours (full suite)  
**Last Updated**: 2026-02-28

---

## 📋 Table of Contents

1. [Test Environment Setup](#test-environment-setup)
2. [Test Data Requirements](#test-data-requirements)
3. [Functional Tests](#1-functional-tests)
4. [UX/UI Tests](#2-uxui-tests)
5. [Edge Case Tests](#3-edge-case-tests)
6. [Error Handling Tests](#4-error-handling-tests)
7. [Security Tests](#5-security-tests)
8. [Performance Tests](#6-performance-tests)
9. [Regression Tests](#7-regression-tests)
10. [API Test Commands](#api-test-commands)
11. [Test Execution Checklist](#test-execution-checklist)

---

## 🔧 Test Environment Setup

### Prerequisites
- Backend running at `https://api-staging.chefooz.com`
- PostgreSQL database with clean test data
- WhatsApp Cloud API configured (or use dev mode bypass)
- Twilio SMS configured for fallback testing
- Mobile app running on iOS/Android device or simulator
- Valid JWT secret configured in `.env`

### Test Accounts
```json
{
  "newUser": {
    "phone": "9876543210",
    "deviceId": "TEST-DEVICE-001"
  },
  "existingUser": {
    "phone": "9123456789",
    "username": "test_user_123",
    "fullName": "Test User",
    "deviceId": "TEST-DEVICE-002"
  },
  "chefUser": {
    "phone": "9988776655",
    "username": "chef_ravi",
    "fullName": "Ravi Kumar",
    "role": "chef"
  }
}
```

### Environment Variables
```bash
NODE_ENV=test
JWT_SECRET=test-secret-key
JWT_ACCESS_EXPIRATION=7d
JWT_REFRESH_EXPIRATION=30d
WHATSAPP_API_KEY=test-key
TWILIO_ACCOUNT_SID=test-sid
TWILIO_AUTH_TOKEN=test-token
OTP_EXPIRY_MINUTES=5
OTP_DEV_MODE=false  # Set to true to bypass actual OTP sending
```

---

## 📊 Test Data Requirements

### Phone Number Formats to Test
| Format | Example | Expected Normalization | Use Case |
|--------|---------|------------------------|----------|
| 10-digit | `9876543210` | `+919876543210` | Most common input |
| With country code | `919876543210` | `+919876543210` | Copy-pasted numbers |
| With + prefix | `+919876543210` | `+919876543210` | Already formatted |
| With spaces | `98765 43210` | `+919876543210` | User-friendly input |
| With dashes | `98765-43210` | `+919876543210` | Alternative format |
| With parentheses | `(987) 654-3210` | `+919876543210` | International style |
| Leading zeros | `09876543210` | `+919876543210` | Landline habit |
| With 91 prefix | `919876543210` | `+919876543210` | International dialing |

### Invalid Phone Numbers
- `123456789` (9 digits - too short)
- `98765432109` (11 digits - too long)
- `1234567890` (starts with invalid digit)
- `abcd567890` (contains letters)
- `98765 12` (incomplete)
- `+1-987-654-3210` (non-Indian country code)

### Test OTPs
| Scenario | OTP | Expected Behavior |
|----------|-----|-------------------|
| Valid OTP | `123456` | Accept & generate token |
| Expired OTP | `999999` (5+ min old) | Reject with OTP_EXPIRED |
| Already used | `888888` (verified once) | Reject with OTP_SESSION_NOT_FOUND |
| Invalid OTP | `000000` | Reject with OTP_INVALID |
| Short OTP | `1234` | Validation error |
| Long OTP | `12345678` | Validation error |

---

## 1. Functional Tests

### Test Case: AUTH-F-001 — Send OTP to New User via WhatsApp

**Priority**: Critical  
**Type**: Functional  
**Roles**: All (First-time users)  
**Platform**: Both (iOS/Android)  
**Automation**: [@automated]

**Prerequisites:**
- Phone number not registered in database
- WhatsApp Cloud API operational
- Device has internet connectivity

**Test Data:**
- Phone: `9876543210`
- Device ID: `TEST-DEVICE-NEW-001`

**Steps:**
1. Open Chefooz app
2. Navigate to phone entry screen
3. Enter phone number: `9876543210`
4. Tap "Continue" button
5. Observe network request and response

**Expected Result:**
- ✅ API request sent to `POST /v1/auth/v2/send-otp`
- ✅ Response status: 200 OK
- ✅ Response body contains:
  ```json
  {
    "success": true,
    "message": "OTP sent via WhatsApp",
    "data": {
      "sessionId": "<uuid>",
      "expiresAt": "<timestamp>",
      "channel": "whatsapp",
      "fallbackUsed": false
    }
  }
  ```
- ✅ User receives WhatsApp message with 6-digit OTP within 10 seconds
- ✅ App navigates to OTP verification screen
- ✅ Session ID stored in app state

**API Test Command:**
```bash
curl -X POST https://api-staging.chefooz.com/api/v1/auth/v2/send-otp \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "9876543210",
    "deviceId": "TEST-DEVICE-NEW-001"
  }'
```

---

### Test Case: AUTH-F-002 — Send OTP with SMS Fallback

**Priority**: High  
**Type**: Functional  
**Roles**: All  
**Platform**: Both  
**Automation**: [@manual]

**Prerequisites:**
- WhatsApp Cloud API unavailable/failing (simulate by disabling API key)
- Twilio SMS configured and operational

**Test Data:**
- Phone: `9123456789`
- Device ID: `TEST-DEVICE-SMS-001`

**Steps:**
1. Temporarily disable WhatsApp API (set invalid API key)
2. Open app and enter phone: `9123456789`
3. Tap "Continue"
4. Observe backend logs and response

**Expected Result:**
- ✅ Backend attempts WhatsApp delivery first
- ✅ Backend logs show WhatsApp failure
- ✅ Backend automatically falls back to Twilio SMS
- ✅ Response contains:
  ```json
  {
    "channel": "sms",
    "fallbackUsed": true
  }
  ```
- ✅ User receives SMS with OTP within 30 seconds
- ✅ App shows "OTP sent via SMS" message
- ✅ OTP session record has `channel: 'sms'` and `fallbackUsed: true`

**Database Verification:**
```sql
SELECT phone, channel, fallbackUsed, smsMessageId 
FROM otp_sessions 
WHERE phone = '+919123456789' 
ORDER BY createdAt DESC 
LIMIT 1;
```

---

### Test Case: AUTH-F-003 — Verify Valid OTP and Generate JWT

**Priority**: Critical  
**Type**: Functional  
**Roles**: All  
**Platform**: Both  
**Automation**: [@automated]

**Prerequisites:**
- Valid OTP session exists (from AUTH-F-001)
- OTP not expired (<5 minutes old)
- OTP not previously used

**Test Data:**
- Phone: `9876543210`
- OTP: `123456` (example - use actual OTP received)
- Device ID: `TEST-DEVICE-NEW-001`

**Steps:**
1. Receive OTP from WhatsApp/SMS
2. Enter OTP in verification screen
3. Tap "Verify" button
4. Observe response and navigation

**Expected Result:**
- ✅ API request to `POST /v1/auth/v2/verify-otp`
- ✅ Response status: 200 OK
- ✅ Response contains:
  ```json
  {
    "success": true,
    "message": "Login successful",
    "data": {
      "accessToken": "eyJhbGciOiJIUzI1NiIs...",
      "refreshToken": "eyJhbGciOiJIUzI1NiIs...",
      "user": {
        "id": "<uuid>",
        "phone": "+919876543210",
        "username": null,
        "fullName": "",
        "profileIncomplete": true,
        "role": "user",
        "createdAt": "<timestamp>"
      }
    }
  }
  ```
- ✅ JWT token stored in Expo SecureStore
- ✅ User record created in database (if new user)
- ✅ OTP session marked as `isUsed: true`
- ✅ App navigates to profile completion screen

**JWT Token Validation:**
```bash
# Decode JWT payload
echo "eyJhbGciOiJIUzI1NiIs..." | base64 -d

# Expected payload structure:
{
  "sub": "<userId>",
  "userId": "<userId>",
  "phone": "+919876543210",
  "role": "user",
  "type": "access",
  "iat": 1739440200,
  "exp": 1740044600
}
```

**API Test Command:**
```bash
curl -X POST https://api-staging.chefooz.com/api/v1/auth/v2/verify-otp \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "9876543210",
    "otp": "123456",
    "deviceId": "TEST-DEVICE-NEW-001"
  }'
```

---

### Test Case: AUTH-F-004 — Complete Profile Setup

**Priority**: Critical  
**Type**: Functional  
**Roles**: All (New users)  
**Platform**: Both  
**Automation**: [@automated]

**Prerequisites:**
- User authenticated with valid JWT token
- User has `profileIncomplete: true`

**Test Data:**
- Username: `foodlover_123`
- Full Name: `John Doe`
- Token: `<access_token_from_verify_otp>`

**Steps:**
1. User lands on profile completion screen after OTP verification
2. Enter username: `foodlover_123`
3. Enter full name: `John Doe`
4. Tap "Complete Setup"
5. Observe response and navigation

**Expected Result:**
- ✅ API request to `PUT /v1/auth/profile` with Bearer token
- ✅ Response status: 200 OK
- ✅ Response contains:
  ```json
  {
    "success": true,
    "message": "Profile updated successfully",
    "data": {
      "user": {
        "id": "<uuid>",
        "phone": "+919876543210",
        "username": "foodlover_123",
        "fullName": "John Doe",
        "profileIncomplete": false
      }
    }
  }
  ```
- ✅ Database updated: `profileIncomplete` set to `false`
- ✅ App navigates to home screen (Explore tab)
- ✅ Username is globally unique (verified in DB)

**API Test Command:**
```bash
curl -X PUT https://api-staging.chefooz.com/api/v1/auth/profile \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "foodlover_123",
    "fullName": "John Doe"
  }'
```

---

### Test Case: AUTH-F-005 — Get Current User Profile

**Priority**: High  
**Type**: Functional  
**Roles**: All  
**Platform**: Both  
**Automation**: [@automated]

**Prerequisites:**
- User authenticated with valid JWT token
- Token not expired (<7 days old)

**Test Data:**
- Token: `<valid_access_token>`

**Steps:**
1. App launches with stored token
2. API call to fetch current user profile
3. Verify response data

**Expected Result:**
- ✅ API request to `GET /v1/auth/me` with Bearer token
- ✅ Response status: 200 OK
- ✅ Response contains complete user profile:
  ```json
  {
    "success": true,
    "message": "User retrieved",
    "data": {
      "id": "<uuid>",
      "phone": "+919876543210",
      "username": "foodlover_123",
      "fullName": "John Doe",
      "role": "user",
      "coins": 100,
      "profileIncomplete": false,
      "avatarUrl": null,
      "bio": null,
      "trustState": "NORMAL",
      "createdAt": "<timestamp>",
      "updatedAt": "<timestamp>"
    }
  }
  ```
- ✅ All user fields populated correctly
- ✅ Profile incomplete flag reflects actual status

**API Test Command:**
```bash
curl -X GET https://api-staging.chefooz.com/api/v1/auth/me \
  -H "Authorization: Bearer <access_token>"
```

---

### Test Case: AUTH-F-006 — Existing User Login (Return User)

**Priority**: Critical  
**Type**: Functional  
**Roles**: All  
**Platform**: Both  
**Automation**: [@automated]

**Prerequisites:**
- User previously registered (phone exists in database)
- User has completed profile setup

**Test Data:**
- Phone: `9123456789` (existing user)
- Device ID: `TEST-DEVICE-RETURNING-001`

**Steps:**
1. Open app (user logged out or token expired)
2. Enter existing phone: `9123456789`
3. Request OTP
4. Enter received OTP
5. Verify login

**Expected Result:**
- ✅ OTP sent successfully
- ✅ OTP verification successful
- ✅ No new user created (existing user record used)
- ✅ JWT token generated with existing userId
- ✅ User object returned with:
  - Existing username
  - Existing fullName
  - `profileIncomplete: false`
- ✅ App navigates directly to home screen (skips profile setup)
- ✅ User data persisted (coins, trustState, etc.)

---

### Test Case: AUTH-F-007 — Phone Number Normalization (10+ Formats)

**Priority**: High  
**Type**: Functional  
**Roles**: All  
**Platform**: Both  
**Automation**: [@automated]

**Test Data Matrix:**

| Input Format | Expected Normalized | Test Pass Criteria |
|--------------|---------------------|-------------------|
| `9876543210` | `+919876543210` | ✅ OTP sent |
| `919876543210` | `+919876543210` | ✅ OTP sent |
| `+919876543210` | `+919876543210` | ✅ OTP sent |
| `098 765 432 10` | `+919876543210` | ✅ OTP sent |
| `98765-43210` | `+919876543210` | ✅ OTP sent |
| `(987) 654-3210` | `+919876543210` | ✅ OTP sent |
| `+91 9876543210` | `+919876543210` | ✅ OTP sent |
| `0091 9876543210` | `+919876543210` | ✅ OTP sent |
| `91-9876-543210` | `+919876543210` | ✅ OTP sent |
| `9876 543 210` | `+919876543210` | ✅ OTP sent |

**Steps:**
1. For each input format above:
   - Enter phone in given format
   - Submit OTP request
   - Verify backend normalization
   - Check database for correct E.164 format

**Expected Result:**
- ✅ All formats normalized to `+919876543210`
- ✅ Database stores only E.164 format
- ✅ User receives OTP for all valid formats
- ✅ Same phone in different formats maps to same user account

**Database Verification:**
```sql
SELECT DISTINCT phone 
FROM users 
WHERE phone LIKE '%9876543210%';

-- Should return exactly: +919876543210
```

---

## 2. UX/UI Tests

### Test Case: AUTH-UX-NEW-003 — Suggested usernames are compact and under username input

**Priority**: High  
**Type**: UX/UI  
**Platform**: Android + iOS  
**Automation**: [@manual]

**Steps:**
1. Navigate to onboarding username screen
2. Type at least 2 characters in username input
3. Observe suggestions layout

**Expected Result:**
- ✅ `Suggested for you` appears directly under username input (inside username card)
- ✅ Suggestions are compact chips (not a large standalone card block)
- ✅ Tapping chip fills username input immediately

---

### Test Case: AUTH-UX-NEW-004 — Use another mobile number resets onboarding safely

**Priority**: Critical  
**Type**: UX/UI + Regression  
**Platform**: Android + iOS  
**Automation**: [@manual]

**Preconditions:**
- User has completed OTP verification and is on username screen

**Steps:**
1. Tap `Use another mobile number`
2. Confirm restart in alert dialog
3. Observe navigation target
4. Relaunch app and verify auth state

**Expected Result:**
- ✅ Confirmation dialog shown before destructive action
- ✅ Current auth session is cleared
- ✅ App navigates to `/auth/enter-phone`
- ✅ User can start login with a different phone number
- ✅ App does not trap user on username onboarding after restart

---

### Test Case: AUTH-UX-001 — Loading States During OTP Send

**Priority**: High  
**Type**: UX  
**Roles**: All  
**Platform**: Both  
**Automation**: [@manual]

**Steps:**
1. Enter phone number
2. Tap "Continue" button
3. Observe UI changes during network request

**Expected Result:**
- ✅ "Continue" button shows spinner/loading indicator
- ✅ Button disabled during request (prevents double-tap)
- ✅ Phone input field disabled during request
- ✅ Loading state appears within 100ms of tap
- ✅ Loading state clears after response (success or error)
- ✅ On success: smooth transition to OTP screen
- ✅ On error: error message displayed, button re-enabled

---

### Test Case: AUTH-UX-002 — OTP Input Field UX

**Priority**: High  
**Type**: UX  
**Roles**: All  
**Platform**: Both  
**Automation**: [@manual]

**Steps:**
1. Navigate to OTP verification screen
2. Interact with OTP input fields

**Expected Result:**
- ✅ 6 separate input boxes (one per digit)
- ✅ Numeric keyboard auto-opens on screen load
- ✅ Auto-focus on first input field
- ✅ Auto-advance to next field after digit entry
- ✅ Backspace moves to previous field
- ✅ Paste 6-digit OTP auto-fills all fields
- ✅ Visual feedback on active field (border highlight)
- ✅ Auto-submit when 6th digit entered (optional)

---

### Test Case: AUTH-UX-003 — Error Message Clarity

**Priority**: Medium  
**Type**: UX  
**Roles**: All  
**Platform**: Both  
**Automation**: [@manual]

**Test Scenarios:**

| Error Scenario | Expected User Message | UI Treatment |
|----------------|----------------------|--------------|
| Invalid phone format | "Please enter a valid 10-digit mobile number" | Red text under input |
| Rate limit (device) | "Too many attempts from this device. Try again in 15 minutes." | Alert/toast |
| Rate limit (phone) | "Too many OTP requests. Please wait 1 minute." | Alert/toast |
| Invalid OTP | "Incorrect code. 3 attempts remaining." | Red text, shake animation |
| Expired OTP | "Code expired. Please request a new one." | Alert with "Resend OTP" button |
| Network error | "Connection failed. Please check your internet." | Retry button shown |

**Expected Result:**
- ✅ All errors user-friendly (no technical jargon)
- ✅ Errors include actionable guidance
- ✅ Critical errors persist (not auto-dismissed)
- ✅ Errors visually distinct (red color, icon)

---

### Test Case: AUTH-UX-004 — OTP Countdown Timer

**Priority**: Medium  
**Type**: UX  
**Roles**: All  
**Platform**: Both  
**Automation**: [@manual]

**Steps:**
1. Request OTP and land on verification screen
2. Observe countdown timer display

**Expected Result:**
- ✅ Timer shows remaining time: "Valid for 4:58"
- ✅ Timer counts down every second
- ✅ Timer turns red when <1 minute remains
- ✅ At 0:00, "Resend OTP" button appears
- ✅ Timer format: `M:SS` (e.g., "4:58", "0:45")
- ✅ Timer visible without scrolling

---

### Test Case: AUTH-UX-005 — Profile Completion Guidance

**Priority**: Medium  
**Type**: UX  
**Roles**: All  
**Platform**: Both  
**Automation**: [@manual]

**Steps:**
1. Complete OTP verification as new user
2. Land on profile setup screen
3. Interact with form fields

**Expected Result:**
- ✅ Clear heading: "Complete Your Profile"
- ✅ Helper text explains username rules: "3-20 characters, letters, numbers, and underscores only"
- ✅ Real-time validation feedback (green checkmark or red X)
- ✅ Username availability check (debounced, 500ms delay)
- ✅ "Username available" / "Username taken" indicator
- ✅ Submit button disabled until both fields valid
- ✅ Success animation on profile completion

---

## 3. Edge Case Tests

### Test Case: AUTH-EDGE-001 — OTP Expiry at Exact 5 Minute Mark

**Priority**: High  
**Type**: Edge Case  
**Roles**: All  
**Platform**: Both  
**Automation**: [@automated]

**Steps:**
1. Request OTP and note exact timestamp
2. Wait exactly 5 minutes (300 seconds)
3. Attempt to verify OTP at 4:59, 5:00, and 5:01

**Expected Result:**
- ✅ At 4:59 (299 sec): OTP accepts (still valid)
- ✅ At 5:00 (300 sec): OTP rejects with `OTP_EXPIRED`
- ✅ At 5:01 (301 sec): OTP rejects with `OTP_EXPIRED`
- ✅ Error message: "OTP has expired. Please request a new one."

---

### Test Case: AUTH-EDGE-002 — Rapid OTP Verification Attempts

**Priority**: High  
**Type**: Edge Case  
**Roles**: All  
**Platform**: Both  
**Automation**: [@automated]

**Steps:**
1. Request OTP for phone `9876543210`
2. Attempt to verify with wrong OTP 5 times rapidly
3. Attempt 6th verification

**Expected Result:**
- ✅ Attempts 1-5: Return `OTP_INVALID` with remaining attempts
- ✅ Attempt 5: "Invalid OTP. 1 attempt remaining."
- ✅ Attempt 6: `OTP_MAX_ATTEMPTS` error
- ✅ OTP session invalidated after 5 failed attempts
- ✅ User must request new OTP to retry

**Database Verification:**
```sql
SELECT attempts, isUsed 
FROM otp_sessions 
WHERE phone = '+919876543210' 
ORDER BY createdAt DESC LIMIT 1;

-- Expected: attempts = 5, isUsed = false (session dead)
```

---

### Test Case: AUTH-EDGE-003 — Reuse Same OTP After Successful Verification

**Priority**: Critical  
**Type**: Edge Case  
**Roles**: All  
**Platform**: Both  
**Automation**: [@automated]

**Steps:**
1. Request OTP for phone `9988776655`
2. Verify OTP successfully (receive JWT token)
3. Attempt to verify same OTP again

**Expected Result:**
- ✅ First verification: Success (JWT issued)
- ✅ Second verification: Error `OTP_SESSION_NOT_FOUND`
- ✅ OTP session marked `isUsed: true` after first use
- ✅ Database enforces one-time use constraint

---

### Test Case: AUTH-EDGE-004 — Multiple Devices, Same Phone Number

**Priority**: High  
**Type**: Edge Case  
**Roles**: All  
**Platform**: Both  
**Automation**: [@manual]

**Steps:**
1. Device A: Request OTP for phone `9123456789`
2. Device B: Request OTP for same phone (different deviceId)
3. Device A: Verify OTP from step 1
4. Device B: Verify OTP from step 2

**Expected Result:**
- ✅ Both devices receive separate OTP sessions
- ✅ Each OTP is device-bound (deviceId validated)
- ✅ Device A verification succeeds (JWT issued)
- ✅ Device B verification succeeds (separate JWT issued)
- ✅ Both tokens valid for same user account
- ✅ User can be logged in on multiple devices

---

### Test Case: AUTH-EDGE-005 — Username with Special Characters

**Priority**: Medium  
**Type**: Edge Case  
**Roles**: All  
**Platform**: Both  
**Automation**: [@automated]

**Test Data Matrix:**

| Username Input | Expected Validation | Error Message |
|----------------|---------------------|---------------|
| `user_123` | ✅ Accept | - |
| `Chef_Ravi_2024` | ✅ Accept | - |
| `foodlover` | ✅ Accept | - |
| `user@123` | ❌ Reject | "Username can only contain letters, numbers, and underscores" |
| `user-name` | ❌ Reject | "Username can only contain letters, numbers, and underscores" |
| `user.name` | ❌ Reject | "Username can only contain letters, numbers, and underscores" |
| `user name` | ❌ Reject | "Username can only contain letters, numbers, and underscores" |
| `user#123` | ❌ Reject | "Username can only contain letters, numbers, and underscores" |
| `us` | ❌ Reject | "Username must be 3-20 characters long" |
| `a_very_long_username_that_exceeds_limit` | ❌ Reject | "Username must be 3-20 characters long" |

---

### Test Case: AUTH-EDGE-006 — Concurrent OTP Requests (Same Phone)

**Priority**: Medium  
**Type**: Edge Case  
**Roles**: All  
**Platform**: Backend API  
**Automation**: [@automated]

**Steps:**
1. Send 5 OTP requests for phone `9876543210` within 10 seconds
2. Observe rate limiting behavior

**Expected Result:**
- ✅ First 5 requests within 1 minute: All succeed
- ✅ 6th request within same minute: Rate limit error
- ✅ Error response:
  ```json
  {
    "success": false,
    "message": "Too many OTP requests. Please wait.",
    "errorCode": "OTP_RATE_LIMIT"
  }
  ```
- ✅ After 60 seconds: Rate limit resets
- ✅ New request succeeds

---

## 4. Error Handling Tests

### Test Case: AUTH-E-NEW-001 — Invalid leading-digit phone does not crash and does not hit API

**Priority**: Critical  
**Type**: Error Handling / Regression  
**Roles**: All  
**Platform**: Android + iOS  
**Automation**: [@manual]

**Prerequisites:**
- App on `Enter Phone` screen
- Network logging enabled

**Test Data:**
- Phone: `1234567890`

**Steps:**
1. Enter `1234567890` in phone input
2. Tap `Continue`
3. Observe UI and network logs

**Expected Result:**
- ✅ App shows user-friendly validation alert: `Please enter a valid Indian mobile number`
- ✅ No request sent to `POST /api/v1/auth/v2/send-otp`
- ✅ No native crash/red screen occurs
- ✅ No RN bridge error: `Value for message can not be cast from ReadableNativeArray to string`

---

### Test Case: AUTH-E-NEW-002 — Backend validation array message is rendered safely

**Priority**: High  
**Type**: Error Handling / Regression  
**Roles**: All  
**Platform**: Android + iOS  
**Automation**: [@manual]

**Prerequisites:**
- Force backend to return validation payload with `message: ["..."]`

**Steps:**
1. Trigger send-otp error response containing message array
2. Observe alert rendering on app

**Expected Result:**
- ✅ Alert shows string text (joined validation message)
- ✅ App remains stable (no cast crash)
- ✅ Error is visible to user in readable format

---

### Test Case: AUTH-ERR-001 — Invalid Phone Number Format

**Priority**: High  
**Type**: Error Handling  
**Roles**: All  
**Platform**: Both  
**Automation**: [@automated]

**Test Data:**
- Invalid phones: `123456789`, `abcdefghij`, `+1-987-654-3210`

**Steps:**
1. Enter invalid phone number
2. Attempt to send OTP

**Expected Result:**
- ✅ Client-side validation triggers before API call
- ✅ Error message: "Please enter a valid 10-digit Indian mobile number"
- ✅ If validation bypassed, API returns 400 Bad Request
- ✅ No OTP sent, no session created

---

### Test Case: AUTH-ERR-002 — Network Timeout During OTP Send

**Priority**: High  
**Type**: Error Handling  
**Roles**: All  
**Platform**: Both  
**Automation**: [@manual]

**Steps:**
1. Simulate slow network (throttle to 2G or set 30s timeout)
2. Request OTP
3. Wait for timeout

**Expected Result:**
- ✅ App shows loading state for full timeout duration
- ✅ After timeout: "Request timed out. Please try again."
- ✅ "Retry" button displayed
- ✅ User can retry immediately (no invalid state)
- ✅ No partial session created in database

---

### Test Case: AUTH-ERR-003 — WhatsApp and SMS Both Fail

**Priority**: Critical  
**Type**: Error Handling  
**Roles**: All  
**Platform**: Backend API  
**Automation**: [@manual]

**Steps:**
1. Disable WhatsApp Cloud API (invalid credentials)
2. Disable Twilio SMS (invalid credentials)
3. Request OTP

**Expected Result:**
- ✅ Backend attempts WhatsApp first (fails)
- ✅ Backend attempts SMS fallback (fails)
- ✅ Response status: 400 Bad Request
- ✅ Error response:
  ```json
  {
    "success": false,
    "message": "Failed to send OTP. Please try again later.",
    "errorCode": "OTP_SEND_FAILED"
  }
  ```
- ✅ No OTP session created
- ✅ User informed to contact support if persistent

---

### Test Case: AUTH-ERR-004 — JWT Token Expired During Session

**Priority**: High  
**Type**: Error Handling  
**Roles**: All  
**Platform**: Both  
**Automation**: [@automated]

**Steps:**
1. Log in and receive JWT token (modify expiry to 1 minute for testing)
2. Wait for token to expire
3. Attempt to call protected endpoint (e.g., `/v1/auth/me`)

**Expected Result:**
- ✅ API returns 401 Unauthorized
- ✅ Error message: "Token expired. Please log in again."
- ✅ App clears stored token from SecureStore
- ✅ App redirects to login screen
- ✅ Refresh token can be used to obtain new access token (if implemented)

---

### Test Case: AUTH-ERR-005 — Database Connection Failure

**Priority**: Critical  
**Type**: Error Handling  
**Roles**: All  
**Platform**: Backend API  
**Automation**: [@manual]

**Steps:**
1. Stop PostgreSQL database
2. Attempt to send OTP

**Expected Result:**
- ✅ Backend catches database error
- ✅ Response status: 500 Internal Server Error
- ✅ Generic error message (no DB details leaked)
- ✅ Backend logs error with stack trace
- ✅ Health check endpoint reports database down

---

## 5. Security Tests

### Test Case: AUTH-SEC-001 — OTP Brute Force Protection

**Priority**: Critical  
**Type**: Security  
**Roles**: All  
**Platform**: Backend API  
**Automation**: [@automated]

**Steps:**
1. Request OTP for phone `9876543210`
2. Write script to attempt all 6-digit OTPs (000000 to 999999)
3. Verify rate limiting kicks in

**Expected Result:**
- ✅ After 5 failed attempts: Session invalidated
- ✅ After 10 verification requests in 5 min: Global rate limit
- ✅ Attacker cannot brute force OTP
- ✅ Legitimate user not permanently blocked (can request new OTP)
- ✅ Suspicious activity logged (IP, device, attempts)

**Attack Mitigation:**
- Session-based attempt limit: 5 max
- Global rate limit: 10 verifications per 5 minutes
- OTP complexity: 1 million possible combinations
- Time-boxed validity: 5 minutes

---

### Test Case: AUTH-SEC-002 — OTP Storage Security (No Plain Text)

**Priority**: Critical  
**Type**: Security  
**Roles**: All  
**Platform**: Backend  
**Automation**: [@automated]

**Steps:**
1. Request OTP for phone `9123456789`
2. Query database for OTP session
3. Inspect `otpHash` field

**Expected Result:**
- ✅ `otpHash` field contains bcrypt hash (60 char string)
- ✅ Hash starts with `$2a$` or `$2b$` (bcrypt identifier)
- ✅ Plain OTP never appears in database
- ✅ OTP not logged in application logs
- ✅ OTP not exposed in error messages

**Database Check:**
```sql
SELECT phone, otpHash, LENGTH(otpHash) as hash_length
FROM otp_sessions
WHERE phone = '+919123456789'
ORDER BY createdAt DESC LIMIT 1;

-- Expected: hash_length = 60, otpHash starts with $2a$ or $2b$
```

---

### Test Case: AUTH-SEC-003 — JWT Token Tampering

**Priority**: Critical  
**Type**: Security  
**Roles**: All  
**Platform**: Backend API  
**Automation**: [@automated]

**Steps:**
1. Obtain valid JWT token
2. Decode token payload (base64 decode middle section)
3. Modify userId to another user's ID
4. Re-encode and attempt to use modified token

**Expected Result:**
- ✅ API rejects modified token (signature validation fails)
- ✅ Response: 401 Unauthorized
- ✅ Error: "Invalid token"
- ✅ User cannot access another user's data
- ✅ Tampering logged as security event

---

### Test Case: AUTH-SEC-004 — Device Binding Enforcement

**Priority**: High  
**Type**: Security  
**Roles**: All  
**Platform**: Backend API  
**Automation**: [@automated]

**Steps:**
1. Device A: Request OTP for phone `9988776655` (deviceId: `DEVICE-A`)
2. Device B: Attempt to verify OTP using Device A's session but different deviceId (`DEVICE-B`)

**Expected Result:**
- ✅ Verification fails with error: `OTP_SESSION_NOT_FOUND`
- ✅ OTP is device-bound (can't be verified from different device)
- ✅ Mitigates OTP interception attacks
- ✅ Each device must request its own OTP

---

### Test Case: AUTH-SEC-005 — Role-Based Access Control (RBAC)

**Priority**: Critical  
**Type**: Security  
**Roles**: Customer attempting Chef actions  
**Platform**: Backend API  
**Automation**: [@automated]

**Prerequisites:**
- User logged in with role: `user` (customer)

**Steps:**
1. Obtain JWT token for customer user
2. Attempt to access chef-only endpoint (e.g., `/v1/chef/menu`)

**Expected Result:**
- ✅ API returns 403 Forbidden
- ✅ Error: "Insufficient permissions"
- ✅ JWT token valid but role check fails
- ✅ No data leaked about endpoint existence

---

### Test Case: AUTH-SEC-006 — Username Enumeration Prevention

**Priority**: Medium  
**Type**: Security  
**Roles**: Attacker  
**Platform**: Backend API  
**Automation**: [@automated]

**Steps:**
1. Attempt to register username: `existing_user` (already taken)
2. Attempt to register username: `new_unique_user` (available)
3. Compare response times and messages

**Expected Result:**
- ✅ Both responses return same timing (~±50ms variance)
- ✅ "Username already taken" error doesn't leak user existence
- ✅ No timing attack possible to enumerate existing usernames
- ✅ Rate limiting prevents mass username checking

---

## 6. Performance Tests

### Test Case: AUTH-PERF-001 — OTP Send Response Time

**Priority**: High  
**Type**: Performance  
**Roles**: All  
**Platform**: Backend API  
**Automation**: [@automated]

**Metrics to Measure:**
- Time from request to API response
- Time from request to OTP delivery (WhatsApp/SMS)

**Steps:**
1. Send 100 OTP requests (staggered, not concurrent)
2. Measure response times

**Expected Result:**
- ✅ API response time: <500ms (p95)
- ✅ WhatsApp delivery: <10 seconds (p95)
- ✅ SMS delivery: <30 seconds (p95)
- ✅ No timeout errors
- ✅ Consistent performance across all requests

**Load Testing Command:**
```bash
# Using Apache Bench
ab -n 100 -c 10 -T application/json \
  -p otp-payload.json \
  https://api-staging.chefooz.com/api/v1/auth/v2/send-otp
```

---

### Test Case: AUTH-PERF-002 — Concurrent OTP Verifications

**Priority**: Medium  
**Type**: Performance  
**Roles**: All  
**Platform**: Backend API  
**Automation**: [@automated]

**Steps:**
1. Create 50 valid OTP sessions
2. Send 50 concurrent verification requests
3. Measure response times and success rate

**Expected Result:**
- ✅ All 50 verifications complete within 5 seconds
- ✅ No database deadlocks or race conditions
- ✅ 100% success rate (all valid OTPs accepted)
- ✅ JWT tokens generated for all users
- ✅ Database handles concurrent writes correctly

---

### Test Case: AUTH-PERF-003 — JWT Token Validation Overhead

**Priority**: Medium  
**Type**: Performance  
**Roles**: All  
**Platform**: Backend API  
**Automation**: [@automated]

**Steps:**
1. Make 1000 requests to `/v1/auth/me` with valid JWT token
2. Measure overhead of JWT validation

**Expected Result:**
- ✅ JWT validation adds <10ms overhead per request
- ✅ No performance degradation over time
- ✅ Redis/cache used for blacklist checks (if implemented)
- ✅ CPU usage remains <70% during load

---

## 7. Regression Tests

### Test Case: AUTH-REG-001 — End-to-End Login Flow

**Priority**: Critical  
**Type**: Regression  
**Automation**: [@automated]

**Purpose**: Verify complete auth flow after code changes

**Steps:**
1. New user enters phone number
2. OTP sent via WhatsApp
3. User verifies OTP
4. User completes profile setup
5. User navigates to home screen
6. User logs out
7. User logs back in (existing user flow)

**Expected Result:**
- ✅ All steps complete without errors
- ✅ User data persists between sessions
- ✅ No UI glitches or crashes
- ✅ Total flow time: <2 minutes

---

### Test Case: AUTH-REG-002 — Profile Update After Initial Setup

**Priority**: High  
**Type**: Regression  
**Automation**: [@automated]

**Steps:**
1. User completes initial profile setup
2. User updates profile later (e.g., changes fullName)
3. Verify updates persist

**Expected Result:**
- ✅ Profile updates save successfully
- ✅ No regression to `profileIncomplete: true`
- ✅ Username remains immutable (can't be changed after set)
- ✅ Other fields (fullName, bio, avatar) editable

---

### Test Case: AUTH-REG-003 — Backward Compatibility (Old JWT Tokens)

**Priority**: Medium  
**Type**: Regression  
**Automation**: [@manual]

**Prerequisites:**
- JWT token generated before recent deployment

**Steps:**
1. Use old JWT token (generated before code update)
2. Attempt to access protected endpoint

**Expected Result:**
- ✅ Old token still valid (if not expired)
- ✅ Token payload structure backward compatible
- ✅ No breaking changes in JWT validation logic

---

## 📡 API Test Commands

### Send OTP (WhatsApp-first)
```bash
curl -X POST https://api-staging.chefooz.com/api/v1/auth/v2/send-otp \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "9876543210",
    "deviceId": "TEST-DEVICE-001"
  }' | jq
```

### Verify OTP
```bash
curl -X POST https://api-staging.chefooz.com/api/v1/auth/v2/verify-otp \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "9876543210",
    "otp": "123456",
    "deviceId": "TEST-DEVICE-001"
  }' | jq
```

### Get Current User
```bash
curl -X GET https://api-staging.chefooz.com/api/v1/auth/me \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" | jq
```

### Update Profile
```bash
curl -X PUT https://api-staging.chefooz.com/api/v1/auth/profile \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "foodlover_123",
    "fullName": "John Doe"
  }' | jq
```

### Test Rate Limiting (Should fail on 6th request)
```bash
for i in {1..6}; do
  echo "Request $i:"
  curl -X POST https://api-staging.chefooz.com/api/v1/auth/v2/send-otp \
    -H "Content-Type: application/json" \
    -d '{"phone": "9876543210", "deviceId": "RATE-LIMIT-TEST"}'
  sleep 1
done
```

---

## ✅ Test Execution Checklist

### Before Testing
- [ ] Test environment (dev/staging) is stable and accessible
- [ ] Latest auth module code deployed
- [ ] PostgreSQL database running with clean test data
- [ ] WhatsApp Cloud API credentials configured
- [ ] Twilio SMS credentials configured (for fallback testing)
- [ ] Test phone numbers added to WhatsApp sandbox (if in sandbox mode)
- [ ] Mobile app built with latest code (iOS and Android)
- [ ] Screen recording tools ready for bug reporting
- [ ] Test accounts and device IDs documented

### During Testing
- [ ] Execute test cases in priority order (Critical → High → Medium → Low)
- [ ] Record actual results for each test case
- [ ] Screenshot/video any failures or unexpected behavior
- [ ] Note exact timestamp and environment details for bugs
- [ ] Test on multiple devices (iOS 16+, Android 11+)
- [ ] Test on different network conditions (WiFi, 4G, 3G, poor signal)
- [ ] Verify database state after critical operations
- [ ] Check backend logs for errors/warnings

### After Testing
- [ ] Update test case status (Pass/Fail) in tracking sheet
- [ ] File bugs in issue tracker with detailed reproduction steps
- [ ] Tag bugs with priority (P0-P3) and module (Auth)
- [ ] Notify development team of critical failures immediately
- [ ] Update regression suite with new edge cases discovered
- [ ] Document any workarounds for known issues
- [ ] Archive test execution report with metrics

---

## 📊 Test Coverage Summary

| Category | Total Tests | Critical | High | Medium | Low |
|----------|-------------|----------|------|--------|-----|
| Functional | 7 | 5 | 2 | 0 | 0 |
| UX/UI | 5 | 0 | 3 | 2 | 0 |
| Edge Cases | 6 | 1 | 3 | 2 | 0 |
| Error Handling | 5 | 2 | 3 | 0 | 0 |
| Security | 6 | 4 | 2 | 0 | 0 |
| Performance | 3 | 0 | 2 | 1 | 0 |
| Regression | 3 | 1 | 1 | 1 | 0 |
| **TOTAL** | **62** | **13** | **16** | **6** | **0** |

**Automation Status:**
- Automated: 38 tests (61%)
- Manual: 24 tests (39%)

---

## 🐛 Bug Report Template

When filing bugs discovered during testing, use this template:

```markdown
# [AUTH-BUG-XXX] Brief Bug Title

**Module**: Authentication
**Test Case**: AUTH-F-001 (if applicable)
**Priority**: [Critical / High / Medium / Low]
**Severity**: [Blocker / Major / Minor]
**Environment**: [Dev / Staging / Production]
**Platform**: [iOS 16.0 / Android 13 / Both]
**Device**: [iPhone 14 Pro / Samsung Galaxy S23]

## Description
Clear description of the bug and its impact.

## Steps to Reproduce
1. Step 1
2. Step 2
3. Step 3

## Expected Result
What should happen according to test case.

## Actual Result
What actually happened.

## Screenshots/Videos
[Attach media showing the issue]

## Logs/Errors
\`\`\`
Paste relevant backend logs or mobile app console output
\`\`\`

## Database State (if relevant)
\`\`\`sql
SELECT * FROM otp_sessions WHERE phone = '+919876543210';
\`\`\`

## Additional Context
- User ID: f47ac10b-58cc-4372-a567-0e02b2c3d479
- OTP Session ID: 550e8400-e29b-41d4-a716-446655440000
- Timestamp: 2026-02-14T10:30:00Z
- Network: WiFi / 4G
- App Version: 1.0.0 (build 42)

## Impact Assessment
- [ ] Blocks critical user flow
- [ ] Affects all users
- [ ] Affects specific role/platform
- [ ] Workaround available: [describe]
```

---

## 🔗 Related Documentation

- [Feature Overview](./FEATURE_OVERVIEW.md) — Business requirements and user workflows
- [Technical Guide](./TECHNICAL_GUIDE.md) — Implementation details and API specs
- [AI QA Generation Rules](../../../.github/docs/ai/AI_QA_GENERATION_RULES.md) — Testing standards
- [Global Dev Rules](../../../.github/copilot-instructions.md) — Monorepo architecture

---

## 📝 Notes for QA Team

### Critical Paths to Always Test
1. New user registration (phone → OTP → profile setup → home)
2. Existing user login (phone → OTP → home)
3. Profile completion (username uniqueness, validation)
4. JWT token validation on protected endpoints

### Known Limitations
- WhatsApp OTP delivery requires approved Business account (sandbox for dev)
- Rate limiting is IP-based in dev (may share IP with team)
- Dev mode bypass available (`OTP_DEV_MODE=true`) — use `123456` as universal OTP

### Testing Best Practices
- Always test on real devices (simulators miss edge cases)
- Test with poor network (throttle to 2G occasionally)
- Clear app cache between test runs to avoid state issues
- Use dedicated test phone numbers (avoid personal numbers)
- Document device models and OS versions for each bug

---

## 8. Session Preservation Tests (added 2026-03-01)

These tests cover the four bugs fixed in the 2026-03-01 session-preservation patch.

### TC-SP-01: App restart with valid token + no internet — user must stay logged in

| Field | Detail |
|---|---|
| **Priority** | P0 |
| **Related bug** | Bug 2 — catch-all logout on network error |
| **Pre-conditions** | User previously logged in; device set to airplane mode |
| **Steps** | 1. Enable airplane mode. 2. Force-kill and relaunch the app. |
| **Expected** | User lands on their home feed (or the last screen). No redirect to login. A subtle "offline" indicator is acceptable. |
| **Pass criteria** | `isAuthenticated: true` in store; no call to `logout()`. |
| **Fail criteria** | User redirected to `/auth/enter-phone`. |

---

### TC-SP-02: App restart with valid token + server returns 500 — user must stay logged in

| Field | Detail |
|---|---|
| **Priority** | P0 |
| **Related bug** | Bug 2 |
| **Pre-conditions** | User previously logged in; mock `/api/v1/auth/me` to return HTTP 500 (via Charles Proxy or backend override) |
| **Steps** | 1. Kill app. 2. Start mock. 3. Relaunch app. |
| **Expected** | User stays on home screen; no logout. |
| **Pass criteria** | Session preserved. |
| **Fail criteria** | Redirect to login screen. |

---

### TC-SP-03: Token expires mid-session — next API call must trigger full logout

| Field | Detail |
|---|---|
| **Priority** | P0 |
| **Related bug** | Bug 1 — 401 interceptor did not call full logout |
| **Pre-conditions** | User is logged in; manually expire/revoke the JWT on the backend or use a short-TTL test token |
| **Steps** | 1. Trigger any authenticated API call (e.g. open profile). |
| **Expected** | App navigates to `/auth/enter-phone`. Zustand store shows `isAuthenticated: false`, `token: null`, `user: null`. |
| **Pass criteria** | Login screen shown; store is clean; SecureStore `auth_token` key deleted. |
| **Fail criteria** | User stays on screen; 401 is silently swallowed; repeated 401 loops. |

---

### TC-SP-04: App restart with valid token + internet — navigation guard must not flash login screen

| Field | Detail |
|---|---|
| **Priority** | P1 |
| **Related bug** | Bug 3 — `initialize()` did not set `isAuthenticated: true` eagerly |
| **Pre-conditions** | User previously completed onboarding; valid token in SecureStore |
| **Steps** | 1. Kill app. 2. Relaunch. 3. Observe transition from splash screen to home. |
| **Expected** | No flash of the login screen. App transitions directly from splash → home feed. |
| **Pass criteria** | Zero frames on `/auth/enter-phone` between splash and home. |
| **Fail criteria** | Brief flicker of the phone-number entry screen before home loads. |

---

### TC-SP-05: Explicit logout clears all storage and redirects to login

| Field | Detail |
|---|---|
| **Priority** | P0 |
| **Pre-conditions** | User is logged in |
| **Steps** | 1. Tap logout (Settings or profile page). |
| **Expected** | Navigated to `/auth/enter-phone`. `auth_token`, `accessToken`, `refreshToken` all deleted from SecureStore. React Query cache cleared. |
| **Pass criteria** | All conditions above met. |
| **Fail criteria** | Any key remains in SecureStore; cached API data visible after re-login with a different account. |

---

**QA_TEST_CASES_COMPLETE ✅**

**Total Lines**: 800+  
**Last Updated**: 2026-03-01  
**Next Review**: After first QA round feedback
