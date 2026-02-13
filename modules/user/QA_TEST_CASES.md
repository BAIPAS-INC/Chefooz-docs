# 👤 User Module — QA Test Cases

**Status**: ✅ **PRODUCTION READY**  
**Last Updated**: 2026-02-14  
**Target Audience**: QA Engineers, Testers, Automation Engineers  
**Related Docs**:
- [User Feature Overview](./FEATURE_OVERVIEW.md)
- [User Technical Guide](./TECHNICAL_GUIDE.md)

---

## 📋 Test Coverage Summary

| Category | Test Cases | Priority Distribution | Automation |
|----------|------------|----------------------|------------|
| Functional | 15 | 8 Critical, 5 High, 2 Medium | 80% |
| UX/UI | 8 | 3 Critical, 3 High, 2 Medium | 50% |
| Edge Cases | 10 | 5 Critical, 3 High, 2 Medium | 70% |
| Error Handling | 8 | 6 Critical, 2 High | 90% |
| Security | 7 | 7 Critical | 100% |
| Performance | 4 | 2 High, 2 Medium | 75% |
| Regression | 5 | 3 Critical, 2 High | 100% |
| **Total** | **57** | **34 Critical, 17 High, 6 Medium** | **82%** |

---

## 🧪 Test Categories

### 1. Functional Tests (Address CRUD)

#### USER-F-001: Create Home Address (Happy Path)
**Priority**: Critical  
**Platform**: Mobile + Backend API  
**Automation**: [@automated]

**Preconditions**:
- User logged in (JWT token available)
- User has 0 addresses

**Steps**:
1. Navigate to Profile → Addresses
2. Tap "+ Add New Address"
3. Fill form:
   - Label: `Home`
   - Full Name: `Rajesh Kumar`
   - Phone: `9876543210`
   - Line 1: `Flat 203, Sunshine Apartments`
   - Line 2: `Koramangala 5th Block`
   - City: `Bangalore`
   - State: `Karnataka`
   - Pincode: `560095`
   - Enable location → Lat: `12.9352`, Lng: `77.6245`
   - Is Default: `true` (checked)
4. Tap "Save Address"
5. Verify API call:
   ```bash
   curl -X POST https://api.chefooz.com/api/v1/user/addresses \
     -H "Authorization: Bearer <JWT>" \
     -H "Content-Type: application/json" \
     -d '{
       "label": "Home",
       "fullName": "Rajesh Kumar",
       "phone": "9876543210",
       "line1": "Flat 203, Sunshine Apartments",
       "line2": "Koramangala 5th Block",
       "city": "Bangalore",
       "state": "Karnataka",
       "pincode": "560095",
       "lat": 12.9352,
       "lng": 77.6245,
       "isDefault": true
     }'
   ```

**Expected Result**:
- API returns 200 OK:
  ```json
  {
    "success": true,
    "message": "Address created successfully",
    "data": {
      "id": "<UUID>",
      "userId": "<user-id>",
      "label": "Home",
      "isDefault": true,
      ...
    }
  }
  ```
- Address appears in address list
- Address marked with "🏠 Default" badge
- Database verification:
  ```sql
  SELECT * FROM addresses WHERE "userId" = '<user-id>' AND label = 'Home';
  -- Should return 1 row with isDefault = true
  ```

---

#### USER-F-002: Add Second Address (Work)
**Priority**: High  
**Platform**: Mobile + Backend API  
**Automation**: [@automated]

**Preconditions**:
- User has 1 existing address (Home, default)

**Steps**:
1. Tap "+ Add New Address"
2. Fill form:
   - Label: `Work`
   - Full Name: `Rajesh Kumar`
   - Phone: `9876543210`
   - Line 1: `Tech Corp Ltd, 12th Floor`
   - City: `Bangalore`
   - State: `Karnataka`
   - Pincode: `560037`
   - Is Default: `false` (unchecked)
3. Save address

**Expected Result**:
- New address created successfully
- Home address remains default
- Address list shows 2 addresses:
  - Home (default)
  - Work

---

#### USER-F-003: Update Address Label
**Priority**: High  
**Platform**: Mobile + Backend API  
**Automation**: [@automated]

**Preconditions**:
- User has address with id `addr_123`, label `Home`

**Steps**:
1. Open address list
2. Tap on "Home" address
3. Change label dropdown: `Home` → `Work`
4. Tap "Save"
5. API call:
   ```bash
   curl -X PUT https://api.chefooz.com/api/v1/user/addresses/addr_123 \
     -H "Authorization: Bearer <JWT>" \
     -H "Content-Type: application/json" \
     -d '{"label": "Work"}'
   ```

**Expected Result**:
- API returns 200 OK
- Address label updated from "Home" to "Work"
- Icon changes: 🏠 → 🏢
- Database updated:
  ```sql
  SELECT label FROM addresses WHERE id = 'addr_123';
  -- Should return 'Work'
  ```

---

#### USER-F-004: Set Default Address
**Priority**: Critical  
**Platform**: Mobile + Backend API  
**Automation**: [@automated]

**Preconditions**:
- User has 3 addresses:
  - addr_home (default: true)
  - addr_work (default: false)
  - addr_other (default: false)

**Steps**:
1. Tap "Work" address
2. Toggle "Set as Default"
3. API call:
   ```bash
   curl -X PUT https://api.chefooz.com/api/v1/user/addresses/addr_work/default \
     -H "Authorization: Bearer <JWT>"
   ```

**Expected Result**:
- API returns 200 OK
- "Work" address now marked as default
- "Home" address no longer default
- Database verification:
  ```sql
  SELECT id, label, "isDefault" FROM addresses WHERE "userId" = '<user-id>';
  -- Expected:
  -- addr_home   | Home  | false
  -- addr_work   | Work  | true   ← Only this should be true
  -- addr_other  | Other | false
  ```

---

#### USER-F-005: Delete Address
**Priority**: High  
**Platform**: Mobile + Backend API  
**Automation**: [@automated]

**Preconditions**:
- User has address with id `addr_other`

**Steps**:
1. Tap "Other" address
2. Tap "Delete Address"
3. Confirm deletion in alert dialog
4. API call:
   ```bash
   curl -X DELETE https://api.chefooz.com/api/v1/user/addresses/addr_other \
     -H "Authorization: Bearer <JWT>"
   ```

**Expected Result**:
- API returns 200 OK
- Address removed from list
- Database verification:
  ```sql
  SELECT * FROM addresses WHERE id = 'addr_other';
  -- Should return 0 rows
  ```

---

#### USER-F-006: Fetch All User Addresses
**Priority**: Critical  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- User has 2 addresses (Home default, Work non-default)

**Steps**:
1. API call:
   ```bash
   curl -X GET https://api.chefooz.com/api/v1/user/addresses \
     -H "Authorization: Bearer <JWT>"
   ```

**Expected Result**:
- API returns 200 OK
- Response contains 2 addresses
- Order: Default first, then by `createdAt DESC`
- Example:
  ```json
  {
    "success": true,
    "data": [
      { "id": "addr_home", "label": "Home", "isDefault": true, ... },
      { "id": "addr_work", "label": "Work", "isDefault": false", ... }
    ]
  }
  ```

---

#### USER-F-007: Address with Geolocation (lat/lng)
**Priority**: High  
**Platform**: Mobile + Backend API  
**Automation**: [@automated]

**Preconditions**:
- User enabling location permissions

**Steps**:
1. Create address with geolocation:
   ```json
   {
     "label": "Home",
     "fullName": "Rajesh",
     "phone": "9876543210",
     "line1": "Flat 203",
     "city": "Bangalore",
     "state": "Karnataka",
     "pincode": "560095",
     "lat": 12.9352,
     "lng": 77.6245
   }
   ```

**Expected Result**:
- Address created with `lat` and `lng` stored
- Database verification:
  ```sql
  SELECT lat, lng FROM addresses WHERE id = '<new-addr-id>';
  -- Should return: 12.9352 | 77.6245
  ```
- Lat/lng used for delivery zone checks in Order module

---

#### USER-F-008: Address without Geolocation
**Priority**: Medium  
**Platform**: Mobile + Backend API  
**Automation**: [@automated]

**Preconditions**:
- User denying location permissions

**Steps**:
1. Create address without lat/lng:
   ```json
   {
     "label": "Home",
     "fullName": "Rajesh",
     "phone": "9876543210",
     "line1": "Flat 203",
     "city": "Bangalore",
     "state": "Karnataka",
     "pincode": "560095"
     // lat and lng omitted
   }
   ```

**Expected Result**:
- Address created successfully
- Database: `lat = NULL`, `lng = NULL`
- Order module assumes deliverable (graceful fallback)

---

### 2. Functional Tests (Username)

#### USER-F-009: Check Username Availability (Available)
**Priority**: Critical  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- Username `newuser123` does not exist in database

**Steps**:
1. API call:
   ```bash
   curl -X GET "https://api.chefooz.com/api/v1/users/check-username?username=newuser123"
   ```

**Expected Result**:
```json
{
  "success": true,
  "message": "Username is available",
  "data": {
    "available": true
  }
}
```

---

#### USER-F-010: Check Username Availability (Taken)
**Priority**: Critical  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- Username `rajesh` exists in `users` table

**Steps**:
1. API call:
   ```bash
   curl -X GET "https://api.chefooz.com/api/v1/users/check-username?username=rajesh"
   ```

**Expected Result**:
```json
{
  "success": true,
  "message": "Username is already taken",
  "data": {
    "available": false,
    "suggestions": ["rajesh_1", "rajesh001", "therajesh"]
  }
}
```

---

#### USER-F-011: Username Case-Insensitive Check
**Priority**: Critical  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- Username `RajeSH` exists in database (mixed case)

**Steps**:
1. Check lowercase: `GET /api/v1/users/check-username?username=rajesh`
2. Check uppercase: `GET /api/v1/users/check-username?username=RAJESH`
3. Check mixed: `GET /api/v1/users/check-username?username=RajESh`

**Expected Result**:
- All 3 requests return `available: false`
- Username uniqueness is case-insensitive
- Database query uses `LOWER(username)`

---

#### USER-F-012: Username Suggestions Generation
**Priority**: High  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- Username `chef` is taken

**Steps**:
1. Check username:
   ```bash
   curl -X GET "https://api.chefooz.com/api/v1/users/check-username?username=chef"
   ```

**Expected Result**:
```json
{
  "success": true,
  "message": "Username is already taken",
  "data": {
    "available": false,
    "suggestions": [
      "chef_1",      // Append underscore + number
      "chef001",     // Append zero-padded number
      "thechef"      // Prepend "the"
    ]
  }
}
```

---

#### USER-F-013: Username Autocomplete Search
**Priority**: High  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- Users with usernames: `rajesh`, `raj_chef`, `raja`

**Steps**:
1. API call:
   ```bash
   curl -X GET "https://api.chefooz.com/api/v1/users/username-suggestions?q=raj"
   ```

**Expected Result**:
```json
{
  "success": true,
  "message": "Username suggestions retrieved",
  "data": {
    "suggestions": [
      {
        "userId": "user_550e8400",
        "username": "raj_chef",
        "fullName": "Raj Chef",
        "avatarUrl": "https://cdn.chefooz.com/avatars/raj.jpg"
      },
      {
        "userId": "user_550e8401",
        "username": "raja",
        "fullName": "Raja Kumar",
        "avatarUrl": null
      },
      {
        "userId": "user_550e8402",
        "username": "rajesh",
        "fullName": "Rajesh Kumar",
        "avatarUrl": null
      }
    ]
  }
}
```
- Max 10 results returned
- Ordered alphabetically by username

---

### 3. Functional Tests (Coin Accrual)

#### USER-F-014: Award Coins with Reputation Multiplier (Gold Tier)
**Priority**: Critical  
**Platform**: Backend Service  
**Automation**: [@automated]

**Preconditions**:
- User `user_123` has Gold tier reputation (`coinMultiplier: 1.1`)
- User current coins: `500`

**Steps**:
1. Trigger coin accrual:
   ```typescript
   await userService.awardCoinsWithReputation('user_123', 100, 'order_complete');
   ```

**Expected Result**:
- Function returns:
  ```typescript
  {
    coinsAwarded: 110,  // floor(100 * 1.1)
    multiplier: 1.1
  }
  ```
- Database verification:
  ```sql
  SELECT coins FROM users WHERE id = 'user_123';
  -- Should return: 610 (500 + 110)
  ```

---

#### USER-F-015: Award Coins with No Reputation Record (Default 1.0×)
**Priority**: High  
**Platform**: Backend Service  
**Automation**: [@automated]

**Preconditions**:
- User `user_new` has no reputation record
- User current coins: `0`

**Steps**:
1. Award coins:
   ```typescript
   await userService.awardCoinsWithReputation('user_new', 50, 'first_order');
   ```

**Expected Result**:
- Function returns:
  ```typescript
  {
    coinsAwarded: 50,  // floor(50 * 1.0) — default multiplier
    multiplier: 1.0
  }
  ```
- User coins updated to `50`

---

## 🎨 UX/UI Tests

#### USER-UX-001: Address Form Auto-Fill from Location
**Priority**: High  
**Platform**: Mobile  
**Automation**: [@manual]

**Preconditions**:
- User grants location permissions

**Steps**:
1. Tap "+ Add New Address"
2. Tap "Use Current Location" button
3. App fetches GPS coordinates
4. Reverse geocode coordinates → formatted address

**Expected Result**:
- Form fields auto-filled:
  - Line 1: Street address
  - City: Bangalore
  - State: Karnataka
  - Pincode: 560095
  - Lat/Lng: Populated
- User can edit auto-filled values before saving

---

#### USER-UX-002: Address List Empty State
**Priority**: Medium  
**Platform**: Mobile  
**Automation**: [@manual]

**Preconditions**:
- User has 0 addresses

**Steps**:
1. Navigate to Profile → Addresses

**Expected Result**:
- Empty state shown:
  - Icon: 📍
  - Text: "No saved addresses"
  - Button: "+ Add Your First Address"
- Tapping button opens address form

---

#### USER-UX-003: Default Address Visual Indicator
**Priority**: High  
**Platform**: Mobile  
**Automation**: [@manual]

**Preconditions**:
- User has 3 addresses, 1 is default

**Steps**:
1. View address list

**Expected Result**:
- Default address has:
  - Badge: "🏠 Default" (green)
  - Positioned at top of list
  - Icon color: Green (#4CAF50)
- Non-default addresses:
  - No badge
  - Grey icon color (#9E9E9E)

---

#### USER-UX-004: Username Real-Time Validation
**Priority**: Critical  
**Platform**: Mobile  
**Automation**: [@manual]

**Preconditions**:
- User on profile completion screen

**Steps**:
1. User types: `r` → No validation yet (too short)
2. User types: `ra` → Still no validation
3. User types: `raj` → API call triggered (debounced 500ms)
4. API returns: Username taken
5. UI shows: ❌ "Username taken" + suggestions

**Expected Result**:
- Validation only triggered after 3+ characters
- Debounced (waits 500ms after user stops typing)
- Clear visual feedback:
  - ✅ Green checkmark if available
  - ❌ Red cross if taken
  - Suggestions clickable (auto-fills input)

---

#### USER-UX-005: Phone Number Input Formatting
**Priority**: High  
**Platform**: Mobile  
**Automation**: [@manual]

**Preconditions**:
- User on add address form

**Steps**:
1. Tap phone input field
2. Keyboard: Numeric only
3. User types: `9876543210`

**Expected Result**:
- Numeric keyboard shown (no letters)
- Max length: 10 digits
- Auto-format: `98765 43210` (space after 5 digits for readability)
- No country code prefix (assumed +91)

---

#### USER-UX-006: Address Label Icons
**Priority**: Medium  
**Platform**: Mobile  
**Automation**: [@manual]

**Preconditions**:
- User has addresses with all 3 labels

**Steps**:
1. View address list

**Expected Result**:
- Home → 🏠 icon
- Work → 🏢 icon
- Other → 📍 icon
- Icons consistent across app (cart, checkout, order)

---

#### USER-UX-007: Delete Address Confirmation
**Priority**: Critical  
**Platform**: Mobile  
**Automation**: [@manual]

**Preconditions**:
- User has address `addr_home`

**Steps**:
1. Tap address
2. Tap "Delete Address"
3. Alert dialog shown:
   - Title: "Delete Address?"
   - Message: "This action cannot be undone."
   - Buttons: "Cancel", "Delete" (red)

**Expected Result**:
- Tapping "Cancel" → No action
- Tapping "Delete" → Address deleted, list refreshed

---

#### USER-UX-008: Username Suggestions Tap Behavior
**Priority**: Medium  
**Platform**: Mobile  
**Automation**: [@manual]

**Preconditions**:
- User enters taken username, suggestions shown

**Steps**:
1. Username input: `chef` (taken)
2. Suggestions shown: `chef_1`, `chef001`, `thechef`
3. User taps `chef001`

**Expected Result**:
- Input field auto-fills with `chef001`
- Validation re-triggered immediately
- If available: ✅ shown

---

## ⚠️ Edge Cases

#### USER-E-001: Create First Address (Auto-Default)
**Priority**: Critical  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- User has 0 addresses

**Steps**:
1. Create address with `isDefault: false` (explicitly unchecked)
2. API call:
   ```json
   {
     "label": "Home",
     "isDefault": false,
     ...
   }
   ```

**Expected Result**:
- Address created with `isDefault: true` (overridden)
- First address always becomes default (business rule)
- Database:
  ```sql
  SELECT "isDefault" FROM addresses WHERE id = '<new-addr>';
  -- Should return: true
  ```

---

#### USER-E-002: Delete Default Address (Auto-Select New Default)
**Priority**: High  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- User has 2 addresses:
  - addr_home (default: true)
  - addr_work (default: false)

**Steps**:
1. Delete `addr_home`
2. API call:
   ```bash
   curl -X DELETE .../addresses/addr_home -H "Authorization: ..."
   ```

**Expected Result**:
- `addr_home` deleted
- `addr_work` remains with `isDefault: false`
- No auto-promotion (user must manually set new default)
- Future enhancement: Auto-promote oldest address

---

#### USER-E-003: Partial Geolocation (Only lat Provided)
**Priority**: Critical  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- None

**Steps**:
1. Create address:
   ```json
   {
     "label": "Home",
     "lat": 12.9352,
     "lng": null,
     ...
   }
   ```

**Expected Result**:
- API returns 400 Bad Request:
  ```json
  {
    "success": false,
    "message": "Both latitude and longitude must be provided together",
    "errorCode": "ADDR_REQUIRE_LAT_LNG"
  }
  ```
- Address not created

---

#### USER-E-004: Username with Consecutive Underscores
**Priority**: High  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- None

**Steps**:
1. Check username:
   ```bash
   curl -X GET ".../check-username?username=chef__raj"
   ```

**Expected Result**:
```json
{
  "success": false,
  "message": "Username cannot contain consecutive underscores",
  "errorCode": "INVALID_USERNAME_FORMAT"
}
```

---

#### USER-E-005: Username Starting with Number
**Priority**: High  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- None

**Steps**:
1. Check username:
   ```bash
   curl -X GET ".../check-username?username=123chef"
   ```

**Expected Result**:
```json
{
  "success": false,
  "message": "Username cannot start with a number",
  "errorCode": "INVALID_USERNAME_FORMAT"
}
```

---

#### USER-E-006: Username Reserved Word
**Priority**: Critical  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- Reserved words: `admin`, `support`, `help`, `chefooz`, `official`, `system`, `root`

**Steps**:
1. Check username:
   ```bash
   curl -X GET ".../check-username?username=admin"
   ```

**Expected Result**:
```json
{
  "success": false,
  "message": "This username is reserved",
  "errorCode": "INVALID_USERNAME_FORMAT"
}
```

---

#### USER-E-007: Update Address Without Ownership
**Priority**: Critical  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- User A owns `addr_home`
- User B logged in

**Steps**:
1. User B attempts to update User A's address:
   ```bash
   curl -X PUT .../addresses/addr_home \
     -H "Authorization: Bearer <USER_B_JWT>" \
     -d '{"label": "Work"}'
   ```

**Expected Result**:
- API returns 403 Forbidden:
  ```json
  {
    "success": false,
    "message": "You do not have permission to update this address",
    "errorCode": "ADDR_FORBIDDEN"
  }
  ```

---

#### USER-E-008: Coin Accrual with Fractional Result
**Priority**: Medium  
**Platform**: Backend Service  
**Automation**: [@automated]

**Preconditions**:
- User has Gold tier (`coinMultiplier: 1.1`)

**Steps**:
1. Award 15 base coins:
   ```typescript
   await userService.awardCoinsWithReputation('user_123', 15, 'referral');
   ```
2. Calculation: `15 × 1.1 = 16.5`

**Expected Result**:
- Result: `floor(16.5) = 16 coins` (no fractional coins)
- Function returns:
  ```typescript
  {
    coinsAwarded: 16,
    multiplier: 1.1
  }
  ```

---

#### USER-E-009: Very Long Username (21 Characters)
**Priority**: High  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- None

**Steps**:
1. Check username (21 chars):
   ```bash
   curl -X GET ".../check-username?username=chef_anika_bangalore_2026"
   ```

**Expected Result**:
```json
{
  "success": false,
  "message": "Username cannot exceed 20 characters",
  "errorCode": "INVALID_USERNAME_FORMAT"
}
```

---

#### USER-E-010: Username with Uppercase Letters
**Priority**: High  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- None

**Steps**:
1. Check username:
   ```bash
   curl -X GET ".../check-username?username=ChefRaj"
   ```

**Expected Result**:
```json
{
  "success": false,
  "message": "Username can only contain lowercase letters, numbers, and underscores",
  "errorCode": "INVALID_USERNAME_FORMAT"
}
```

---

## 🚨 Error Handling

#### USER-ERR-001: Create Address Without JWT Token
**Priority**: Critical  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- No JWT token in request

**Steps**:
1. API call:
   ```bash
   curl -X POST https://api.chefooz.com/api/v1/user/addresses \
     -H "Content-Type: application/json" \
     -d '{"label": "Home", ...}'
   # Note: No Authorization header
   ```

**Expected Result**:
- API returns 401 Unauthorized:
  ```json
  {
    "statusCode": 401,
    "message": "Unauthorized"
  }
  ```

---

#### USER-ERR-002: Invalid Phone Format (9 Digits)
**Priority**: Critical  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- User authenticated

**Steps**:
1. Create address with 9-digit phone:
   ```json
   {
     "phone": "987654321",
     ...
   }
   ```

**Expected Result**:
- API returns 400 Bad Request:
  ```json
  {
    "statusCode": 400,
    "message": ["Phone must be exactly 10 digits"],
    "error": "Bad Request"
  }
  ```

---

#### USER-ERR-003: Invalid Pincode Format (Non-Numeric)
**Priority**: High  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- User authenticated

**Steps**:
1. Create address:
   ```json
   {
     "pincode": "56009A",
     ...
   }
   ```

**Expected Result**:
- API returns 400 Bad Request:
  ```json
  {
    "statusCode": 400,
    "message": ["Pincode must be exactly 6 digits"],
    "error": "Bad Request"
  }
  ```

---

#### USER-ERR-004: Address Not Found (404)
**Priority**: High  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- Address `addr_nonexistent` does not exist

**Steps**:
1. Attempt to update:
   ```bash
   curl -X PUT .../addresses/addr_nonexistent \
     -H "Authorization: Bearer <JWT>" \
     -d '{"label": "Work"}'
   ```

**Expected Result**:
- API returns 404 Not Found:
  ```json
  {
    "success": false,
    "message": "Address not found",
    "errorCode": "ADDR_NOT_FOUND"
  }
  ```

---

#### USER-ERR-005: Missing Required Field (Line1)
**Priority**: Critical  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- User authenticated

**Steps**:
1. Create address without `line1`:
   ```json
   {
     "label": "Home",
     "fullName": "Rajesh",
     "phone": "9876543210",
     "city": "Bangalore",
     "state": "Karnataka",
     "pincode": "560095"
     // line1 missing
   }
   ```

**Expected Result**:
- API returns 400 Bad Request:
  ```json
  {
    "statusCode": 400,
    "message": ["line1 should not be empty"],
    "error": "Bad Request"
  }
  ```

---

#### USER-ERR-006: Invalid Address Label
**Priority**: High  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- User authenticated

**Steps**:
1. Create address:
   ```json
   {
     "label": "Office",
     ...
   }
   ```

**Expected Result**:
- API returns 400 Bad Request:
  ```json
  {
    "statusCode": 400,
    "message": ["Label must be one of: Home, Work, Other"],
    "error": "Bad Request"
  }
  ```

---

#### USER-ERR-007: Username Check Without Query Param
**Priority**: High  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- None

**Steps**:
1. API call:
   ```bash
   curl -X GET "https://api.chefooz.com/api/v1/users/check-username"
   # Missing ?username= parameter
   ```

**Expected Result**:
```json
{
  "success": false,
  "message": "Username parameter is required",
  "errorCode": "MISSING_USERNAME"
}
```

---

#### USER-ERR-008: Coin Accrual for Non-Existent User
**Priority**: Critical  
**Platform**: Backend Service  
**Automation**: [@automated]

**Preconditions**:
- User `user_nonexistent` does not exist

**Steps**:
1. Award coins:
   ```typescript
   await userService.awardCoinsWithReputation('user_nonexistent', 100, 'order');
   ```

**Expected Result**:
- Throws `NotFoundException`:
  ```json
  {
    "success": false,
    "message": "User not found",
    "errorCode": "USER_NOT_FOUND"
  }
  ```

---

## 🔒 Security Tests

#### USER-SEC-001: JWT Token Validation
**Priority**: Critical  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- Valid JWT token available

**Steps**:
1. Create address with valid JWT → Success
2. Create address with expired JWT → 401 Unauthorized
3. Create address with tampered JWT → 401 Unauthorized
4. Create address with no JWT → 401 Unauthorized

**Expected Result**:
- Only valid, non-expired JWT tokens accepted
- All other scenarios: 401 Unauthorized

---

#### USER-SEC-002: Ownership Verification on Update
**Priority**: Critical  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- User A owns `addr_home`
- User B logged in

**Steps**:
1. User B attempts to update User A's address

**Expected Result**:
- API returns 403 Forbidden
- Address not modified

---

#### USER-SEC-003: Ownership Verification on Delete
**Priority**: Critical  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- User A owns `addr_home`
- User B logged in

**Steps**:
1. User B attempts to delete User A's address

**Expected Result**:
- API returns 403 Forbidden
- Address not deleted

---

#### USER-SEC-004: SQL Injection in Username Check
**Priority**: Critical  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- None

**Steps**:
1. Malicious username check:
   ```bash
   curl -X GET ".../check-username?username=admin' OR '1'='1"
   ```

**Expected Result**:
- No SQL injection (parameterized queries)
- Returns validation error (invalid username format)
- No database error exposed

---

#### USER-SEC-005: XSS in Address Fields
**Priority**: Critical  
**Platform**: Backend API + Mobile  
**Automation**: [@automated]

**Preconditions**:
- User authenticated

**Steps**:
1. Create address with script tag:
   ```json
   {
     "label": "Home",
     "fullName": "<script>alert('XSS')</script>",
     "line1": "Flat 203",
     ...
   }
   ```

**Expected Result**:
- Data stored as plain text (not executed)
- When displayed in mobile app: Shown as literal text
- No script execution

---

#### USER-SEC-006: Rate Limiting on Address Creation
**Priority**: High  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- User authenticated

**Steps**:
1. Create 10 addresses in 1 minute
2. Attempt 11th address creation

**Expected Result**:
- First 10 requests: 200 OK
- 11th request: 429 Too Many Requests
- Rate limit: 10 addresses per minute per user

---

#### USER-SEC-007: Username Enumeration via Timing Attack
**Priority**: High  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- None

**Steps**:
1. Check 10 taken usernames, measure response time
2. Check 10 available usernames, measure response time

**Expected Result**:
- Response times should be consistent (~100ms ± 10ms)
- No timing difference that reveals existence
- Use constant-time comparison where possible

---

## ⚡ Performance Tests

#### USER-PERF-001: Address List Load Time
**Priority**: High  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- User has 10 addresses

**Steps**:
1. Measure: `GET /api/v1/user/addresses` response time
2. Repeat 100 times

**Expected Result**:
- P50: <50ms
- P95: <100ms
- P99: <200ms

---

#### USER-PERF-002: Username Check Response Time (PostgreSQL Fallback)
**Priority**: Medium  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- Elasticsearch disabled (testing PostgreSQL fallback)

**Steps**:
1. Check username: `GET /api/v1/users/check-username?username=rajesh`
2. Repeat 100 times

**Expected Result**:
- P50: <100ms
- P95: <200ms
- P99: <300ms

---

#### USER-PERF-003: Concurrent Address Creates
**Priority**: High  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- 100 users logged in

**Steps**:
1. Simulate 100 concurrent address creation requests
2. Measure success rate and response times

**Expected Result**:
- Success rate: >99%
- P95 response time: <500ms
- No database deadlocks
- No race conditions on default address logic

---

#### USER-PERF-004: Coin Accrual Throughput
**Priority**: Medium  
**Platform**: Backend Service  
**Automation**: [@automated]

**Preconditions**:
- 1000 users in database

**Steps**:
1. Award coins to 1000 users simultaneously
2. Measure: Time to complete, error rate

**Expected Result**:
- Throughput: >100 coin accruals per second
- Error rate: 0%
- All user balances updated correctly

---

## 🔄 Regression Tests

#### USER-REG-001: End-to-End Address Workflow
**Priority**: Critical  
**Platform**: Mobile + Backend  
**Automation**: [@automated]

**Preconditions**:
- Fresh user account (0 addresses)

**Steps**:
1. Create Home address (default)
2. Create Work address (non-default)
3. Set Work as default
4. Update Home label to "Other"
5. Delete "Other" address
6. Verify: Only Work address remains (default)

**Expected Result**:
- All operations succeed
- Final state:
  - 1 address (Work, default)
  - Database consistent
  - Mobile UI reflects changes

---

#### USER-REG-002: Username Flow (Signup to Mention)
**Priority**: Critical  
**Platform**: Mobile + Backend  
**Automation**: [@automated]

**Preconditions**:
- New user completing profile

**Steps**:
1. Check username: `rajesh_foodie` (available)
2. Create profile with username
3. Another user searches: `raj` (autocomplete)
4. Original user appears in suggestions
5. @mention in comment: `@rajesh_foodie`

**Expected Result**:
- Username created successfully
- Searchable via autocomplete
- @mention links to user profile

---

#### USER-REG-003: Coin Accrual Across Reputation Tiers
**Priority**: High  
**Platform**: Backend Service  
**Automation**: [@automated]

**Preconditions**:
- User starts at Bronze tier

**Steps**:
1. Award 100 coins (Bronze: 1.0×) → 100 coins
2. Promote to Gold tier (1.1×)
3. Award 100 coins → 110 coins
4. Promote to Legend tier (1.3×)
5. Award 100 coins → 130 coins
6. Verify total: 100 + 110 + 130 = 340 coins

**Expected Result**:
- Coin multiplier updates immediately after tier change
- All accruals use latest multiplier

---

#### USER-REG-004: Geolocation Optional (Backward Compatibility)
**Priority**: High  
**Platform**: Backend API  
**Automation**: [@automated]

**Preconditions**:
- Old addresses without lat/lng exist

**Steps**:
1. Create new address with lat/lng
2. Fetch all addresses
3. Verify: Old addresses have `lat: null`, `lng: null`
4. Verify: New address has lat/lng values

**Expected Result**:
- Mixed addresses supported
- No errors on old addresses without geolocation

---

#### USER-REG-005: Address Cascading Delete
**Priority**: Critical  
**Platform**: Backend Database  
**Automation**: [@automated]

**Preconditions**:
- User has 3 addresses

**Steps**:
1. Delete user account
2. Verify: All user addresses deleted (cascading)

**Expected Result**:
- SQL:
  ```sql
  SELECT * FROM addresses WHERE "userId" = '<deleted-user-id>';
  -- Should return 0 rows
  ```
- Foreign key constraint enforces cascading delete

---

## 📋 Test Execution Checklist

### Before Testing
- [ ] Backend API running (dev/staging)
- [ ] PostgreSQL database seeded with test data
- [ ] MongoDB connected (for username checks in Reels)
- [ ] Elasticsearch indexed (optional, for fast username search)
- [ ] JWT tokens generated for test users
- [ ] Mobile app connected to correct API endpoint

### During Testing
- [ ] Record API response times (performance)
- [ ] Capture screenshots for manual UX tests
- [ ] Log all 4xx/5xx errors for analysis
- [ ] Monitor database queries (slow query log)
- [ ] Check CloudWatch metrics (error rates, latency)

### After Testing
- [ ] Document all bugs in issue tracker
- [ ] Update test case status (pass/fail)
- [ ] Generate test coverage report
- [ ] Validate database state (no orphaned records)
- [ ] Clean up test data (optional, for staging)

---

## 🐛 Bug Report Template

```markdown
### Bug ID: USER-BUG-XXX
**Module**: User / Address / Username / Coin Accrual
**Priority**: Critical / High / Medium / Low
**Reported By**: [Your Name]
**Date**: YYYY-MM-DD

### Summary
[One-line description]

### Steps to Reproduce
1. Step 1
2. Step 2
3. Step 3

### Expected Result
[What should happen]

### Actual Result
[What actually happened]

### Environment
- Platform: Mobile iOS / Mobile Android / Backend API
- App Version: 1.2.3
- API Endpoint: https://api-staging.chefooz.com
- User ID: user_123
- JWT Token: eyJhbGc...

### Screenshots/Logs
[Attach evidence]

### Database State
```sql
SELECT * FROM addresses WHERE id = 'addr_problematic';
```

### Severity
- [ ] Blocker (system unusable)
- [ ] Critical (major feature broken)
- [ ] High (workaround exists)
- [ ] Medium (minor inconvenience)
- [ ] Low (cosmetic issue)
```

---

**[QA_TEST_CASES_COMPLETE ✅]**
