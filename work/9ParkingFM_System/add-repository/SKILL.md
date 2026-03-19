---
name: add-repository
description: "Generate an Infrastructure Repository implementation from a Domain Repository interface. Use this skill whenever the user says /add-repository, or asks to generate a repository implementation, implement a repository interface, create the infrastructure layer for a repository, or scaffold a repository class from its interface. Also trigger when the user mentions implementing IXxxRepository, creating a repository class, or bridging Domain interfaces to Infrastructure — even if they don't use the exact command name."
---

# Add Repository Implementation

Generate an Infrastructure-layer Repository class from an existing Domain Repository interface. The user writes the interface with method signatures and comment-based requirements; this skill produces the EF Core implementation.

## Context: What the user provides

The user creates a Domain Repository interface like this:

```csharp
namespace MyApp.Domain.OrderInfo;

public interface IOrderRepository
{
    // 新增訂單
    Task AddAsync(Order order, CancellationToken cancellationToken = default);

    // 用 ID 查詢，找不到回 null
    Task<Order?> GetAsync(int orderId, CancellationToken cancellationToken = default);

    // 查詢某客戶的所有訂單
    Task<List<Order>> GetByCustomerIdAsync(int customerId, CancellationToken cancellationToken = default);

    // 刪除訂單
    void Delete(Order order);
}
```

Key characteristics:
- Method signatures may be **rough** — input parameters and return types might need refinement
- **Comments above each method** describe the intended behavior in natural language
- The interface lives in `Domain/{AggregateFolder}/`

## Step 1: Gather context

Before generating, read these files to understand the full picture:

1. **The interface file** — the user will point you to it, or you find it in the Domain layer
2. **The corresponding Aggregate Root** — in the same folder as the interface, to understand entity structure, properties, Value Objects, and computed properties
3. **AppDbContext.cs** — to find the correct `DbSet<T>` property name and see what entities are available
4. **Existing repository implementations** — look in `Infrastructure/Persistence/Configuration/Domain/` for examples of how this project implements repositories, so you can match the style

## Step 2: Refine the interface (if needed)

Because the user said input/return types may be vague, review the interface and suggest improvements before implementing. Common refinements:

- Add `CancellationToken cancellationToken = default` if missing
- Suggest `Task<T?>` vs `Task<T>` based on whether "not found" is a valid case
- For delete methods that require business validation, suggest accepting a wrapper type (like `DeletedOrder`) instead of the raw entity
- For methods returning collections, suggest `Task<List<T>>` or `Task<IReadOnlyList<T>>`

**Show the user your suggested refinements and wait for confirmation before proceeding.** If the interface looks good as-is, skip this step.

## Step 3: Generate the Repository implementation

### File location

Place the implementation in:
```
{InfrastructureProject}/Persistence/Configuration/Domain/{AggregateFolder}/{Name}Repository.cs
```

This co-locates the repository with the EF Core configuration, matching the project convention.

### Template

```csharp
using Microsoft.EntityFrameworkCore;
using {DomainNamespace};

namespace {InfrastructureNamespace}.Persistence.Configuration.Domain.{AggregateFolder};

public class {Name}Repository : I{Name}Repository
{
    private readonly AppDbContext _appDbContext;

    public {Name}Repository(AppDbContext appDbContext)
    {
        _appDbContext = appDbContext;
    }

    // ... method implementations
}
```

### Method implementation patterns

Generate each method based on its comment description and signature. Follow these patterns:

**Add/Create:**
```csharp
public async Task AddAsync(Order order, CancellationToken cancellationToken = default)
    => await _appDbContext.AddAsync(order, cancellationToken);
```

**Get by ID (return null):**
```csharp
public async Task<Order?> GetAsync(int orderId, CancellationToken cancellationToken = default)
{
    return await _appDbContext.OrderInfo
        .Where(x => x.OrderId == orderId)
        .SingleOrDefaultAsync(cancellationToken);
}
```

**Get by ID (throw if not found):**
```csharp
public async Task<Order?> GetAsync(int orderId, CancellationToken cancellationToken = default)
{
    var entity = await _appDbContext.OrderInfo
        .Where(x => x.OrderId == orderId)
        .SingleOrDefaultAsync(cancellationToken);

    if (entity is null)
        throw new NotFoundException(orderId);

    return entity;
}
```

**Get by ID with Value Object key:**
```csharp
public async Task<Company?> GetAsync(Guid companyId, CancellationToken cancellationToken = default)
{
    return await _appDbContext.Company
        .Where(x => ((Guid)x.CompanyId) == companyId)
        .SingleOrDefaultAsync(cancellationToken);
}
```
Note: Value Object IDs need explicit cast in LINQ queries.

**Query by foreign key:**
```csharp
public async Task<List<Order>> GetByCustomerIdAsync(int customerId, CancellationToken cancellationToken = default)
{
    return await _appDbContext.OrderInfo
        .Where(x => x.CustomerId == customerId)
        .ToListAsync(cancellationToken);
}
```

**Delete (simple):**
```csharp
public void Delete(Order order)
    => _appDbContext.OrderInfo.Remove(order);
```

**Delete (with wrapper type):**
```csharp
public void Delete(DeletedOrder deletedOrder)
    => _appDbContext.OrderInfo.Remove(deletedOrder.Value);
```

**Aggregate query (e.g., next sort order):**
```csharp
public async Task<int> GetNextSortOrderAsync(int? parentId, CancellationToken cancellationToken = default)
{
    var maxSortOrder = await _appDbContext.OrderInfo
        .Where(x => x.ParentId == parentId)
        .MaxAsync(x => (int?)x.SortOrder, cancellationToken);

    return (maxSortOrder ?? 0) + 1;
}
```

**Link/Unlink via join table:**
```csharp
public async Task LinkAsync(int menuInfoId, int actionInfoId, CancellationToken cancellationToken = default)
{
    var existing = await _appDbContext.MenuAction
        .Where(x => x.MenuInfoID == menuInfoId && x.ActionInfoID == actionInfoId)
        .SingleOrDefaultAsync(cancellationToken);

    if (existing is not null)
    {
        existing.IsEnabled = true;
    }
    else
    {
        await _appDbContext.MenuAction.AddAsync(new MenuAction
        {
            MenuInfoID = menuInfoId,
            ActionInfoID = actionInfoId,
            IsEnabled = true,
            SortOrder = 0
        }, cancellationToken);
    }
}
```

**Populating computed properties:**
If the Aggregate has `Ignore`-d properties (like `LinkedMenusId`) that come from join tables, populate them after loading:
```csharp
public async Task<Action?> GetAsync(int actionInfoId, CancellationToken cancellationToken = default)
{
    var action = await _appDbContext.ActionInfo
        .Where(x => x.ActionInfoId == actionInfoId)
        .SingleOrDefaultAsync(cancellationToken);

    if (action is null)
        return null;

    var linkedMenuIds = await _appDbContext.MenuAction
        .Where(x => x.ActionInfoID == actionInfoId)
        .Select(x => x.MenuInfoID)
        .ToListAsync(cancellationToken);

    action.SetLinkedMenuIds(linkedMenuIds);

    return action;
}
```

### Key conventions

- Use `_appDbContext.{DbSetName}` — get the DbSet name from AppDbContext
- Use `.Where().SingleOrDefaultAsync()` instead of `.FindAsync()` for consistency
- Async methods return `Task<T>`, sync methods (like Delete) return `void`
- All async methods accept `CancellationToken cancellationToken = default`
- When querying Value Object IDs, cast to the underlying type: `((Guid)x.CompanyId)`

## Step 4: Confirm with user

Before writing, show the user:
1. Any suggested refinements to the interface (if applicable)
2. The full generated repository class
3. The file path where it will be written

Wait for user confirmation before writing the file.

## Step 5: Remind about DI registration

After generating, remind the user they may need to register the repository in their DI container:

```csharp
services.AddScoped<I{Name}Repository, {Name}Repository>();
```

Point out the file where other repositories are registered (typically an `Installer` or `ServiceCollectionExtensions` class in the Infrastructure project).
