# 🔐 Authentication Module — Technical Implementation Guide

**Audience**: Backend Developers / Full-Stack Developers  
**Complexity**: Intermediate  
**Estimated Implementation Time**: Already implemented (reference guide)  
**Prerequisites**: 
- NestJS and TypeORM knowledge
- Understanding of JWT authentication
- Expo/React Native basics for frontend integration
- WhatsApp Cloud API and Twilio SMS setup

---

## 📋 Overview

The Authentication module implements a **WhatsApp-first OTP authentication system** with SMS fallback, providing passwordless phone number-based authentication for the Chefooz platform. This guide provides comprehensive technical documentation for implementing, maintaining, and troubleshooting the authentication system.

**What You'll Learn:**
- 📌 Complete authentication architecture with flow diagrams
- 📌 Database schema and entity relationships
- 📌 API endpoint specifications with request/response examples
- 📌 Security implementation (bcrypt, JWT, rate limiting)
- 📌 Frontend integration patterns for Expo apps
- 📌 Testing strategies and debugging techniques
- 📌 Performance considerations and scaling strategies

**Key Features:**
- ✅ WhatsApp Cloud API primary delivery (faster, higher deliverability)
- ✅ Automatic SMS fallback via Twilio
- ✅ Device-based and phone-based rate limiting
- ✅ bcrypt-hashed OTP storage (never plain text)
- ✅ JWT-based session management (7-day access, 30-day refresh)
- ✅ Auto-user creation on first login
- ✅ Profile completion workflow
- ✅ Development mode bypass for testing

---

## 🏗️ Architecture Overview

### System Architecture

```
┌─────────────────┐      ┌──────────────────┐      ┌─────────────────┐
│   Mobile App    │─────▶│   NestJS API     │─────▶│   PostgreSQL    │
│   (Expo)        │◀─────│   (Auth Module)  │◀─────│   (TypeORM)     │
└─────────────────┘      └──────────────────┘      └─────────────────┘
                                  │
                                  ├─────────────────┐
                                  │                 │
                         ┌────────▼───────┐  ┌─────▼──────┐
                         │  WhatsApp      │  │  Twilio    │
                         │  Cloud API     │  │  SMS API   │
                         └────────────────┘  └────────────┘
```

### Module Structure

```
apps/chefooz-apis/src/modules/auth/
├── auth.controller.ts        → HTTP endpoints
├── auth.service.ts           → Business logic (profile, me)
├── otp.service.ts           → OTP generation & verification
├── auth.module.ts           → Module definition
├── dto/
│   ├── send-otp-v2.dto.ts   → Send OTP request validation
│   ├── verify-otp-v2.dto.ts → Verify OTP request validation
│   └── update-profile.dto.ts → Profile update validation
├── guards/
│   └── jwt.guard.ts         → JWT authentication guard
└── strategies/
    └── jwt.strategy.ts      → Passport JWT strategy

apps/chefooz-apis/src/database/entities/
├── user.entity.ts           → User table
└── otp-session.entity.ts    → OTP sessions table

libs/types/src/lib/
├── auth.types.ts            → Shared auth types
└── otp.types.ts             → Shared OTP types

libs/api-client/src/lib/clients/
├── otp.client.ts            → API client functions
└── otp.hooks.ts             → React Query hooks

apps/chefooz-app/src/app/auth/
├── enter-phone.tsx          → Phone entry screen
├── verify-otp-v2.tsx        → OTP verification screen
└── profile-setup.tsx        → Profile completion screen
```

---

## 🗄️ Database Schema

### Users Table

**Entity**: `apps/chefooz-apis/src/database/entities/user.entity.ts`

```typescript
@Entity('users')
export class User {
  @PrimaryGeneratedColumn('uuid')
  id: string;                      // UUID primary key

  @Column({ unique: true, length: 15 })
  phone: string;                   // E.164 format: +919876543210

  @Column({ unique: true, nullable: true })
  username: string | null;         // Unique username (set after onboarding)

  @Column({ nullable: true })
  fullName: string;                // User's display name

  @Column({ type: 'enum', enum: ['user', 'chef', 'admin', 'rider'], default: 'user' })
  role: 'user' | 'chef' | 'admin' | 'rider';

  @Column({ type: 'boolean', default: true })
  profileIncomplete: boolean;      // Triggers onboarding flow

  @Column({ type: 'int', default: 0 })
  coins: number;                   // Virtual currency balance

  @Column({ type: 'varchar', length: 20, default: 'NORMAL' })
  trustState: 'NORMAL' | 'WARNED' | 'RESTRICTED' | 'FRICTION_REQUIRED';

  @Column({ nullable: true, length: 500 })
  avatarUrl?: string;              // Profile picture URL

  @Column({ type: 'text', nullable: true })
  bio?: string;                    // User bio (max 300 chars)

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

**Indexes:**
- Primary Key: `id` (UUID)
- Unique: `phone`, `username`
- Indexed: `username`, `trustState`

**Constraints:**
- `phone`: Must be unique, E.164 format with country code
- `username`: Must be unique when set (null during onboarding)
- `role`: Enum validation at DB level

---

### OTP Sessions Table

**Entity**: `apps/chefooz-apis/src/database/entities/otp-session.entity.ts`

```typescript
@Entity('otp_sessions')
export class OtpSession {
  @PrimaryGeneratedColumn('uuid')
  id: string;                      // Session ID (returned to client)

  @Column({ length: 15 })
  phone: string;                   // E.164 format

  @Column({ type: 'varchar', length: 255 })
  otpHash: string;                 // bcrypt hash (never store plain OTP)

  @Column({ type: 'timestamp' })
  expiresAt: Date;                 // 5 minutes from creation

  @CreateDateColumn()
  createdAt: Date;

  @Column({ type: 'int', default: 0 })
  attempts: number;                // Failed verification attempts (max 5)

  @Column({ type: 'enum', enum: ['whatsapp', 'sms'], default: 'whatsapp' })
  channel: 'whatsapp' | 'sms';    // Delivery channel used

  @Column({ length: 255 })
  deviceId: string;                // Device identifier for rate limiting

  @Column({ default: false })
  isUsed: boolean;                 // One-time use flag

  @Column({ type: 'varchar', nullable: true })
  whatsappMessageId?: string;      // WhatsApp delivery receipt

  @Column({ type: 'varchar', nullable: true })
  smsMessageId?: string;           // Twilio SID

  @Column({ default: false })
  fallbackUsed: boolean;           // Did we fallback to SMS?
}
```

**Indexes:**
- Primary Key: `id` (UUID)
- Composite: `(phone, deviceId)` for rate limiting queries
- Indexed: `phone`, `deviceId`, `expiresAt`, `isUsed`

**Lifecycle:**
1. Created on OTP send
2. Updated on verification attempts
3. Marked `isUsed = true` on successful verification
4. Expired sessions cleaned up periodically (TODO: cron job)

---

## 🔌 API Endpoints

### Base URL

```
Production:  https://api.chefooz.com/api/v1/auth
Development: http://localhost:3333/api/v1/auth
```

### Authentication Flow

```mermaid
sequenceDiagram
    participant Client as Mobile App
    participant API as NestJS Backend
    participant WhatsApp as WhatsApp API
    participant SMS as Twilio SMS
    participant DB as PostgreSQL

    Client->>API: POST /v1/auth/v2/send-otp
    API->>DB: Check rate limits
    API->>DB: Create OTP session
    API->>WhatsApp: Send OTP template
    alt WhatsApp Success
        WhatsApp-->>API: Message ID
        API-->>Client: { sessionId, channel: "whatsapp" }
    else WhatsApp Fails
        API->>SMS: Send OTP via Twilio
        SMS-->>API: SID
        API-->>Client: { sessionId, channel: "sms", fallbackUsed: true }
    end

    Client->>API: POST /v1/auth/v2/verify-otp
    API->>DB: Find OTP session
    API->>API: Verify bcrypt hash
    API->>DB: Find or create User
    API->>API: Generate JWT tokens
    API->>DB: Mark session as used
    API-->>Client: { accessToken, refreshToken, user }
```

---

### 1. Send OTP

**Endpoint**: `POST /v1/auth/v2/send-otp`

**Rate Limits**:
- 5 requests per minute per IP/phone
- 3 requests per 15 minutes per device
- 3 requests per 15 minutes per phone
- 10 requests per day per phone

**Request DTO**:

```typescript
interface SendOtpRequest {
  phone: string;      // Format: "9876543210" or "+919876543210"
  deviceId: string;   // Unique device identifier
}
```

**Validation**:
- `phone`: Must match regex `^(\+91|91)?[6-9][0-9]{9}$`
- `deviceId`: Required, non-empty string

**Request Examples**:

**cURL:**
```bash
curl -X POST http://localhost:3333/api/v1/auth/v2/send-otp \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "9876543210",
    "deviceId": "iPhone-14-1234567890"
  }'
```

**Axios (Frontend):**
```typescript
import { apiClient } from '@chefooz-app/api-client';

const response = await apiClient.post('/v1/auth/v2/send-otp', {
  phone: '+919876543210',
  deviceId: await getDeviceId(), // From expo-device
});
```

**React Query Hook:**
```typescript
import { useSendOtp } from '@chefooz-app/api-client';

const { mutate: sendOtp, isPending } = useSendOtp({
  onSuccess: (data) => {
    console.log('OTP sent via', data.data.channel);
    // Navigate to verify screen
  },
  onError: (error) => {
    Alert.alert('Error', error.message);
  },
});

// Trigger OTP
sendOtp({ phone: '+919876543210', deviceId: 'device-123' });
```

**Success Response (200)**:

```json
{
  "success": true,
  "message": "OTP sent via WhatsApp",
  "data": {
    "sessionId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "expiresAt": "2026-02-14T10:35:00.000Z",
    "channel": "whatsapp",
    "fallbackUsed": false
  }
}
```

**Error Responses**:

**Rate Limit Exceeded (429)**:
```json
{
  "success": false,
  "message": "Too many OTP requests from this device. Please try again in 15 minutes.",
  "errorCode": "OTP_DEVICE_RATE_LIMIT"
}
```

**Invalid Phone (400)**:
```json
{
  "success": false,
  "message": "Phone number must be valid Indian mobile number",
  "errorCode": "VALIDATION_ERROR"
}
```

**Service Unavailable (400)**:
```json
{
  "success": false,
  "message": "Failed to send OTP via WhatsApp and SMS",
  "errorCode": "OTP_SEND_FAILED"
}
```

---

### 2. Verify OTP

**Endpoint**: `POST /v1/auth/v2/verify-otp`

**Rate Limits**:
- 10 verification attempts per 5 minutes
- Max 5 attempts per OTP session

**Request DTO**:

```typescript
interface VerifyOtpRequest {
  phone: string;      // Must match send-otp phone
  otp: string;        // 6-digit numeric code
  deviceId: string;   // Must match send-otp deviceId
}
```

**Validation**:
- `phone`: Must match regex `^(\+91|91)?[6-9][0-9]{9}$`
- `otp`: Must be exactly 6 digits, regex `^[0-9]{6}$`
- `deviceId`: Required, non-empty string

**Request Examples**:

**cURL:**
```bash
curl -X POST http://localhost:3333/api/v1/auth/v2/verify-otp \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "9876543210",
    "otp": "123456",
    "deviceId": "iPhone-14-1234567890"
  }'
```

**Axios:**
```typescript
const response = await apiClient.post('/v1/auth/v2/verify-otp', {
  phone: '+919876543210',
  otp: '123456',
  deviceId: 'device-123',
});
```

**React Query Hook:**
```typescript
import { useVerifyOtp } from '@chefooz-app/api-client';

const { mutate: verifyOtp, isPending } = useVerifyOtp({
  onSuccess: async (data) => {
    // Store tokens in secure storage
    await SecureStore.setItemAsync('accessToken', data.data.accessToken);
    await SecureStore.setItemAsync('refreshToken', data.data.refreshToken);
    
    // Update auth store
    setUser(data.data.user);
    await setToken(data.data.accessToken);
    
    // Navigate based on profile completion
    if (data.data.user.profileIncomplete) {
      router.replace('/onboarding/username');
    } else {
      router.replace('/(tabs)/home');
    }
  },
  onError: (error) => {
    Alert.alert('Verification Failed', error.message);
  },
});

// Trigger verification
verifyOtp({ phone: '+919876543210', otp: '123456', deviceId: 'device-123' });
```

**Success Response (200)**:

```json
{
  "success": true,
  "message": "Login successful",
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "user": {
      "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
      "phone": "+919876543210",
      "username": null,
      "fullName": "",
      "avatarUrl": null,
      "bio": null,
      "profileIncomplete": true,
      "createdAt": "2026-02-14T10:30:00.000Z"
    }
  }
}
```

**JWT Token Payload**:

```typescript
// Access Token (7 days)
{
  "sub": "f47ac10b-58cc-4372-a567-0e02b2c3d479",  // User ID
  "userId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "phone": "+919876543210",
  "role": "user",
  "scope": "access",
  "type": "access",
  "iss": "chefooz-api",
  "aud": "chefooz-app",
  "iat": 1739440200,
  "exp": 1740044600
}

// Refresh Token (30 days)
{
  "sub": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "userId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "phone": "+919876543210",
  "role": "user",
  "scope": "refresh",
  "type": "refresh",
  "iss": "chefooz-api",
  "aud": "chefooz-app",
  "iat": 1739440200,
  "exp": 1741827400
}
```

**Error Responses**:

**Invalid OTP (401)**:
```json
{
  "success": false,
  "message": "Invalid OTP. 3 attempts remaining.",
  "errorCode": "OTP_INVALID"
}
```

**Expired OTP (400)**:
```json
{
  "success": false,
  "message": "OTP has expired. Please request a new one.",
  "errorCode": "OTP_EXPIRED"
}
```

**Session Not Found (404)**:
```json
{
  "success": false,
  "message": "OTP session not found or already used",
  "errorCode": "OTP_SESSION_NOT_FOUND"
}
```

**Max Attempts Exceeded (400)**:
```json
{
  "success": false,
  "message": "Too many failed attempts. Please request a new OTP.",
  "errorCode": "OTP_MAX_ATTEMPTS"
}
```

---

### 3. Get Current User

**Endpoint**: `GET /v1/auth/me`

**Authentication**: Required (Bearer Token)

**Request Examples**:

**cURL:**
```bash
curl -X GET http://localhost:3333/api/v1/auth/me \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

**Axios:**
```typescript
// Token automatically added by axios interceptor
const response = await apiClient.get('/v1/auth/me');
```

**Success Response (200)**:

```json
{
  "success": true,
  "message": "User retrieved",
  "data": {
    "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "phone": "+919876543210",
    "username": "foodlover123",
    "fullName": "John Doe",
    "role": "user",
    "coins": 100,
    "profileIncomplete": false,
    "avatarUrl": "https://cdn.chefooz.com/avatars/123.jpg",
    "bio": "Food enthusiast",
    "trustState": "NORMAL",
    "createdAt": "2026-02-14T10:30:00.000Z",
    "updatedAt": "2026-02-14T11:00:00.000Z"
  }
}
```

**Error Responses**:

**Unauthorized (401)**:
```json
{
  "statusCode": 401,
  "message": "Unauthorized"
}
```

---

### 4. Update Profile

**Endpoint**: `PUT /v1/auth/profile`

**Authentication**: Required (Bearer Token)

**Purpose**: Complete user profile after first login (username + fullName)

**Request DTO**:

```typescript
interface UpdateProfileRequest {
  username: string;    // 3-20 chars, alphanumeric + underscore
  fullName: string;    // 2-50 chars
}
```

**Request Examples**:

**cURL:**
```bash
curl -X PUT http://localhost:3333/api/v1/auth/profile \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "Content-Type: application/json" \
  -d '{
    "username": "foodlover123",
    "fullName": "John Doe"
  }'
```

**Axios:**
```typescript
const response = await apiClient.put('/v1/auth/profile', {
  username: 'foodlover123',
  fullName: 'John Doe',
});
```

**Success Response (200)**:

```json
{
  "success": true,
  "message": "Profile updated successfully",
  "data": {
    "user": {
      "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
      "phone": "+919876543210",
      "username": "foodlover123",
      "fullName": "John Doe",
      "profileIncomplete": false,
      "createdAt": "2026-02-14T10:30:00.000Z",
      "updatedAt": "2026-02-14T11:00:00.000Z"
    }
  }
}
```

**Error Responses**:

**Username Taken (409)**:
```json
{
  "success": false,
  "message": "Username is already taken",
  "errorCode": "USERNAME_TAKEN"
}
```

---

## 🔒 Security Implementation

### 1. OTP Hashing with bcrypt

**Implementation**: `apps/chefooz-apis/src/modules/auth/otp.service.ts:65-75`

```typescript
/**
 * Hash OTP using bcrypt (never store plain text)
 */
private async hashOtp(otp: string): Promise<string> {
  return bcrypt.hash(otp, 10); // 10 salt rounds
}

/**
 * Verify OTP hash
 */
private async verifyOtpHash(otp: string, hash: string): Promise<boolean> {
  return bcrypt.compare(otp, hash);
}
```

**Why bcrypt?**
- Slow by design (prevents brute force)
- Automatic salting (unique hash each time)
- Industry standard for password/OTP hashing
- Future-proof (adjustable work factor)

**Security Best Practices**:
- ✅ Never log actual OTP in production
- ✅ Store only bcrypt hash in database
- ✅ Use constant-time comparison (bcrypt.compare)
- ✅ Implement attempt limits (max 5 per session)

---

### 2. Rate Limiting

**Implementation**: Device-based and phone-based rate limiting

```typescript
/**
 * Rate Limits (per device):
 * - 3 OTP requests per 15 minutes
 */
private async checkDeviceRateLimit(deviceId: string): Promise<void> {
  const fifteenMinutesAgo = new Date(Date.now() - 15 * 60 * 1000);
  
  const recentAttempts = await this.otpSessionRepository.count({
    where: {
      deviceId,
      createdAt: MoreThan(fifteenMinutesAgo),
    },
  });

  if (recentAttempts >= 3) {
    throw new HttpException(
      {
        success: false,
        message: 'Too many OTP requests from this device. Please try again in 15 minutes.',
        errorCode: 'OTP_DEVICE_RATE_LIMIT',
      },
      HttpStatus.TOO_MANY_REQUESTS,
    );
  }
}

/**
 * Rate Limits (per phone):
 * - 3 OTP requests per 15 minutes
 * - 10 OTP requests per day
 */
private async checkPhoneRateLimit(phone: string): Promise<void> {
  const fifteenMinutesAgo = new Date(Date.now() - 15 * 60 * 1000);
  const oneDayAgo = new Date(Date.now() - 24 * 60 * 60 * 1000);

  const recent15Min = await this.otpSessionRepository.count({
    where: { phone, createdAt: MoreThan(fifteenMinutesAgo) },
  });

  if (recent15Min >= 3) {
    throw new HttpException({
      success: false,
      message: 'Too many OTP requests for this number. Please try again in 15 minutes.',
      errorCode: 'OTP_PHONE_RATE_LIMIT',
    }, HttpStatus.TOO_MANY_REQUESTS);
  }

  const recentDay = await this.otpSessionRepository.count({
    where: { phone, createdAt: MoreThan(oneDayAgo) },
  });

  if (recentDay >= 10) {
    throw new HttpException({
      success: false,
      message: 'Daily OTP limit reached. Please try again tomorrow.',
      errorCode: 'OTP_DAILY_LIMIT',
    }, HttpStatus.TOO_MANY_REQUESTS);
  }
}
```

**Rate Limit Summary**:

| Type | Limit | Window | Purpose |
|------|-------|--------|---------|
| Device | 3 requests | 15 minutes | Prevent device-level spam |
| Phone | 3 requests | 15 minutes | Prevent number-level spam |
| Phone | 10 requests | 24 hours | Daily abuse prevention |
| Verification | 5 attempts | Per session | Brute force protection |
| Global | 5 requests | 60 seconds | IP-based rate limit (controller) |

---

### 3. JWT Security

**Token Generation**: `apps/chefooz-apis/src/modules/auth/otp.service.ts:509-525`

```typescript
// Create JWT payload using domain layer (Phase 3.8.1)
const accessPayload = createJwtPayload(
  user.id,
  user.phone,
  user.role || USER_ROLE.USER,
  'access'
);

const accessToken = this.jwtService.sign(accessPayload, { 
  expiresIn: '7d' 
});

const refreshPayload = createJwtPayload(
  user.id,
  user.phone,
  user.role || USER_ROLE.USER,
  'refresh'
);

const refreshToken = this.jwtService.sign(refreshPayload, { 
  expiresIn: '30d' 
});
```

**JWT Validation Strategy**: `apps/chefooz-apis/src/modules/auth/strategies/jwt.strategy.ts:55-80`

```typescript
async validate(payload: JwtPayload): Promise<User> {
  // Phase 3.8.1: Validate JWT payload structure
  const validationResult = validateJwtPayload(payload as any);
  if (!validationResult.valid) {
    throw new UnauthorizedException(validationResult.reason || 'Invalid token');
  }

  // Ensure this is an access token (not refresh)
  if (payload.type && !isAccessToken(payload as any)) {
    throw new UnauthorizedException('Invalid token type');
  }
  
  // Extract user ID (support both 'sub' and 'userId')
  const userId = payload.sub || payload.userId;
  if (!userId) {
    throw new UnauthorizedException('Invalid token - missing user ID');
  }
  
  // Fetch user from database
  const user = await this.userRepository.findOne({
    where: { id: userId },
  });

  if (!user) {
    throw new UnauthorizedException('User not found');
  }

  // Check trust state (Phase 3.8.1)
  if (shouldForceReauth(user.trustState, false)) {
    throw new UnauthorizedException('Re-authentication required');
  }

  return user;
}
```

**Security Features**:
- ✅ Token type validation (access vs refresh)
- ✅ Issuer and audience claims
- ✅ Trust state validation
- ✅ User existence verification
- ✅ Configurable expiration
- ✅ Secure secret storage (env vars)

**JWT Configuration**: `.env`

```bash
JWT_SECRET=your-super-secure-secret-key-min-32-chars
# In production: Use strong random string (64+ chars)
# Generate: openssl rand -base64 64
```

---

### 4. Phone Number Normalization

**Implementation**: `apps/chefooz-apis/src/modules/auth/otp.service.ts:85-105`

```typescript
/**
 * Normalize phone to E.164 format with +91 prefix
 * Ensures consistent storage for global support
 */
private normalizePhone(phone: string): string {
  const digits = phone.replace(/\D/g, '');
  
  // If already has 91 prefix (12 digits), add +
  if (digits.startsWith('91') && digits.length === 12) {
    return `+${digits}`;
  }
  
  // If 10 digits, add +91
  if (digits.length === 10) {
    return `+91${digits}`;
  }
  
  // If already correct format or other country code
  return phone.startsWith('+') ? phone : `+${digits}`;
}

/**
 * Get last 10 digits (legacy compatibility)
 */
private getLast10Digits(phone: string): string {
  const digits = phone.replace(/\D/g, '');
  return digits.slice(-10);
}
```

**Why E.164 Format?**
- ✅ Global standard for phone numbers
- ✅ WhatsApp/Twilio compatibility
- ✅ Unique identification across countries
- ✅ Future-proof for international expansion

---

## 📱 Frontend Integration

### Expo Mobile App Setup

**File**: `apps/chefooz-app/src/app/auth/enter-phone.tsx`

**Key Features**:
1. Device ID generation (persistent)
2. Phone number validation
3. Rate limit error handling
4. Navigation to verification screen

```typescript
import { useSendOtp } from '@chefooz-app/api-client';
import * as SecureStore from 'expo-secure-store';
import * as Device from 'expo-device';

export default function EnterPhoneScreen() {
  const [phone, setPhone] = useState('');
  
  // Get or create device ID
  const getDeviceId = async (): Promise<string> => {
    let deviceId = await SecureStore.getItemAsync('deviceId');
    
    if (!deviceId) {
      const deviceName = Device.deviceName || 'unknown';
      const modelName = Device.modelName || 'unknown';
      deviceId = `${deviceName}-${modelName}-${Date.now()}`;
      await SecureStore.setItemAsync('deviceId', deviceId);
    }
    
    return deviceId;
  };

  const { mutate: sendOtp, isPending } = useSendOtp({
    onSuccess: async (envelope) => {
      // Navigate to verify screen
      router.push({
        pathname: '/auth/verify-otp-v2',
        params: {
          phone: envelope.data.phone,
          deviceId: await getDeviceId(),
          sessionId: envelope.data.sessionId,
          channel: envelope.data.channel,
        },
      });
    },
    onError: (error: any) => {
      const message = error.response?.data?.message || 'Failed to send OTP';
      Alert.alert('Error', message);
    },
  });

  const handleSendOtp = async () => {
    const deviceId = await getDeviceId();
    sendOtp({ 
      phone: `+91${phone}`, 
      deviceId 
    });
  };

  return (
    <View>
      <TextInput
        value={phone}
        onChangeText={setPhone}
        keyboardType="phone-pad"
        maxLength={10}
      />
      <Button
        title="Send OTP"
        onPress={handleSendOtp}
        disabled={phone.length < 10 || isPending}
      />
    </View>
  );
}
```

---

### OTP Verification Screen

**File**: `apps/chefooz-app/src/app/auth/verify-otp-v2.tsx`

**Key Features**:
1. 6-digit OTP input
2. Auto-focus between inputs
3. Resend timer (60 seconds)
4. Token storage in SecureStore
5. Navigation based on profile completion

```typescript
import { useVerifyOtp, useSendOtp } from '@chefooz-app/api-client';
import { useAuthStore } from '../../store/auth.store';

export default function VerifyOtpV2Screen() {
  const [otp, setOtp] = useState('');
  const [resendTimer, setResendTimer] = useState(60);
  
  const setUser = useAuthStore((state) => state.setUser);
  const setToken = useAuthStore((state) => state.setToken);

  // Resend timer
  useEffect(() => {
    if (resendTimer > 0) {
      const timer = setTimeout(() => setResendTimer(prev => prev - 1), 1000);
      return () => clearTimeout(timer);
    }
  }, [resendTimer]);

  const { mutate: verifyOtp, isPending } = useVerifyOtp({
    onSuccess: async (data) => {
      // Clear old tokens
      await SecureStore.deleteItemAsync('accessToken');
      await SecureStore.deleteItemAsync('refreshToken');
      
      // Store new tokens
      await SecureStore.setItemAsync('accessToken', data.data.accessToken);
      await SecureStore.setItemAsync('refreshToken', data.data.refreshToken);
      
      // Update auth store
      await setToken(data.data.accessToken);
      setUser(data.data.user);
      
      // Navigate based on profile completion
      if (data.data.user.profileIncomplete) {
        router.replace('/onboarding/username');
      } else {
        router.replace('/(tabs)/home');
      }
    },
    onError: (error) => {
      Alert.alert('Verification Failed', error.message);
      setOtp('');
    },
  });

  const handleVerify = () => {
    if (otp.length === 6) {
      verifyOtp({ phone, otp, deviceId });
    }
  };

  return (
    <View>
      {/* 6-digit OTP input boxes */}
      <OtpInput value={otp} onChange={setOtp} />
      
      <Button
        title="Verify"
        onPress={handleVerify}
        disabled={otp.length < 6 || isPending}
      />
      
      {resendTimer > 0 ? (
        <Text>Resend in {resendTimer}s</Text>
      ) : (
        <TouchableOpacity onPress={handleResendOtp}>
          <Text>Resend OTP</Text>
        </TouchableOpacity>
      )}
    </View>
  );
}
```

---

### Axios Configuration

**File**: `libs/api-client/src/lib/axios-config.ts:40-85`

**Token Injection Interceptor**:

```typescript
// Request interceptor - add auth token
apiClient.interceptors.request.use(async (config) => {
  // Get token from secure storage
  const token = await tokenStorage.getToken();
  
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  
  // Normalize /v1/ to /api/v1/
  if (config.url?.startsWith('/v1/')) {
    config.url = `/api${config.url}`;
  }
  
  return config;
});

// Response interceptor - handle 401
apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      // Clear tokens and redirect to login
      await tokenStorage.clearToken();
      // Trigger logout in auth store
    }
    return Promise.reject(error);
  }
);
```

---

## 🧪 Testing Strategies

### Unit Tests

**File**: `apps/chefooz-apis/src/modules/auth/auth.service.spec.ts`

```typescript
describe('AuthService', () => {
  let service: AuthService;
  let userRepository: Repository<User>;
  let jwtService: JwtService;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        AuthService,
        {
          provide: getRepositoryToken(User),
          useValue: {
            findOne: jest.fn(),
            create: jest.fn(),
            save: jest.fn(),
          },
        },
        {
          provide: JwtService,
          useValue: {
            sign: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get<AuthService>(AuthService);
    userRepository = module.get(getRepositoryToken(User));
    jwtService = module.get<JwtService>(JwtService);
  });

  describe('updateProfile', () => {
    it('should update user profile successfully', async () => {
      const mockUser = {
        id: 'user-123',
        phone: '+919876543210',
        username: null,
        fullName: '',
      };

      jest.spyOn(userRepository, 'findOne').mockResolvedValue(mockUser as User);
      jest.spyOn(userRepository, 'save').mockResolvedValue({
        ...mockUser,
        username: 'newuser',
        fullName: 'John Doe',
        profileIncomplete: false,
      } as User);

      const result = await service.updateProfile('user-123', {
        username: 'newuser',
        fullName: 'John Doe',
      });

      expect(result.user.username).toBe('newuser');
      expect(result.user.profileIncomplete).toBe(false);
    });

    it('should throw ConflictException if username is taken', async () => {
      jest.spyOn(userRepository, 'findOne')
        .mockResolvedValueOnce({ id: 'user-123' } as User) // Current user
        .mockResolvedValueOnce({ id: 'user-456' } as User); // Existing username

      await expect(
        service.updateProfile('user-123', {
          username: 'existing',
          fullName: 'John',
        })
      ).rejects.toThrow(ConflictException);
    });
  });
});
```

---

### Integration Tests

**File**: `apps/chefooz-apis-e2e/src/auth/auth.e2e-spec.ts`

```typescript
describe('Auth E2E', () => {
  let app: INestApplication;
  let deviceId: string;
  let phone: string;

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleRef.createNestApplication();
    await app.init();
    
    deviceId = `test-device-${Date.now()}`;
    phone = '+919876543210';
  });

  describe('OTP Flow', () => {
    let sessionId: string;

    it('POST /auth/v2/send-otp should send OTP', async () => {
      const response = await request(app.getHttpServer())
        .post('/api/v1/auth/v2/send-otp')
        .send({ phone, deviceId })
        .expect(200);

      expect(response.body.success).toBe(true);
      expect(response.body.data.sessionId).toBeDefined();
      sessionId = response.body.data.sessionId;
    });

    it('POST /auth/v2/verify-otp should authenticate user', async () => {
      // In dev mode, default OTP is 123456
      const response = await request(app.getHttpServer())
        .post('/api/v1/auth/v2/verify-otp')
        .send({ phone, otp: '123456', deviceId })
        .expect(200);

      expect(response.body.success).toBe(true);
      expect(response.body.data.accessToken).toBeDefined();
      expect(response.body.data.user.phone).toBe(phone);
    });
  });

  describe('Protected Routes', () => {
    let accessToken: string;

    beforeAll(async () => {
      // Get token
      const response = await request(app.getHttpServer())
        .post('/api/v1/auth/v2/verify-otp')
        .send({ phone, otp: '123456', deviceId });
      
      accessToken = response.body.data.accessToken;
    });

    it('GET /auth/me should return user profile', async () => {
      const response = await request(app.getHttpServer())
        .get('/api/v1/auth/me')
        .set('Authorization', `Bearer ${accessToken}`)
        .expect(200);

      expect(response.body.data.id).toBeDefined();
      expect(response.body.data.phone).toBe(phone);
    });

    it('PUT /auth/profile should update profile', async () => {
      const response = await request(app.getHttpServer())
        .put('/api/v1/auth/profile')
        .set('Authorization', `Bearer ${accessToken}`)
        .send({ username: 'testuser', fullName: 'Test User' })
        .expect(200);

      expect(response.body.data.user.username).toBe('testuser');
      expect(response.body.data.user.profileIncomplete).toBe(false);
    });
  });

  afterAll(async () => {
    await app.close();
  });
});
```

---

### Manual Testing (Development)

**Test OTP Flow (Dev Mode)**:

```bash
# 1. Start backend
npx nx run chefooz-apis:serve

# 2. Send OTP (use default OTP: 123456 in dev)
curl -X POST http://localhost:3333/api/v1/auth/v2/send-otp \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "9876543210",
    "deviceId": "test-device-123"
  }'

# Response:
# {
#   "success": true,
#   "message": "OTP sent (dev mode - use 123456)",
#   "data": {
#     "sessionId": "uuid-here",
#     "expiresAt": "2026-02-14T10:35:00Z",
#     "channel": "sms",
#     "fallbackUsed": false
#   }
# }

# 3. Verify OTP
curl -X POST http://localhost:3333/api/v1/auth/v2/verify-otp \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "9876543210",
    "otp": "123456",
    "deviceId": "test-device-123"
  }'

# Response: JWT tokens + user data

# 4. Test protected endpoint
TOKEN="<access-token-from-step-3>"
curl -X GET http://localhost:3333/api/v1/auth/me \
  -H "Authorization: Bearer $TOKEN"
```

---

## 🚨 Troubleshooting Guide

### Common Issues

#### 1. "OTP session not found"

**Symptoms**: 404 error when verifying OTP

**Causes**:
- Device ID mismatch between send and verify
- Phone number format mismatch
- OTP expired (>5 minutes)
- Session already used

**Solutions**:
```typescript
// Ensure device ID is consistent
const deviceId = await SecureStore.getItemAsync('deviceId');
console.log('Device ID:', deviceId);

// Ensure phone format is consistent
const normalizedPhone = phone.replace(/\D/g, '');
console.log('Normalized phone:', normalizedPhone);

// Check backend logs for OTP session creation
// Look for: "OTP sent to +919876543210 via whatsapp"
```

---

#### 2. "Too many OTP requests"

**Symptoms**: 429 error when sending OTP

**Causes**:
- Rate limit exceeded (3 per 15 min or 10 per day)
- Multiple devices using same phone
- Testing without clearing sessions

**Solutions**:
```sql
-- Check OTP sessions (PostgreSQL)
SELECT phone, device_id, created_at, is_used
FROM otp_sessions
WHERE phone = '+919876543210'
ORDER BY created_at DESC
LIMIT 10;

-- Clear test sessions (dev only)
DELETE FROM otp_sessions
WHERE phone = '+919876543210';
```

---

#### 3. "Invalid token" or "Unauthorized"

**Symptoms**: 401 error on protected endpoints

**Causes**:
- Token expired (>7 days)
- Token not in Authorization header
- JWT secret mismatch
- User deleted after token issued

**Solutions**:
```typescript
// Check token in SecureStore
const token = await SecureStore.getItemAsync('accessToken');
console.log('Token:', token ? `${token.substring(0, 30)}...` : 'NOT FOUND');

// Decode JWT to inspect payload (jwt.io)
// Check expiration: exp timestamp

// Force re-login
await SecureStore.deleteItemAsync('accessToken');
await SecureStore.deleteItemAsync('refreshToken');
router.replace('/auth/enter-phone');
```

**Backend Debugging**:
```typescript
// In jwt.guard.ts, check logs:
// 🛡️ Authorization header: Bearer eyJ...
// 🛡️ Token (first 30 chars): eyJhbGciOiJIUzI1NiIsInR5cCI...
// 🛡️ Decoded User from JWT: { sub: 'uuid', phone: '+91...' }
```

---

#### 4. WhatsApp/SMS not sending

**Symptoms**: OTP not received, but no error in app

**Causes**:
- WhatsApp template not approved
- Twilio credentials incorrect
- Phone number not registered with WhatsApp
- Network timeout

**Solutions**:
```bash
# Check environment variables
echo $WHATSAPP_TOKEN
echo $WHATSAPP_PHONE_NUMBER_ID
echo $TWILIO_ACCOUNT_SID
echo $TWILIO_AUTH_TOKEN
echo $TWILIO_PHONE_NUMBER

# Check backend logs for delivery errors
# Look for:
# - "WhatsApp OTP sent to 919876543210"
# - "Falling back to SMS..."
# - "SMS OTP sent to +919876543210"

# Test WhatsApp API directly
curl -X POST https://graph.facebook.com/v17.0/$PHONE_NUMBER_ID/messages \
  -H "Authorization: Bearer $WHATSAPP_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "messaging_product": "whatsapp",
    "to": "919876543210",
    "type": "template",
    "template": {
      "name": "chefooz_login_otp",
      "language": { "code": "en" }
    }
  }'
```

---

#### 5. TypeScript errors in types

**Symptoms**: Type errors for `SendOtpResponse`, `VerifyOtpResponse`

**Causes**:
- NX cache not updated
- TypeScript server not restarted

**Solutions**:
```bash
# Clear NX cache
npx nx reset

# Restart TypeScript server in VS Code
# Cmd+Shift+P → "TypeScript: Restart TS Server"

# Rebuild types lib
npx nx build types
```

---

## ⚡ Performance Considerations

### Database Optimization

**Indexes**:
```sql
-- OTP Sessions indexes (already applied via TypeORM decorators)
CREATE INDEX idx_otp_phone ON otp_sessions(phone);
CREATE INDEX idx_otp_device_id ON otp_sessions(device_id);
CREATE INDEX idx_otp_expires_at ON otp_sessions(expires_at);
CREATE INDEX idx_otp_is_used ON otp_sessions(is_used);
CREATE INDEX idx_otp_phone_device ON otp_sessions(phone, device_id);

-- Users indexes
CREATE INDEX idx_user_phone ON users(phone);
CREATE INDEX idx_user_username ON users(username);
CREATE INDEX idx_user_trust_state ON users(trust_state);
```

**Query Optimization**:
```typescript
// Use composite index for rate limit queries
const recentAttempts = await this.otpSessionRepository.count({
  where: {
    phone,
    deviceId,
    createdAt: MoreThan(fifteenMinutesAgo),
  },
});
// Uses idx_otp_phone_device + idx_otp_expires_at
```

---

### Caching Strategy

**Redis/Valkey for Rate Limiting** (Future Enhancement):

```typescript
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';

@Injectable()
export class OtpService {
  constructor(
    @Inject(CACHE_MANAGER) private cacheManager: Cache,
  ) {}

  async checkRateLimitCached(key: string, limit: number, ttl: number): Promise<boolean> {
    const current = await this.cacheManager.get<number>(key) || 0;
    
    if (current >= limit) {
      return false; // Rate limit exceeded
    }
    
    await this.cacheManager.set(key, current + 1, ttl);
    return true; // Allow request
  }
}
```

**Benefits**:
- Reduces DB load for rate limit checks
- Faster response times
- Distributed rate limiting across multiple instances

---

### Connection Pooling

**TypeORM Configuration**:

```typescript
// apps/chefooz-apis/src/database/database.module.ts
TypeOrmModule.forRoot({
  type: 'postgres',
  host: process.env.DB_HOST,
  port: parseInt(process.env.DB_PORT),
  database: process.env.DB_NAME,
  username: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  
  // Connection pooling
  extra: {
    max: 20,              // Max connections
    min: 5,               // Min connections
    idleTimeoutMillis: 30000,
    connectionTimeoutMillis: 2000,
  },
});
```

---

## 🚀 Deployment Checklist

### Environment Variables

**Required (.env)**:

```bash
# Database
DB_HOST=localhost
DB_PORT=5432
DB_NAME=chefooz
DB_USER=postgres
DB_PASSWORD=your-secure-password

# JWT
JWT_SECRET=your-super-secure-secret-minimum-32-characters-long
# Generate: openssl rand -base64 64

# WhatsApp Cloud API
WHATSAPP_TOKEN=your-whatsapp-business-token
WHATSAPP_PHONE_NUMBER_ID=your-phone-number-id

# Twilio SMS
TWILIO_ACCOUNT_SID=your-twilio-sid
TWILIO_AUTH_TOKEN=your-twilio-token
TWILIO_PHONE_NUMBER=+1234567890

# Optional
OTP_ENABLED=true                  # Set to "false" for dev mode (always use 123456)
NODE_ENV=production               # Set to "development" for debug logs
```

---

### Production Checklist

- [ ] Generate strong JWT secret (64+ chars)
- [ ] Configure WhatsApp Business API
- [ ] Approve WhatsApp message template
- [ ] Configure Twilio SMS sender ID
- [ ] Set up SSL/TLS for API
- [ ] Enable CORS for allowed origins
- [ ] Set up monitoring (Sentry, DataDog)
- [ ] Configure log aggregation
- [ ] Set up database backups
- [ ] Enable rate limiting at API gateway level
- [ ] Set up CDN for static assets
- [ ] Configure firewall rules
- [ ] Set up health check endpoints
- [ ] Enable request/response logging
- [ ] Set up alerting for OTP failures
- [ ] Test SMS/WhatsApp delivery in production
- [ ] Set up database connection pooling
- [ ] Configure auto-scaling rules
- [ ] Test rate limits under load
- [ ] Enable database query logging (temporarily)
- [ ] Set up Redis/Valkey for caching (optional)

---

### Security Hardening

**Production Security**:

```typescript
// apps/chefooz-apis/src/main.ts

import helmet from 'helmet';
import rateLimit from 'express-rate-limit';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // Security headers
  app.use(helmet());
  
  // Global rate limiting
  app.use(
    rateLimit({
      windowMs: 15 * 60 * 1000, // 15 minutes
      max: 100, // Limit each IP to 100 requests per window
      message: 'Too many requests from this IP',
    })
  );
  
  // CORS
  app.enableCors({
    origin: ['https://app.chefooz.com', 'https://admin.chefooz.com'],
    credentials: true,
  });
  
  // Disable X-Powered-By header
  app.use((req, res, next) => {
    res.removeHeader('X-Powered-By');
    next();
  });
  
  await app.listen(3333);
}
```

---

## 📚 Related Documentation

- **FEATURE_OVERVIEW.md**: Business logic and user flows (850+ lines)
- **OTP_AUTH_SETUP.md**: Quick setup guide for local development
- **User Module**: Profile management and settings
- **Chef Module**: Chef-specific authentication flows
- **Cart Module**: Authenticated cart operations
- **Order Module**: Order placement with auth tokens

---

## 🔗 External Resources

- **WhatsApp Cloud API**: https://developers.facebook.com/docs/whatsapp/cloud-api
- **Twilio SMS API**: https://www.twilio.com/docs/sms
- **JWT Best Practices**: https://datatracker.ietf.org/doc/html/rfc8725
- **bcrypt Documentation**: https://github.com/kelektiv/node.bcrypt.js
- **NestJS Authentication**: https://docs.nestjs.com/security/authentication
- **TypeORM Relations**: https://typeorm.io/relations
- **Expo SecureStore**: https://docs.expo.dev/versions/latest/sdk/securestore/

---

## 📞 Support

For technical issues:
1. Check troubleshooting guide above
2. Review backend logs: `npx nx run chefooz-apis:serve --verbose`
3. Test API directly with cURL
4. Check database sessions: `SELECT * FROM otp_sessions ORDER BY created_at DESC LIMIT 10;`
5. Open GitHub issue with logs and steps to reproduce

---

## ✅ TECHNICAL_GUIDE_COMPLETE ✅

**Document Version**: 1.0.0  
**Last Updated**: February 14, 2026  
**Lines**: 900+  
**Status**: Production Ready  

---

**Implementation Files**:
- Backend: `apps/chefooz-apis/src/modules/auth/` (7 files)
- Frontend: `apps/chefooz-app/src/app/auth/` (3 screens)
- Shared Types: `libs/types/src/lib/auth.types.ts`, `libs/types/src/lib/otp.types.ts`
- API Client: `libs/api-client/src/lib/clients/otp.{client,hooks}.ts`
- Database: `apps/chefooz-apis/src/database/entities/{user,otp-session}.entity.ts`
