---
name: add-aggregate-efcore
description: "Generate EF Core Code First configuration from an existing Domain Aggregate Root class. Use this skill whenever the user says /add-aggregate-efcore, or asks to generate EF Core configuration, entity type configuration, DbContext mapping, or database mapping for a DDD aggregate. Also trigger when the user mentions creating IEntityTypeConfiguration, adding a DbSet, or mapping domain entities to EF Core — even if they don't use the exact command name."
---

# Add Aggregate EF Core Configuration

Generate EF Core `IEntityTypeConfiguration<T>` and update `AppDbContext` from an existing Domain Aggregate Root class that already has its fields defined.

## Workflow

The user has already created the Aggregate Root `.cs` file with:
- All properties defined (with `private set`)
- Optional **inline comments** for metadata:
  - `// VO: {TypeName}` — this property is a Value Object, generate `HasConversion`
  - `// MaxLength: {N}` — set `HasMaxLength(N)` for this string property
  - `// Ignore` — this property should not be persisted (use `builder.Ignore()`)
  - `// FK: {TargetEntity}` — this property is a foreign key

Your job: read the file, generate the EF Core configuration, and update AppDbContext.

## Step 1: Locate the project structure

Before generating anything, understand the project layout:

1. Find the `.sln` file or root namespace by scanning `*.csproj` files near the working directory
2. Identify the **Infrastructure project** — look for a project containing `Persistence/AppDbContext.cs`
3. Identify the **Domain project** — the project containing the user's Aggregate Root file
4. Extract the root namespace (e.g., `YP.FinancialManagementSystem`) from the csproj `<RootNamespace>` or by convention from the project name

Key paths to locate:
- `AppDbContext.cs` — where to add the `DbSet<T>`
- `Infrastructure/Persistence/Configuration/Domain/` — where to create the configuration class

## Step 2: Read and parse the Aggregate Root

Read the user-provided `.cs` file. Extract:

### Properties
For each property, determine:
- **Name** and **Type**
- **Nullability** — `string?` means nullable, `string` means required
- **Comment annotations** — check the trailing comment on the same line

### Primary Key
The first property ending in `Id` is typically the primary key. Rules:
- If the type is `int` → use `.ValueGeneratedOnAdd()`
- If the type is a Value Object (like `CompanyId`) → use `.HasConversion()`
- If the type is `Guid` → just set as key

### Comment annotation parsing

```csharp
public CompanyId CompanyId { get; private set; }           // VO: CompanyId
public string FullName { get; private set; }               // MaxLength: 50
public string? Description { get; private set; }           // MaxLength: 200
public IReadOnlyList<int> ChildrenIds { get; private set; } // Ignore
public int CategoryId { get; private set; }                // FK: Category
```

## Step 3: Generate the Configuration class

### File location
Place the configuration in:
```
{InfrastructureProject}/Persistence/Configuration/Domain/{AggregateFolder}/{Name}Configuration.cs
```

Where `{AggregateFolder}` matches the folder name of the Aggregate Root in the Domain project.

### Template

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using {DomainNamespace};

namespace {InfrastructureNamespace}.Persistence.Configuration.Domain.{AggregateFolder};

internal class {Name}Configuration : IEntityTypeConfiguration<{AggregateType}>
{
    public void Configure(EntityTypeBuilder<{AggregateType}> builder)
    {
        builder.ToTable("{TableName}");

        // Primary key
        builder.HasKey(x => x.{PrimaryKeyProp});

        // Primary key generation strategy
        // For int: builder.Property(x => x.{PK}).ValueGeneratedOnAdd();
        // For VO:  builder.Property(x => x.{PK}).HasConversion(x => x.Value, v => new {VOType}(v));

        // Properties — generate one line per property
        // string (required):  builder.Property(x => x.Name).IsRequired().HasMaxLength({N});
        // string (nullable):  builder.Property(x => x.Name).HasMaxLength({N});
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
public virtual DbSet<{AggregateType}> {TableName} { get; set; }
```

- Add the corresponding `using` statement if needed
- The table name should match the `ToTable()` value in the configuration
- Use the aggregate's folder name as the table name by default (e.g., folder `CompanyInfo` → table `Company` or `CompanyInfo`)

## Step 5: Confirm with user

Before writing any files, show the user:
1. The generated configuration class (full code)
2. The line to be added to AppDbContext
3. The file paths where these will be written

Wait for user confirmation before writing.

## Example

Given this Aggregate Root at `Domain/OrderInfo/Order.cs`:

```csharp
namespace MyApp.Domain.OrderInfo;

public class Order : Entity
{
    public int OrderId { get; private set; }
    public string OrderNumber { get; private set; }        // MaxLength: 20
    public decimal TotalAmount { get; private set; }
    public string? Note { get; private set; }              // MaxLength: 500
    public int CustomerId { get; private set; }            // FK: Customer
    public DateTime CreatedAt { get; private set; }
    public IReadOnlyList<int> ItemIds { get; private set; } // Ignore

    private Order() { }
}
```

The skill generates:

**`Infrastructure/Persistence/Configuration/Domain/OrderInfo/OrderConfiguration.cs`:**
```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using MyApp.Domain.OrderInfo;

namespace MyApp.Infrastructure.Persistence.Configuration.Domain.OrderInfo;

internal class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("OrderInfo");

        builder.HasKey(x => x.OrderId);
        builder.Property(x => x.OrderId).ValueGeneratedOnAdd();

        builder.Property(x => x.OrderNumber).IsRequired().HasMaxLength(20);
        builder.Property(x => x.TotalAmount);
        builder.Property(x => x.Note).HasMaxLength(500);
        builder.Property(x => x.CustomerId);
        builder.Property(x => x.CreatedAt);

        builder.Ignore(x => x.ItemIds);

        builder.HasOne<Customer>()
            .WithMany()
            .HasForeignKey(x => x.CustomerId)
            .OnDelete(DeleteBehavior.Restrict);

        builder.Property<byte[]>("Version").IsRowVersion();
    }
}
```

**AppDbContext.cs addition:**
```csharp
public virtual DbSet<Order> OrderInfo { get; set; }
```
