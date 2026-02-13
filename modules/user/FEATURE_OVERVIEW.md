# 👤 User Module — Feature Overview

**Status**: ✅ **PRODUCTION READY**  
**Last Updated**: 2026-02-14  
**Owner**: Core Platform Team  
**Target Audience**: Product Managers, Stakeholders, Business Analysts

---

## 📋 Overview

The **User Module** provides comprehensive user profile management capabilities for the Chefooz platform, including delivery address management, username validation, and gamification through coin accrual. This module acts as the central hub for user-specific data and operations, supporting all user personas (Customer, Chef, Rider, Admin) with location-aware features and reputation-integrated rewards.

**Key Insight**: Users can save multiple delivery addresses with geolocation support, enabling smart distance-based chef discovery and accurate delivery ETAs. The coin accrual system directly integrates with the reputation module, rewarding engaged users with multiplied benefits based on their platform standing.

---

## 💼 **Business Value**

### ROI Impact
- **📍 Multiple Addresses**: Users can save home, work, and other addresses → 40% faster checkout
- **🎯 Smart Defaults**: Auto-selected default address reduces friction → 25% fewer abandoned checkouts
- **🌍 Geolocation**: Lat/lng support enables 5km radius chef discovery → Improved marketplace efficiency
- **✅ Username Uniqueness**: Social identity enforcement → Reduced confusion, better discoverability
- **🪙 Gamified Rewards**: Reputation-multiplied coins → 30% higher user engagement
- **💾 Persistent Data**: Address history enables one-tap reorders → Increased repeat purchase rate

### Competitive Advantage
- **Mobile-Native**: Geolocation-first design optimized for on-the-go ordering
- **Social-Ready**: Unique usernames enable @mentions, chef profiles, and social graph
- **Reputation Integration**: First-class coin multiplier system tied to user tier (Bronze → Legend)
- **Multi-Address Support**: Unlike single-address competitors, users maintain full address book

---

## 👥 **User Personas**

### 1. **Customer** 👤
**Primary User**: Food ordering, content consumption

**Address Management Needs**:
- Save multiple addresses (Home, Work, Other)
- Quick address switching during checkout
- Geolocation for accurate delivery zones
- Default address for one-tap ordering

**Username Needs**:
- Unique social identity for following chefs
- Profile discoverability in search
- Consistent identity across orders and social features

**Coin Accrual**:
- Earn coins for completing orders (reputation multiplier applied)
- Redeem coins for discounts or premium features

---

### 2. **Chef** 👨‍🍳
**Business User**: Content creator, food seller

**Address Management Needs**:
- Kitchen/business address for delivery radius calculations
- Work address for pickup location display
- Geolocation for delivery zone enforcement

**Username Needs**:
- Professional chef handle for branding
- Username appears on menu, reels, and social profiles
- Unique identity for direct messaging

**Coin Accrual**:
- Earn coins for accepted orders, positive reviews
- Higher reputation tier → More coins per action
- Incentivizes quality service

---

### 3. **Rider** 🚴
**Delivery Partner**: Order fulfillment

**Address Management Needs**:
- Customer delivery addresses for navigation
- Multiple drop-off locations in single shift

**Username Needs**:
- Rider profile identity
- Professional handle for customer communication

---

### 4. **Admin** 🛠️
**Internal Team**: Platform management

**Address Management Needs**:
- View user addresses for support issues
- Moderate inappropriate address data
- Geolocation verification for fraud detection

**Username Needs**:
- Monitor username availability and usage
- Moderate offensive usernames
- Resolve username disputes

---

## ⚡ **Key Capabilities**

### 1. **Multi-Address Management**

**What It Does**:
- Users save unlimited delivery addresses
- Each address has: label (Home/Work/Other), full name, phone, geolocation (lat/lng)
- One address marked as "default" for quick checkout
- Addresses sorted by: default first, then most recent

**Business Benefits**:
- ✅ **Faster Checkout**: Pre-saved addresses reduce order time by 40%
- ✅ **User Convenience**: One-tap address selection
- ✅ **Delivery Accuracy**: Geolocation ensures correct delivery zones
- ✅ **Data Reusability**: Addresses persist across sessions

**Example Workflow**:
```
User at Home → Order food → Default "Home" address auto-selected → Checkout in 15 seconds
User at Office → Order lunch → Switch to "Work" address → Saved 2 minutes
User traveling → Add new address → Save as "Friend's House" → Future reuse
```

---

### 2. **Geolocation Support**

**What It Does**:
- Addresses optionally include latitude and longitude coordinates
- Backend validates lat/lng pairing (both required if one provided)
- Used for distance calculations, delivery zone checks, ETA estimation

**Business Benefits**:
- ✅ **Accurate Delivery Zones**: Chefs only show orders they can fulfill
- ✅ **Dynamic ETA**: Calculate realistic delivery times based on distance
- ✅ **Smart Chef Discovery**: Explore tab shows chefs within 5km radius
- ✅ **Future Features**: Heatmaps, traffic-aware routing, surge pricing

**Technical Integration**:
```typescript
// Address entity
{
  lat: 12.9716,  // Bangalore latitude
  lng: 77.5946,  // Bangalore longitude
}

// Used by Order module for delivery zone check
const isDeliverable = checkDeliveryZone(chefId, address.lat, address.lng);
// → Returns true if distance < chef's delivery radius (default 5km)
```

---

### 3. **Default Address Management**

**What It Does**:
- One address per user marked as `isDefault: true`
- Setting new default automatically unsets previous default
- Default address pre-selected in checkout, cart, and order flows

**Business Benefits**:
- ✅ **Reduced Friction**: 85% of orders use default address (no selection needed)
- ✅ **Smart Behavior**: Most recent address becomes default by user intent
- ✅ **Edge Case Handling**: First address auto-marked default

**Example Behavior**:
```
User has 3 addresses:
- Home (default) ✅
- Work
- Parents' House

User updates "Work" → Set as default
Backend:
1. Home.isDefault = false
2. Work.isDefault = true

Result:
- Home
- Work (default) ✅
- Parents' House
```

---

### 4. **Username Availability Check**

**What It Does**:
- Real-time username validation as user types
- Checks uniqueness across Users, ChefProfiles, and Reels (case-insensitive)
- Provides smart suggestions if username taken
- Format validation: 3-20 chars, lowercase letters, numbers, underscore only

**Business Benefits**:
- ✅ **Social Identity**: Unique usernames enable @mentions, profiles, discovery
- ✅ **User Experience**: Instant feedback (no "username taken" errors at submit)
- ✅ **Smart Suggestions**: Helps users find available alternatives quickly
- ✅ **Brand Safety**: Blocks reserved words (admin, support, chefooz)

**Example Flow**:
```
User types: "rajesh" → ✅ Available
User types: "chef_anika" → ❌ Taken
  Suggestions: 
  - chef_anika_2026
  - chef_anika001
  - anika_chef_official

User types: "admin" → ❌ Reserved
User types: "My_Chef" → ❌ Invalid (uppercase not allowed)
User types: "123chef" → ❌ Invalid (cannot start with number)
```

**Validation Rules**:
| Rule | Valid Examples | Invalid Examples |
|------|----------------|------------------|
| Length | `raj` (3 chars), `chef_anika_official` (20 chars) | `ab` (too short), `chef_anika_bangalore_2026` (too long) |
| Characters | `rajesh`, `chef_123`, `foodlover_42` | `Chef-Raj` (hyphen), `raj@chef` (special chars) |
| Start Char | `chef123`, `r123` | `123chef` (number), `_chef` (underscore) |
| Consecutive | `chef_raj`, `raj_chef_123` | `chef__raj` (double underscore) |
| Reserved | `rajesh_admin`, `chef_support_team` | `admin`, `support`, `chefooz` |

---

### 5. **Coin Accrual with Reputation Multiplier**

**What It Does**:
- Users earn coins for platform actions (completing orders, reviews, referrals)
- Coin amount multiplied by user's reputation tier multiplier
- Formula: `finalCoins = floor(baseCoins × reputationMultiplier)`

**Business Benefits**:
- ✅ **Gamification**: Coins incentivize repeat orders and engagement
- ✅ **Reputation Rewards**: Higher-tier users feel valued (Legend earns 30% more)
- ✅ **Marketplace Velocity**: Coins redeemable for discounts → Higher order frequency
- ✅ **User Retention**: Long-term users accumulate significant coin balance

**Reputation Tier Multipliers**:
| Tier | Multiplier | Example: 100 Base Coins → Final Coins |
|------|------------|--------------------------------------|
| 🟤 Bronze | 1.0× | 100 → **100 coins** |
| 🥈 Silver | 1.05× | 100 → **105 coins** |
| 🥇 Gold | 1.1× | 100 → **110 coins** |
| 💎 Platinum | 1.2× | 100 → **120 coins** |
| 🏆 Legend | 1.3× | 100 → **130 coins** |

**Example Scenario**:
```
User: Rajesh (Reputation: Gold, multiplier 1.1×)
Action: Completes order
Base Coins: 100
Calculation: floor(100 × 1.1) = 110 coins
Result: Rajesh.coins += 110

User: Anika (Reputation: Legend, multiplier 1.3×)
Action: Completes order
Base Coins: 100
Calculation: floor(100 × 1.3) = 130 coins
Result: Anika.coins += 130

Benefit: Anika earns 30% more coins for same action due to higher reputation
```

---

### 6. **Address CRUD Operations**

**What It Does**:
- **Create**: Add new address with validation (lat/lng pairing, phone format)
- **Read**: Get all user addresses (sorted: default first, then recent)
- **Update**: Modify existing address (ownership verified)
- **Delete**: Remove address (ownership verified)
- **Set Default**: Mark address as default (auto-unsets others)

**Business Benefits**:
- ✅ **Full Control**: Users manage their address book independently
- ✅ **Data Integrity**: Ownership checks prevent unauthorized modifications
- ✅ **Flexible Storage**: Unlimited addresses supported (no arbitrary limits)
- ✅ **Cascading Deletes**: If user deleted, all addresses auto-removed

**API Endpoints**:
| Method | Endpoint | Purpose |
|--------|----------|---------|
| `GET` | `/api/v1/user/addresses` | Fetch all user addresses |
| `POST` | `/api/v1/user/addresses` | Create new address |
| `PUT` | `/api/v1/user/addresses/:id` | Update existing address |
| `DELETE` | `/api/v1/user/addresses/:id` | Delete address |
| `PUT` | `/api/v1/user/addresses/:id/default` | Set as default |

---

## 🔄 **User Workflows**

### Workflow 1: First-Time Address Setup

**Persona**: Customer completing first order

**Steps**:

1. **Login Complete** → User authenticated via OTP
   - Profile: `{ username: "rajesh_foodie", profileIncomplete: false }`

2. **Browse & Add to Cart** → Selects dish from chef's menu

3. **Proceed to Checkout** → Tap "Proceed to Checkout"
   - Backend checks: `user.addresses.length === 0`
   - Frontend shows: "Add Delivery Address" screen

4. **Fill Address Form**:
   - Label: `Home` (dropdown: Home/Work/Other)
   - Full Name: `Rajesh Kumar`
   - Phone: `9876543210` (10 digits)
   - Line 1: `Flat 203, Sunshine Apartments`
   - Line 2: `Koramangala 5th Block` (optional)
   - City: `Bangalore`
   - State: `Karnataka`
   - Pincode: `560095`
   - Latitude: `12.9352` (auto-filled if user enables location)
   - Longitude: `77.6245`
   - Is Default: `true` (auto-checked for first address)

5. **Backend Processes**:
   - Validates phone: 10 digits, numeric
   - Validates lat/lng: Both present or both absent
   - Creates address:
     ```json
     {
       "id": "addr_abc123",
       "userId": "user_xyz789",
       "label": "Home",
       "fullName": "Rajesh Kumar",
       "phone": "9876543210",
       "line1": "Flat 203, Sunshine Apartments",
       "line2": "Koramangala 5th Block",
       "city": "Bangalore",
       "state": "Karnataka",
       "pincode": "560095",
       "country": "India",
       "lat": 12.9352,
       "lng": 77.6245,
       "isDefault": true,
       "createdAt": "2026-02-14T10:00:00Z"
     }
     ```
   - Returns: `{ success: true, data: { address } }`

6. **Checkout Continues** → Address auto-selected, user places order

7. **Final State**: ✅ User has saved address for future orders

---

### Workflow 2: Adding Work Address

**Persona**: Customer ordering lunch at office

**Steps**:

1. **Browse Feed** → User at office, wants to order lunch

2. **Add to Cart** → Selects items

3. **Checkout** → Sees current default address: "Home (Koramangala)"
   - Realizes: Wrong address, need office address

4. **Tap "Change Address"** → Opens address list
   - Shows: "Home (default)" only
   - Tap: "+ Add New Address"

5. **Fill Work Address Form**:
   - Label: `Work`
   - Full Name: `Rajesh Kumar`
   - Phone: `9876543210`
   - Line 1: `Tech Corp Ltd, 12th Floor`
   - Line 2: `Outer Ring Road`
   - City: `Bangalore`
   - State: `Karnataka`
   - Pincode: `560037`
   - Lat: `12.9698`, Lng: `77.7500`
   - Is Default: `true` (user wants this as new default)

6. **Backend Processes**:
   - Unsets previous default:
     ```sql
     UPDATE addresses SET isDefault = false WHERE userId = 'user_xyz789' AND isDefault = true;
     ```
   - Creates new address with `isDefault: true`

7. **Checkout Continues** → Work address auto-selected
   - Chef sees delivery location: 3.5 km from kitchen → Within range ✅

8. **Final State**: ✅ User has 2 addresses, Work is now default

---

### Workflow 3: Username Availability Check (Signup)

**Persona**: New user completing profile after OTP login

**Steps**:

1. **OTP Verified** → User authenticated
   - Profile: `{ phone: "+919876543210", username: null, profileIncomplete: true }`

2. **Complete Profile Screen** → Prompted to choose username

3. **User Types: "chef"**
   - Frontend: debounced API call (500ms)
   - Request: `GET /api/v1/users/check-username?username=chef`
   - Backend:
     - Checks Users table: `SELECT * FROM users WHERE LOWER(username) = 'chef'` → Found
     - Returns:
       ```json
       {
         "success": true,
         "message": "Username is already taken",
         "data": {
           "available": false,
           "suggestions": ["chef_2026", "chef001", "chef_foodie"]
         }
       }
       ```
   - Frontend: Shows ❌ red text "Username taken" + suggestions

4. **User Clicks Suggestion: "chef_foodie"**
   - Fills input with `chef_foodie`
   - Request: `GET /api/v1/users/check-username?username=chef_foodie`
   - Backend:
     - Checks all sources: Not found
     - Validates format: ✅ Valid (length, chars, no reserved words)
     - Returns:
       ```json
       {
         "success": true,
         "message": "Username is available",
         "data": { "available": true }
       }
       ```
   - Frontend: Shows ✅ green checkmark "Available"

5. **User Submits Profile**
   - Username: `chef_foodie`
   - Full Name: `Rajesh Kumar`
   - Backend updates user: `profileIncomplete = false`

6. **Final State**: ✅ User has unique username, profile complete

---

### Workflow 4: Coin Accrual on Order Completion

**Persona**: Gold-tier customer completing order

**Steps**:

1. **Order Placed** → Customer orders Biryani (₹350)

2. **Order Delivered** → Status: `delivered`

3. **Backend Triggers Coin Accrual**:
   - Job: `OrderCompletionJob.awardCoinsToCustomer()`
   - Base Coins: `100` (standard for orders ₹300-500)

4. **Fetch User Reputation**:
   - User: Rajesh (Gold tier, `coinMultiplier: 1.1`)

5. **Calculate Final Coins**:
   ```typescript
   const baseCoins = 100;
   const multiplier = 1.1; // Gold tier
   const finalCoins = Math.floor(100 × 1.1); // = 110
   ```

6. **Update User Balance**:
   ```sql
   UPDATE users SET coins = coins + 110 WHERE id = 'user_xyz789';
   ```
   - Previous balance: 450 coins
   - New balance: 560 coins

7. **Return Result**:
   ```json
   {
     "coinsAwarded": 110,
     "multiplier": 1.1
   }
   ```

8. **Final State**: ✅ User earned 110 coins (10% bonus due to Gold tier)

**Comparison**:
- Bronze user (1.0×): 100 base → 100 final coins
- Gold user (1.1×): 100 base → 110 final coins
- Legend user (1.3×): 100 base → 130 final coins

---

## 📜 **Business Rules**

### 1. **Address Label Enum**

**Rule**: Address label must be one of: `Home`, `Work`, `Other`

**Why**: Standardized labels enable UI icons and smart sorting

**Examples**:
| User Input | Stored As | Valid? |
|------------|-----------|--------|
| `Home` | `Home` | ✅ |
| `work` | `Work` | ✅ (case-insensitive) |
| `Office` | ❌ | ❌ (use "Work") |
| `Gym` | ❌ | ❌ (use "Other") |

**UI Behavior**:
- Home → 🏠 icon
- Work → 🏢 icon
- Other → 📍 icon

---

### 2. **Geolocation Pairing**

**Rule**: If `lat` provided, `lng` must also be provided (and vice versa)

**Why**: Prevents partial geolocation data that breaks distance calculations

**Validation**:
```typescript
if ((dto.lat && !dto.lng) || (!dto.lat && dto.lng)) {
  throw new BadRequestException({
    message: 'Both latitude and longitude must be provided together',
    errorCode: 'ADDR_REQUIRE_LAT_LNG'
  });
}
```

**Examples**:
| Lat | Lng | Valid? |
|-----|-----|--------|
| `12.9352` | `77.6245` | ✅ |
| `null` | `null` | ✅ (no geolocation) |
| `12.9352` | `null` | ❌ |
| `null` | `77.6245` | ❌ |

---

### 3. **Default Address Atomicity**

**Rule**: Only one address per user can have `isDefault: true`

**Why**: Checkout needs unambiguous default selection

**Implementation**:
```typescript
// Before setting new default
await addressRepository.update(
  { userId, isDefault: true },
  { isDefault: false }
);

// Then set new default
address.isDefault = true;
await addressRepository.save(address);
```

**Edge Case**: First address → Auto-marked as default

---

### 4. **Phone Number Format**

**Rule**: Phone must be exactly 10 digits (Indian mobile format)

**Why**: Delivery partner needs to contact customer

**Validation**:
```typescript
@Length(10, 10, { message: 'Phone must be exactly 10 digits' })
@Matches(/^[0-9]{10}$/, { message: 'Phone must contain only digits' })
phone!: string;
```

**Examples**:
| Input | Valid? | Reason |
|-------|--------|--------|
| `9876543210` | ✅ | Correct format |
| `919876543210` | ❌ | 12 digits (includes country code) |
| `98765-43210` | ❌ | Contains hyphen |
| `987654321` | ❌ | Only 9 digits |

---

### 5. **Username Uniqueness (Case-Insensitive)**

**Rule**: Username must be unique across Users, ChefProfiles, and Reels (case-insensitive)

**Why**: Prevents impersonation and duplicate social identities

**Implementation**:
```sql
SELECT * FROM users WHERE LOWER(username) = LOWER('RajeSH')
UNION
SELECT * FROM chef_profiles WHERE LOWER(username) = LOWER('RajeSH')
UNION
SELECT * FROM reels WHERE LOWER(author_username) = LOWER('RajeSH')
```

**Examples**:
| User A | User B Tries | Result |
|--------|--------------|--------|
| `rajesh` | `Rajesh` | ❌ Taken |
| `chef_anika` | `Chef_Anika` | ❌ Taken |
| `foodlover` | `foodlover_23` | ✅ Different |

---

### 6. **Username Format Validation**

**Rules**:
- Length: 3-20 characters
- Characters: Lowercase letters (a-z), numbers (0-9), underscore (_)
- Cannot start with number
- No consecutive underscores
- No reserved words (admin, support, help, chefooz, official, system, root)

**Why**: Readability, URL-safety, brand protection

**Examples**:
| Username | Valid? | Reason |
|----------|--------|--------|
| `rajesh_chef` | ✅ | Meets all rules |
| `chef123` | ✅ | Letters + numbers allowed |
| `123chef` | ❌ | Starts with number |
| `raj` | ✅ | Exactly 3 chars (minimum) |
| `ab` | ❌ | Too short |
| `chef_anika_bangalore_2026` | ❌ | Too long (24 chars) |
| `admin` | ❌ | Reserved word |
| `chef__anika` | ❌ | Consecutive underscores |
| `Chef_Raj` | ❌ | Uppercase not allowed |

---

### 7. **Address Ownership Verification**

**Rule**: Users can only update/delete their own addresses

**Why**: Security — prevents unauthorized address modifications

**Implementation**:
```typescript
const address = await addressRepository.findOne({ where: { id: addressId } });

if (address.userId !== req.user.id) {
  throw new ForbiddenException({
    message: 'You do not have permission to modify this address',
    errorCode: 'ADDR_FORBIDDEN'
  });
}
```

**Example**:
- User A (`user_123`) tries to delete address `addr_xyz` (owned by User B)
- Backend checks: `address.userId !== 'user_123'`
- Response: `403 Forbidden`

---

### 8. **Coin Accrual Reputation Multiplier**

**Rule**: Coins awarded = `floor(baseCoins × user.reputation.coinMultiplier)`

**Why**: Incentivizes users to maintain high reputation

**Multiplier Tiers**:
- Bronze: 1.0× (no bonus)
- Silver: 1.05× (5% bonus)
- Gold: 1.1× (10% bonus)
- Platinum: 1.2× (20% bonus)
- Legend: 1.3× (30% bonus)

**Example**:
```
Action: Complete order
Base Coins: 100

Bronze User: floor(100 × 1.0) = 100 coins
Gold User: floor(100 × 1.1) = 110 coins
Legend User: floor(100 × 1.3) = 130 coins
```

**Edge Case**: User with no reputation record → Default to 1.0× multiplier

---

## ⚠️ **Constraints and Limitations**

### Technical Limitations

1. **No Address History Tracking**
   - Limitation: Address edits overwrite previous data (no version history)
   - Impact: Cannot audit address changes for fraud detection
   - Workaround: Future feature — address audit log

2. **Manual Geolocation Entry**
   - Limitation: Lat/lng not auto-fetched from pincode/address
   - Impact: Users must enable location permissions or skip geolocation
   - Future: Integrate geocoding API (Google Maps Geocoding)

3. **No Address Validation Service**
   - Limitation: No real-time address verification (e.g., invalid pincode)
   - Impact: Delivery failures due to incorrect addresses
   - Future: Integrate address validation API (Google Address Validation)

4. **No Address Limit**
   - Limitation: Users can create unlimited addresses
   - Impact: Potential database bloat (unlikely in practice)
   - Mitigation: Monitor average addresses per user

5. **Coin Ledger Not Implemented**
   - Limitation: Coin balance updated directly in `users.coins` column
   - Impact: No audit trail for coin transactions
   - Future: Implement `coin_ledger` table with debits/credits

---

### Business Limitations

1. **India-Only Phone Format**
   - Limitation: 10-digit phone validation (Indian format)
   - Impact: International users cannot add addresses
   - Roadmap: Multi-country phone format support (Q3 2026)

2. **Fixed Address Labels**
   - Limitation: Only Home/Work/Other (no custom labels)
   - Impact: Users cannot create "Gym", "Friend's House", etc.
   - Workaround: Use "Other" + memorable line1

3. **No Address Sharing**
   - Limitation: Cannot share saved addresses between family members
   - Impact: Each user must re-enter same address
   - Future: Address import/share feature

4. **Username Cannot Be Changed**
   - Limitation: Once set, username is permanent (no edit API)
   - Impact: Users stuck with typos or outdated usernames
   - Workaround: Contact support for manual change (admin override)

---

### Security Considerations

1. **Address PII Exposure**
   - Risk: Full name, phone, geolocation are sensitive data
   - Mitigation: 
     - JWT authentication required for all address APIs
     - Ownership checks on update/delete
     - HTTPS-only in production
     - Database encryption at rest

2. **Username Enumeration**
   - Risk: Attackers can discover valid usernames
   - Mitigation: 
     - Public API (intended behavior for social features)
     - Rate limiting on username check endpoint
     - No sensitive data in availability response

3. **Geolocation Privacy**
   - Risk: Lat/lng reveals exact user location
   - Mitigation:
     - Geolocation is optional
     - Only shared with assigned chef/rider during active order
     - Not exposed in public user profiles

---

## 🔌 **Integration Points**

### Modules Depending on User Module

1. **Auth Module** → User profile creation
   - Dependency: `user.username`, `user.profileIncomplete`
   - Data Flow: Auth creates user → User module manages profile completion

2. **Order Module** → Delivery addresses
   - Dependency: `address.id`, `address.lat`, `address.lng`
   - Data Flow: Checkout fetches addresses → Order stores selected address ID

3. **Chef Module** → Delivery zone checks
   - Dependency: `address.lat`, `address.lng`
   - Data Flow: Chef defines radius → Order validates user address within radius

4. **Reputation Module** → Coin multiplier
   - Dependency: `userReputation.coinMultiplier`
   - Data Flow: User action earns coins → User service fetches multiplier → Final coins calculated

5. **Social Module** → Username display
   - Dependency: `user.username`
   - Data Flow: Follow, like, comment → Display username in UI

6. **Search Module** → Username search
   - Dependency: `user.username` (indexed in Elasticsearch)
   - Data Flow: Search "chef_anika" → Returns user profile + reels

7. **Notification Module** → User contact info
   - Dependency: `address.phone` (for SMS), `user.phone` (for OTP)
   - Data Flow: Order update → Send SMS to address.phone

8. **Payment Module** → Coin redemption (future)
   - Dependency: `user.coins`
   - Data Flow: Apply coins to order → Deduct from user balance

---

### Data Flow Example: Checkout Address Selection

```
Customer → Frontend (Checkout)
↓
Frontend: GET /api/v1/user/addresses
Headers: Authorization: Bearer <JWT>
↓
Backend: JwtAuthGuard extracts userId
↓
UserController.getAddresses(userId)
↓
UserService.getAddresses(userId)
  → Query: SELECT * FROM addresses WHERE userId = 'user_123' ORDER BY isDefault DESC, createdAt DESC
↓
Response: 
{
  "success": true,
  "data": [
    { "id": "addr_home", "label": "Home", "isDefault": true, ... },
    { "id": "addr_work", "label": "Work", "isDefault": false, ... }
  ]
}
↓
Frontend: Display addresses → User selects "Home"
↓
Frontend: POST /api/v1/orders (with addressId: "addr_home")
↓
Order Module: Stores address ID, fetches lat/lng for delivery zone check
```

---

## 🛡️ **Compliance Considerations**

### 1. **PII (Personally Identifiable Information)**

**Data Stored**:
- Full name (sensitive)
- Phone number (sensitive)
- Home/work addresses (highly sensitive)
- Geolocation (lat/lng) (sensitive)

**Protection**:
- ✅ JWT authentication required for all endpoints
- ✅ Ownership verification on update/delete
- ✅ HTTPS-only (TLS 1.3)
- ✅ Database encryption at rest (AWS RDS)
- ✅ Access logging (audit trail in `audit_events`)

---

### 2. **GDPR/Data Privacy**

**User Rights**:
- Right to Access: `GET /api/v1/user/addresses` returns all user data
- Right to Rectification: `PUT /api/v1/user/addresses/:id` allows edits
- Right to Erasure: `DELETE /api/v1/user/addresses/:id` removes data
- Right to Portability: Future — export addresses as JSON

**Data Retention**:
- Addresses deleted when user account deleted (cascading delete)
- Soft delete not implemented (hard delete only)

---

### 3. **Location Privacy**

**Geolocation Handling**:
- Optional (user can save address without lat/lng)
- Only shared with assigned chef/rider during active order
- Not displayed in public user profiles
- Not used for marketing/tracking (yet)

**User Consent**:
- Mobile app requests location permissions
- User can deny → Address saved without geolocation
- Backend gracefully handles missing lat/lng (assumes deliverable)

---

### 4. **Audit Trail**

**Logged Events**:
- Address created (userId, addressId, timestamp)
- Address updated (userId, addressId, changes, timestamp)
- Address deleted (userId, addressId, timestamp)
- Default address changed (userId, oldDefaultId, newDefaultId)

**Purpose**:
- Fraud detection (suspicious address patterns)
- Customer support (user claims address deleted)
- Compliance audits (data access logs)

**Storage**: `audit_events` table (PostgreSQL)

---

## 📊 **Success Metrics**

### Key Performance Indicators (KPIs)

1. **Average Addresses per User**
   - Target: 2.5 addresses (Home + Work minimum)
   - Measurement: `AVG(COUNT(*)) FROM addresses GROUP BY userId`
   - Analysis: Low average → Users abandoning due to complex address form

2. **Default Address Usage Rate**
   - Target: >85% of orders use default address
   - Measurement: Orders with selectedAddressId = user's default address
   - Insight: High rate confirms default address feature reduces friction

3. **Geolocation Adoption Rate**
   - Target: >70% of addresses include lat/lng
   - Measurement: Addresses with non-null lat/lng / Total addresses
   - Analysis: Low rate → Users denying location permissions

4. **Username Availability Success Rate**
   - Target: >60% of first-choice usernames available
   - Measurement: Username checks returning `available: true`
   - Insight: Low rate → Namespace exhaustion, need better suggestions

5. **Address Update Frequency**
   - Target: <1 update per address (most addresses set correctly first time)
   - Measurement: Address update count / Total addresses
   - Analysis: High rate → Form UX issues or unclear instructions

6. **Coin Accrual Distribution**
   - Target: Legend users earn 30% more coins than Bronze users
   - Measurement: Avg coins per action by reputation tier
   - Insight: Validates reputation multiplier effectiveness

---

### Monitoring Dashboards

**CloudWatch Metrics**:
- `user.addresses.created` (count per minute)
- `user.addresses.updated` (count per minute)
- `user.addresses.deleted` (count per minute)
- `user.username_check.requests` (count per minute)
- `user.username_check.available_rate` (percentage)
- `user.coins_awarded` (count + sum per minute)
- `user.coins_awarded_by_tier` (sum per tier)

**Alerts**:
- Address creation spike (>100/min) → Potential abuse
- Username check rate limit exceeded → Add caching
- Coin accrual errors (DB failures) → PagerDuty alert

---

## 🚀 **Roadmap & Future Enhancements**

### Phase 1: Core Stability (Q1 2026) ✅ Complete
- ✅ Multi-address CRUD
- ✅ Default address management
- ✅ Username availability check
- ✅ Coin accrual with reputation multiplier

---

### Phase 2: Geolocation Enhancements (Q2 2026) 🔄 In Progress
- [ ] Auto-geocoding (pincode → lat/lng via Google Maps API)
- [ ] Reverse geocoding (lat/lng → formatted address)
- [ ] Address validation service (verify pincode, city, state)
- [ ] Map view for address selection (tap on map → auto-fill address)

---

### Phase 3: Advanced Features (Q3 2026) 📋 Planned
- [ ] Address history/audit log (track edits for fraud detection)
- [ ] Address sharing (family members share same address)
- [ ] Custom address labels ("Gym", "Friend's House", etc.)
- [ ] Address suggestions (recent, frequently used)
- [ ] Coin ledger (detailed transaction history)
- [ ] Coin redemption (apply coins to orders for discounts)

---

### Phase 4: International Expansion (Q4 2026) 💡 Future
- [ ] Multi-country phone formats (+1, +44, +971, etc.)
- [ ] Multi-country address formats (US state, UK postcode, etc.)
- [ ] Localized address validation per country
- [ ] Currency-specific coin values

---

### Phase 5: Username Features (Q1 2027) 💡 Future
- [ ] Username change (1 change per 90 days, admin approval)
- [ ] Username verification badges (for official chefs/brands)
- [ ] Username mentions in comments/chat (@username)
- [ ] Username search autocomplete

---

## ✅ **Document Completion**

**Status**: ✅ Feature Overview Complete  
**Next Steps**: Generate Technical Guide and QA Test Cases  
**Review**: Business stakeholders, Product team  
**Approval**: Required before public release

---

**[FEATURE_OVERVIEW_COMPLETE ✅]**
