---
name: SCRATCHPAD_EXAMPLE
description: 'Complete end-to-end example of SCRATCHPAD.md showing Architect, Planner, and Builder outputs.'
---

# SCRATCHPAD: CSS-21342 Add Stripe Payment Processing

> **Filename:** `SCRATCHPAD-CSS-21342.md` — Each issue gets its own versioned scratchpad file enabling parallel sessions. See [SKILLS.md](./SKILLS.md) for versioning details.

## Architectural Context

**Status**: complete

**Global Invariants**:
- Use `decimal.Decimal` for all currency calculations (no floats)
- All database operations must be transactional
- External service calls must be mockable via interfaces
- Tests must not depend on execution order (`t.Parallel()` safe)
- Audit trail required for all financial state changes

**Primary Files**:
- `internal/billing/stripe_processor.go`: Implements Stripe payment processing
- `internal/billing/stripe_processor_test.go`: Test suite for Stripe processor
- `internal/payment/gateway.go`: Payment gateway interface (already exists; Stripe processor implements it)

**Secondary Files**:
- `internal/audit/logger.go`: Audit trail logging (may need new event types)
- `internal/credentials/stripe_creds.go`: Stripe API credential management
- `internal/errors/billing_errors.go`: Payment-specific error definitions

**Patterns & References**:
- Table-driven tests: See `internal/bill/pricing_engine_test.go` (look for `tests := []struct` pattern)
- Payment gateway interface: See `internal/payment/gateway.go` (PaymentGateway interface)
- Decimal handling: See `internal/price/calculator.go` (decimal math examples)
- Error patterns: Reference `internal/apierrors/errors.go` for how to define custom error types
- Mock patterns: Reference `internal/docusign/mock.go` for external service mocking

**Dependency Graph**:
```
stripe_processor.go
  ├── payment/gateway.go (interface implementation)
  ├── audit/logger.go (log state changes)
  ├── credentials/stripe_creds.go (load Stripe API key)
  ├── decimal package (currency calculations)
  └── stripe-go SDK (external service)
```

**Notes**:
- No circular dependencies identified
- `PaymentGateway` interface already exists; Stripe processor just implements it
- Audit logging uses existing package; no new API needed
- Stripe SDK is v1.145.0; supports idempotency keys and webhook verification

---

## Test Design

**Issue Reference**: CSS-21342

**Acceptance Criteria & Test Mapping**:

| Criterion | Test Name | Type | Scenario |
|-----------|-----------|------|----------|
| AC1: Process single payment | `TestProcessPayment_ValidAmount` | unit | Happy path: valid amount, valid currency |
| AC1: Process single payment | `TestProcessPayment_DecimalPrecision` | unit | Edge case: $10.99 (2 decimals), no rounding errors |
| AC1: Process single payment | `TestProcessPayment_InvalidAmount` | unit | Error case: negative amount, expect error |
| AC2: Reject duplicate payments | `TestRejectDuplicatePayment` | unit | Error case: same idempotency key, expect ErrDuplicatePayment |
| AC3: Log audit trail | `TestAuditTrailLogged` | integration | Verify audit event created in DB after payment |
| AC4: Integrate Stripe API | `TestProcessPayment_StripeAPICall` | integration | Mock Stripe API verify correct charge request |
| AC5: Handle Stripe errors | `TestProcessPayment_StripeNetworkTimeout` | integration | Mock Stripe timeout, verify retry logic |
| AC5: Handle Stripe errors | `TestProcessPayment_StripeRejected` | integration | Mock Stripe decline, verify error propagated to user |

**Batch Structure**:

### Batch #1: Unit Tests (4 test cases)
- `TestProcessPayment_ValidAmount`: Valid payment structure (amount: Decimal("10.00"), currency: "USD"), verify processing returns success
- `TestProcessPayment_DecimalPrecision`: Amount with fractional cents ($10.99 → Decimal(1099, 2)), verify no rounding errors in calculation
- `TestProcessPayment_InvalidAmount`: Negative amount (-$50.00), expect error type `ErrInvalidAmount`
- `TestRejectDuplicatePayment`: Attempt duplicate payment (same idempotency key), expect error type `ErrDuplicatePayment`

### Batch #2: Integration Tests (4 test cases)
- `TestAuditTrailLogged`: Process payment, verify `audit_events` table contains new entry with status "payment_processed"
- `TestProcessPayment_StripeAPICall`: Mock Stripe API, verify `charges.create()` called with correct parameters (amount, currency, idempotency_key)
- `TestProcessPayment_StripeNetworkTimeout`: Mock Stripe to timeout after 2 attempts, verify exponential backoff + eventual failure
- `TestProcessPayment_StripeRejected`: Mock Stripe to return 402 (card declined), verify `ErrPaymentDeclined` returned to caller

**Test Data Requirements**:

Fixtures:
- `ValidPaymentRequest`: `{ Amount: Decimal("100.00"), Currency: "USD", IdempotencyKey: "stripe-test-123", CustomerId: "cust-001" }`
- `InvalidPaymentRequest`: `{ Amount: Decimal("-50.00"), Currency: "USD", ... }` (negative amount)
- `DuplicatePaymentRequest`: `{ Amount: Decimal("100.00"), Currency: "USD", IdempotencyKey: "stripe-123" }` (same key as existing)
- `EdgeCasePaymentRequest`: `{ Amount: Decimal("10.99"), Currency: "USD", ... }` (tests decimal precision)

Mocks:
- `StripeAPIClient`: Mock Stripe charge creation
  - Success scenario: Return charge ID "ch-123456"
  - Timeout scenario: Delay 5s, then context.DeadlineExceeded
  - Rejection scenario: Return { "status": "failed", "error": "card_declined" }
  
- `AuditLogger`: Already exists in codebase; no mock needed (we test actual logging)

Database State:
 - `payments` table: Must be empty before each test (Builder will add `t.Cleanup()` to delete test payments)
- `audit_events` table: Must be empty before each test
- Foreign keys: `payments.customer_id` → `customers.id` (create test customer beforehand)

**Coverage Target**: 85% line coverage, 80% branch coverage

**Technical Context**:
- Affected Packages: `internal/billing`, `internal/payment`
- External Dependencies: Stripe API (v1.145.0), PostgreSQL DB
- Database: Uses `decimal` type for currency fields; all amounts stored as integers (cents) with decimal precision
- Existing Patterns: 
  - Payment gateway interface pattern in `internal/payment/gateway.go`
  - Decimal math helpers in `internal/price/calculator.go`
  - Error definitions in `internal/apierrors/`
  - Audit logging in `internal/audit/logger.go`

**Notes for Implementer**:
- Stripe SDK already imported; no new dependencies needed
- Use `stripe.Client.Charges.New()` for API calls
- Idempotency keys prevent duplicate charges; test this!
- Decimal precision: $10.99 = 1099 cents (scale 2); verify no float rounding

---

## Implementation Progress

### Batch #1: Unit Tests — PASS ✅
- [x] `TestProcessPayment_ValidAmount` — GREEN (1m 30s)
  - Implementation: Created `ProcessPayment(ctx, req)` function
  - Assertion: Returns success, no error
- [x] `TestProcessPayment_DecimalPrecision` — GREEN (2m)
  - Implementation: Validated amount calculation with Decimal
  - Assertion: $10.99 → 1099 cents, no loss of precision
- [x] `TestProcessPayment_InvalidAmount` — GREEN (1m)
  - Implementation: Added validation `if amount < 0 { return ErrInvalidAmount }`
  - Assertion: `errors.Is(err, ErrInvalidAmount)` passes
- [x] `TestRejectDuplicatePayment` — GREEN (1m 45s)
  - Implementation: Check idempotency key in DB before processing
  - Assertion: Returns `ErrDuplicatePayment`

**Refactoring Completed**:
- Extracted `validatePayment()` helper to reduce duplication
- Created `NewTestPayment()` fixture builder in `internal/testutil/fixtures.go`
- Standardized error handling to use `errors.Is()` pattern

**Coverage**: 88% line, 87% branch (exceeds 85%/80% target)

**Test Time**: ~6m 15s total for batch

---

### Batch #2: Integration Tests — PASS ✅
- [x] `TestAuditTrailLogged` — GREEN (2m 30s)
  - Implementation: Called `auditLogger.LogPaymentProcessed()` after Stripe charge
  - Assertion: `SELECT FROM audit_events WHERE type='payment_processed'` returns 1 row
- [x] `TestProcessPayment_StripeAPICall` — GREEN (3m)
  - Implementation: Injected mock Stripe client via interface
  - Assertion: Mock verified `charges.create()` called with correct parameters
- [x] `TestProcessPayment_StripeNetworkTimeout` — GREEN (2m 45s)
  - Implementation: Mock returns timeout, code retries with backoff
  - Assertion: After 2 attempts, returns `ErrStripeNetworkFailure`
- [x] `TestProcessPayment_StripeRejected` — GREEN (1m 45s)
  - Implementation: Mock returns 402, code translates to `ErrPaymentDeclined`
  - Assertion: User sees proper error message

**Refactoring Completed**:
- Extracted mock Stripe client to `internal/mocktesting/stripe_mock.go`
- Created `WaitForAuditEvent()` helper to polling audit table (avoids timing issues)
- Added retry logic with exponential backoff (0.5s, 1s, 2s)

**Coverage**: 92% line, 91% branch (exceeds target)

**Test Time**: ~10m total for batch

---

## Approval Log

**Architect Approval**: 2025-03-04T10:30:00Z — Approved
- Context map reviewed and complete
- 5 primary/secondary files identified
- 5 key patterns referenced from existing code

**Planner Approval**: 2025-03-04T11:45:00Z — Approved
- Test plan covers all 5 acceptance criteria
- 8 unit + integration tests planned
- Coverage target 85% confirmed with user
- User asked: "Should we test webhook handling?" Planner reply: "Out of scope for first PR; captured as follow-up issue"

**User Confirmations**:
- Checkpoint #1: 2025-03-04T14:20:00Z — Batch #1 (Unit Tests) — Approved ✅
  - Comment: "Good coverage, all tests are clear. Proceed to Batch #2."
- Checkpoint #2: 2025-03-04T17:30:00Z — Batch #2 (Integration Tests) — Approved ✅
  - Comment: "Integration tests look good. Tests properly mock external API."

---

## Roadblocks & Decisions

**Resolved Decisions**:
- **Decision**: Use in-memory mock for Stripe instead of real sandbox
  - Rationale: Sandbox requires network calls, slow (5s per test); mock is instant (<100ms)
  - Implementation: Created `internal/mocktesting/stripe_mock.go`

- **Decision**: Audit logging tested against real DB, not mocked
  - Rationale: Audit is critical for financial operations; must test real DB behavior
  - Implementation: Tests run against test DB (recreated per test)

- **Decision**: Exponential backoff for Stripe timeouts (0.5s, 1s, 2s)
  - Rationale: Follows industry standard for network resilience; prevents overwhelming API
  - Implementation: Added `RetryWithBackoff()` helper in stripe_processor.go

**No Active Roadblocks** — All tests passing, implementation complete.

---

## Final Summary

**Tests Completed**: 8 (4 unit + 4 integration)
**Coverage Achieved**: 92% line, 91% branch (target: 85%/80%) ✅
**All Tests Passing**: Yes ✅
**Code Quality**:
- Follows table-driven test pattern (reference: pricing_engine_test.go)
- Uses interface-based mocking (PaymentGateway interface)
- Proper error handling with `errors.Is()`
- Decimal precision verified for all financial calculations
- Audit trail tested with real DB
- No flakiness detected (ran tests 10x, all passed)
- No race conditions detected (`go test -race ./...` passed)

**Preparation for PR**:
- ✅ All tests pass: `make test`
- ✅ Coverage meets threshold: 92% line (target 85%)
- ✅ Linting passed: `make lint`
- ✅ No race conditions: `go test -race ./...`
- ✅ Code formatted: `make fmt`
- ✅ Ready for code review and merge
