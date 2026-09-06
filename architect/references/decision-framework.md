# Senior Engineering Decision Framework

As an architect or senior engineer, your role is to identify and resolve decisions that have **high downstream impact** while deliberately deferring minor implementation details.

---

## 🧭 1. The Decision Hierarchy

Decisions are not all equal. Evaluate them in order of gravity:

```text
┌─────────────────────────────────────────────────────────────┐
│ 1. System & Module Boundaries                               │
│    (Where does this live? Who owns this state? Coupling)    │
├─────────────────────────────────────────────────────────────┤
│ 2. Data & Persistence Model                                 │
│    (Single source of truth, Consistency, Read vs Write)     │
├─────────────────────────────────────────────────────────────┤
│ 3. Interfaces & Contracts                                   │
│    (Sync vs Async, REST, gRPC, Commands vs Events, Result)  │
├─────────────────────────────────────────────────────────────┤
│ 4. Error Handling & Invariant Enforcement                   │
│    (Domain invariants, Fallback strategies, RFC 7807)       │
├─────────────────────────────────────────────────────────────┤
│ 5. Cross-Cutting Concerns                                   │
│    (Auth, Observability, Concurrency, Caching, Rate limits) │
└─────────────────────────────────────────────────────────────┘
```

---

## ⚖️ 2. Trade-Off Evaluation Matrix

Whenever proposing an architectural decision, evaluate the options using the **Trilemma Matrix**:

| Dimension | Option A (e.g., Simple / Direct) | Option B (e.g., Decoupled / Event-Driven) | Option C (e.g., Highly Optimized) |
| :--- | :--- | :--- | :--- |
| **Simplicity & Time to Ship** | High (few moving parts) | Medium (orchestration needed) | Low (complex infrastructure) |
| **Flexibility & Extensibility** | Low (tighter coupling) | High (isolated changes) | Medium |
| **Operational & Mental Overhead** | Very Low | Medium (monitoring, eventual consistency) | High (cache invalidation, tuning) |
| **Recommended When...** | Scope is narrow, invariants local | Multiple bounded contexts communicate | High-throughput, proven bottleneck |

### Senior Engineer Rule of Thumb:
> *Always recommend the simplest solution that honestly satisfies the known requirements, while keeping the door open for future evolution.*

---

## 🔍 3. Common Architectural Decision Categories

### A. State Ownership & Persistence
- **Write Consistency**: Does this need immediate consistency (ACID transaction within a single aggregate) or eventual consistency across aggregates?
- **Separation of Concerns**: Is this purely a read optimization (CQRS / Dapper projection) or a state mutation (EF Core / Aggregate)?
- **Soft Delete vs Hard Delete vs State Machine**: Does data get erased, archived, or transitioned through explicit domain states (e.g., `Draft -> Submitted -> Approved -> Cancelled`)?

### B. Communication & Dispatching
- **Synchronous vs Asynchronous**: Does the user need the result immediately in the HTTP request (Minimal API + in-process `IDispatcher`), or should it be offloaded to a background queue / worker?
- **In-Process vs Distributed**: Is an in-process mediator sufficient, or is distributed message bus (e.g., RabbitMQ, Azure Service Bus) justified by independent scaling requirements?

### C. Error & Edge Case Strategy
- **Expected Business Failures**: Modeled explicitly via the **Result Pattern** (`Result<T>`, domain `Error`) — no exceptions for control flow.
- **Catastrophic / Infrastructure Failures**: Handled via global exception middleware / ProblemDetails.
- **Concurrency Conflicts**: Optimistic concurrency (`RowVersion` / concurrency token) vs pessimistic locking.

---

## 🎯 4. Anti-Patterns to Avoid

- **Analysis Paralysis**: Asking 20 theoretical questions before writing a single line of code. Stop once high-impact decisions are resolved.
- **The Blank Page Trap**: Asking "How do you want to do error handling?" without offering an opinionated recommendation and rationale.
- **Premature Optimization**: Designing microservices or multi-tier caching for a feature with 10 requests a day.
- **Prescribing Implementation Minutiae**: Debating private variable naming or loop constructs during the architectural design phase.
