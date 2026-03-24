---
name: 9pfms:add-ddd-usecase
description: "Generate a Wolverine HTTP Endpoint (Command/Query + Validator + DTO) for a DDD project. Use this skill whenever the user says /9pfms:add-ddd-usecase, or asks to create a use case, add a command, add a query, create an endpoint, scaffold CQRS files, or build application-layer logic for an aggregate. Also trigger when the user describes a business operation they want to implement (e.g., 'I need to create an order', 'add update functionality for products') — even without mentioning CQRS or use case explicitly."
---

# Add Use Case (Wolverine HTTP Endpoint)

Generate Application-layer Wolverine HTTP Endpoint files from a user's oral description, then optionally cascade changes to Domain Interface and Infrastructure Repository.

This is a **3-phase guided workflow** with confirmation at each phase:
1. Generate Application layer files
2. Identify and add missing Domain Interface methods
3. Implement those methods in Infrastructure Repository

## Wolverine Endpoint Architecture

The project uses **Wolverine HTTP Endpoints** (not MassTransit Mediator):

- Each use case is a **static class** with a **static `Handle` method**
- Command, Validator, DTO, and Endpoint live in the **same file** (one file per endpoint)
- All files for a feature are **flat** in `Application/{Feature}/` (no sub-folders per use case)
- Wolverine **auto-manages transactions** — no `IUnitOfWork` needed for Commands
- Dependencies are **directly injected** as `Handle` method parameters
- Return types use `IResult` (`Results.Ok()`, `Results.Created()`, etc.)
- Query endpoints use `[NonTransactional]` attribute
- Read-side queries use `I{Feature}QueryService` (not `ReadService`)

## Phase 1: Generate Application Layer

### Gather context first

Before generating, read the project to understand:

1. **The target Aggregate Root** — read its `.cs` file to understand properties, factory methods, and domain methods (Create, Update, Delete, etc.)
2. **Existing endpoints** — look at `Application/{Feature}/` to see naming conventions and existing endpoints for this aggregate
3. **The Domain Repository Interface** — to know what methods already exist
4. **AppDbContext.cs** — to know available DbSets

### Determine the use case type

Based on the user's description, determine:

| User says | Type | HTTP Method | Generate |
|---|---|---|---|
| "Create...", "Add...", "新增..." | Command | POST | Command + Validator + Endpoint |
| "Update...", "Edit...", "修改..." | Command | PUT | Command + Validator + Endpoint |
| "Delete...", "Remove...", "刪除..." | Command | DELETE | Endpoint (param from route) |
| "Get...", "查詢..." (single) | Query | GET | DTO + Endpoint |
| "Browse...", "List...", "列表..." | Query | GET | DTO + Endpoint |
| "Link...", "Bind...", "綁定..." | Command | POST | Command + Endpoint |
| "Unlink...", "Unbind...", "解除..." | Command | DELETE | Command + Endpoint |

### File naming convention

Each endpoint is **one file** named `{UseCaseName}Endpoint.cs`:
```
Application/{Feature}/
├── CreateMenuEndpoint.cs       (Command + Validator + Endpoint)
├── UpdateMenuEndpoint.cs       (Command + Validator + Endpoint)
├── DeleteMenuEndpoint.cs       (Endpoint only, ID from route)
├── GetMenuEndpoint.cs          (DTO + Endpoint)
├── BrowseMenuEndpoint.cs       (DTO + Endpoint)
├── LinkinMenuActionEndpoint.cs (Command + Endpoint)
├── I{Feature}QueryService.cs   (Query service interface)
```

### Code patterns

#### Command Endpoint (Create)

```csharp
using FluentValidation;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Wolverine.Http;
using {DomainNamespace};
using {DomainNamespace}.Aggregates.{AggregateFolder};

namespace {AppNamespace}.{Feature};

/// <summary>
/// {Description}參數
/// </summary>
/// <param name="{Param1}">{Param1Description}</param>
/// <param name="{Param2}">{Param2Description}</param>
public sealed record {Name}Command(
    {PropertyType} {PropertyName},
    ...);

/// <summary>
/// <see cref="{Name}Command"/> 驗證器
/// </summary>
public class {Name}CommandValidator : AbstractValidator<{Name}Command>
{
    public {Name}CommandValidator()
    {
        RuleFor(x => x.StringProp).NotEmpty().MaximumLength(30);
        // Match Domain constraints
    }
}

/// <summary>
/// {Description}
/// </summary>
public static class {Name}Endpoint
{
    /// <summary>
    /// {Description}
    /// </summary>
    [Tags("{TagName}")]
    [WolverinePost("/api/{route}")]
    [ProducesResponseType(StatusCodes.Status201Created)]
    public static async Task<IResult> Handle(
        {Name}Command command,
        I{Aggregate}Repository {aggregateRepo},
        IDateTimeProvider dateTimeProvider,
        CancellationToken cancellationToken)
    {
        // Convert Guid to strongly-typed ID if needed
        var parentId = command.ParentId.HasValue ? new {AggregateId}(command.ParentId.Value) : null;

        var entity = {Aggregate}.Create(
            command.Prop1,
            command.Prop2,
            ...,
            dateTimeProvider);

        await {aggregateRepo}.AddAsync(entity, cancellationToken);

        return Results.Created();
    }
}
```

#### Command Endpoint (Update)

```csharp
using FluentValidation;
using Microsoft.AspNetCore.Http;
using Wolverine.Http;
using Microsoft.AspNetCore.Mvc;
using {AppNamespace}.Exceptions;
using {DomainNamespace};
using {DomainNamespace}.Aggregates.{AggregateFolder};

namespace {AppNamespace}.{Feature};

/// <summary>
/// {Description}參數
/// </summary>
public sealed record {Name}Command(
    Guid {AggregateId},
    {PropertyType} {PropertyName},
    ...);

public class {Name}CommandValidator : AbstractValidator<{Name}Command>
{
    public {Name}CommandValidator()
    {
        RuleFor(x => x.{AggregateId}).NotEmpty();
        RuleFor(x => x.StringProp).NotEmpty().MaximumLength(30);
    }
}

/// <summary>
/// {Description}
/// </summary>
public static class {Name}Endpoint
{
    /// <summary>
    /// {Description}
    /// </summary>
    [Tags("{TagName}")]
    [WolverinePut("api/{route}")]
    [ProducesResponseType(StatusCodes.Status200OK)]
    public static async Task<IResult> Handle(
        {Name}Command command,
        I{Aggregate}Repository {aggregateRepo},
        IDateTimeProvider dateTimeProvider,
        CancellationToken cancellationToken)
    {
        var id = new {StrongId}(command.{AggregateId});
        var entity = await {aggregateRepo}.GetByIdAsync(id, cancellationToken)
            ?? throw new NotFoundApplicationException(nameof({Aggregate}), id);

        entity.Update(
            command.Prop1,
            command.Prop2,
            ...,
            dateTimeProvider);

        return Results.Ok();
    }
}
```

#### Command Endpoint (Delete)

Delete endpoints typically take the ID from the route, not a command body:

```csharp
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Wolverine.Http;
using {AppNamespace}.Exceptions;
using {DomainNamespace}.Aggregates.{AggregateFolder};

namespace {AppNamespace}.{Feature};

/// <summary>
/// {Description}
/// </summary>
public static class {Name}Endpoint
{
    /// <summary>
    /// {Description}
    /// </summary>
    [Tags("{TagName}")]
    [WolverineDelete("/api/{route}/{entityId}")]
    [ProducesResponseType(StatusCodes.Status200OK)]
    public static async Task<IResult> Handle(
        Guid {entityId},
        I{Aggregate}Repository {aggregateRepo})
    {
        var id = new {StrongId}({entityId});
        var entity = await {aggregateRepo}.GetByIdAsync(id)
            ?? throw new NotFoundApplicationException(nameof({Aggregate}), id);

        {aggregateRepo}.Delete(entity);

        return Results.Ok();
    }
}
```

#### Query Endpoint (Single item)

```csharp
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Wolverine.Http;
using {AppNamespace}.Exceptions;
using {DomainNamespace}.Aggregates.{AggregateFolder};

namespace {AppNamespace}.{Feature};

/// <summary>
/// {Description} 回應資料
/// </summary>
/// <param name="{Param1}">{Param1Description}</param>
public sealed record {Name}Dto(
    Guid {AggregateId},
    {PropertyType} {PropertyName},
    ...);

/// <summary>
/// {Description}
/// </summary>
public static class {Name}Endpoint
{
    /// <summary>
    /// {Description}
    /// </summary>
    [Tags("{TagName}")]
    [WolverineGet("/api/{route}/{entityId}")]
    [ProducesResponseType<{Name}Dto>(StatusCodes.Status200OK)]
    public static async Task<IResult> Handle(
        Guid {entityId},
        I{Feature}QueryService {queryService},
        CancellationToken cancellationToken)
    {
        var id = new {StrongId}({entityId});
        var result = await {queryService}.GetByIdAsync(id, cancellationToken)
            ?? throw new NotFoundApplicationException(nameof({Aggregate}), id);
        return Results.Ok(result);
    }
}
```

#### Query Endpoint (List/Browse)

```csharp
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Wolverine.Attributes;
using Wolverine.Http;
using {DomainNamespace}.Aggregates.{AggregateFolder};

namespace {AppNamespace}.{Feature};

/// <summary>
/// {Description} 資料
/// </summary>
/// <param name="{Param1}">{Param1Description}</param>
public sealed record {Name}Dto(
    Guid {AggregateId},
    {PropertyType} {PropertyName},
    ...);

/// <summary>
/// {Description}
/// </summary>
public static class {Name}Endpoint
{
    /// <summary>
    /// {Description}
    /// </summary>
    [Tags("{TagName}")]
    [WolverineGet("/api/{route}")]
    [NonTransactional]
    [ProducesResponseType<IReadOnlyList<{Name}Dto>>(StatusCodes.Status200OK)]
    public static async Task<IReadOnlyList<{Name}Dto>> Handle(
        I{Feature}QueryService {queryService},
        CancellationToken cancellationToken)
    {
        return await {queryService}.{MethodName}(cancellationToken);
    }
}
```

Note: Browse/List endpoints can return the DTO list directly (no `IResult` wrapper needed).

#### Link/Unlink Command Endpoint

```csharp
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Wolverine.Http;
using {DomainNamespace}.Aggregates.{AggregateFolder};
using {DomainNamespace}.Aggregates.{RelatedAggregateFolder};

namespace {AppNamespace}.{Feature};

/// <summary>
/// {Description}參數
/// </summary>
/// <param name="{Id1}">{Id1Description}</param>
/// <param name="{Id2}">{Id2Description}</param>
public sealed record {Name}Command(Guid {Id1}, Guid {Id2});

/// <summary>
/// {Description}
/// </summary>
public static class {Name}Endpoint
{
    /// <summary>
    /// {Description}
    /// </summary>
    [Tags("{TagName}")]
    [WolverinePost("api/{route}")]
    [ProducesResponseType(StatusCodes.Status200OK)]
    public static async Task<IResult> Handle(
        {Name}Command command,
        I{Aggregate}Repository {aggregateRepo},
        CancellationToken cancellationToken)
    {
        var id1 = new {StrongId1}(command.{Id1});
        var id2 = new {StrongId2}(command.{Id2});

        await {aggregateRepo}.{LinkMethod}(id1, id2, cancellationToken);

        return Results.Ok();
    }
}
```

### Key conventions

- **XML documentation**: All Command records, DTO records, and Endpoint classes/methods must have `<summary>` tags. Command/DTO records also have `<param>` tags for each property.
- **Strongly-typed ID conversion**: Always convert `Guid` parameters to strongly-typed IDs in the Handle method (e.g., `new MenuId(command.MenuId)`)
- **Not found handling**: Use `?? throw new NotFoundApplicationException(nameof({Aggregate}), id)` pattern
- **No IUnitOfWork**: Wolverine handles transactions automatically. Do NOT inject or call `IUnitOfWork`.
- **Validator**: Only generate for Commands with input that needs validation. Implement `AbstractValidator<T>`. Match Domain constraints.
- **Route convention**: `/api/{feature-kebab-case}` (e.g., `/api/menu-info`)
- **Tags**: Use `[Tags("{中文 feature name}")]` matching existing tags in the feature

### Present to user and wait for confirmation

Show all generated files with full code. Ask if the content looks right before writing.

---

## Phase 2: Update Domain Interface

After the Application layer is confirmed, analyze the Endpoint code:

1. Read the existing Domain Repository Interface (`I{Aggregate}Repository.cs`)
2. Check which repository methods the Endpoint calls
3. Identify any **missing methods** that need to be added

For example, if the Endpoint calls `_repo.GetNextSortOrderAsync(...)` but the interface doesn't have this method, propose adding it.

### Present changes

Show the user:
- Current interface content
- Proposed new method signatures to add
- Where the method will be inserted

Wait for confirmation before modifying the file.

If the Endpoint also needs a new **Query Service method** (for queries), propose adding it to `I{Feature}QueryService.cs` as well.

---

## Phase 3: Update Infrastructure Repository

After the Domain Interface is confirmed, generate the implementation:

1. Read the existing Infrastructure Repository (`{Aggregate}Repository.cs`)
2. For each new interface method, generate the EF Core implementation
3. Follow the same patterns documented in the add-repository skill:
   - Use `DbSet` (from base class) for aggregate's own table queries
   - Use `_dbContext.{DbSetName}` for cross-entity queries
   - Use `.Where().SingleOrDefaultAsync()` pattern
   - Value Object IDs can be compared directly with `==` in LINQ

### Present changes

Show the user:
- The new method implementations to be added
- Where they'll be inserted in the existing file

Wait for confirmation before modifying.

If a Query Service implementation is needed, also generate and present it.

---

## Phase 4: Remind about DI registration

After all files are written, check if any new services need DI registration:
- New Query Service interface → `services.AddScoped<I{Feature}QueryService, {Feature}QueryService>()`
- New Repository (unlikely at this stage but possible)

Point out the DI registration file location.

---

## Example walkthrough

**User says**: "我需要一個建立目錄的功能，輸入上層目錄ID、名稱、路由、圖示、是否顯示"

**Phase 1** — Generate `Application/MenuInfo/CreateMenuEndpoint.cs`:
```csharp
using FluentValidation;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Wolverine.Http;
using YP.FinancialManagementSystem.Domain;
using YP.FinancialManagementSystem.Domain.Aggregates.MenuInfo;

namespace YP.FinancialManagementSystem.Application.MenuInfo;

/// <summary>
/// 建立目錄參數
/// </summary>
/// <param name="ParentId">上層目錄識別碼</param>
/// <param name="MenuName">目錄名稱</param>
/// <param name="Route">路由</param>
/// <param name="Icon">目錄圖示</param>
/// <param name="IsVisible">是否顯示</param>
public sealed record CreateMenuCommand(
    Guid? ParentId,
    string MenuName,
    string Route,
    string? Icon,
    bool IsVisible);

/// <summary>
/// <see cref="CreateMenuCommand"/> 驗證器
/// </summary>
public class CreateMenuCommandValidator : AbstractValidator<CreateMenuCommand>
{
    public CreateMenuCommandValidator()
    {
        RuleFor(x => x.MenuName)
            .NotEmpty()
            .MaximumLength(30);

        RuleFor(x => x.Route)
            .NotEmpty()
            .MaximumLength(30);
    }
}

/// <summary>
/// 建立目錄
/// </summary>
public static class CreateMenuEndpoint
{
    /// <summary>
    /// 建立目錄
    /// </summary>
    [Tags("目錄管理")]
    [WolverinePost("/api/menu-info")]
    [ProducesResponseType(StatusCodes.Status201Created)]
    public static async Task<IResult> Handle(
        CreateMenuCommand command,
        IMenuRepository menuRepository,
        IDateTimeProvider dateTimeProvider,
        CancellationToken cancellationToken)
    {
        var parentId = command.ParentId.HasValue ? new MenuId(command.ParentId.Value) : null;

        var sortOrder = await menuRepository.GetNextSortOrderAsync(parentId, cancellationToken);

        var menu = Menu.Create(
            parentId,
            command.MenuName,
            command.Route,
            command.Icon,
            command.IsVisible,
            sortOrder,
            dateTimeProvider);

        await menuRepository.AddAsync(menu, cancellationToken);

        return Results.Created();
    }
}
```

→ User confirms ✓

**Phase 2** — Endpoint calls `GetNextSortOrderAsync` and `AddAsync`:
- `AddAsync` exists in `IRepository<T, TId>` → no change
- `GetNextSortOrderAsync` exists in `IMenuRepository` → no change

→ No Domain changes needed ✓

**Phase 3** — No new repository methods → skip

**Phase 4** — "Repository already registered. No DI changes needed."
