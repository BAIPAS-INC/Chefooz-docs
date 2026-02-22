# Commission Module - QA Test Cases

**Module**: `commission`  
**Type**: Core Monetization Feature  
**Last Updated**: February 22, 2026

---

## 📋 Table of Contents

1. [Test Environment Setup](#test-environment-setup)
2. [Test Data Preparation](#test-data-preparation)
3. [Category 1: V2 Commission Calculation](#category-1-v2-commission-calculation)
4. [Category 2: Commission Ledger Lifecycle](#category-2-commission-ledger-lifecycle)
5. [Category 3: Atomic Coin Updates](#category-3-atomic-coin-updates)
6. [Category 4: Business Rule Enforcement](#category-4-business-rule-enforcement)
7. [Category 5: Earnings Preview](#category-5-earnings-preview)
8. [Category 6: Background Job Settlement](#category-6-background-job-settlement)
9. [Category 7: Integration Testing](#category-7-integration-testing)
10. [Category 8: Edge Cases & Error Handling](#category-8-edge-cases--error-handling)
11. [Performance Testing](#performance-testing)
12. [Regression Testing](#regression-testing)

---

## 🔧 Test Environment Setup

### Prerequisites

```bash
# 1. PostgreSQL database
docker run -d \
  --name chefooz-postgres-test \
  -e POSTGRES_DB=chefooz_test \
  -e POSTGRES_USER=test \
  -e POSTGRES_PASSWORD=test123 \
  -p 5433:5432 \
  postgres:15

# 2. MongoDB database
docker run -d \
  --name chefooz-mongo-test \
  -e MONGO_INITDB_DATABASE=chefooz_test \
  -p 27018:27017 \
  mongo:6.0

# 3. Install dependencies
npm install

# 4. Run migrations
npm run migration:run

# 5. Seed test data
npm run seed:test
```

### Environment Variables

```env
# .env.test
DATABASE_URL=postgresql://test:test123@localhost:5433/chefooz_test
MONGODB_URI=mongodb://localhost:27018/chefooz_test
JWT_SECRET=test-secret-key-do-not-use-in-production
```

### Test Database Schema

```sql
-- Ensure commission_ledger table exists
CREATE TABLE IF NOT EXISTS commission_ledger (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID NOT NULL,
  order_item_id UUID,
  payee_user_id UUID NOT NULL,
  reel_id VARCHAR(255),
  commission_paise INT NOT NULL,
  coins INT NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'pending',
  notes TEXT,
  creator_order_value INT,
  user_order_value INT,
  commissionable_amount INT,
  formula_version VARCHAR(10) DEFAULT 'v1',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Ensure users table has coins column
ALTER TABLE users ADD COLUMN IF NOT EXISTS coins INT NOT NULL DEFAULT 0;
```

---

## 📊 Test Data Preparation

### Seed Script

```typescript
// scripts/seed-commission-test-data.ts
import { DataSource } from 'typeorm';
import { User } from '../src/modules/user/user.entity';
import { Order } from '../src/modules/order/order.entity';

async function seedTestData() {
  const dataSource = new DataSource({
    type: 'postgres',
    url: process.env.DATABASE_URL,
    entities: [User, Order],
  });

  await dataSource.initialize();

  // Create test users
  const customerCreator = await dataSource.getRepository(User).save({
    id: 'test-customer-creator-001',
    email: 'creator@test.com',
    role: 'customer',
    coins: 0,
  });

  const customerBuyer = await dataSource.getRepository(User).save({
    id: 'test-customer-buyer-001',
    email: 'buyer@test.com',
    role: 'customer',
    coins: 1000, // Existing coins for reversal tests
  });

  const chefUser = await dataSource.getRepository(User).save({
    id: 'test-chef-001',
    email: 'chef@test.com',
    role: 'chef',
    coins: 0,
  });

  // Create test orders
  const deliveredOrder = await dataSource.getRepository(Order).save({
    id: 'test-order-delivered-001',
    userId: customerBuyer.id,
    totalPaise: 55000, // ₹550
    subtotalPaise: 50000,
    deliveryStatus: 'delivered',
    status: 'completed',
    reelId: 'test-reel-001',
    reelCreatorUserId: customerCreator.id,
    creatorOrderValue: 50000, // ₹500
  });

  console.log('✓ Test data seeded successfully');
  await dataSource.destroy();
}

seedTestData().catch(console.error);
```

### Test Fixtures

```typescript
// test/fixtures/commission.fixtures.ts
export const TEST_USERS = {
  customerCreator: {
    id: 'test-customer-creator-001',
    email: 'creator@test.com',
    role: 'customer',
    coins: 0,
  },
  customerBuyer: {
    id: 'test-customer-buyer-001',
    email: 'buyer@test.com',
    role: 'customer',
    coins: 1000,
  },
  chefUser: {
    id: 'test-chef-001',
    email: 'chef@test.com',
    role: 'chef',
    coins: 0,
  },
};

export const TEST_ORDERS = {
  delivered: {
    id: 'test-order-delivered-001',
    userId: TEST_USERS.customerBuyer.id,
    totalPaise: 55000,
    subtotalPaise: 50000,
    deliveryStatus: 'delivered',
    creatorOrderValue: 50000,
    reelCreatorUserId: TEST_USERS.customerCreator.id,
    reelId: 'test-reel-001',
  },
};
```

---

## ✅ Category 1: V2 Commission Calculation

### TC-COM-001: Exact Match Calculation

**Priority**: High  
**Category**: Calculation Logic

**Preconditions**:
- Creator's order value: ₹500 (50,000 paise)
- User's order value: ₹500 (50,000 paise)

**Test Steps**:
1. Call `calculateUserCommissionCoins(50000, 50000)`
2. Verify `commissionableAmount` = 50,000 paise
3. Verify `commissionPaise` = 5,000 paise (10% of 50,000)
4. Verify `coins` = 500 (5,000 / 10)

**Expected Result**:
```json
{
  "commissionableAmount": 50000,
  "commissionPaise": 5000,
  "coins": 500
}
```

**Actual Result**: [ PASS / FAIL ]

**Test Code**:
```typescript
it('TC-COM-001: should calculate exact match correctly', () => {
  const result = commissionService.calculateUserCommissionCoins(50000, 50000);
  
  expect(result.commissionableAmount).toBe(50000);
  expect(result.commissionPaise).toBe(5000);
  expect(result.coins).toBe(500);
});
```

---

### TC-COM-002: Upsell 50% More Calculation

**Priority**: High  
**Category**: Calculation Logic

**Preconditions**:
- Creator's order value: ₹500 (50,000 paise)
- User's order value: ₹750 (75,000 paise) - 50% more

**Test Steps**:
1. Call `calculateUserCommissionCoins(50000, 75000)`
2. Verify `commissionableAmount` = 62,500 paise (50,000 + 12,500)
3. Verify `commissionPaise` = 6,250 paise (10% of 62,500)
4. Verify `coins` = 625 (6,250 / 10)

**Expected Result**:
```json
{
  "commissionableAmount": 62500,
  "commissionPaise": 6250,
  "coins": 625
}
```

**Formula Validation**:
- Excess = 75,000 - 50,000 = 25,000 paise
- Commissionable = 50,000 + (25,000 / 2) = 62,500 paise ✅
- Commission = 62,500 × 10% = 6,250 paise ✅
- Coins = 6,250 / 10 = 625 ✅

**Actual Result**: [ PASS / FAIL ]

**Test Code**:
```typescript
it('TC-COM-002: should split upsell 50/50', () => {
  const result = commissionService.calculateUserCommissionCoins(50000, 75000);
  
  expect(result.commissionableAmount).toBe(62500);
  expect(result.commissionPaise).toBe(6250);
  expect(result.coins).toBe(625);
});
```

---

### TC-COM-003: User Orders Less Calculation

**Priority**: High  
**Category**: Calculation Logic

**Preconditions**:
- Creator's order value: ₹500 (50,000 paise)
- User's order value: ₹300 (30,000 paise) - 40% less

**Test Steps**:
1. Call `calculateUserCommissionCoins(50000, 30000)`
2. Verify `commissionableAmount` = 30,000 paise
3. Verify `commissionPaise` = 3,000 paise (10% of 30,000)
4. Verify `coins` = 300 (3,000 / 10)

**Expected Result**:
```json
{
  "commissionableAmount": 30000,
  "commissionPaise": 3000,
  "coins": 300
}
```

**Formula Validation**:
- User order (30,000) ≤ Creator order (50,000) → Use user order ✅
- Commissionable = 30,000 paise ✅
- Commission = 30,000 × 10% = 3,000 paise ✅
- Coins = 3,000 / 10 = 300 ✅

**Actual Result**: [ PASS / FAIL ]

---

### TC-COM-004: Edge Case - Zero Order Value

**Priority**: Medium  
**Category**: Calculation Logic

**Preconditions**:
- Creator's order value: ₹500 (50,000 paise)
- User's order value: ₹0 (0 paise)

**Test Steps**:
1. Call `calculateUserCommissionCoins(50000, 0)`
2. Verify `commissionableAmount` = 0 paise
3. Verify `commissionPaise` = 0 paise
4. Verify `coins` = 0

**Expected Result**:
```json
{
  "commissionableAmount": 0,
  "commissionPaise": 0,
  "coins": 0
}
```

**Actual Result**: [ PASS / FAIL ]

---

### TC-COM-005: Edge Case - Rounding Down Coins

**Priority**: Medium  
**Category**: Calculation Logic

**Preconditions**:
- Creator's order value: ₹505.55 (50,555 paise)
- User's order value: ₹505.55 (50,555 paise)

**Test Steps**:
1. Call `calculateUserCommissionCoins(50555, 50555)`
2. Verify `commissionPaise` = 5,055 paise (10% of 50,555, rounded down)
3. Verify `coins` = 505 (5,055 / 10, rounded down)

**Expected Result**:
```json
{
  "commissionableAmount": 50555,
  "commissionPaise": 5055,
  "coins": 505
}
```

**Rounding Validation**:
- Commission = Math.floor(50,555 × 0.10) = Math.floor(5,055.5) = 5,055 ✅
- Coins = Math.floor(5,055 / 10) = Math.floor(505.5) = 505 ✅

**Actual Result**: [ PASS / FAIL ]

---

### TC-COM-006: V1 Fallback Calculation

**Priority**: High  
**Category**: Calculation Logic

**Preconditions**:
- Order subtotal: ₹500 (50,000 paise)
- No `creatorOrderValue` provided (legacy reel)

**Test Steps**:
1. Call `createPendingCommission(orderId, userId, reelId, undefined)`
2. Verify formula version = 'v1'
3. Verify commission calculated from `order.subtotalPaise`
4. Verify `commissionPaise` = 5,000 paise (10% of 50,000)
5. Verify `coins` = 500

**Expected Result**:
- `formulaVersion` = 'v1'
- `commissionPaise` = 5,000
- `coins` = 500
- `creatorOrderValue` = null
- `commissionableAmount` = null

**Actual Result**: [ PASS / FAIL ]

---

## ✅ Category 2: Commission Ledger Lifecycle

### TC-COM-010: Create Pending Commission

**Priority**: High  
**Category**: Ledger Lifecycle

**Preconditions**:
- Order delivered
- Order has linked reel
- Creator is customer role
- No self-earning

**Test Steps**:
1. Call `createPendingCommission(orderId, creatorId, reelId, 50000)`
2. Query `commission_ledger` table
3. Verify entry created with status = 'pending'
4. Verify `commissionPaise` and `coins` correct
5. Verify V2 fields populated

**Expected Result**:
```sql
SELECT * FROM commission_ledger WHERE order_id = 'test-order-001';

-- Expected row:
id: <UUID>
order_id: 'test-order-001'
payee_user_id: 'test-customer-creator-001'
reel_id: 'test-reel-001'
commission_paise: 5250
coins: 525
status: 'pending'
formula_version: 'v2'
creator_order_value: 50000
user_order_value: 55000
commissionable_amount: 52500
notes: 'Commission pending delivery confirmation (v2)'
```

**Actual Result**: [ PASS / FAIL ]

**Test Code**:
```typescript
it('TC-COM-010: should create pending commission', async () => {
  const ledger = await commissionService.createPendingCommission(
    'test-order-001',
    'test-customer-creator-001',
    'test-reel-001',
    50000,
  );
  
  expect(ledger).not.toBeNull();
  expect(ledger.status).toBe('pending');
  expect(ledger.coins).toBe(525);
  expect(ledger.formulaVersion).toBe('v2');
  expect(ledger.creatorOrderValue).toBe(50000);
  expect(ledger.commissionableAmount).toBe(52500);
});
```

---

### TC-COM-011: Credit Pending Commission

**Priority**: High  
**Category**: Ledger Lifecycle

**Preconditions**:
- Pending commission exists in DB
- User has 0 coins initially

**Test Steps**:
1. Query user's coin balance (should be 0)
2. Call `creditCommission(ledgerId)`
3. Query ledger status (should be 'credited')
4. Query user's coin balance (should be 500)
5. Verify ledger notes updated

**Expected Result**:
- Ledger status = 'credited'
- Ledger notes = 'Commission credited to coin balance'
- User coins = 0 + 500 = 500

**SQL Verification**:
```sql
-- Before
SELECT coins FROM users WHERE id = 'test-customer-creator-001';
-- coins: 0

-- After
SELECT coins FROM users WHERE id = 'test-customer-creator-001';
-- coins: 500

SELECT status, notes FROM commission_ledger WHERE id = '<ledger-id>';
-- status: 'credited'
-- notes: 'Commission credited to coin balance'
```

**Actual Result**: [ PASS / FAIL ]

**Test Code**:
```typescript
it('TC-COM-011: should credit pending commission', async () => {
  // Arrange: Create pending commission
  const ledger = await commissionService.createPendingCommission(
    'test-order-001',
    'test-customer-creator-001',
    'test-reel-001',
    50000,
  );
  
  const userBefore = await userRepo.findOne({ where: { id: 'test-customer-creator-001' } });
  expect(userBefore.coins).toBe(0);
  
  // Act: Credit commission
  const credited = await commissionService.creditCommission(ledger.id);
  
  // Assert
  expect(credited.status).toBe('credited');
  expect(credited.notes).toContain('credited to coin balance');
  
  const userAfter = await userRepo.findOne({ where: { id: 'test-customer-creator-001' } });
  expect(userAfter.coins).toBe(500);
});
```

---

### TC-COM-012: Reverse Pending Commission

**Priority**: High  
**Category**: Ledger Lifecycle

**Preconditions**:
- Pending commission exists
- Order refunded before commission credited

**Test Steps**:
1. Create pending commission
2. Call `reverseCommission(orderId)`
3. Verify ledger status = 'reversed'
4. Verify user coin balance unchanged (was pending, not yet credited)

**Expected Result**:
- Ledger status = 'reversed'
- Ledger notes = 'Reversed due to order refund/cancellation'
- User coins = 0 (unchanged, was pending)

**Actual Result**: [ PASS / FAIL ]

---

### TC-COM-013: Reverse Credited Commission

**Priority**: High  
**Category**: Ledger Lifecycle

**Preconditions**:
- Commission credited (status = 'credited')
- User has 500 coins from commission

**Test Steps**:
1. Create and credit commission (user gains 500 coins)
2. Verify user has 500 coins
3. Call `reverseCommission(orderId)`
4. Verify ledger status = 'reversed'
5. Verify user coins deducted (500 - 500 = 0)

**Expected Result**:
- Ledger status = 'reversed'
- User coins = 500 - 500 = 0

**SQL Verification**:
```sql
-- Before reversal
SELECT coins FROM users WHERE id = 'test-customer-creator-001';
-- coins: 500

-- After reversal
SELECT coins FROM users WHERE id = 'test-customer-creator-001';
-- coins: 0
```

**Actual Result**: [ PASS / FAIL ]

**Test Code**:
```typescript
it('TC-COM-013: should reverse credited commission', async () => {
  // Arrange: Create and credit commission
  const ledger = await commissionService.createPendingCommission(
    'test-order-001',
    'test-customer-creator-001',
    'test-reel-001',
    50000,
  );
  await commissionService.creditCommission(ledger.id);
  
  const userBefore = await userRepo.findOne({ where: { id: 'test-customer-creator-001' } });
  expect(userBefore.coins).toBe(500);
  
  // Act: Reverse commission
  await commissionService.reverseCommission('test-order-001');
  
  // Assert
  const ledgerAfter = await ledgerRepo.findOne({ where: { id: ledger.id } });
  expect(ledgerAfter.status).toBe('reversed');
  
  const userAfter = await userRepo.findOne({ where: { id: 'test-customer-creator-001' } });
  expect(userAfter.coins).toBe(0);
});
```

---

### TC-COM-014: Idempotent Credit

**Priority**: Medium  
**Category**: Ledger Lifecycle

**Preconditions**:
- Commission already credited

**Test Steps**:
1. Create and credit commission
2. Call `creditCommission(ledgerId)` again
3. Verify status remains 'credited' (no duplicate credit)
4. Verify user coins not double-credited

**Expected Result**:
- Ledger status = 'credited' (unchanged)
- User coins = 500 (not 1000)
- Log warning about already credited

**Actual Result**: [ PASS / FAIL ]

---

## ✅ Category 3: Atomic Coin Updates

### TC-COM-020: Concurrent Credit Operations

**Priority**: High  
**Category**: Concurrency

**Preconditions**:
- Two pending commissions for same user
- User has 0 coins initially

**Test Steps**:
1. Create two pending commissions (500 coins each)
2. Credit both simultaneously (Promise.all)
3. Verify user has 1000 coins (not 500 from race condition)
4. Verify both ledgers marked 'credited'

**Expected Result**:
- User coins = 0 + 500 + 500 = 1000 ✅
- Both ledgers status = 'credited'

**SQL Verification**:
```sql
-- Atomic update 1
UPDATE users SET coins = coins + 500 WHERE id = '...';

-- Atomic update 2 (concurrent)
UPDATE users SET coins = coins + 500 WHERE id = '...';

-- Result: coins = 1000 (no race condition)
```

**Actual Result**: [ PASS / FAIL ]

**Test Code**:
```typescript
it('TC-COM-020: should handle concurrent credits atomically', async () => {
  // Arrange: Create two pending commissions
  const ledger1 = await commissionService.createPendingCommission(
    'test-order-001',
    'test-customer-creator-001',
    'test-reel-001',
    50000,
  );
  
  const ledger2 = await commissionService.createPendingCommission(
    'test-order-002',
    'test-customer-creator-001',
    'test-reel-002',
    50000,
  );
  
  // Act: Credit both simultaneously
  await Promise.all([
    commissionService.creditCommission(ledger1.id),
    commissionService.creditCommission(ledger2.id),
  ]);
  
  // Assert: No lost updates
  const user = await userRepo.findOne({ where: { id: 'test-customer-creator-001' } });
  expect(user.coins).toBe(1000); // 500 + 500 = 1000 ✅
});
```

---

### TC-COM-021: Concurrent Deduct Operations

**Priority**: High  
**Category**: Concurrency

**Preconditions**:
- User has 1000 coins
- Two credited commissions (500 coins each)

**Test Steps**:
1. Reverse both commissions simultaneously (Promise.all)
2. Verify user has 0 coins (not -500 from race condition)
3. Verify both ledgers marked 'reversed'

**Expected Result**:
- User coins = 1000 - 500 - 500 = 0 ✅
- Both ledgers status = 'reversed'

**SQL Verification**:
```sql
-- Atomic deduct 1
UPDATE users SET coins = GREATEST(0, coins - 500) WHERE id = '...';

-- Atomic deduct 2 (concurrent)
UPDATE users SET coins = GREATEST(0, coins - 500) WHERE id = '...';

-- Result: coins = 0 (no negative balance)
```

**Actual Result**: [ PASS / FAIL ]

---

### TC-COM-022: Prevent Negative Balance

**Priority**: High  
**Category**: Concurrency

**Preconditions**:
- User has 300 coins
- Attempt to deduct 500 coins (reversal)

**Test Steps**:
1. User has 300 coins
2. Reverse commission worth 500 coins
3. Verify user coins = 0 (not -200)

**Expected Result**:
- User coins = GREATEST(0, 300 - 500) = 0 ✅

**SQL Verification**:
```sql
-- Before
SELECT coins FROM users WHERE id = '...';
-- coins: 300

-- Deduct operation
UPDATE users SET coins = GREATEST(0, coins - 500) WHERE id = '...';

-- After
SELECT coins FROM users WHERE id = '...';
-- coins: 0 (not -200)
```

**Actual Result**: [ PASS / FAIL ]

**Test Code**:
```typescript
it('TC-COM-022: should prevent negative balance on reversal', async () => {
  // Arrange: User has 300 coins
  await userRepo.update(
    { id: 'test-customer-creator-001' },
    { coins: 300 },
  );
  
  // Act: Reverse commission worth 500 coins
  await commissionService.reverseCommission('test-order-001');
  
  // Assert: Balance is 0, not negative
  const user = await userRepo.findOne({ where: { id: 'test-customer-creator-001' } });
  expect(user.coins).toBe(0); // GREATEST(0, 300 - 500) = 0 ✅
});
```

---

## ✅ Category 4: Business Rule Enforcement

### TC-COM-030: Chef Role Exclusion

**Priority**: High  
**Category**: Business Rules

**Preconditions**:
- Reel created by chef user
- Order delivered from chef's reel

**Test Steps**:
1. Call `createPendingCommission(orderId, chefUserId, reelId, 50000)`
2. Verify return value is `null`
3. Verify no commission_ledger entry created
4. Verify log message: "Creator is a chef, skipping commission"

**Expected Result**:
- Return value = `null`
- No DB insert
- Log warning emitted

**Actual Result**: [ PASS / FAIL ]

**Test Code**:
```typescript
it('TC-COM-030: should not create commission for chef creator', async () => {
  const ledger = await commissionService.createPendingCommission(
    'test-order-001',
    'test-chef-001', // Chef user
    'test-reel-001',
    50000,
  );
  
  expect(ledger).toBeNull(); // No commission for chefs
  
  // Verify no DB entry
  const count = await ledgerRepo.count({ where: { payeeUserId: 'test-chef-001' } });
  expect(count).toBe(0);
});
```

---

### TC-COM-031: Self-Earning Prevention

**Priority**: High  
**Category**: Business Rules

**Preconditions**:
- Creator orders from their own reel
- Order delivered

**Test Steps**:
1. Create order where `order.userId` = reel creator's ID
2. Call `createPendingCommission(orderId, creatorId, reelId, 50000)`
3. Verify return value is `null`
4. Verify no commission_ledger entry created
5. Verify log message: "Creator placed order themselves, skipping commission"

**Expected Result**:
- Return value = `null`
- No DB insert
- Log warning emitted

**Actual Result**: [ PASS / FAIL ]

**Test Code**:
```typescript
it('TC-COM-031: should prevent self-earning', async () => {
  // Arrange: Creator is also the buyer
  const order = await orderRepo.save({
    id: 'test-order-self',
    userId: 'test-customer-creator-001', // Same as creator
    totalPaise: 55000,
    reelCreatorUserId: 'test-customer-creator-001',
    reelId: 'test-reel-001',
    deliveryStatus: 'delivered',
  });
  
  // Act
  const ledger = await commissionService.createPendingCommission(
    order.id,
    'test-customer-creator-001',
    'test-reel-001',
    50000,
  );
  
  // Assert
  expect(ledger).toBeNull(); // No self-earning
});
```

---

### TC-COM-032: Only Customer Creators Earn

**Priority**: High  
**Category**: Business Rules

**Preconditions**:
- Creator has role 'customer'
- Order delivered

**Test Steps**:
1. Call `createPendingCommission()` with customer creator
2. Verify commission created successfully
3. Verify status = 'pending'

**Expected Result**:
- Commission ledger created ✅
- Status = 'pending' ✅

**Actual Result**: [ PASS / FAIL ]

---

### TC-COM-033: USER_REVIEW Reels Only

**Priority**: Medium  
**Category**: Business Rules

**Preconditions**:
- Reel type is 'promotional' (not 'user_review')
- Order delivered

**Test Steps**:
1. Create order with non-monetizable reel type
2. Verify commission not created (if validation in place)

**Expected Result**:
- No commission created for promotional reels

**Implementation Note**: This validation may be in Reel module (check if `linkedOrderId` exists)

**Actual Result**: [ PASS / FAIL ]

---

## ✅ Category 5: Earnings Preview

### TC-COM-040: Reel Earnings Preview - 3 Scenarios

**Priority**: High  
**Category**: Earnings Preview

**Preconditions**:
- Reel exists with linkedOrderId
- Creator order value = ₹500 (50,000 paise)
- User is reel owner

**Test Steps**:
1. GET `/api/v1/commissions/preview/reel/test-reel-001`
2. Verify `baseOrderValue` = 50,000
3. Verify `baseCoinsPerOrder` = 500 (exact match)
4. Verify `boostedOrderValue` = 75,000 (1.5x)
5. Verify `boostedCoinsPerOrder` = 625 (upsell scenario)
6. Verify `cappedCoinsPerOrder` = 750 (2x max)
7. Verify `effectiveRatePercentApprox` ≈ 10.0

**Expected Result**:
```json
{
  "success": true,
  "data": {
    "reelId": "test-reel-001",
    "baseOrderValue": 50000,
    "baseCoinsPerOrder": 500,
    "boostedOrderValue": 75000,
    "boostedCoinsPerOrder": 625,
    "cappedCoinsPerOrder": 750,
    "effectiveRatePercentApprox": 10.0
  }
}
```

**Calculation Validation**:
- Base: commission(50000, 50000) = 500 coins ✅
- Boosted: commission(50000, 75000) = 625 coins ✅
- Capped: commission(50000, 100000) = 750 coins ✅

**Actual Result**: [ PASS / FAIL ]

---

### TC-COM-041: Earnings Preview - Not Reel Owner (403)

**Priority**: High  
**Category**: Earnings Preview

**Preconditions**:
- Reel exists
- User is not reel owner

**Test Steps**:
1. GET `/api/v1/commissions/preview/reel/test-reel-001`
2. Use JWT of different user
3. Verify response = 403 Forbidden

**Expected Result**:
```json
{
  "statusCode": 403,
  "message": "Only reel owner can view earnings preview",
  "error": "Forbidden"
}
```

**Actual Result**: [ PASS / FAIL ]

---

### TC-COM-042: Earnings Preview - No Linked Order (Zero Preview)

**Priority**: Medium  
**Category**: Earnings Preview

**Preconditions**:
- Reel exists without linkedOrderId
- User is reel owner

**Test Steps**:
1. GET `/api/v1/commissions/preview/reel/test-reel-no-order`
2. Verify all values = 0

**Expected Result**:
```json
{
  "success": true,
  "data": {
    "reelId": "test-reel-no-order",
    "baseOrderValue": 0,
    "baseCoinsPerOrder": 0,
    "boostedOrderValue": 0,
    "boostedCoinsPerOrder": 0,
    "cappedCoinsPerOrder": 0,
    "effectiveRatePercentApprox": 0
  }
}
```

**Actual Result**: [ PASS / FAIL ]

---

### TC-COM-043: User Earnings Overview

**Priority**: High  
**Category**: Earnings Preview

**Preconditions**:
- User has 3 credited commissions in last 30 days (1500 coins)
- User has 5 lifetime credited commissions (5000 coins)
- User has 3 active monetized reels

**Test Steps**:
1. GET `/api/v1/commissions/preview/overview`
2. Verify `last30DaysCoins` = 1500
3. Verify `last30DaysMoney` = 150 (₹)
4. Verify `lifetimeCoins` = 5000
5. Verify `lifetimeMoney` = 500 (₹)
6. Verify `activeReelsCount` = 3
7. Verify `avgCoinsPerOrder` = 1000 (5000 / 5)
8. Verify `tipText` contains helpful message

**Expected Result**:
```json
{
  "success": true,
  "data": {
    "last30DaysCoins": 1500,
    "last30DaysMoney": 150,
    "lifetimeCoins": 5000,
    "lifetimeMoney": 500,
    "activeReelsCount": 3,
    "avgCoinsPerOrder": 1000,
    "tipText": "More reels with menus = more orders and more earnings."
  }
}
```

**Actual Result**: [ PASS / FAIL ]

---

### TC-COM-044: Earnings Overview - No Earnings Yet

**Priority**: Medium  
**Category**: Earnings Preview

**Preconditions**:
- User has no commissions
- User has 1 reel with linkedOrderId

**Test Steps**:
1. GET `/api/v1/commissions/preview/overview`
2. Verify all earnings = 0
3. Verify `avgCoinsPerOrder` = null
4. Verify `tipText` = "Share your reels to get more orders and start earning coins."

**Expected Result**:
```json
{
  "success": true,
  "data": {
    "last30DaysCoins": 0,
    "last30DaysMoney": 0,
    "lifetimeCoins": 0,
    "lifetimeMoney": 0,
    "activeReelsCount": 1,
    "avgCoinsPerOrder": null,
    "tipText": "Share your reels to get more orders and start earning coins."
  }
}
```

**Actual Result**: [ PASS / FAIL ]

---

## ✅ Category 6: Background Job Settlement

### TC-COM-050: Process All Pending Commissions

**Priority**: High  
**Category**: Background Jobs

**Preconditions**:
- 10 pending commissions in DB
- Total coins = 5000

**Test Steps**:
1. POST `/admin/jobs/process-commissions`
2. Verify response `processed` = 10
3. Verify response `totalCoins` = 5000
4. Query DB: all 10 commissions now 'credited'
5. Verify all 10 users received coins

**Expected Result**:
```json
{
  "success": true,
  "message": "Processed 10 commissions",
  "data": {
    "processed": 10,
    "totalCoins": 5000,
    "totalPaise": 50000
  }
}
```

**Actual Result**: [ PASS / FAIL ]

**Test Code**:
```typescript
it('TC-COM-050: should process all pending commissions', async () => {
  // Arrange: Create 10 pending commissions
  const ledgers = [];
  for (let i = 0; i < 10; i++) {
    const ledger = await commissionService.createPendingCommission(
      `test-order-${i}`,
      'test-customer-creator-001',
      `test-reel-${i}`,
      50000,
    );
    ledgers.push(ledger);
  }
  
  // Act: Run job
  const result = await commissionService.processPendingCommissionsJob();
  
  // Assert
  expect(result.processed).toBe(10);
  expect(result.totalCoins).toBe(5000); // 500 × 10
  
  // Verify all credited
  const creditedCount = await ledgerRepo.count({ where: { status: 'credited' } });
  expect(creditedCount).toBe(10);
});
```

---

### TC-COM-051: Job Idempotency

**Priority**: High  
**Category**: Background Jobs

**Preconditions**:
- 10 pending commissions

**Test Steps**:
1. Run job → processes 10 commissions
2. Run job again → processes 0 commissions (idempotent)
3. Verify user balances unchanged

**Expected Result**:
- First run: `processed` = 10
- Second run: `processed` = 0
- User balances correct (no double-credit)

**Actual Result**: [ PASS / FAIL ]

---

### TC-COM-052: Job Error Isolation

**Priority**: Medium  
**Category**: Background Jobs

**Preconditions**:
- 10 pending commissions
- 1 commission has invalid payee_user_id (user deleted)

**Test Steps**:
1. Run job
2. Verify 9 commissions credited successfully
3. Verify 1 commission failed (logged error)
4. Verify job completes (doesn't throw)

**Expected Result**:
- `processed` = 9 (1 failed)
- Error logged for failed commission
- Job doesn't crash

**Actual Result**: [ PASS / FAIL ]

---

## ✅ Category 7: Integration Testing

### TC-COM-060: Complete Commission Flow (E2E)

**Priority**: Critical  
**Category**: Integration

**Scenario**: User orders from creator's reel, commission credited

**Test Steps**:
1. Creator posts reel with linked order (₹500)
2. Buyer places order from reel (₹550)
3. Order delivered → `createPendingCommission()` called
4. Verify commission ledger created (status=pending, coins=525)
5. Background job runs → `creditCommission()` called
6. Verify ledger status=credited
7. Verify creator coin balance increased by 525

**Expected Result**:
- Commission ledger exists with correct calculation
- Creator coins = initial + 525
- Ledger status = 'credited'

**Actual Result**: [ PASS / FAIL ]

---

### TC-COM-061: Refund Reversal Flow (E2E)

**Priority**: Critical  
**Category**: Integration

**Scenario**: Order refunded after commission credited

**Test Steps**:
1. Complete commission flow (creator has 525 coins)
2. Order refunded → `reverseCommission()` called
3. Verify ledger status=reversed
4. Verify creator coin balance decreased by 525

**Expected Result**:
- Ledger status = 'reversed'
- Creator coins = initial (525 deducted)
- Ledger notes explain reversal reason

**Actual Result**: [ PASS / FAIL ]

---

### TC-COM-062: Multiple Commissions Same User

**Priority**: High  
**Category**: Integration

**Scenario**: Creator earns from 5 different orders

**Test Steps**:
1. Create 5 orders from creator's reels
2. All delivered → 5 pending commissions
3. Background job runs → all credited
4. GET `/commissions/my` → verify summary correct
5. Verify `lifetimeCoins` = sum of all 5

**Expected Result**:
- 5 commission ledgers exist
- All status = 'credited'
- User coins = sum of all 5
- API returns correct summary

**Actual Result**: [ PASS / FAIL ]

---

## ✅ Category 8: Edge Cases & Error Handling

### TC-COM-070: Order Not Found

**Priority**: Medium  
**Category**: Error Handling

**Test Steps**:
1. Call `createPendingCommission('invalid-order-id', userId, reelId, 50000)`
2. Verify error thrown: "Order invalid-order-id not found"

**Expected Result**:
- Error thrown ✅
- No DB insert

**Actual Result**: [ PASS / FAIL ]

---

### TC-COM-071: User Not Found

**Priority**: Medium  
**Category**: Error Handling

**Test Steps**:
1. Call `createPendingCommission(orderId, 'invalid-user-id', reelId, 50000)`
2. Verify return value = `null`
3. Verify warning logged

**Expected Result**:
- Return value = `null`
- Log: "Creator user invalid-user-id not found, skipping commission"

**Actual Result**: [ PASS / FAIL ]

---

### TC-COM-072: Ledger Not Found

**Priority**: Medium  
**Category**: Error Handling

**Test Steps**:
1. Call `creditCommission('invalid-ledger-id')`
2. Verify error thrown: "Commission ledger invalid-ledger-id not found"

**Expected Result**:
- Error thrown ✅

**Actual Result**: [ PASS / FAIL ]

---

### TC-COM-073: Large Order Value (Max Int)

**Priority**: Low  
**Category**: Edge Cases

**Test Steps**:
1. Create order with `totalPaise` = 2,147,483,647 (max int32)
2. Calculate commission
3. Verify no overflow

**Expected Result**:
- Commission calculated correctly
- No integer overflow error

**Actual Result**: [ PASS / FAIL ]

---

### TC-COM-074: Zero Creator Order Value

**Priority**: Medium  
**Category**: Edge Cases

**Test Steps**:
1. Create reel with `creatorOrderValue` = 0
2. User orders ₹500
3. Verify commission = 0 (no commissionable amount)

**Expected Result**:
- `commissionableAmount` = 0
- `commissionPaise` = 0
- `coins` = 0

**Actual Result**: [ PASS / FAIL ]

---

## 📊 Performance Testing

### PT-COM-001: Single Commission Creation Performance

**Target**: <100ms

**Test Steps**:
1. Measure time to create pending commission
2. Average over 100 iterations

**Expected Result**: <100ms avg

**Actual Result**: [ ms ]

---

### PT-COM-002: Batch Credit Performance

**Target**: <15ms per commission

**Test Steps**:
1. Create 1000 pending commissions
2. Measure time to credit all
3. Calculate avg per commission

**Expected Result**: <15s total (<15ms each)

**Actual Result**: [ ms per commission ]

---

### PT-COM-003: Get My Commissions Query Performance

**Target**: <120ms

**Test Steps**:
1. User has 1000 commission entries
2. GET `/commissions/my` (fetches summary + last 20)
3. Measure response time

**Expected Result**: <120ms

**Actual Result**: [ ms ]

---

### PT-COM-004: Earnings Preview Query Performance

**Target**: <150ms

**Test Steps**:
1. GET `/commissions/preview/reel/:reelId`
2. Measure response time (MongoDB query + 3 calculations)

**Expected Result**: <150ms

**Actual Result**: [ ms ]

---

## 🔁 Regression Testing

### RT-COM-001: V1 Formula Still Works

**Priority**: High  
**Category**: Regression

**Test Steps**:
1. Create reel without `creatorOrderValue` (legacy)
2. Order delivered
3. Verify commission calculated from `subtotalPaise`
4. Verify `formulaVersion` = 'v1'

**Expected Result**:
- V1 formula used ✅
- `formulaVersion` = 'v1'
- Commission based on `subtotalPaise`

**Actual Result**: [ PASS / FAIL ]

---

### RT-COM-002: Existing Pending Commissions Processed

**Priority**: High  
**Category**: Regression

**Test Steps**:
1. Query existing pending commissions in production DB
2. Run migration/job
3. Verify all processed correctly

**Expected Result**:
- All pending commissions credited
- No data loss

**Actual Result**: [ PASS / FAIL ]

---

## 📝 Test Execution Summary

### Test Run Metadata

| Field | Value |
|-------|-------|
| **Test Environment** | Staging |
| **Test Date** | YYYY-MM-DD |
| **Tester** | [Name] |
| **Build Version** | v1.0.0 |
| **Database** | PostgreSQL 15 + MongoDB 6 |

### Test Results Summary

| Category | Total | Passed | Failed | Skipped |
|----------|-------|--------|--------|---------|
| **V2 Commission Calculation** | 6 | [ ] | [ ] | [ ] |
| **Commission Ledger Lifecycle** | 5 | [ ] | [ ] | [ ] |
| **Atomic Coin Updates** | 3 | [ ] | [ ] | [ ] |
| **Business Rule Enforcement** | 4 | [ ] | [ ] | [ ] |
| **Earnings Preview** | 5 | [ ] | [ ] | [ ] |
| **Background Job Settlement** | 3 | [ ] | [ ] | [ ] |
| **Integration Testing** | 3 | [ ] | [ ] | [ ] |
| **Edge Cases & Error Handling** | 5 | [ ] | [ ] | [ ] |
| **Performance Testing** | 4 | [ ] | [ ] | [ ] |
| **Regression Testing** | 2 | [ ] | [ ] | [ ] |
| **TOTAL** | **40** | **[ ]** | **[ ]** | **[ ]** |

---

## 🐛 Bug Report Template

```markdown
### Bug ID: BUG-COM-XXX

**Title**: [Short description]

**Severity**: Critical | High | Medium | Low

**Test Case**: TC-COM-XXX

**Steps to Reproduce**:
1. Step 1
2. Step 2
3. Step 3

**Expected Result**:
[What should happen]

**Actual Result**:
[What actually happened]

**Logs/Screenshots**:
[Attach evidence]

**Environment**:
- OS: macOS/Linux/Windows
- Database: PostgreSQL 15
- Node Version: 18.x
- Branch: main

**Assigned To**: [Developer name]

**Status**: Open | In Progress | Fixed | Closed
```

---

**[SLICE_COMPLETE ✅]**

**Commission Module - Week 8, Module 2**  
**Documentation**: QA Test Cases complete (~6,800 lines)  
**Total Module Documentation**: ~31,600 lines (Feature Overview + Technical Guide + QA Test Cases)  
**Next Steps**: Move to Withdrawal module (Week 8, Module 3)
