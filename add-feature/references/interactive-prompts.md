# Interactive Discovery & Clarification Prompts

When invoked to add a feature to an ASP.NET Core solution, you must **actively verify and prompt the user** for any missing architectural details before generating code.

---

## 🎯 1. Interactive Discovery Flowchart

```text
┌────────────────────────────────────────────────────────┐
│ 1. Analyze User Request & Existing Workspace Context    │
└───────────────────────────┬────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────┐
│ 2. Check for Missing Critical Feature Specifications:   │
│    - Feature / Use Case Name & Intent                  │
│    - Database Table Name & Schema Definition           │
│    - Operation Type (Command vs Query vs Full Slice)   │
│    - Business Rules, Invariants & Validation           │
│    - API Route & HTTP Method                           │
└───────────────────────────┬────────────────────────────┘
                            │
          ┌─────────────────┴─────────────────┐
          │ Any critical detail missing?      │
          ├─────────────────┬─────────────────┤
          │       YES       │       NO        │
          ▼                 ▼                 ▼
┌──────────────────┐               ┌──────────────────┐
│ Prompt User with │               │ Proceed to       │
│ Targeted Form    │               │ Implementation   │
└──────────────────┘               └──────────────────┘
```

---

## 📋 2. Core Clarification Questionnaire

Use the structured format below to gather missing information from the user:

### A. Feature Identification & Intent
1. **Feature Name**: What is the name of the feature / usecase? (e.g., `CreateOrder`, `CancelSubscription`, `GetProductById`, `ListActiveUsers`)
2. **Feature Group / Aggregate**: Which aggregate or domain module does this belong to? (e.g., `Orders`, `Customers`, `Billing`, `Catalog`)
3. **CQRS Type**:
   - **Command** (State mutating: Creates, Updates, Deletes, Transitions)
   - **Query** (Read-only: Projections, Search, Single Item, Paginated List)
   - **Full Aggregate Slice** (Entity + Create Command + Get Query + EF Mapping + Dapper Read)

### B. Database Schema & Table Details
1. **Target Table Name**: What is the exact database table name? (e.g., `dbo.Orders`, `billing.Invoices`, `catalog.Products`)
2. **Primary Key Strategy**:
   - `Guid` (Default, generated via `NEWID()` or client-side `Guid.NewGuid()`)
   - `int` / `bigint` (`IDENTITY(1,1)`)
   - Composite Key
3. **Columns & Data Types**:
   - Column names, C# data types, SQL data types (e.g., `nvarchar(200)`, `decimal(18,2)`, `datetimeoffset`)
   - Nullability (`NOT NULL` vs `NULL`)
   - Max lengths and string constraints
4. **Foreign Keys & Relationships**:
   - Parent table / child tables (e.g., `CustomerId` referencing `Customers(Id)`)
   - Navigation properties and cascading behavior

### C. Business Rules & Validation
1. **Invariants & Domain Logic**:
   - What business invariants must the Domain entity protect? (e.g., "Cannot cancel an order that has already shipped", "Price cannot be negative")
2. **Validation Rules (FluentValidation 11.6.0)**:
   - Required fields, max lengths, email formats, numerical ranges, regex patterns.
3. **Domain Errors (`*Errors.cs`)**:
   - What error codes and descriptions should be returned on business failures? (e.g., `OrderErrors.NotFound(id)`, `OrderErrors.AlreadyCompleted`)

### D. Transport & API Specification
1. **Endpoint Route**: What is the RESTful endpoint URI? (e.g., `POST /api/v1/orders`, `GET /api/v1/orders/{id}`)
2. **HTTP Method**: `POST`, `GET`, `PUT`, `PATCH`, `DELETE`
3. **Status Codes**:
   - Success: `201 Created` with `CreatedAtRoute`, `200 OK` with payload, `204 NoContent`
   - Validation failure: `400 Bad Request` / `ValidationProblem`
   - Business failure: `404 Not Found`, `409 Conflict`, `422 Unprocessable Entity` via `ProblemDetails`

---

## 💬 3. Example Prompt Template for Agents

When prompting the user in chat, present the questions clearly:

```markdown
To scaffold the feature cleanly into your ASP.NET Core solution adhering to our Clean Architecture and CQRS standards, please confirm or provide the following details:

1. **Database Table Name**: What is the exact table name (and schema) in SQL Server? (e.g., `dbo.SupportTickets`)
2. **Columns & Types**: What fields make up this table? (e.g., `Id (Guid, PK)`, `Title (nvarchar(200))`, `Description (nvarchar(max))`, `Status (nvarchar(50))`, `CreatedAtUtc (datetimeoffset)`)
3. **Use Case / Feature Details**:
   - What specific operation are we building? (e.g., `CreateSupportTicketCommand` or `GetTicketByIdQuery`)
   - What business validation rules or domain invariants should apply?
4. **API Endpoint Route**: What should the Minimal API route and method be? (e.g., `POST /api/tickets`)
```
