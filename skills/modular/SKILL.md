---
name: modular
description: "Enforce SOLID principles and module boundaries. Apply when designing a new module, reviewing existing code structure, or catching tight coupling. Trigger on: 'design this module', 'structure this code', 'how should I organize this', 'is this too coupled', 'this is getting messy', 'refactor this structure'. Code that works is not the same as code that can be changed. Modularity is what makes code changeable without breaking what you haven't touched."
user-invocable: true
argument-hint: "[module or system to review or design]"
---

# Modular — SOLID Principles and Module Boundaries

## INPUT

Consumes: `SPEC.md` from `/spec` and `SCHEMA.md` from `/db-review`

Uses: interface definitions (from SPEC.md) and table ownership (from SCHEMA.md) to draw module boundaries.

---

## THE LOOP

**TRIGGER:** Interfaces are defined in SPEC.md. Schema is reviewed in SCHEMA.md. Before writing code.

**CYCLE:**
1. Name every module — if the name requires "and," split it
2. Apply each SOLID principle — find violations, propose fixes
3. Draw module boundaries — who owns what, who calls whom
4. Verify no circular dependencies
5. Confirm each module is independently testable

**EXIT CONDITION:** Every module has a single responsibility, depends on interfaces not implementations, and can be changed without touching other modules. MODULE_MAP.md is written.

**The single test of modularity:** Can you change this module without looking at — or breaking — any other module? If yes: modular. If no: coupled.

---

## S — SINGLE RESPONSIBILITY

One module, one reason to change.

Bad:
```typescript
// UserService handles auth, emails, billing, and profile — 4 reasons to change
class UserService {
  login(email, password) { ... }
  sendWelcomeEmail(user) { ... }
  chargeCreditCard(user, amount) { ... }
  updateProfile(user, data) { ... }
}
```

Good:
```typescript
class AuthService    { login(email, password) { ... } }
class EmailService   { sendWelcomeEmail(user) { ... } }
class BillingService { charge(user, amount) { ... } }
class ProfileService { update(user, data) { ... } }
```

**The test:** Name your module. If the name requires "and" — it has two responsibilities.

`PaymentProcessor` — one responsibility.
`PaymentProcessorAndEmailSender` — two.

**For Strapi:** Each API folder is one content type. One service per resource. Controllers handle HTTP only — no business logic in controllers. Business logic lives in services.

---

## O — OPEN/CLOSED

Open for extension. Closed for modification.

Adding a new behavior: add new code, don't edit existing code.

Bad — adding Razorpay means editing the existing function:
```typescript
function processPayment(method: string, amount: number) {
  if (method === 'stripe') { ... }
  if (method === 'paypal') { ... }
  // To add Razorpay: you MUST edit this function
}
```

Good — adding Razorpay means adding a new class:
```typescript
interface PaymentProvider {
  charge(amount: number): Promise<ChargeResult>
}

class StripeProvider implements PaymentProvider { ... }
class PayPalProvider implements PaymentProvider { ... }
class RazorpayProvider implements PaymentProvider { ... }  // No edits to existing code
```

**Signal:** If adding a new variant requires finding all the if/switch statements that need updating — extract to an interface.

---

## L — LISKOV SUBSTITUTION

Swap implementations without breaking callers.

If module A depends on interface B, any implementation of B must work for A without A knowing which one it's using.

Practical meaning: your business logic works identically whether:
- Files go to local disk or Cloudinary
- Emails go to SendGrid or Resend
- Cache is Redis or in-memory
- Database is SQLite (dev) or Postgres (prod)

Bad:
```typescript
import { cloudinary } from 'cloudinary'

async function saveProfilePicture(file: Buffer) {
  return cloudinary.uploader.upload(file)  // Coupled to Cloudinary
}
```

Good:
```typescript
interface FileStorage {
  upload(file: Buffer, path: string): Promise<string>
}

async function saveProfilePicture(storage: FileStorage, file: Buffer) {
  return storage.upload(file, 'profiles/')  // Any FileStorage implementation works
}
```

**For Strapi:** The service layer is naturally substitutable — your controller calls `strapi.service('api::article.article').find()` regardless of what database is underneath. Don't break it by importing implementation details directly.

---

## I — INTERFACE SEGREGATION

Don't force modules to depend on what they don't use.

Bad — read-only consumer forced to depend on write operations:
```typescript
interface UserRepository {
  findById(id: string): Promise<User>
  create(data: UserData): Promise<User>
  update(id: string, data: Partial<UserData>): Promise<User>
  delete(id: string): Promise<void>
}

class ReportGenerator {
  constructor(private users: UserRepository) {}  // Only uses findById
}
```

Good:
```typescript
interface UserReader { findById(id: string): Promise<User> }
interface UserWriter {
  create(data: UserData): Promise<User>
  update(id: string, data: Partial<UserData>): Promise<User>
  delete(id: string): Promise<void>
}

class ReportGenerator {
  constructor(private users: UserReader) {}  // Depends only on what it needs
}
```

**Signal:** If a module imports something and only uses 20% of it — split the interface.

---

## D — DEPENDENCY INVERSION

Depend on abstractions. Don't import concrete implementations in business logic.

Bad:
```typescript
import Stripe from 'stripe'

async function processSubscription(userId: string, plan: string) {
  const stripe = new Stripe(process.env.STRIPE_KEY)
  return stripe.subscriptions.create({ ... })
}
```

Good:
```typescript
interface SubscriptionProvider {
  createSubscription(userId: string, plan: string): Promise<Subscription>
  cancelSubscription(subscriptionId: string): Promise<void>
}

async function processSubscription(
  provider: SubscriptionProvider,
  userId: string,
  plan: string
) {
  return provider.createSubscription(userId, plan)
}
```

**For Strapi:** Business logic lives in `src/api/[resource]/services/`. External integrations live in `src/plugins/` or `config/`. Services call plugin interfaces, never import payment SDKs directly.

---

## MODULE BOUNDARY RULES

### Rule 1: Modules communicate through interfaces, not internals

```
✓ OrderService calls PaymentService.charge(amount)
✗ OrderService imports PaymentService's internal validateCard() helper
✗ OrderService queries the payments table directly
```

### Rule 2: No circular dependencies

Module A cannot depend on module B if module B depends on module A. Circular = incorrectly split.

Detect: `npx madge --circular src/`

Fix: extract the shared concern into a third module that both A and B depend on.

### Rule 3: Data flows one direction

```
HTTP Request
    ↓
Controller (validate input, call service)
    ↓
Service (business logic, call repository)
    ↓
Repository / Strapi Entity Service (data access)
    ↓
Database
```

Nothing reaches upward. Repository doesn't call service. Service doesn't call controller.

### Rule 4: Each module is independently testable

If testing module A requires setting up B, C, and D — the modules are too coupled.

```typescript
// Good: PaymentService testable with a mock SubscriptionProvider
const mockProvider: SubscriptionProvider = {
  createSubscription: jest.fn().mockResolvedValue({ id: 'sub_123' })
}
const service = new PaymentService(mockProvider)
```

### Rule 5: Module size heuristic

Too large:
- File exceeds 200 lines
- More than 5 public methods
- You need to scroll to understand what it does
- "What does this file do?" has a multi-sentence answer

Too small:
- Only wraps one function call
- Exists solely to "separate" code that always changes together

Right size: does one thing completely, clear interface, understood in 2 minutes.

---

## STRAPI-SPECIFIC MODULE STRUCTURE

```
src/api/[resource]/
├── content-types/[resource]/schema.json  ← Data shape only
├── controllers/[resource].ts             ← HTTP only: validate, call service, return response
├── services/[resource].ts                ← Business logic only
├── routes/[resource].ts                  ← Route definitions
└── middlewares/                          ← Auth, rate limit
```

Controller (HTTP only):
```typescript
async create(ctx) {
  const { amount, currency } = ctx.request.body
  const result = await strapi.service('api::payment.payment').process(amount, currency)
  ctx.body = { data: result }
}
```

Service (business logic only):
```typescript
async process(amount: number, currency: string) {
  if (amount <= 0) throw new Error('Amount must be positive')
  const charge = await strapi.plugin('payment-provider').service('provider').charge(amount, currency)
  await strapi.db.query('api::payment.payment').create({ data: { amount, chargeId: charge.id } })
  return charge
}
```

The controller doesn't know how payments are processed. The service doesn't know about HTTP.

---

## MODULE REVIEW CHECKLIST

```
Single Responsibility:
[ ] Module name requires no "and"
[ ] Module has one reason to change
[ ] Public interface has ≤ 5 methods

Open/Closed:
[ ] Adding new behavior requires adding code, not editing existing code
[ ] No if/switch on type that expands with every new type

Liskov:
[ ] Swapping an implementation doesn't change callers
[ ] Implementations are injected, not imported directly in business logic

Interface Segregation:
[ ] No module depends on methods it doesn't use
[ ] Read and write interfaces are separated where consumers split naturally

Dependency Inversion:
[ ] Business logic imports interfaces, not concrete implementations
[ ] External SDKs are isolated behind a service/plugin

Boundaries:
[ ] No circular dependencies (npx madge --circular src/)
[ ] Data flows one direction
[ ] Each module is testable with mocked dependencies
[ ] Module fits size heuristic (< 200 lines, ≤ 5 public methods, 2-minute read)
```

---

## OUTPUT: MODULE_MAP.md

```markdown
# MODULE_MAP.md

## Module Inventory
| Module | Responsibility | Public Interface | Owns Table(s) |
|--------|---------------|-----------------|---------------|
| AuthService | Authentication | login, logout, refresh | sessions |
| OrderService | Order lifecycle | create, cancel, status | orders |

## Dependency Graph
[Module A] → [Interface] ← [Module B]
No circular dependencies confirmed: npx madge output

## Violations Found
| Principle | Location | Issue | Fix |
|-----------|----------|-------|-----|

## Refactor Plan (if needed)
Step 1: [Smallest, safest change]
Step 2: [Next change]
Step 3: [Verify with /tdd]

## Feeds into
/tdd (TEST_REPORT.md) — module boundaries define mock boundaries for tests
```

---

## FEEDS INTO

`/tdd` — module interfaces define mock boundaries. Each module is testable in isolation.
