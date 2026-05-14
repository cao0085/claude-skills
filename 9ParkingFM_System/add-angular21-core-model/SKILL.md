---
name: add-angular21-core-model
description: "Scaffold a typed core model from DB column definitions in an Angular 21 project. Use this skill whenever: user runs /add-angular21-core-model, provides DB column definitions to map to TypeScript, or asks to create/generate a base model — especially when project-specific file conventions, naming rules, or alignment to existing model files are involved."
---

# Add Angular Core Model

Generate a TypeScript interface from DB column definitions and save it to `src/app/core/models/`.

---

## Workflow

### Step 1: Collect Info

1. If the user hasn't provided any info, ask for:
   - DB column definitions (name + type + nullable)
   - Model/entity name (e.g., `Company`)
   - Feature group and feature name (e.g., group: `admin`, feature: `company`)
   - Any enum fields? If yes, ask for enum name, members, and their display labels
     (e.g., `CompanyType: LimitedCompany=0 有限公司, CorporationLimited=1 股份有限公司`)
2. If the user only provides a column list, infer the model name from the column names
   (e.g., columns prefixed with `Company` → model name `Company`)
3. Confirm the following before generating:
   - Core model filename (e.g., `company.ts`)
   - Feature path (e.g., `src/app/features/admin/company/models.ts`)

### Step 2: Map Types

Apply the following rules to each column:

**DB → TypeScript Type Mapping**

| DB Type | TypeScript | Notes |
|---|---|---|
| `uniqueidentifier` | `string` | GUID |
| `int` / `bigint` | `number` | |
| `nvarchar` / `varchar` | `string` | |
| `bit` | `boolean` | |
| `datetime2` / `datetime` | `Date` | |
| `decimal` / `float` | `number` | |
| `nvarchar(max)` | `string` | |

**Nullability Rules**
- `NOT NULL` → plain type, e.g., `string`
- `NULL` → union with null, e.g., `string | null`

> ⚠️ Never use `?` optional for nullable columns — always use `| null`

**Naming Convention**
- DB columns are PascalCase → TypeScript properties use **camelCase**
- e.g., `MenuInfoId` → `menuInfoId`, `IsActive` → `isActive`

**Enum Fields**
- If the user provides enum info, use the enum name as the field type instead of `number`
- e.g., `companyType: CompanyType` instead of `companyType: number`

### Step 3: Generate Files

Generate two files:

**File 1 — Core Model**
- Path: `src/app/core/models/{camelCaseModelName}.ts`

Output format (no enum):

```typescript
// Base DB Model - aligned to database (frontend uses camelCase)
export interface {ModelName} {
    {camelCaseField}: {TsType};           // {OriginalDbColumn}: {DbType} NOT NULL
    {camelCaseField}: {TsType} | null;    // {OriginalDbColumn}: {DbType} NULL
}
```

Output format (with enum) — declare enum before the interface, append frontend extensions after:

```typescript
export enum {EnumName} {
    {Member} = {value},
}

// Base DB Model - aligned to database (frontend uses camelCase)
export interface {ModelName} {
    {camelCaseField}: {TsType};           // {OriginalDbColumn}: {DbType} NOT NULL
    {camelCaseField}: {EnumName};         // {OriginalDbColumn}: int NOT NULL
}

// Frontend extensions
export const {ENUM_NAME}_LABEL: Record<{EnumName}, string> = {
    [{EnumName}.{Member}]: '{display label}',
};

export const {ENUM_NAME}_OPTIONS = Object.entries({ENUM_NAME}_LABEL).map(
    ([value, label]) => ({ label, value: Number(value) as {EnumName} })
);

@Pipe({ name: '{camelCaseEnumName}', standalone: true })
export class {EnumName}Pipe implements PipeTransform {
    transform(value: {EnumName}): string {
        return {ENUM_NAME}_LABEL[value] ?? '';
    }
}
```

Add an inline comment at the end of each field: `// {OriginalDbColumn}: {DbType} {NOT NULL | NULL}`

**File 2 — Feature Model**
- Path: `src/app/features/{group}/{feature-name}/models.ts`

This file is the single entry point for all models used within the feature.
All access to core models must go through this file — never import directly from `@app/core/models/` inside the feature.

At this stage, only generate the re-export. The user will extend it later with `Pick<>`, `extends`, DTOs, etc.

Output format (no enum):

```typescript
import { {ModelName} } from '@app/core/models/{camelCaseModelName}';
export type { {ModelName} } from '@app/core/models/{camelCaseModelName}';
```

Output format (with enum):

```typescript
import { {ModelName} } from '@app/core/models/{camelCaseModelName}';
export type { {ModelName} } from '@app/core/models/{camelCaseModelName}';
export { {EnumName}, {EnumName}Pipe, {ENUM_NAME}_OPTIONS } from '@app/core/models/{camelCaseModelName}';
```

---

## Example

**Input:**
```
Id uuid NOT NULL
CompanyCode nvarchar(20) NOT NULL
FullName nvarchar(100) NOT NULL
ShortName nvarchar(50) NOT NULL
EnglishName nvarchar(100) NULL
CompanyType int NOT NULL  (enum: LimitedCompany=0 有限公司, CorporationLimited=1 股份有限公司, SoleProprietorship=2 獨資, Partnership=3 合夥)
IsActive bit NOT NULL
```

**Output `src/app/core/models/company.ts`:**
```typescript
export enum CompanyType {
    LimitedCompany = 0,
    CorporationLimited = 1,
    SoleProprietorship = 2,
    Partnership = 3,
}

// Base DB Model - aligned to database (frontend uses camelCase)
export interface Company {
    id: string;                   // Id: uuid NOT NULL
    companyCode: string;          // CompanyCode: nvarchar(20) NOT NULL
    fullName: string;             // FullName: nvarchar(100) NOT NULL
    shortName: string;            // ShortName: nvarchar(50) NOT NULL
    englishName: string | null;   // EnglishName: nvarchar(100) NULL
    companyType: CompanyType;     // CompanyType: int NOT NULL
    isActive: boolean;            // IsActive: bit NOT NULL
}

// Frontend extensions
export const COMPANY_TYPE_LABEL: Record<CompanyType, string> = {
    [CompanyType.LimitedCompany]: '有限公司',
    [CompanyType.CorporationLimited]: '股份有限公司',
    [CompanyType.SoleProprietorship]: '獨資',
    [CompanyType.Partnership]: '合夥',
};

export const COMPANY_TYPE_OPTIONS = Object.entries(COMPANY_TYPE_LABEL).map(
    ([value, label]) => ({ label, value: Number(value) as CompanyType })
);

@Pipe({ name: 'companyType', standalone: true })
export class CompanyTypePipe implements PipeTransform {
    transform(value: CompanyType): string {
        return COMPANY_TYPE_LABEL[value] ?? '';
    }
}
```

**Output `src/app/features/admin/company/models.ts`:**
```typescript
import { Company } from '@app/core/models/company';
export type { Company } from '@app/core/models/company';
export { CompanyType, CompanyTypePipe, COMPANY_TYPE_OPTIONS } from '@app/core/models/company';
```

---

## Project Context

- Path: `src/app/core/models/`
- Purpose: Align to DB schema; consumed by feature `models.ts` via `Pick<>`, `extends`, or re-export
- Naming: filename matches interface name (camelCase filename / PascalCase interface name)