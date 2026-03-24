---
name: 9pfms:add-ddd-aggregate-efcore
description: "Generate EF Core Code First configuration from an existing Domain Aggregate Root or Entity class. Use this skill whenever the user says /9pfms:add-ddd-aggregate-efcore, or asks to generate EF Core configuration, entity type configuration, DbContext mapping, or database mapping for a DDD aggregate. Also trigger when the user mentions creating IEntityTypeConfiguration, adding a DbSet, or mapping domain entities to EF Core — even if they don't use the exact command name."
---

# Add Aggregate EF Core Configuration

Generate EF Core `IEntityTypeConfiguration<T>`, update `AppDbContext`, and generate a Domain Repository interface from an existing Domain Aggregate Root or Entity class that already has its fields defined.

## Domain Base Class Architecture

The project uses a generic base class hierarchy:

### `Entity<TId>`
- Abstract base class for all domain entities
- Provides `public TId Id { get; protected set; }` — the primary key
- `TId` is constrained to `notnull`
- Includes equality comparison based on `Id`

### `AggregateRoot<TId>`
- Inherits from `Entity<TId>` and implements `IHasDomainEvents`
- Use this for **aggregate roots** (top-level domain objects that own a consistency boundary)
- Provides domain event support (`AddDomainEvent`, `ClearDomainEvents`)

### Strongly-Typed IDs
- Defined as `public sealed record {Name}Id(Guid Value)` with an explicit operator to `Guid`
- Used as the `TId` generic parameter (e.g., `AggregateRoot<MenuId>`, `Entity<Guid>`)

### How to distinguish
- **Aggregate Root**: inherits `AggregateRoot<TId>` — e.g., `public class Menu : AggregateRoot<MenuId>`
- **Entity**: inherits `Entity<TId>` — e.g., `public class MenuAction : Entity<Guid>`
- The `Id` property is **NOT** declared in the class itself — it comes from the base class

### Repository Base Architecture

The project provides generic repository interfaces and base implementations:

**`IRepository<T, TId>`** (Domain interface) — standard CRUD contract:
```csharp
public interface IRepository<T, TId> where T : AggregateRoot<TId> where TId : notnull
{
    Task<T?> GetByIdAsync(TId id, CancellationToken cancellationToken = default);
    Task AddAsync(T entity, CancellationToken cancellationToken = default);
    void Update(T entity);
    void Delete(T entity);
}
```

**`Repository<T, TId>`** (Infrastructure base class at `Infrastructure/Shared/Repository.cs`) — default EF Core implementations:
- Injects `DbContext` and exposes `DbSet` as `protected readonly DbSet<T> DbSet`
- `GetByIdAsync` uses `.Equals()` for TId comparison (supports Record/Struct IDs)
- `AddAsync` uses `DbSet.AddAsync()`
- `Update` uses `DbSet.Update()`
- `Delete` uses `DbSet.Remove()`

Concrete repositories: Domain interface inherits `IRepository<T, TId>` and adds custom methods; Implementation inherits `Repository<T, TId>`.

## Workflow

The user has already created the Aggregate Root or Entity `.cs` file with:
- Inheritance from `AggregateRoot<TId>` or `Entity<TId>`
- All properties defined (with `private set`)
- Optional **inline comments** for metadata:
  - `// VO: {TypeName}` — this property is a Value Object, generate `HasConversion`
  - `// MaxLength: {N}` — set `HasMaxLength(N)` for this string property
  - `// Ignore` — this property should not be persisted (use `builder.Ignore()`)
  - `// FK: {TargetEntity}` — this property is a foreign key

Your job: read the file, generate the EF Core configuration, update AppDbContext, and (for Aggregate Roots only) generate the Domain Repository interface.

## Step 1: Locate the project structure

Before generating anything, understand the project layout:

1. Find the `.sln` file or root namespace by scanning `*.csproj` files near the working directory
2. Identify the **Infrastructure project** — look for a project containing `Persistence/AppDbContext.cs`
3. Identify the **Domain project** — the project containing the user's Aggregate Root file
4. Extract the root namespace (e.g., `YP.FinancialManagementSystem`) from the csproj `<RootNamespace>` or by convention from the project name

Key paths to locate:
- `AppDbContext.cs` — where to add the `DbSet<T>`
- `Infrastructure/Persistence/Configuration/Domain/` — where to create the configuration class

## Step 2: Read and parse the Aggregate Root / Entity

Read the user-provided `.cs` file. Extract:

### Base class and TId
- Determine if the class inherits `AggregateRoot<TId>` or `Entity<TId>`
- Extract the `TId` type (e.g., `MenuId`, `Guid`, `int`)
- The primary key is always the `Id` property inherited from the base class

### Properties
For each property, determine:
- **Name** and **Type**
- **Nullability** — `string?` means nullable, `string` means required
- **Comment annotations** — check the trailing comment on the same line

### Primary Key (`Id`)
The `Id` property comes from the base class. Configuration rules based on `TId` type:
- If `TId` is a **strongly-typed ID** (e.g., `MenuId`) → use `.HasConversion(x => x.Value, v => new {VOType}(v))`
- If `TId` is `int` → use `.ValueGeneratedOnAdd()`
- If `TId` is `Guid` → just set as key (no special config)

### Comment annotation parsing

```csharp
public string FullName { get; private set; }               // MaxLength: 50
public string? Description { get; private set; }           // MaxLength: 200
public IReadOnlyList<int> ChildrenIds { get; private set; } // Ignore
public int CategoryId { get; private set; }                // FK: Category
```

### Strongly-typed ID properties (non-PK)
Properties whose type ends in `Id` and is a strongly-typed record (e.g., `MenuId`, `ActionId`) should automatically get `HasConversion` — no `// VO:` comment needed. Example:
```csharp
public MenuId MenuId { get; private set; }    // auto-detected as VO
public ActionId ActionId { get; private set; } // auto-detected as VO
```

## Step 3: Generate the Configuration class

### File location
Place the configuration in:
```
{InfrastructureProject}/Persistence/Configuration/Domain/{AggregateFolder}/{Name}Configuration.cs
```

Where `{AggregateFolder}` matches the folder name of the Aggregate Root in the Domain project (under `Aggregates/`).

### Template

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using {DomainNamespace};

namespace {InfrastructureNamespace}.Persistence.Configuration.Domain.{AggregateFolder};

internal class {Name}Configuration : IEntityTypeConfiguration<{EntityType}>
{
    public void Configure(EntityTypeBuilder<{EntityType}> builder)
    {
        builder.ToTable("{TableName}");

        // Primary key — always `Id` from base class
        builder.HasKey(x => x.Id);

        // Primary key generation strategy based on TId type:
        // For strongly-typed ID (e.g., MenuId):
        //   builder.Property(x => x.Id).HasConversion(x => x.Value, v => new MenuId(v));
        // For int:
        //   builder.Property(x => x.Id).ValueGeneratedOnAdd();
        // For Guid:
        //   (no additional config needed)

        // Properties — generate one line per property
        // string (required):  builder.Property(x => x.Name).IsRequired().HasMaxLength({N});
        // string (nullable):  builder.Property(x => x.Name).HasMaxLength({N});
        // Strongly-typed ID:  builder.Property(x => x.Prop).HasConversion(x => x.Value, v => new {VOType}(v));
        // VO property:        builder.Property(x => x.Prop).HasConversion(x => x.Value, v => new {VOType}(v));
        // DateTime:           builder.Property(x => x.CreatedAt); (plain, no special config needed)
        // Ignored:            builder.Ignore(x => x.Prop);

        // Foreign keys (if any)
        // builder.HasOne<{Target}>()
        //     .WithMany()
        //     .HasForeignKey(x => x.{FKProp})
        //     .OnDelete(DeleteBehavior.Restrict);

        // Always add row version for optimistic concurrency
        builder.Property<byte[]>("Version").IsRowVersion();
    }
}
```

### Property mapping rules

| Property Type | Nullable? | Comment | Configuration |
|---|---|---|---|
| `string` | No | `// MaxLength: N` | `.IsRequired().HasMaxLength(N)` |
| `string?` | Yes | `// MaxLength: N` | `.HasMaxLength(N)` |
| `int`, `bool`, `decimal` | No | — | `builder.Property(x => x.Prop);` |
| `int?`, `bool?` | Yes | — | `builder.Property(x => x.Prop);` |
| `DateTime` | No | — | `builder.Property(x => x.Prop);` |
| `DateTime?` | Yes | — | `builder.Property(x => x.Prop);` |
| Strongly-typed ID (e.g., `MenuId`, `ActionId`) | — | (auto-detected) | `.HasConversion(x => x.Value, v => new {Type}(v))` |
| Value Object type | — | `// VO: {Type}` | `.HasConversion(x => x.Value, v => new {Type}(v))` |
| `IReadOnlyList<>`, `IReadOnlyCollection<>` | — | `// Ignore` | `builder.Ignore(x => x.Prop);` |
| Any type | — | `// Ignore` | `builder.Ignore(x => x.Prop);` |
| `int` (foreign key) | — | `// FK: {Target}` | Add `HasOne<{Target}>().WithMany().HasForeignKey()` |

### Naming conflicts
If the Aggregate class name conflicts with a System type (e.g., `Action`), add a using alias at the top:
```csharp
using DomainAction = {DomainNamespace}.Action;
```
Then use `DomainAction` throughout the configuration.

## Step 4: Update AppDbContext

Add a new `DbSet<T>` property to `AppDbContext.cs`:

```csharp
public virtual DbSet<{EntityType}> {TableName} { get; set; }
```

- Add the corresponding `using` statement if needed
- The table name should match the `ToTable()` value in the configuration
- Use the aggregate's folder name as the table name by default (e.g., folder `MenuInfo` → table `MenuInfo`)

## Step 5: Generate the Domain Repository Interface (Aggregate Roots only)

**This step only applies when the class inherits `AggregateRoot<TId>`.** Do NOT generate a repository interface for `Entity<TId>` classes — entities are managed through their aggregate root's repository.

### File location
Place the interface in the same folder as the Aggregate Root:
```
{DomainProject}/Aggregates/{AggregateFolder}/I{Name}Repository.cs
```

### Template

```csharp
using {DomainNamespace};

namespace {DomainNamespace}.Aggregates.{AggregateFolder};

public interface I{Name}Repository : IRepository<{AggregateType}, {TId}>
{
    // 使用者可在此新增自訂方法簽名，之後呼叫 /9pfms:add-ddd-repository 產生實作
}
```

- The interface inherits `IRepository<T, TId>` which already provides `GetByIdAsync`, `AddAsync`, `Update`, `Delete`
- Generate the interface with an **empty body** (only the inheritance) — the user will add custom method signatures manually before calling `/9pfms:add-ddd-repository`
- Add the necessary `using` statements for any referenced types

## Step 6: Confirm with user

Before writing any files, show the user:
1. The generated configuration class (full code)
2. The line to be added to AppDbContext
3. The generated repository interface (for Aggregate Roots only)
4. The file paths where these will be written

Wait for user confirmation before writing.

After confirmation, remind the user of the next steps:
> 如果需要自訂查詢方法，可以在 `I{Name}Repository` 中新增方法簽名，然後呼叫 `/9pfms:add-ddd-repository` 產生 Infrastructure 實作。

## Example

Given this Aggregate Root at `Domain/Aggregates/MenuInfo/Menu.cs`:

```csharp
namespace YP.FinancialManagementSystem.Domain.Aggregates.MenuInfo;

public class Menu : AggregateRoot<MenuId>
{
    public MenuId? ParentId { get; private set; }
    public string MenuName { get; private set; } = null!;    // MaxLength: 50
    public string? Route { get; private set; }               // MaxLength: 200
    public string? Icon { get; private set; }                // MaxLength: 100
    public int SortOrder { get; private set; }
    public bool IsVisible { get; private set; }
    public bool IsActive { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime UpdatedAt { get; private set; }
    public IReadOnlyList<int> ChildrenIds { get; private set; } = []; // Ignore

    private Menu() { }
}
```

The skill generates **3 outputs**:

### 1. `Infrastructure/Persistence/Configuration/Domain/MenuInfo/MenuConfiguration.cs`:
```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using YP.FinancialManagementSystem.Domain.Aggregates.MenuInfo;

namespace YP.FinancialManagementSystem.Infrastructure.Persistence.Configuration.Domain.MenuInfo;

internal class MenuConfiguration : IEntityTypeConfiguration<Menu>
{
    public void Configure(EntityTypeBuilder<Menu> builder)
    {
        builder.ToTable("MenuInfo");

        builder.HasKey(x => x.Id);
        builder.Property(x => x.Id).HasConversion(x => x.Value, v => new MenuId(v));

        builder.Property(x => x.ParentId).HasConversion(x => x!.Value, v => new MenuId(v));
        builder.Property(x => x.MenuName).IsRequired().HasMaxLength(50);
        builder.Property(x => x.Route).HasMaxLength(200);
        builder.Property(x => x.Icon).HasMaxLength(100);
        builder.Property(x => x.SortOrder);
        builder.Property(x => x.IsVisible);
        builder.Property(x => x.IsActive);
        builder.Property(x => x.CreatedAt);
        builder.Property(x => x.UpdatedAt);

        builder.Ignore(x => x.ChildrenIds);

        builder.Property<byte[]>("Version").IsRowVersion();
    }
}
```

### 2. AppDbContext.cs addition:
```csharp
public virtual DbSet<Menu> MenuInfo { get; set; }
```

### 3. `Domain/Aggregates/MenuInfo/IMenuRepository.cs`:
```csharp
namespace YP.FinancialManagementSystem.Domain.Aggregates.MenuInfo;

public interface IMenuRepository : IRepository<Menu, MenuId>
{
    // 使用者可在此新增自訂方法簽名，之後呼叫 /9pfms:add-ddd-repository 產生實作
}
```

Then the user adds custom methods:
```csharp
public interface IMenuRepository : IRepository<Menu, MenuId>
{
    Task<int> GetNextSortOrderAsync(MenuId? parentId, CancellationToken cancellationToken = default);
    Task LinkinActionAsync(MenuId menuId, ActionId actionId, CancellationToken cancellationToken = default);
}
```

And calls `/9pfms:add-ddd-repository` to generate the implementation.

---

Given this **Entity** at `Domain/Aggregates/MenuInfo/MenuAction.cs`:

```csharp
using YP.FinancialManagementSystem.Domain.Aggregates.ActionInfo;

namespace YP.FinancialManagementSystem.Domain.Aggregates.MenuInfo;

public class MenuAction : Entity<Guid>
{
    public MenuId MenuId { get; private set; }
    public ActionId ActionId { get; private set; }
    public int SortOrder { get; private set; }
    public bool IsEnabled { get; private set; }

    private MenuAction() { }
}
```

The skill generates **only 2 outputs** (no repository interface for Entity):

### 1. `Infrastructure/Persistence/Configuration/Domain/MenuInfo/MenuActionConfiguration.cs`:
```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using YP.FinancialManagementSystem.Domain.Aggregates.ActionInfo;
using YP.FinancialManagementSystem.Domain.Aggregates.MenuInfo;

namespace YP.FinancialManagementSystem.Infrastructure.Persistence.Configuration.Domain.MenuInfo;

internal class MenuActionConfiguration : IEntityTypeConfiguration<MenuAction>
{
    public void Configure(EntityTypeBuilder<MenuAction> builder)
    {
        builder.ToTable("MenuAction");

        builder.HasKey(x => x.Id);

        builder.Property(x => x.MenuId).HasConversion(x => x.Value, v => new MenuId(v));
        builder.Property(x => x.ActionId).HasConversion(x => x.Value, v => new ActionId(v));
        builder.Property(x => x.SortOrder);
        builder.Property(x => x.IsEnabled);

        builder.Property<byte[]>("Version").IsRowVersion();
    }
}
```

### 2. AppDbContext.cs addition:
```csharp
public virtual DbSet<MenuAction> MenuAction { get; set; }
```
