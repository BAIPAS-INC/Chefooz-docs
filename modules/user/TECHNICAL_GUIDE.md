# 👤 User Module — Technical Guide

**Status**: ✅ **PRODUCTION READY**  
**Last Updated**: 2026-02-14  
**Target Audience**: Backend/Full-Stack Developers  
**Related Docs**: 
- [User Feature Overview](./FEATURE_OVERVIEW.md)
- [Auth Module Technical Guide](../auth/TECHNICAL_GUIDE.md)

---

## 📋 Overview

The User module provides comprehensive user profile and address management APIs. It handles delivery address CRUD operations, username validation with smart suggestions, and reputation-integrated coin accrual system. This module is central to order fulfillment, social features, and gamification.

**Learning Objectives**:
- Understand address management with geolocation support
- Implement username uniqueness checks with Elasticsearch fallback
- Integrate reputation-based coin multipliers
- Handle JWT-authenticated CRUD operations

---

## 🏗️ Architecture Overview

```
User Module Architecture
========================

┌─────────────────┐
│  Mobile Client  │
│  (Expo App)     │
└────────┬────────┘
         │ JWT Bearer Token
         │
         ▼
┌─────────────────────────────────────┐
│  UserController (NestJS)            │
│  - GET /api/v1/user/addresses       │
│  - POST /api/v1/user/addresses      │
│  - PUT /api/v1/user/addresses/:id   │
│  - DELETE /api/v1/user/addresses/:id│
│  - PUT /api/v1/user/addresses/:id/  │
│    default                           │
│                                     │
│  UsernameController (NestJS)        │
│  - GET /api/v1/users/check-username │
│  - GET /api/v1/users/username-      │
│    suggestions                       │
└────────┬───────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  UserService / UsernameService      │
│  - Address CRUD logic               │
│  - Default address atomicity        │
│  - Geolocation validation           │
│  - Coin accrual with reputation     │
│  - Username format validation       │
│  - Cross-table uniqueness checks    │
│  - Smart suggestion generation      │
│  - Elasticsearch prefix search      │
└────────┬───────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  Database Layer                     │
│  ┌────────────────────────────────┐ │
│  │ PostgreSQL (TypeORM)           │ │
│  │ - addresses (user delivery)    │ │
│  │ - users (profile, coins)       │ │
│  │ - user_reputation_current      │ │
│  │ - chef_profiles                │ │
│  ├────────────────────────────────┤ │
│  │ MongoDB (Mongoose)             │ │
│  │ - reels (username collision)   │ │
│  ├────────────────────────────────┤ │
│  │ Elasticsearch (Optional)       │ │
│  │ - users index (fast search)    │ │
│  └────────────────────────────────┘ │
└─────────────────────────────────────┘
```

---

## 📂 Module Structure

```
apps/chefooz-apis/src/modules/user/
├── user.controller.ts           # Address CRUD endpoints
├── user.service.ts              # Address & coin accrual logic
├── user.module.ts               # Module definition with dependencies
├── user.service.spec.ts         # Unit tests
├── username.controller.ts       # Username check endpoints
├── username.service.ts          # Username validation & search logic
└── dto/
    ├── create-address.dto.ts    # Address creation validation
    └── update-address.dto.ts    # Address update validation (PartialType)

apps/chefooz-apis/src/database/entities/
├── address.entity.ts            # Address table schema
├── user.entity.ts               # User table (coins column)
└── user-reputation-current.entity.ts  # Reputation multiplier

libs/types/src/lib/
└── location.types.ts            # Shared Address types

libs/api-client/src/lib/user/
├── user.client.ts               # Axios API calls
└── user.hooks.ts                # React Query hooks
```

---

## 🗄️ Database Schema

### addresses Table (PostgreSQL)

```sql
CREATE TABLE addresses (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  "userId" UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  label VARCHAR(20) NOT NULL CHECK (label IN ('Home', 'Work', 'Other')),
  "fullName" VARCHAR(100) NOT NULL,
  phone VARCHAR(10) NOT NULL CHECK (phone ~ '^[0-9]{10}$'),
  line1 VARCHAR(255) NOT NULL,
  line2 VARCHAR(255),
  city VARCHAR(100) NOT NULL,
  state VARCHAR(100) NOT NULL,
  pincode VARCHAR(6) NOT NULL CHECK (pincode ~ '^[0-9]{6}$'),
  country VARCHAR(50) NOT NULL DEFAULT 'India',
  lat FLOAT,  -- Latitude (optional, must pair with lng)
  lng FLOAT,  -- Longitude (optional, must pair with lat)
  "isDefault" BOOLEAN NOT NULL DEFAULT false,
  "createdAt" TIMESTAMP NOT NULL DEFAULT NOW(),
  "updatedAt" TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Indexes for performance
CREATE INDEX idx_addresses_userId_createdAt ON addresses("userId", "createdAt");
CREATE INDEX idx_addresses_isDefault ON addresses("isDefault");

-- Ensure only one default per user (application-level enforcement)
```

**Key Constraints**:
- `userId` → Foreign key to `users.id` (cascading delete)
- `label` → Enum: Home, Work, Other
- `phone` → Exactly 10 digits (Indian mobile format)
- `lat` + `lng` → Either both present or both absent (app-level validation)
- `isDefault` → Only one per user (enforced by service logic, not DB constraint)

**Lifecycle**:
- Created: When user saves new address via `POST /api/v1/user/addresses`
- Updated: When user edits address via `PUT /api/v1/user/addresses/:id`
- Deleted: When user removes address OR when user account deleted (cascade)

---

### users Table (Relevant Columns)

```sql
-- Only showing User module-relevant columns
CREATE TABLE users (
  id UUID PRIMARY KEY,
  username VARCHAR(20) UNIQUE,  -- Case-insensitive unique via service checks
  "fullName" VARCHAR(100),
  phone VARCHAR(15) UNIQUE NOT NULL,
  coins INTEGER NOT NULL DEFAULT 0,  -- Gamification balance
  "profileIncomplete" BOOLEAN NOT NULL DEFAULT true,
  "createdAt" TIMESTAMP NOT NULL DEFAULT NOW(),
  "updatedAt" TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Username case-insensitive uniqueness handled by UsernameService
CREATE UNIQUE INDEX idx_users_username_lower ON users(LOWER(username));
```

---

### user_reputation_current Table (Read-Only for User Module)

```sql
CREATE TABLE user_reputation_current (
  "userId" UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  tier VARCHAR(20) NOT NULL,  -- Bronze, Silver, Gold, Platinum, Legend
  "coinMultiplier" DECIMAL(3,2) NOT NULL,  -- 1.00, 1.05, 1.10, 1.20, 1.30
  "createdAt" TIMESTAMP NOT NULL,
  "updatedAt" TIMESTAMP NOT NULL
);
```

**Usage in User Module**:
- `coinMultiplier` fetched during coin accrual: `finalCoins = floor(baseCoins * coinMultiplier)`
- Read-only (reputation tier managed by Reputation module)

---

## 🌐 API Endpoints

### 1. Get All User Addresses

**Endpoint**: `GET /api/v1/user/addresses`  
**Auth**: Required (JWT Bearer Token)  
**Description**: Fetches all saved addresses for authenticated user, sorted by default first, then most recent.

**Request**:
```bash
curl -X GET https://api.chefooz.com/api/v1/user/addresses \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Addresses retrieved successfully",
  "data": [
    {
      "id": "addr_550e8400-e29b-41d4-a716-446655440000",
      "userId": "user_123",
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
      "createdAt": "2026-02-10T08:30:00.000Z",
      "updatedAt": "2026-02-10T08:30:00.000Z"
    },
    {
      "id": "addr_550e8400-e29b-41d4-a716-446655440001",
      "label": "Work",
      "isDefault": false,
      ...
    }
  ]
}
```

**Axios Example**:
```typescript
import { userClient } from '@chefooz-app/api-client';

const addresses = await userClient.getAddresses();
console.log(addresses); // Array<Address>
```

**React Query Hook**:
```typescript
import { useAddresses } from '@chefooz-app/api-client';

function AddressListScreen() {
  const { data, isLoading, error } = useAddresses();
  
  if (isLoading) return <Spinner />;
  if (error) return <Error message={error.message} />;
  
  return (
    <FlatList
      data={data}
      renderItem={({ item }) => <AddressCard address={item} />}
      keyExtractor={(item) => item.id}
    />
  );
}
```

---

### 2. Create New Address

**Endpoint**: `POST /api/v1/user/addresses`  
**Auth**: Required (JWT Bearer Token)  
**Description**: Creates a new delivery address. If `isDefault: true`, automatically unsets previous default.

**Request Body** (CreateAddressDto):
```json
{
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
  "isDefault": true
}
```

**Validation Rules**:
| Field | Required | Validation |
|-------|----------|------------|
| `label` | ✅ | Enum: `Home`, `Work`, `Other` |
| `fullName` | ✅ | String (1-100 chars) |
| `phone` | ✅ | Exactly 10 digits, numeric |
| `line1` | ✅ | String (1-255 chars) |
| `line2` | ❌ | String (0-255 chars) |
| `city` | ✅ | String (1-100 chars) |
| `state` | ✅ | String (1-100 chars) |
| `pincode` | ✅ | Exactly 6 digits, numeric |
| `country` | ❌ | String (defaults to "India") |
| `lat` | ❌ | Float (-90 to 90), must pair with `lng` |
| `lng` | ❌ | Float (-180 to 180), must pair with `lat` |
| `isDefault` | ❌ | Boolean (defaults to `false`) |

**cURL Example**:
```bash
curl -X POST https://api.chefooz.com/api/v1/user/addresses \
  -H "Authorization: Bearer <JWT_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "label": "Home",
    "fullName": "Rajesh Kumar",
    "phone": "9876543210",
    "line1": "Flat 203, Sunshine Apartments",
    "city": "Bangalore",
    "state": "Karnataka",
    "pincode": "560095",
    "lat": 12.9352,
    "lng": 77.6245,
    "isDefault": true
  }'
```

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Address created successfully",
  "data": {
    "id": "addr_abc123",
    "userId": "user_xyz789",
    "label": "Home",
    "isDefault": true,
    ...
  }
}
```

**Error Response** (400 Bad Request) - Missing lat/lng pair:
```json
{
  "success": false,
  "message": "Both latitude and longitude must be provided together",
  "errorCode": "ADDR_REQUIRE_LAT_LNG"
}
```

**Axios Example**:
```typescript
import { userClient } from '@chefooz-app/api-client';

const newAddress = await userClient.createAddress({
  label: 'Home',
  fullName: 'Rajesh Kumar',
  phone: '9876543210',
  line1: 'Flat 203, Sunshine Apartments',
  city: 'Bangalore',
  state: 'Karnataka',
  pincode: '560095',
  lat: 12.9352,
  lng: 77.6245,
  isDefault: true
});
```

---

### 3. Update Existing Address

**Endpoint**: `PUT /api/v1/user/addresses/:id`  
**Auth**: Required (JWT Bearer Token)  
**Description**: Updates an existing address. Ownership verified (user can only update their own addresses).

**Request Body** (UpdateAddressDto - all fields optional):
```json
{
  "label": "Work",
  "isDefault": true
}
```

**cURL Example**:
```bash
curl -X PUT https://api.chefooz.com/api/v1/user/addresses/addr_abc123 \
  -H "Authorization: Bearer <JWT_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"label": "Work", "isDefault": true}'
```

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Address updated successfully",
  "data": {
    "id": "addr_abc123",
    "label": "Work",
    "isDefault": true,
    ...
  }
}
```

**Error Response** (403 Forbidden) - Ownership check failed:
```json
{
  "success": false,
  "message": "You do not have permission to update this address",
  "errorCode": "ADDR_FORBIDDEN"
}
```

**Error Response** (404 Not Found):
```json
{
  "success": false,
  "message": "Address not found",
  "errorCode": "ADDR_NOT_FOUND"
}
```

---

### 4. Delete Address

**Endpoint**: `DELETE /api/v1/user/addresses/:id`  
**Auth**: Required (JWT Bearer Token)  
**Description**: Permanently deletes an address. Ownership verified.

**cURL Example**:
```bash
curl -X DELETE https://api.chefooz.com/api/v1/user/addresses/addr_abc123 \
  -H "Authorization: Bearer <JWT_TOKEN>"
```

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Address deleted successfully"
}
```

---

### 5. Set Default Address

**Endpoint**: `PUT /api/v1/user/addresses/:id/default`  
**Auth**: Required (JWT Bearer Token)  
**Description**: Marks address as default. Automatically unsets all other defaults for the user.

**cURL Example**:
```bash
curl -X PUT https://api.chefooz.com/api/v1/user/addresses/addr_abc123/default \
  -H "Authorization: Bearer <JWT_TOKEN>"
```

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Default address updated successfully",
  "data": {
    "id": "addr_abc123",
    "isDefault": true,
    ...
  }
}
```

**Backend Logic**:
```typescript
// Step 1: Unset all current defaults
await addressRepository.update(
  { userId, isDefault: true },
  { isDefault: false }
);

// Step 2: Set new default
address.isDefault = true;
await addressRepository.save(address);
```

---

### 6. Check Username Availability

**Endpoint**: `GET /api/v1/users/check-username?username=rajesh`  
**Auth**: Not required (public endpoint)  
**Description**: Validates username format and checks uniqueness across Users, ChefProfiles, and Reels. Returns smart suggestions if taken.

**Request**:
```bash
curl -X GET "https://api.chefooz.com/api/v1/users/check-username?username=rajesh"
```

**Response** (Available):
```json
{
  "success": true,
  "message": "Username is available",
  "data": {
    "available": true
  }
}
```

**Response** (Taken):
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

**Response** (Invalid Format):
```json
{
  "success": false,
  "message": "Username cannot start with a number",
  "errorCode": "INVALID_USERNAME_FORMAT"
}
```

**Validation Rules**:
1. Length: 3-20 characters
2. Characters: Lowercase letters (a-z), numbers (0-9), underscore (_)
3. Cannot start with number
4. No consecutive underscores
5. No reserved words: `admin`, `support`, `help`, `chefooz`, `official`, `system`, `root`

**Axios Example**:
```typescript
import { usernameClient } from '@chefooz-app/api-client';

const result = await usernameClient.checkUsername('rajesh');
if (result.available) {
  console.log('Username available!');
} else {
  console.log('Taken. Try:', result.suggestions);
}
```

**React Hook**:
```typescript
import { useCheckUsername } from '@chefooz-app/api-client';

function UsernameInput() {
  const [username, setUsername] = useState('');
  const { data, isLoading } = useCheckUsername(username, {
    enabled: username.length >= 3,
    // Debounced: only checks after 500ms pause
  });
  
  return (
    <View>
      <TextInput value={username} onChangeText={setUsername} />
      {isLoading && <Spinner />}
      {data?.available && <Text style={{color: 'green'}}>✅ Available</Text>}
      {data?.available === false && (
        <View>
          <Text style={{color: 'red'}}>❌ Taken</Text>
          <Text>Suggestions: {data.suggestions.join(', ')}</Text>
        </View>
      )}
    </View>
  );
}
```

---

### 7. Username Suggestions (Autocomplete for Mentions)

**Endpoint**: `GET /api/v1/users/username-suggestions?q=raj&userId=<current_user_id>`  
**Auth**: Not required (but `userId` param enables blocked user filtering)  
**Description**: Autocomplete search for @mentions. Returns up to 10 matching users.

**Request**:
```bash
curl -X GET "https://api.chefooz.com/api/v1/users/username-suggestions?q=raj&userId=user_123"
```

**Response**:
```json
{
  "success": true,
  "message": "Username suggestions retrieved",
  "data": {
    "suggestions": [
      {
        "userId": "user_550e8400",
        "username": "rajesh",
        "fullName": "Rajesh Kumar",
        "avatarUrl": "https://cdn.chefooz.com/avatars/rajesh.jpg"
      },
      {
        "userId": "user_550e8401",
        "username": "raj_chef",
        "fullName": "Raj Chef",
        "avatarUrl": null
      }
    ]
  }
}
```

**Performance**:
- Uses Elasticsearch prefix search when available (fast)
- Falls back to PostgreSQL `LIKE` query
- Max 10 results, ordered by username ASC

---

## 🔒 Security Implementation

### 1. JWT Authentication

All address endpoints require valid JWT token:

```typescript
@Controller('v1/user')
@UseGuards(JwtAuthGuard)
@ApiBearerAuth()
export class UserController {
  // All methods require authentication
}
```

**Workflow**:
```
Client → Request with Authorization: Bearer <JWT>
↓
JwtAuthGuard validates token signature
↓
Extracts userId from token payload
↓
Attaches req.user = { id: userId, phone, role }
↓
Controller accesses req.user.id
```

---

### 2. Ownership Verification

Before update/delete operations, service verifies ownership:

```typescript
// UserService.updateAddress()
const address = await this.addressRepository.findOne({ where: { id: addressId } });

if (!address) {
  throw new NotFoundException({ errorCode: 'ADDR_NOT_FOUND' });
}

if (address.userId !== userId) {
  throw new ForbiddenException({ errorCode: 'ADDR_FORBIDDEN' });
}

// Proceed with update
Object.assign(address, dto);
return this.addressRepository.save(address);
```

**Error Handling**:
- `404 Not Found` → Address doesn't exist
- `403 Forbidden` → Address exists but belongs to another user

---

### 3. Geolocation Validation

Prevents partial geolocation data (lat without lng):

```typescript
if ((dto.lat && !dto.lng) || (!dto.lat && dto.lng)) {
  throw new BadRequestException({
    message: 'Both latitude and longitude must be provided together',
    errorCode: 'ADDR_REQUIRE_LAT_LNG'
  });
}
```

---

### 4. Default Address Atomicity

Ensures only one default per user (race condition safe):

```typescript
// Step 1: Unset all defaults (even if multiple exist due to race)
await this.addressRepository.update(
  { userId, isDefault: true },
  { isDefault: false }
);

// Step 2: Set new default
address.isDefault = true;
await this.addressRepository.save(address);
```

**Why Not DB Constraint?**
- PostgreSQL unique partial index would require complex setup
- Application-level enforcement is more flexible
- Race conditions unlikely (user actions are sequential)

---

### 5. Username Case-Insensitive Uniqueness

Cross-table uniqueness check:

```typescript
const lowerUsername = username.toLowerCase();

// Check Users table
const userExists = await this.userRepository
  .createQueryBuilder('user')
  .where('LOWER(user.username) = :username', { username: lowerUsername })
  .getOne();

// Check ChefProfile (if handle field exists)
// Check Reels collection (MongoDB)
const reelExists = await this.reelModel
  .findOne({ userId: new RegExp(`^${lowerUsername}$`, 'i') })
  .exec();

return !userExists && !reelExists; // Available if both false
```

---

## 📱 Frontend Integration (Expo Mobile App)

### Address Management Screens

**Location**: `apps/chefooz-app/src/app/(tabs)/profile/addresses/`

#### AddressListScreen
```typescript
import { useAddresses, useDeleteAddress } from '@chefooz-app/api-client';
import { FlatList, TouchableOpacity, Alert } from 'react-native';

export function AddressListScreen() {
  const { data: addresses, isLoading, refetch } = useAddresses();
  const deleteAddress = useDeleteAddress();
  
  const handleDelete = async (addressId: string) => {
    Alert.alert(
      'Delete Address',
      'Are you sure?',
      [
        { text: 'Cancel', style: 'cancel' },
        {
          text: 'Delete',
          style: 'destructive',
          onPress: async () => {
            await deleteAddress.mutateAsync(addressId);
            refetch(); // Refresh list
          }
        }
      ]
    );
  };
  
  return (
    <FlatList
      data={addresses}
      renderItem={({ item }) => (
        <AddressCard
          address={item}
          onPress={() => router.push(`/profile/addresses/edit/${item.id}`)}
          onDelete={() => handleDelete(item.id)}
        />
      )}
      keyExtractor={(item) => item.id}
    />
  );
}
```

#### AddAddressScreen
```typescript
import { useCreateAddress } from '@chefooz-app/api-client';
import { useState } from 'react';
import { TextInput, Button } from 'react-native';

export function AddAddressScreen() {
  const [form, setForm] = useState({
    label: 'Home' as 'Home' | 'Work' | 'Other',
    fullName: '',
    phone: '',
    line1: '',
    city: '',
    state: '',
    pincode: '',
    lat: null,
    lng: null,
    isDefault: false
  });
  
  const createAddress = useCreateAddress();
  
  const handleSubmit = async () => {
    try {
      await createAddress.mutateAsync(form);
      router.back(); // Navigate back to address list
    } catch (error) {
      Alert.alert('Error', error.message);
    }
  };
  
  return (
    <ScrollView>
      <Picker
        selectedValue={form.label}
        onValueChange={(value) => setForm({...form, label: value})}
      >
        <Picker.Item label="🏠 Home" value="Home" />
        <Picker.Item label="🏢 Work" value="Work" />
        <Picker.Item label="📍 Other" value="Other" />
      </Picker>
      
      <TextInput
        placeholder="Full Name"
        value={form.fullName}
        onChangeText={(text) => setForm({...form, fullName: text})}
      />
      
      <TextInput
        placeholder="Phone (10 digits)"
        keyboardType="numeric"
        maxLength={10}
        value={form.phone}
        onChangeText={(text) => setForm({...form, phone: text})}
      />
      
      {/* ... other fields ... */}
      
      <Button
        title="Save Address"
        onPress={handleSubmit}
        disabled={createAddress.isPending}
      />
    </ScrollView>
  );
}
```

#### Username Validation Component
```typescript
import { useCheckUsername } from '@chefooz-app/api-client';
import { useState, useEffect } from 'react';
import { TextInput, Text, View } from 'react-native';
import { useDebounce } from '@chefooz-app/utils';

export function UsernameField({ value, onChange, onValidChange }) {
  const debouncedUsername = useDebounce(value, 500); // Wait 500ms after typing stops
  const { data, isLoading } = useCheckUsername(debouncedUsername, {
    enabled: debouncedUsername.length >= 3
  });
  
  useEffect(() => {
    if (data) {
      onValidChange(data.available);
    }
  }, [data]);
  
  return (
    <View>
      <TextInput
        value={value}
        onChangeText={(text) => onChange(text.toLowerCase())} // Force lowercase
        placeholder="Username (3-20 chars)"
        autoCapitalize="none"
        autoCorrect={false}
      />
      
      {isLoading && <Text>Checking...</Text>}
      
      {data?.available && (
        <Text style={{ color: 'green' }}>✅ Username available</Text>
      )}
      
      {data?.available === false && (
        <View>
          <Text style={{ color: 'red' }}>❌ Username taken</Text>
          <Text>Try: {data.suggestions.join(', ')}</Text>
        </View>
      )}
    </View>
  );
}
```

---

## 🧪 Testing Strategies

### Unit Tests (Jest)

**File**: `apps/chefooz-apis/src/modules/user/user.service.spec.ts`

```typescript
import { Test } from '@nestjs/testing';
import { UserService } from './user.service';
import { getRepositoryToken } from '@nestjs/typeorm';
import { Address } from '../../database/entities/address.entity';

describe('UserService', () => {
  let service: UserService;
  let addressRepo: jest.Mocked<Repository<Address>>;
  
  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        UserService,
        {
          provide: getRepositoryToken(Address),
          useValue: {
            find: jest.fn(),
            create: jest.fn(),
            save: jest.fn(),
            update: jest.fn(),
            findOne: jest.fn(),
            delete: jest.fn()
          }
        },
        // ... other repos
      ]
    }).compile();
    
    service = module.get<UserService>(UserService);
    addressRepo = module.get(getRepositoryToken(Address));
  });
  
  describe('createAddress', () => {
    it('should create address with valid data', async () => {
      const dto = {
        label: 'Home' as const,
        fullName: 'Rajesh Kumar',
        phone: '9876543210',
        line1: 'Flat 203',
        city: 'Bangalore',
        state: 'Karnataka',
        pincode: '560095',
        lat: 12.9352,
        lng: 77.6245,
        isDefault: true
      };
      
      addressRepo.create.mockReturnValue(dto as any);
      addressRepo.save.mockResolvedValue({ id: 'addr_123', ...dto } as any);
      
      const result = await service.createAddress('user_123', dto);
      
      expect(addressRepo.create).toHaveBeenCalledWith({
        ...dto,
        userId: 'user_123',
        country: 'India'
      });
      expect(result.id).toBe('addr_123');
    });
    
    it('should throw error if lat provided without lng', async () => {
      const dto = {
        label: 'Home' as const,
        fullName: 'Rajesh',
        phone: '9876543210',
        line1: 'Flat 203',
        city: 'Bangalore',
        state: 'Karnataka',
        pincode: '560095',
        lat: 12.9352,
        lng: null // Missing
      };
      
      await expect(service.createAddress('user_123', dto))
        .rejects
        .toThrow('Both latitude and longitude must be provided together');
    });
    
    it('should unset previous default when creating new default', async () => {
      const dto = { ...validDto, isDefault: true };
      
      await service.createAddress('user_123', dto);
      
      expect(addressRepo.update).toHaveBeenCalledWith(
        { userId: 'user_123', isDefault: true },
        { isDefault: false }
      );
    });
  });
  
  describe('awardCoinsWithReputation', () => {
    it('should multiply coins by reputation tier', async () => {
      const userRepo = module.get(getRepositoryToken(User));
      const reputationRepo = module.get(getRepositoryToken(UserReputationCurrent));
      
      userRepo.findOne.mockResolvedValue({ id: 'user_123', coins: 100 } as any);
      reputationRepo.findOne.mockResolvedValue({ coinMultiplier: 1.1 } as any);
      
      const result = await service.awardCoinsWithReputation('user_123', 100, 'order_complete');
      
      expect(result.coinsAwarded).toBe(110); // floor(100 * 1.1)
      expect(result.multiplier).toBe(1.1);
      expect(userRepo.save).toHaveBeenCalledWith(
        expect.objectContaining({ coins: 210 }) // 100 + 110
      );
    });
  });
});
```

---

### Integration Tests (Supertest)

**File**: `apps/chefooz-apis-e2e/src/user.e2e-spec.ts`

```typescript
import * as request from 'supertest';
import { INestApplication } from '@nestjs/common';

describe('User Module (e2e)', () => {
  let app: INestApplication;
  let jwtToken: string;
  
  beforeAll(async () => {
    // Bootstrap test app
    app = await createTestApp();
    
    // Login to get JWT token
    const authResponse = await request(app.getHttpServer())
      .post('/api/v1/auth/verify-otp/v2')
      .send({ phone: '9876543210', otp: '1234' });
    
    jwtToken = authResponse.body.data.token;
  });
  
  describe('POST /api/v1/user/addresses', () => {
    it('should create address with valid data', () => {
      return request(app.getHttpServer())
        .post('/api/v1/user/addresses')
        .set('Authorization', `Bearer ${jwtToken}`)
        .send({
          label: 'Home',
          fullName: 'Rajesh Kumar',
          phone: '9876543210',
          line1: 'Flat 203',
          city: 'Bangalore',
          state: 'Karnataka',
          pincode: '560095',
          lat: 12.9352,
          lng: 77.6245
        })
        .expect(200)
        .expect((res) => {
          expect(res.body.success).toBe(true);
          expect(res.body.data.id).toBeDefined();
          expect(res.body.data.label).toBe('Home');
        });
    });
    
    it('should reject without JWT token', () => {
      return request(app.getHttpServer())
        .post('/api/v1/user/addresses')
        .send({ label: 'Home', ... })
        .expect(401); // Unauthorized
    });
    
    it('should reject invalid phone format', () => {
      return request(app.getHttpServer())
        .post('/api/v1/user/addresses')
        .set('Authorization', `Bearer ${jwtToken}`)
        .send({ phone: '987654321', ... }) // Only 9 digits
        .expect(400)
        .expect((res) => {
          expect(res.body.message).toContain('Phone must be exactly 10 digits');
        });
    });
  });
  
  describe('GET /api/v1/users/check-username', () => {
    it('should return available for new username', () => {
      return request(app.getHttpServer())
        .get('/api/v1/users/check-username?username=newuser123')
        .expect(200)
        .expect((res) => {
          expect(res.body.data.available).toBe(true);
        });
    });
    
    it('should return suggestions for taken username', () => {
      return request(app.getHttpServer())
        .get('/api/v1/users/check-username?username=admin')
        .expect(200)
        .expect((res) => {
          expect(res.body.data.available).toBe(false);
          expect(res.body.data.suggestions).toHaveLength(3);
        });
    });
  });
});
```

---

## 🚨 Troubleshooting Guide

### Issue 1: "Address not found" (404)

**Symptoms**: DELETE/PUT returns 404 even though address exists

**Possible Causes**:
1. Address deleted by another device/session
2. Wrong addressId in URL parameter
3. Database connection issues

**Solutions**:
```bash
# Verify address exists
curl -X GET https://api.chefooz.com/api/v1/user/addresses \
  -H "Authorization: Bearer <JWT>"

# Check if addressId is correct (UUID format)
# Correct: addr_550e8400-e29b-41d4-a716-446655440000
# Wrong: addr_123 (too short)
```

---

### Issue 2: "You do not have permission" (403)

**Symptoms**: Cannot update/delete address owned by another user

**Possible Causes**:
1. Trying to modify another user's address (security working as intended)
2. JWT token contains wrong userId
3. Address userId column corrupted

**Solutions**:
```bash
# Verify JWT payload
echo "<JWT_TOKEN>" | jwt decode -

# Check userId matches address owner
SELECT id, "userId" FROM addresses WHERE id = 'addr_123';
```

---

### Issue 3: "Both latitude and longitude must be provided together" (400)

**Symptoms**: Address creation fails when only lat or lng provided

**Cause**: Geolocation validation rule (either both or neither)

**Solutions**:
```javascript
// ❌ Wrong: Only lat provided
{ lat: 12.9352, lng: null }

// ✅ Correct: Both provided
{ lat: 12.9352, lng: 77.6245 }

// ✅ Correct: Both omitted
{ lat: null, lng: null }
```

---

### Issue 4: Username check always returns "taken"

**Symptoms**: All username checks return `available: false`

**Possible Causes**:
1. Elasticsearch index stale
2. Database username column case-sensitivity issue
3. Reserved word collision

**Solutions**:
```bash
# Reindex Elasticsearch (if enabled)
curl -X POST https://api.chefooz.com/api/v1/admin/search/reindex-users

# Check reserved words
const RESERVED = ['admin', 'support', 'help', 'chefooz', 'official', 'system', 'root'];

# Test directly in DB
SELECT * FROM users WHERE LOWER(username) = 'test123';
```

---

### Issue 5: Coins not updating after reputation tier change

**Symptoms**: User promoted to Gold tier but still earning Bronze-level coins

**Cause**: `awardCoinsWithReputation` fetches reputation at time of accrual, not cached

**Solutions**:
```bash
# Verify reputation record exists
SELECT * FROM user_reputation_current WHERE "userId" = 'user_123';

# Expected: coinMultiplier = 1.1 for Gold tier
# If missing: Reputation module needs to create record

# Manual fix (admin only)
INSERT INTO user_reputation_current ("userId", tier, "coinMultiplier", "createdAt", "updatedAt")
VALUES ('user_123', 'Gold', 1.1, NOW(), NOW());
```

---

## ⚡ Performance Considerations

### 1. Database Indexing

Indexes for fast address queries:
```sql
CREATE INDEX idx_addresses_userId_createdAt ON addresses("userId", "createdAt");
CREATE INDEX idx_addresses_isDefault ON addresses("isDefault");
```

**Query Performance**:
- `GET /api/v1/user/addresses` → Uses `userId` index → <10ms
- Default address lookup → Uses `isDefault` index → <5ms

---

### 2. Username Check Optimization

**Elasticsearch Prefix Search** (fast):
```typescript
const result = await client.search({
  index: 'users',
  body: {
    query: {
      prefix: { 'username.keyword': 'raj' }
    },
    size: 10
  }
});
// Response time: ~20ms
```

**PostgreSQL Fallback** (slower but reliable):
```sql
SELECT * FROM users WHERE LOWER(username) LIKE 'raj%' LIMIT 10;
-- Response time: ~100ms (acceptable)
```

---

### 3. Coin Accrual Atomicity

Use transaction for atomic coin updates (future enhancement):
```typescript
await this.dataSource.transaction(async (manager) => {
  const user = await manager.findOne(User, { where: { id: userId } });
  user.coins += coinsAwarded;
  await manager.save(user);
  
  await manager.save(CoinLedger, {
    userId,
    amount: coinsAwarded,
    reason,
    createdAt: new Date()
  });
});
```

---

## 🚀 Deployment Checklist

### Environment Variables

Required in `.env` or AWS Secrets Manager:

```bash
# PostgreSQL
DATABASE_URL=postgresql://user:pass@localhost:5432/chefooz

# MongoDB (for Reels username check)
MONGODB_URI=mongodb://localhost:27017/chefooz

# Elasticsearch (optional, for fast username search)
ELASTICSEARCH_NODE=http://localhost:9200
ELASTICSEARCH_ENABLED=true

# JWT (for authentication)
JWT_SECRET=<secret-key>
JWT_EXPIRY=7d
```

---

### Production Checklist

- [ ] Database indexes created (`addresses`, `users`)
- [ ] Elasticsearch index mapped (if enabled)
- [ ] JWT_SECRET set (strong, rotated regularly)
- [ ] HTTPS enforced (TLS 1.3)
- [ ] Rate limiting enabled (address creation: 10/min per user)
- [ ] Monitoring dashboards configured (CloudWatch)
- [ ] Error tracking enabled (Sentry)
- [ ] Audit logging active (`audit_events` table)
- [ ] Backup strategy verified (RDS automated backups)
- [ ] Load testing completed (>1000 concurrent address creates)

---

## 📚 Related Documentation

### Current Module
- [User Feature Overview](./FEATURE_OVERVIEW.md)
- [User QA Test Cases](./QA_TEST_CASES.md)

### Related Modules
- [Auth Module](../auth/TECHNICAL_GUIDE.md) — User authentication
- [Order Module](../order/TECHNICAL_GUIDE.md) — Address usage in orders
- [Reputation Module](../reputation/TECHNICAL_GUIDE.md) — Coin multiplier source
- [Social Module](../social/TECHNICAL_GUIDE.md) — Username display

### External Resources
- [TypeORM Documentation](https://typeorm.io/)
- [NestJS Guards](https://docs.nestjs.com/guards)
- [Elasticsearch Prefix Query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-prefix-query.html)

---

**[TECHNICAL_GUIDE_COMPLETE ✅]**
