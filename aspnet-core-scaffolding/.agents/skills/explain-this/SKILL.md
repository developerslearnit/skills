---
name: explain-this
description: Analyzes and explains what a specific piece of code, method, class, file, or architectural component does. Provides structured, clear breakdowns covering intent, execution flow, design patterns, dependencies, error handling, edge cases, and performance considerations. Use whenever asked to explain, dissect, understand, review, or walk through code.
metadata:
  author: Adesina Mark Omoniyi
  version: "1.0"
  tags:
    - code-explanation
    - architecture-analysis
    - code-review
    - csharp
    - dotnet
    - clean-architecture
---

# Explain This Code

## Purpose

Use this skill whenever you need to analyze, explain, or teach what a specific code snippet, function, class, file, or architecture component does.

The goal is to deliver an explanation that is **crystal clear, technically accurate, context-aware, and structured for fast comprehension**.

---

## Explanation Framework

When explaining code, follow this standardized multi-layer structure:

```text
┌────────────────────────────────────────────────────────┐
│ 1. Executive Summary (1-2 sentences: purpose & outcome)│
├────────────────────────────────────────────────────────┤
│ 2. Signature & Contract (Inputs, Output, Side-effects) │
├────────────────────────────────────────────────────────┤
│ 3. Step-by-Step Execution Walkthrough                  │
├────────────────────────────────────────────────────────┤
│ 4. Design Patterns & Architectural Idioms              │
├────────────────────────────────────────────────────────┤
│ 5. Error Handling & Edge Cases                         │
├────────────────────────────────────────────────────────┤
│ 6. Performance & Concurrency Considerations            │
├────────────────────────────────────────────────────────┤
│ 7. Practical Usage Example                             │
└────────────────────────────────────────────────────────┘
```

---

## Section Guidelines

### 1. Executive Summary
- State **what** the code does and **why** it exists in 1–2 plain English sentences.
- Avoid technical jargon in the first sentence; give the big picture immediately.

### 2. Signature & Contract
- **Inputs**: What parameters are passed, what are their constraints (e.g., required, nullable, default values)?
- **Output / Return**: What is returned on success? (e.g., `Result<TValue>`, `Task<T>`, DTO).
- **Side Effects**: Does it mutate database state, publish events, modify external resources, or write to logs?

### 3. Step-by-Step Execution Walkthrough
- Break down the execution flow in chronological order.
- Group logical steps (e.g., Step 1: Input Validation & Preconditions, Step 2: Domain Entity Construction, Step 3: Persistence & Commit).
- Use concise bullet points with short code snippets illustrating critical transitions.

### 4. Design Patterns & Architectural Idioms
Highlight recognized design patterns and explain why they are used:
- **Result Pattern**: Modeling failure as values rather than exceptions.
- **CQRS / Dispatcher**: Separating state-mutation (Commands) from read operations (Queries).
- **Domain-Driven Design (DDD)**: Aggregate roots protecting invariants, Value Objects ensuring encapsulation.
- **Unit of Work & Repository**: Transaction boundaries and decoupled persistence.
- **Dependency Injection**: Decoupling caller from concrete implementations.

### 5. Error Handling & Edge Cases
Identify how edge cases and potential failure modes are handled:
- What happens if an input is invalid, `null`, or out of range?
- How are business rule violations communicated (e.g., `Result.Failure(Error.NotFound)`)?
- How is cancellation handled (`CancellationToken` propagation)?
- Are database deadlocks, unique constraint violations, or network timeouts accounted for?

### 6. Performance & Concurrency Considerations
- **Asynchronous I/O**: Proper use of `async`/`await` without thread blocking (`.Result` / `.Wait()`).
- **Memory Allocations**: Value types, `readonly struct`, `IReadOnlyList<T>`, string allocations.
- **Database Access**: EF Core Change Tracking (`AsNoTracking` for queries), Dapper parameterization, N+1 query avoidance.

### 7. Practical Usage Example
Provide a clean, idiomatic caller snippet demonstrating how the code is invoked in a real-world scenario (e.g., from an API endpoint, background service, or unit test).

---

## Pattern-Specific Explanations

### Explaining CQRS Command Handlers
When explaining a command handler (e.g., `ICommandHandler<TCommand, TResponse>`):
1. **Command Intent**: What business action is triggered?
2. **Invariants Enforced**: What domain rules are validated prior to state changes?
3. **Aggregate Interaction**: How the domain aggregate is created or mutated.
4. **Persistence & Transaction**: How the repository and Unit of Work commit changes.
5. **Returned Outcome**: The strongly typed `Result<TValue>` returned to the caller.

### Explaining CQRS Query Handlers
When explaining a query handler (e.g., `IQueryHandler<TQuery, TResponse>`):
1. **Query Purpose**: What projection or read model is retrieved?
2. **Data Source & Mechanism**: Dapper SQL query, stored procedure, or EF read.
3. **Nullability & Not Found**: How missing data is represented (`Result.Failure(Error.NotFound)`).
4. **Pagination & Filtering**: How search parameters and limits are applied.

### Explaining Domain Entities & Value Objects
When explaining domain models:
1. **Identity & Lifecycle**: Is it an Entity (tracked by ID) or Value Object (immutable, compared by value)?
2. **Encapsulation**: How private constructors and factory methods (`Create(...)`) protect invariants.
3. **Domain Events**: What events are recorded during mutation.

---

## Output Quality Rules

1. **Be Concise Yet Thorough**: Avoid fluff; focus on actionable clarity.
2. **Format with Markdown**: Use bolding, syntax-highlighted code blocks, tables, and alert callouts (`> [!NOTE]`, `> [!TIP]`, `> [!IMPORTANT]`).
3. **Use Visual Flowcharts**: When control flow has branching logic or multi-service orchestration, include a Mermaid sequence or flowchart diagram.
4. **Highlight Non-Obvious Gotchas**: Point out potential traps (e.g., missing cancellation tokens, silent exceptions, race conditions).
5. **Always Reference Exact Types and Line Numbers**: Use markdown links to the file and symbol names where applicable.

---

## Reference Documents

- [Explanation Patterns & Examples](references/explanation-patterns.md)
