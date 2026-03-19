---
name: add-usecase
description: "Generate a CQRS Use Case (Command/Query + Handler + Validator + DTO) for a DDD project using MassTransit Mediator. Use this skill whenever the user says /add-usecase, or asks to create a use case, add a command, add a query, create a handler, scaffold CQRS files, or build application-layer logic for an aggregate. Also trigger when the user describes a business operation they want to implement (e.g., 'I need to create an order', 'add update functionality for products') — even without mentioning CQRS or use case explicitly."
---

# Add Use Case (CQRS Command/Query)

Generate Application-layer CQRS files from a user's oral description, then optionally cascade changes to Domain Interface and Infrastructure Repository.

This is a **3-phase guided workflow** with confirmation at each phase:
1. Generate Application layer files
2. Identify and add missing Domain Interface methods
3. Implement those methods in Infrastructure Repository

## Phase 1: Generate Application Layer

### Gather context first

Before generating, read the project to understand:

1. **The target Aggregate Root** — read its `.cs` file to understand properties, factory methods, and domain methods (Create, Update, Delete, etc.)
2. **Existing use cases** — look at `Application/{Feature}/` to see naming conventions and existing commands/queries for this aggregate
3. **The Domain Repository Interface** — to know what methods already exist
4. **AppDbContext.cs** — to know available DbSets

### Determine the use case type

Based on the user's description, determine:

| User says | Type | Generate |
|---|---|---|
| "Create...", "Add...", "新增..." | Command | Command + Handler + Validator + Response |
| "Update...", "Edit...", "修改..." | Command | Command + Handler + Validator + Response |
| "Delete...", "Remove...", "刪除..." | Command | Command + Handler + Response |
| "Get...", "Query...", "List...", "查詢..." | Query | Query + Handler + DTO/Response |

### File structure to generate

```
Application/{Feature}/{UseCaseName}/
├── {Name}Command.cs (or {Name}Query.cs)
├── {Name}CommandHandler.cs (or {Name}QueryHandler.cs)
├── {Name}CommandValidator.cs (optional, for Commands with input validation)
└── {Name}Response.cs (or {Name}Dto.cs)
```

### Code patterns

#### Command (sealed record)

```csharp
namespace {AppNamespace}.{Feature}.{UseCaseName};

public sealed record {Name}Command(
    {PropertyType} {PropertyName},
    ...);

public sealed record {Name}Response(bool Success);
```

For update/delete commands, the first property is usually the entity ID.

#### Command Handler (MassTransit IConsumer)

```csharp
using MassTransit;
using {DomainNamespace};

namespace {AppNamespace}.{Feature}.{UseCaseName};

public sealed class {Name}CommandHandler : IConsumer<{Name}Command>
{
    private readonly I{Aggregate}Repository _{aggregateRepo};
    private readonly IUnitOfWork _unitOfWork;

    public {Name}CommandHandler(
        I{Aggregate}Repository {aggregateRepo},
        IUnitOfWork unitOfWork)
    {
        _{aggregateRepo} = {aggregateRepo};
        _unitOfWork = unitOfWork;
    }

    public async Task Consume(ConsumeContext<{Name}Command> context)
    {
        var command = context.Message;

        // Business logic here — call aggregate methods

        await _unitOfWork.SaveAsync(context.CancellationToken);
        await context.RespondAsync(new {Name}Response(true));
    }
}
```

**Handler logic varies by operation type:**

**Create**: Call aggregate's static factory method, then repository AddAsync
```csharp
var entity = {Aggregate}.Create(command.Prop1, command.Prop2, ...);
await _{repo}.AddAsync(entity, context.CancellationToken);
await _unitOfWork.SaveAsync(context.CancellationToken);
```

**Update**: Fetch from repository, call aggregate's Update method
```csharp
var entity = await _{repo}.GetAsync(command.Id, context.CancellationToken);
if (entity is null)
    throw new KeyNotFoundException($"{Aggregate} with id {command.Id} not found.");
entity.Update(command.Prop1, command.Prop2, ...);
await _unitOfWork.SaveAsync(context.CancellationToken);
```

**Delete**: Fetch from repository, call domain delete logic, then repository Delete
```csharp
var entity = await _{repo}.GetAsync(command.Id, context.CancellationToken);
if (entity is null)
    throw new KeyNotFoundException($"{Aggregate} with id {command.Id} not found.");
_{repo}.Delete(entity);
await _unitOfWork.SaveAsync(context.CancellationToken);
```

#### Command Validator (FluentValidation)

```csharp
using FluentValidation;

namespace {AppNamespace}.{Feature}.{UseCaseName};

public class {Name}CommandValidator : AbstractValidator<{Name}Command>, IValidator
{
    public {Name}CommandValidator()
    {
        RuleFor(x => x.PropertyName).NotEmpty();
        RuleFor(x => x.StringProp).NotEmpty().MaximumLength(30);
        RuleFor(x => x.IdProp).GreaterThan(0);
    }
}
```

Validation rules should match the Domain constraints (e.g., if the aggregate enforces MaxLength 30, the validator should too).

Only generate a validator if the command has properties that need validation. Simple commands like `RemoveOrderCommand(int OrderId)` might only need `GreaterThan(0)` — use judgment on whether a validator file is warranted.

#### Query (sealed record)

```csharp
namespace {AppNamespace}.{Feature}.{UseCaseName};

public sealed record {Name}Query({params if needed});
```

#### Query Handler

Queries use a **Read Service** (not the repository) for CQRS separation:

```csharp
using MassTransit;

namespace {AppNamespace}.{Feature}.{UseCaseName};

public sealed class {Name}QueryHandler : IConsumer<{Name}Query>
{
    private readonly I{Feature}ReadService _readService;

    public {Name}QueryHandler(I{Feature}ReadService readService)
    {
        _readService = readService;
    }

    public async Task Consume(ConsumeContext<{Name}Query> context)
    {
        var result = await _readService.{MethodName}(context.CancellationToken);
        await context.RespondAsync(result);
    }
}
```

#### DTO (for queries)

```csharp
namespace {AppNamespace}.{Feature}.{UseCaseName};

public sealed record {Name}Dto(
    {PropertyType} {PropertyName},
    ...);

public sealed record {Name}Response(IReadOnlyList<{Name}Dto> Items);
// OR for single item:
// The DTO itself is the response
```

### Present to user and wait for confirmation

Show all generated files with full code. Ask if the content looks right before writing.

---

## Phase 2: Update Domain Interface

After the Application layer is confirmed, analyze the Handler code:

1. Read the existing Domain Repository Interface (`I{Aggregate}Repository.cs`)
2. Check which repository methods the Handler calls
3. Identify any **missing methods** that need to be added

For example, if the Handler calls `_repo.GetByCustomerIdAsync(...)` but the interface doesn't have this method, propose adding it.

### Present changes

Show the user:
- Current interface content
- Proposed new method signatures to add
- Where the method will be inserted

Wait for confirmation before modifying the file.

If the Handler also needs a new **Read Service method** (for queries), propose adding it to `I{Feature}ReadService.cs` as well.

---

## Phase 3: Update Infrastructure Repository

After the Domain Interface is confirmed, generate the implementation:

1. Read the existing Infrastructure Repository (`{Aggregate}Repository.cs`)
2. For each new interface method, generate the EF Core implementation
3. Follow the same patterns documented in the add-repository skill:
   - Use `_appDbContext.{DbSetName}` for queries
   - Use `.Where().SingleOrDefaultAsync()` pattern
   - Cast Value Object IDs in LINQ: `((Guid)x.CompanyId)`
   - Populate computed/Ignored properties after loading

### Present changes

Show the user:
- The new method implementations to be added
- Where they'll be inserted in the existing file

Wait for confirmation before modifying.

If a Read Service implementation is needed, also generate and present it.

---

## Phase 4: Remind about DI registration

After all files are written, check if any new services need DI registration:
- New Read Service interface → `services.AddScoped<I{Feature}ReadService, {Feature}ReadService>()`
- New Repository (unlikely at this stage but possible)

Point out the DI registration file location.

---

## Example walkthrough

**User says**: "我需要一個 CreateOrder 的功能，輸入客戶 ID 和金額，建立訂單"

**Phase 1** — Generate:
- `Application/OrderManager/CreateOrder/CreateOrderCommand.cs`
- `Application/OrderManager/CreateOrder/CreateOrderCommandHandler.cs`
- `Application/OrderManager/CreateOrder/CreateOrderCommandValidator.cs`
- `Application/OrderManager/CreateOrder/CreateOrderResponse.cs`

→ User confirms ✓

**Phase 2** — Handler calls `_orderRepo.AddAsync()` — check `IOrderRepository`:
- `AddAsync` already exists → no change needed
- Handler also needs `GetNextOrderNumberAsync()` → propose adding to interface

→ User confirms ✓

**Phase 3** — Implement `GetNextOrderNumberAsync()` in `OrderRepository.cs`

→ User confirms ✓

**Phase 4** — Remind: "Repository already registered. No DI changes needed."
