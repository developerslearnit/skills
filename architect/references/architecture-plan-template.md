# Architecture Blueprint & Implementation Plan Template

Use this template to generate the final output at the end of an architecture thinking session.

---

## 📋 Implementation Blueprint Template

```markdown
# Architecture Blueprint — [Feature Name]

## 1. Executive Summary
[A concise paragraph summarizing what we are building, the problem it solves, and the business/technical value.]

## 2. Ubiquitous Language & Domain Terms
- **[Term 1]**: [Agreed definition and scope]
- **[Term 2]**: [Agreed definition and scope]
- **[Term 3]**: [Agreed definition and scope]

## 3. Key Architectural Decisions & Rationale
| Decision | Chosen Approach | Rationale & Trade-offs |
| :--- | :--- | :--- |
| **[e.g., Persistence]** | EF Core for writes, Dapper for reads | High write integrity with sub-millisecond query performance |
| **[e.g., Dispatching]** | In-process `IDispatcher` | Avoids heavy distributed dependencies while preserving CQRS decoupling |
| **[e.g., Error Handling]** | Result Pattern (`Result<T>`) | Deterministic, explicit domain error returns without exception overhead |

## 4. Invariants, Rules & Constraints
- **Invariant 1**: [e.g., A subscription cannot be activated without an active payment method.]
- **Invariant 2**: [e.g., Ticket titles must be between 5 and 200 characters.]
- **Constraint**: [e.g., Must support SQL Server 2022 compatibility level.]

## 5. System & Component Map
```text
[Api Endpoint / Trigger]
        │
        ▼
[Application Command / Query Handler]
   ├── [Domain Entity / Invariant Protection]
   └── [Repository Interface (Domain)]
              │
              ▼
[Infrastructure Persistence Implementation (EF / Dapper)]
```

## 6. Assumptions & Non-Goals
### Assumptions
- [Assumption 1]
- [Assumption 2]

### Non-Goals (Out of Scope for this iteration)
- [Explicitly excluded item 1]
- [Explicitly excluded item 2]

## 7. Ordered Implementation Sequence
1. **Domain Layer**:
   - [ ] Define Entity / Aggregate root with invariant factory methods.
   - [ ] Define domain error definitions (`*Errors.cs`).
   - [ ] Define repository interfaces.
2. **Application Layer**:
   - [ ] Define Command/Query records and DTOs.
   - [ ] Implement FluentValidation validator.
   - [ ] Implement Command/Query handler orchestrating domain logic.
3. **Infrastructure Layer**:
   - [ ] Create EF Core entity configuration and table mapping.
   - [ ] Implement EF Core write repository.
   - [ ] Implement Dapper read repository with parameterized SQL.
   - [ ] Register services in DI container.
4. **API Layer**:
   - [ ] Wire Minimal API endpoint and route group.
   - [ ] Map Result outcomes to ProblemDetails / HTTP status codes.
5. **Verification & Tests**:
   - [ ] Unit tests for Domain and Application handlers.
   - [ ] End-to-end build and test execution.
```
