---
name: modular
description: "Enforce SOLID principles and module boundaries in every system. Apply when designing a new module, reviewing existing code structure, or catching tight coupling. Trigger on: 'design this module', 'structure this code', 'how should I organize this', 'is this too coupled', 'this is getting messy', 'refactor this structure', 'how do modules talk to each other'. Code that works is not the same as code that can be changed. Modularity is what makes code changeable without breaking what you haven't touched."
user-invocable: true
argument-hint: "[module or system to review or design]"
---

# Modular — SOLID Principles and Module Boundaries

The reason systems become unmaintainable isn't that they grow. It's that they grow without boundaries. Each shortcut that reaches across a module boundary, each function that does two things, each import of a concrete implementation — these are deposits into a debt account that eventually forecloses.

SOLID principles are the engineering community's best-tested answer to "how do we prevent this?" They're not academic — they're scars from production systems that became too tangled to change.

**The single test of modularity:** Can you change this module without looking at — or breaking — any other module?

If yes: modular. If no: coupled.

---

## THE FIVE PRINCIPLES — PRACTICAL, NOT ACADEMIC

### S — Single Responsibility

**One module, one reason to change.**

A module has too many responsibilities if changing one behavior requires you to understand and test multiple unrelated behaviors.

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
// Each service has one reason to change
class AuthService      { login(email, password) { ... } }
class EmailService     { sendWelcomeEmail(user) { ... } }
class BillingService   { charge(user, amount) { ... } }
class ProfileService   { update(user, data) { ... } }
```

**The test:** Name your module. If the name requires "and" — it has two responsibilities.

`PaymentProcessor` — one responsibility.
`PaymentProcessorAndEmailSender` — two.

**For Strapi:** Each API folder is one content type. One service file per resource. Strapi controllers handle HTTP only — no business logic in controllers. Business logic lives in services.

### O — Open/Closed

**Open for extension. Closed for modification.**

When you add a new behavior, you should be able to add new code — not edit existing code. Editing existing code risks breaking existing behavior.

Bad — adding a new payment provider requires editing the existing handler:
```typescript
// Adding Razorpay means editing this existing function
function processPayment(method: string, amount: number) {
  if (method === 'stripe') { /* Stripe logic */ }
  if (method === 'paypal') { /* PayPal logic */ }
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
class RazorpayProvider implements PaymentProvider { ... }  // New — no edits to existing code

function processPayment(provider: PaymentProvider, amount: number) {
  return provider.charge(amount)
}
```

**The practical signal:** If adding a new variant requires searching for all the `if/else` or `switch` statements that need updating — the code is closed for extension. Extract to an interface.

### L — Liskov Substitution

**Swap implementations without breaking callers.**

If module A depends on interface B, any implementation of B must work for A without A knowing which implementation it's using.

Practical meaning: your business logic should work identically whether:
- Files go to local disk or Cloudinary
- Emails go to SendGrid or Resend
- Cache is Redis or in-memory
- Database is SQLite (dev) or Postgres (prod)

Bad — business logic knows it's using Cloudinary specifically:
```typescript
import { cloudinary } from 'cloudinary'

async function saveProfilePicture(file: Buffer) {
  // Directly coupled to Cloudinary — can't swap without touching this function
  return cloudinary.uploader.upload(file)
}
```

Good — business logic depends on a storage interface:
```typescript
interface FileStorage {
  upload(file: Buffer, path: string): Promise<string>
}

// Strapi's upload plugin implements this interface
// Local dev: LocalStorage implements it
// Prod: CloudinaryStorage implements it
// Business logic never imports either directly

async function saveProfilePicture(storage: FileStorage, file: Buffer) {
  return storage.upload(file, 'profiles/')
}
```

**For Strapi:** Strapi's service layer is naturally substitutable — your controller calls `strapi.service('api::article.article').find()` regardless of what database is underneath. This is Liskov in practice. Don't break it by importing implementation details directly.

### I — Interface Segregation

**Don't force modules to depend on what they don't use.**

A module that only reads data shouldn't need to import a write-capable interface. A module that only sends emails shouldn't depend on the full UserService.

Bad — read-only consumer forced to depend on write operations:
```typescript
interface UserRepository {
  findById(id: string): Promise<User>
  findByEmail(email: string): Promise<User>
  create(data: UserData): Promise<User>
  update(id: string, data: Partial<UserData>): Promise<User>
  delete(id: string): Promise<void>
}

// This module only reads — but must depend on the full interface
class ReportGenerator {
  constructor(private users: UserRepository) {}
  async generate() { const u = await this.users.findById(...) }
}
```

Good — separate read and write interfaces:
```typescript
interface UserReader {
  findById(id: string): Promise<User>
  findByEmail(email: string): Promise<User>
}

interface UserWriter {
  create(data: UserData): Promise<User>
  update(id: string, data: Partial<UserData>): Promise<User>
  delete(id: string): Promise<void>
}

// ReportGenerator depends only on what it needs
class ReportGenerator {
  constructor(private users: UserReader) {}
}
```

**Practical signal:** If your module imports something and only uses 20% of it — split the interface.

### D — Dependency Inversion

**Depend on abstractions. Don't import concrete implementations in business logic.**

High-level modules (business logic) should not depend on low-level modules (Stripe, Cloudinary, SendGrid). Both should depend on an abstraction (an interface).

Bad — business logic directly imports Stripe:
```typescript
import Stripe from 'stripe'  // Business logic knows about Stripe

async function processSubscription(userId: string, plan: string) {
  const stripe = new Stripe(process.env.STRIPE_KEY)
  // Tightly coupled — testing requires Stripe, swapping requires editing this function
  return stripe.subscriptions.create({ ... })
}
```

Good — business logic imports an interface:
```typescript
// In Strapi: define the interface in a service
interface SubscriptionProvider {
  createSubscription(userId: string, plan: string): Promise<Subscription>
  cancelSubscription(subscriptionId: string): Promise<void>
}

// The concrete implementation lives in a separate plugin/service
// Swapping from Stripe to Cashfree means changing one file, not hunting through business logic
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

A module is a black box. Other modules can call its public interface. They cannot access its internal state, its private functions, or its database queries directly.

```
✓ OrderService calls PaymentService.charge(amount)
✗ OrderService imports PaymentService's internal validateCard() helper
✗ OrderService queries the payments table directly
```

### Rule 2: No circular dependencies

Module A cannot depend on module B if module B depends on module A. Circular dependencies mean the modules are actually one module that has been incorrectly split.

Detect with: `npx madge --circular src/`

Fix by: extracting the shared concern into a third module that both A and B depend on.

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

Nothing in this chain should reach upward. The repository doesn't call the service. The service doesn't call the controller. Data flows down, results flow up.

### Rule 4: Each module is independently testable

If testing module A requires setting up module B, C, and D — the modules are too coupled. Each module should be testable with its dependencies mocked at the interface level.

```typescript
// Good: PaymentService is testable with a mock SubscriptionProvider
const mockProvider: SubscriptionProvider = {
  createSubscription: jest.fn().mockResolvedValue({ id: 'sub_123' })
}
const service = new PaymentService(mockProvider)
```

### Rule 5: Module size heuristic

A module is too large if:
- Its file exceeds 200 lines
- It has more than 5 public methods
- You need to scroll to understand what it does
- New team members ask "what does this file do?" and the answer is more than one sentence

A module is too small if:
- It only wraps one function call
- It exists solely to "separate" code that always changes together

The right size: a module that does one thing completely, with a clear interface, that you can understand in under 2 minutes.

---

## STRAPI-SPECIFIC MODULE STRUCTURE

```
src/api/[resource]/
├── content-types/[resource]/schema.json  ← Data shape (only)
├── controllers/[resource].ts             ← HTTP only: validate input, call service, return response
├── services/[resource].ts                ← Business logic only: orchestrates, enforces rules
├── routes/[resource].ts                  ← Route definitions
└── middlewares/                          ← Request-level concerns (auth, rate limit)
```

**Controller responsibility (HTTP only):**
```typescript
// controllers/payment.ts
async create(ctx) {
  const { amount, currency } = ctx.request.body  // validate input
  const result = await strapi.service('api::payment.payment').process(amount, currency)
  ctx.body = { data: result }  // format response
}
```

**Service responsibility (business logic only):**
```typescript
// services/payment.ts
async process(amount: number, currency: string) {
  // Business rules live here
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
[ ] Module's public interface has ≤ 5 methods

Open/Closed:
[ ] Adding new behavior requires adding code, not editing existing code
[ ] No if/switch on type that must be extended every time a new type is added

Liskov:
[ ] Swapping an implementation (Stripe → Razorpay, local → Cloudinary) doesn't change callers
[ ] Implementations are injected, not imported directly in business logic

Interface Segregation:
[ ] No module depends on methods it doesn't use
[ ] Read and write interfaces are separated where consumers are naturally split

Dependency Inversion:
[ ] Business logic imports interfaces, not concrete implementations
[ ] External SDKs (Stripe, Cloudinary, SendGrid) are isolated behind a service/plugin

Boundaries:
[ ] No circular dependencies (`npx madge --circular src/`)
[ ] Data flows one direction (controller → service → repository → DB)
[ ] Each module is testable with mocked dependencies
[ ] Module fits the size heuristic (< 200 lines, ≤ 5 public methods, understood in 2 min)
```

---

## OUTPUT FORMAT

```
## Module Review: [name]

### Responsibility Assessment
Current responsibilities: [list]
Should be: [list of separated modules if needed]
Verdict: ✓ Single responsibility / ✗ Split into [A] and [B]

### Coupling Assessment
Dependencies: [list what this module imports]
Which should be interfaces: [list concrete imports that should be abstracted]
Circular deps: ✓ None / ✗ [A → B → A]

### Interface Quality
Current public interface: [list public methods]
Verdict: ✓ Minimal / ✗ Too large — split into [ReadInterface] and [WriteInterface]

### Refactor Plan (if needed)
Step 1: [what to extract first — smallest, safest change]
Step 2: [next change]
Step 3: [verify with tests — run /tdd]
```
