---
name: 9pfms:add-angular21-core-model
description: "Create a new core model file in @app/core/models/ that maps DB columns to a TypeScript interface. Trigger when user says /add-angular21-core-model, provides DB column definitions, or asks to create a base model aligned to the database."
---

# Add Angular Core Model

根據使用者提供的 DB 欄位，建立對應的 TypeScript interface 並存入 `src/app/core/models/`。

---

## Step 1: 收集資訊

使用者會提供：
- **Model 名稱**（e.g., `Company`）
- **DB 欄位清單**（欄位名稱、型別、是否 NULL）

若使用者只給欄位清單，從欄位名稱推斷 Model 名稱。

---

## Step 2: 型別對應規則

### DB → TypeScript 型別對照

| DB 型別 | TypeScript | 備註 |
|---|---|---|
| `uniqueidentifier` | `string` | Guid |
| `int` / `bigint` | `number` | |
| `nvarchar` / `varchar` | `string` | |
| `bit` | `boolean` | |
| `datetime2` / `datetime` | `Date` | |
| `decimal` / `float` | `number` | |
| `nvarchar(max)` | `string` | |

### NULL 規則
- `NOT NULL` → 一般型別，e.g., `string`
- `NULL` → union with null，e.g., `string | null`

### 命名規則
- DB 欄位是 PascalCase → TypeScript 屬性用 **camelCase**
- e.g., `MenuInfoId` → `menuInfoId`、`IsActive` → `isActive`

---

## Step 3: 產生檔案

檔名：`{camelCaseModelName}.ts`（e.g., `company.ts`）
路徑：`src/app/core/models/{camelCaseModelName}.ts`

格式：

```typescript
// 基礎 DB Model - 對齊資料庫 (前端使用 camelCase)
export interface {ModelName} {
    {camelCaseField}: {TsType};           // {OriginalDbColumn}: {DbType} NOT NULL
    {camelCaseField}: {TsType} | null;    // {OriginalDbColumn}: {DbType} NULL
    // ...
}
```

每個欄位行末加上 `// {原始DB欄位名}: {DB型別} {NOT NULL | NULL}` 的 inline comment。

---

## 範例

**使用者輸入：**
```
CompanyInfoId uniqueidentifier NOT NULL
CompanyName nvarchar(100) NOT NULL
TaxId nvarchar(20) NULL
IsActive bit NOT NULL
CreatedAt datetime2 NOT NULL
```

**產出 `companyInfo.ts`：**
```typescript
// 基礎 DB Model - 對齊資料庫 (前端使用 camelCase)
export interface CompanyInfo {
    companyInfoId: string;        // CompanyInfoId: uniqueidentifier NOT NULL
    companyName: string;          // CompanyName: nvarchar(100) NOT NULL
    taxId: string | null;         // TaxId: nvarchar(20) NULL
    isActive: boolean;            // IsActive: bit NOT NULL
    createdAt: Date;              // CreatedAt: datetime2 NOT NULL
}
```

---

## Project context

- 路徑：`src/app/core/models/`
- 用途：對齊 DB，供各 feature 的 `models.ts` 以 `Pick<>` / `extends` / re-export 使用
- 命名：檔名與 interface 名稱一致（camelCase 檔名 / PascalCase interface）
