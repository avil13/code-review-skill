# Architecture Review Guide

A guide for reviewing architecture and design: whether structure is sound and decisions are appropriate.

## SOLID checklist

### S — Single Responsibility Principle (SRP)

**What to check:**
- Does this class/module have only one reason to change?
- Do its methods all serve one coherent purpose?
- Could you explain what the class does in one sentence to a non-developer?

**Signals during review:**
```
⚠️ Class names with vague words like "And", "Manager", "Handler", "Processor"
⚠️ A class longer than ~200–300 lines
⚠️ More than ~5–7 public methods
⚠️ Methods operating on unrelated data
```

**Review prompts:**
- "What is this class responsible for? Could it be split?"
- "If requirement X changes, which methods change? What if Y changes?"

### O — Open/Closed Principle (OCP)

**What to check:**
- Does adding a feature require editing existing code?
- Can new behavior be added by extension (inheritance, composition)?
- Are there large if/else or switch chains for different types?

**Signals during review:**
```
⚠️ Long switch/if-else chains on type
⚠️ New features require edits to core classes
⚠️ Type checks (instanceof, typeof) scattered through the code
```

**Review prompts:**
- "If we add a new type X, which files change?"
- "Will this switch grow every time we add a type?"

### L — Liskov Substitution Principle (LSP)

**What to check:**
- Can subclasses fully replace their base type?
- Do subclasses change expectations of base methods?
- Do subclasses throw exceptions the base did not document?

**Signals during review:**
```
⚠️ Explicit downcasts
⚠️ Subclass methods throw NotImplementedException
⚠️ Empty implementations or methods that only return
⚠️ Call sites that must know the concrete subtype
```

**Review prompts:**
- "If we use the subclass everywhere we used the base, do callers still work?"
- "Does this override honor the base type’s contract?"

### I — Interface Segregation Principle (ISP)

**What to check:**
- Are interfaces small and focused?
- Are implementors forced to support methods they don’t need?
- Do clients depend on methods they never call?

**Signals during review:**
```
⚠️ Interfaces with more than ~5–7 methods
⚠️ Implementations with empty methods or NotImplementedException
⚠️ Vague names (IManager, IService)
⚠️ Different clients each use only part of the interface
```

**Review prompts:**
- "Does every implementation use every method on this interface?"
- "Could we split this into smaller, role-specific interfaces?"

### D — Dependency Inversion Principle (DIP)

**What to check:**
- Do high-level modules depend on abstractions, not concrete details?
- Is dependency injection used instead of `new` everywhere?
- Are abstractions owned by high-level policy, not low-level details?

**Signals during review:**
```
⚠️ High-level code news concrete low-level classes
⚠️ Imports of concrete implementations instead of interfaces/abstract types
⚠️ Config and connection strings embedded in business logic
⚠️ Hard to unit-test a class in isolation
```

**Review prompts:**
- "Can dependencies be mocked in tests?"
- "If we swap database or API implementation, how many places change?"

---

## Architectural anti-patterns

### Critical anti-patterns

| Anti-pattern | Signals | Impact |
|--------------|---------|--------|
| **Big ball of mud** | No clear module boundaries; anything may call anything | Hard to understand, change, and test |
| **God object** | One class does too much; knows and touches everything | Tight coupling; reuse and testing suffer |
| **Spaghetti code** | Control flow is tangled (goto, deep nesting); hard to follow | Hard to understand and maintain |
| **Lava flow** | Old code nobody dares touch; little docs or tests | Technical debt accumulates |

### Design anti-patterns

| Anti-pattern | Signals | Suggestion |
|--------------|---------|------------|
| **Golden hammer** | Same tool/pattern for every problem | Match the solution to the problem |
| **Over-engineering** | Complex solution for a simple problem; pattern abuse | YAGNI: start simple, add complexity when needed |
| **Boat anchor** | Unused code written “for later” | Remove dead code; add it when you need it |
| **Copy-paste programming** | Same logic in many places | Extract shared helpers or modules |

### Sample review comments

```markdown
🔴 [blocking] "This class is ~2000 lines; consider splitting into focused types"
🟡 [important] "This logic is duplicated in three places—extract a shared helper?"
💡 [suggestion] "This switch could become a strategy map for easier extension"
```

---

## Coupling and cohesion

### Coupling types (best to worst)

| Type | Description | Example |
|------|-------------|---------|
| **Message coupling** ✅ | Data passed via parameters | `calculate(price, quantity)` |
| **Data coupling** ✅ | Share simple data structures | `processOrder(orderDTO)` |
| **Stamp coupling** ⚠️ | Share a large structure but use only part of it | Pass full `User` but only read `name` |
| **Control coupling** ⚠️ | Flags change callee behavior | `process(data, isAdmin=true)` |
| **Common coupling** ❌ | Shared global state | Many modules read/write the same globals |
| **Content coupling** ❌ | Reach into another module’s internals | Mutate another class’s private fields |

### Cohesion types (best to worst)

| Type | Description | Quality |
|------|-------------|---------|
| **Functional** | Elements do one task together | ✅ Best |
| **Sequential** | Output of one step feeds the next | ✅ Good |
| **Communicational** | Operations share the same data | ⚠️ Acceptable |
| **Temporal** | Things run at the same time | ⚠️ Weaker |
| **Logical** | Related in idea but not in function | ❌ Poor |
| **Coincidental** | No real relationship | ❌ Worst |

### Reference metrics

```yaml
Coupling:
  CBO (coupling between objects):
    good: < 5
    warning: 5-10
    risky: > 10

  Ce (efferent coupling):
    description: how many external types this type depends on
    good: < 7

  Ca (afferent coupling):
    description: how many types depend on this one
    high_values_mean: large blast radius; needs a stable API

Cohesion:
  LCOM4 (lack of cohesion of methods):
    "1": single responsibility ✅
    "2-3": consider splitting ⚠️
    "> 3": should split ❌
```

### Review prompts

- "How many other modules does this one depend on? Can we reduce that?"
- "How many places break if we change this class?"
- "Do these methods all relate to the same data?"

---

## Layered architecture

### Clean Architecture layers

```
┌─────────────────────────────────────┐
│         Frameworks & Drivers        │ ← Outermost: Web, DB, UI
├─────────────────────────────────────┤
│         Interface Adapters          │ ← Controllers, gateways, presenters
├─────────────────────────────────────┤
│          Application Layer          │ ← Use cases, application services
├─────────────────────────────────────┤
│            Domain Layer             │ ← Entities, domain services
└─────────────────────────────────────┘
          ↑ Dependencies point inward ↑
```

### Dependency rule

**Rule: source dependencies must point inward only**

```typescript
// ❌ Breaks the rule: Domain depends on Infrastructure
// domain/User.ts
import { MySQLConnection } from '../infrastructure/database';

// ✅ Good: Domain defines ports; Infrastructure implements
// domain/UserRepository.ts (interface)
interface UserRepository {
  findById(id: string): Promise<User>;
}

// infrastructure/MySQLUserRepository.ts (implementation)
class MySQLUserRepository implements UserRepository {
  findById(id: string): Promise<User> { /* ... */ }
}
```

### Review checklist

**Layer boundaries:**
- [ ] Does the domain layer depend on externals (DB, HTTP, filesystem)?
- [ ] Does the application layer talk to the database or external APIs directly?
- [ ] Does the controller contain business rules?
- [ ] Any cross-layer shortcuts (UI → repository)?

**Separation of concerns:**
- [ ] Is business logic separate from presentation?
- [ ] Is data access behind a dedicated boundary?
- [ ] Are config and environment concerns centralized?

### Sample review comments

```markdown
🔴 [blocking] "Domain entity imports a DB connection—violates the dependency rule"
🟡 [important] "Controller holds business calculations; move to a service layer"
💡 [suggestion] "Consider DI to decouple these components"
```

---

## Design patterns

### When patterns help

| Pattern | Good fit | Poor fit |
|---------|----------|----------|
| **Factory** | Many product types; type chosen at runtime | Single fixed type |
| **Strategy** | Swappable algorithms at runtime | One algorithm, never changes |
| **Observer** | One-to-many updates on state change | A direct call is enough |
| **Singleton** | True process-wide singleton (e.g. config) | Something you could inject instead |
| **Decorator** | Add behavior dynamically; avoid subclass explosion | Fixed responsibilities, no composition need |

### Over-engineering signals

```
⚠️ "Patternitis" signals:

1. Simple if/else replaced with strategy + factory + registry
2. Interface with only one implementation
3. Abstraction "for someday" with no current need
4. Line count jumped mainly because of pattern plumbing
5. New contributors need a long ramp to follow the structure
```

### Review principles

```markdown
✅ Patterns used well:
- Solve a real extensibility or clarity problem
- Code is easier to read and test
- Adding features gets simpler

❌ Pattern overuse:
- Using patterns for their own sake
- Extra complexity without payoff
- Violates YAGNI
```

### Review prompts

- "What concrete problem does this pattern solve?"
- "What breaks if we don’t use it?"
- "Is the abstraction worth its complexity?"

---

## Extensibility

### Extensibility checklist

**Feature extensibility:**
- [ ] Does a new feature require changing core code?
- [ ] Are there extension points (hooks, plugins, events)?
- [ ] Is configuration externalized (files, env vars)?

**Data extensibility:**
- [ ] Can the model evolve (new fields, versions)?
- [ ] Is growth in data volume considered?
- [ ] Are queries indexed appropriately?

**Load extensibility:**
- [ ] Can we scale out (more instances)?
- [ ] Is there sticky state (session, local-only cache)?
- [ ] Are DB connections pooled?

### Extension-point design

```typescript
// ✅ Good: events / hooks
class OrderService {
  private hooks: OrderHooks;

  async createOrder(order: Order) {
    await this.hooks.beforeCreate?.(order);
    const result = await this.save(order);
    await this.hooks.afterCreate?.(result);
    return result;
  }
}

// ❌ Poor: everything hard-coded
class OrderService {
  async createOrder(order: Order) {
    await this.sendEmail(order);        // hard-coded
    await this.updateInventory(order);  // hard-coded
    await this.notifyWarehouse(order);  // hard-coded
    return await this.save(order);
  }
}
```

### Sample review comments

```markdown
💡 [suggestion] "If we add another payment method later, is this design easy to extend?"
🟡 [important] "This behavior is hard-coded—consider config or a strategy"
📚 [learning] "An event-driven style could make this feature easier to extend"
```

---

## Code structure

### Directory layout

**By feature/domain (preferred):**
```
src/
├── user/
│   ├── User.ts           (entity)
│   ├── UserService.ts    (service)
│   ├── UserRepository.ts (data access)
│   └── UserController.ts (API)
├── order/
│   ├── Order.ts
│   ├── OrderService.ts
│   └── ...
└── shared/
    ├── utils/
    └── types/
```

**By technical layer (less ideal):**
```
src/
├── controllers/     ← mixes domains
│   ├── UserController.ts
│   └── OrderController.ts
├── services/
├── repositories/
└── models/
```

### Naming

| Kind | Convention | Examples |
|------|------------|----------|
| Classes | PascalCase, noun | `UserService`, `OrderRepository` |
| Methods | camelCase, verb | `createUser`, `findOrderById` |
| Interfaces | `I` prefix or none | `IUserService` or `UserService` |
| Constants | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |
| Private fields | Leading `_` or `#` | `_cache` or `#cache` |

### File size guidelines

```yaml
Suggested_limits:
  per_file: < 300 lines
  per_function: < 50 lines
  per_class: < 200 lines
  function_parameters: < 4
  nesting_depth: < 4 levels

When_over_limits:
  - Split into smaller units
  - Prefer composition over inheritance
  - Extract helpers or types
```

### Sample review comments

```markdown
🟢 [nit] "This 500-line file could be split by responsibility"
🟡 [important] "Prefer feature-based folders over technical layers"
💡 [suggestion] "`process` is vague—maybe `calculateOrderTotal`?"
```

---

## Quick reference

### Five-minute architecture pass

```markdown
□ Do dependencies flow the right way (outer → inner)?
□ Any circular dependencies?
□ Is core logic free of framework/UI/DB details?
□ SOLID respected in the changed areas?
□ Obvious anti-patterns?
```

### Red flags (address)

```markdown
🔴 God object — one class over ~1000 lines
🔴 Cycle — A → B → C → A
🔴 Domain layer depends on a framework
🔴 Hard-coded secrets or environment-specific values
🔴 External services with no seam for testing/mocking
```

### Yellow flags (consider fixing)

```markdown
🟡 CBO > 10
🟡 More than five parameters on a method
🟡 Nesting deeper than four levels
🟡 Duplicated block over ~10 lines
🟡 Interface with a single implementation
```

---

## Tooling

| Tool | Use | Languages |
|------|-----|-----------|
| **SonarQube** | Quality, coupling | Many |
| **NDepend** | Dependencies, architecture rules | .NET |
| **JDepend** | Package dependencies | Java |
| **Madge** | Module dependency graphs | JavaScript/TypeScript |
| **ESLint** | Style, complexity | JavaScript/TypeScript |
| **CodeScene** | Hotspots, technical debt | Many |

---

## References

- [Clean Architecture - Uncle Bob](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [SOLID Principles in Code Review - JetBrains](https://blog.jetbrains.com/upsource/2015/08/31/what-to-look-for-in-a-code-review-solid-principles-2/)
- [Software Architecture Anti-Patterns](https://medium.com/@christophnissle/anti-patterns-in-software-architecture-3c8970c9c4f5)
- [Coupling and Cohesion in System Design](https://www.geeksforgeeks.org/system-design/coupling-and-cohesion-in-system-design/)
- [Design Patterns - Refactoring Guru](https://refactoring.guru/design-patterns)
