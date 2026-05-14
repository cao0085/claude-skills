---
name: add-angular21-template-table
description: "Scaffold the index.html table list template for an Angular 21 feature. Use this skill whenever: user runs /add-angular21-template-table, asks to generate a table layout template, or wants to create a list page with header, skeleton, nz-table, and action buttons using ng-zorro components."
---

# Add Angular Table List Template

Generate `index/index.html` for a feature with a standard table list layout.

---

## Workflow

### Step 1: Collect Info

1. Read the following files before generating:
   - `index/index.ts` — extract: `refresh()` method, signal names, signal types, CRUD methods, pipe imports
   - `models.ts` — find the type used in `fetched{Xxx}` signal to resolve column fields
2. If files are not provided, ask for:
   - Feature name and display title (e.g., `company`, `企業列表`)
   - The type used in the list signal (e.g., `Notification`, `BrowseCompanyDto`)
   - Available signals and methods from `index.ts`
3. Confirm before generating:
   - Page title (e.g., `企業列表`)
   - Primary entity variable name used in `@for` (e.g., `company`)
   - Which fields to show in table (if type has many fields, ask user to pick key ones)

### Step 2: Determine Table Data Type and Columns

1. Read `index.ts` → find `refresh()` → locate the `fetched{Xxx}` signal type:
   ```typescript
   readonly fetched{Xxx} = signal<{Type}[]>([]);
   ```
2. Go to `models.ts` and find `{Type}` definition:
   - If `{Type}` is a `Pick<>` / `Omit<>` / `extends` DTO → use those fields only
   - If `{Type}` is the core model directly (mock situation) → use core model fields, skip audit fields (`createdAt`, `updatedAt`, `readAt`, etc.)

> ⚠️ Never assume the type is `BrowseXxxDto` — always trace from `refresh()` → signal type → `models.ts`

For each resolved field, apply the following rendering rules:

| Field type | Render as |
|---|---|
| `boolean` (e.g. `isActive`) | `<nz-tag>` with success/default color |
| Enum type (e.g. `companyType`) | `{{ value \| enumPipe }}` — check index.ts imports for pipe name |
| `string` / `number` / `Date` | `{{ value }}` |
| Nullable (`string \| null`) | `{{ value ?? '-' }}` |

**Column width guidelines:**
- ID / code fields → `nzWidth="120px"`
- Short name / label → `nzWidth="160px"`
- Status / boolean → `nzWidth="100px"` + `nzAlign="center"`
- Action column → `nzWidth="150px"` + `nzAlign="center"`
- Long text / name → no width (auto expand)

### Step 3: Generate index.html

**Fixed structure:**

```html
<!-- ── Page Header ─────────────────────────────────── -->
<div nz-flex nzJustify="space-between" nzAlign="center" class="feature-page-header">
  <h3 class="page-title">{PageTitle}</h3>
  <button nz-button nzType="primary" (click)="isCreate{Model}ModalVisible.set(true)">
    <nz-icon nzType="plus" />新增
  </button>
</div>

<!-- ── Table ───────────────────────────────────────── -->
<nz-skeleton [nzLoading]="isLoading()" [nzActive]="true" [nzParagraph]="{ rows: 5 }">
<nz-table
  #{camelCaseModel}Table
  [nzData]="fetched{Model}s()"
  nzSize="default"
  [nzPageSize]="10"
  [nzShowSizeChanger]="true"
>
  <thead>
    <tr>
      {columns}
      <th nzWidth="100px" nzAlign="center">狀態</th>
      <th nzWidth="150px" nzAlign="center">操作</th>
    </tr>
  </thead>
  <tbody>
    @for ({camelCaseModel} of {camelCaseModel}Table.data; track {camelCaseModel}.id) {
      <tr>
        {cells}
        <td nzAlign="center">
          <nz-tag [nzColor]="{camelCaseModel}.isActive ? 'success' : 'default'">
            {{ {camelCaseModel}.isActive ? '啟用' : '停用' }}
          </nz-tag>
        </td>
        <td nzAlign="center">
          <button nz-button nzType="link" nzSize="small" (click)="openDetail({camelCaseModel})">詳細</button>
          <nz-divider nzType="vertical" />
          <button nz-button nzType="link" nzSize="small" nzDanger (click)="delete{Model}({camelCaseModel})">刪除</button>
        </td>
      </tr>
    }
  </tbody>
</nz-table>
</nz-skeleton>

<!-- ── Create Modal ────────────────────────────────── -->
<app-create-{kebab-model}-modal
  [isVisible]="isCreate{Model}ModalVisible()"
  (visibleChange)="isCreate{Model}ModalVisible.set($event)"
  (create)="on{Model}Created($event)"
/>

<!-- ── Detail Modal ────────────────────────────────── -->
<app-detail-modal
  [{camelCaseModel}Id]="selected{Model}Id()"
  (closed)="selected{Model}Id.set(null)"
  (update)="on{Model}Updated($event)"
/>
```

> ⚠️ If the model has no `isActive` field, remove the status column entirely
> ⚠️ If no `openDetail()` method in index.ts, remove the detail modal block and detail button
> ⚠️ Inline styles are only acceptable for one-off spacing; prefer `var(--xxx)` CSS variables for any sizing/color values

---

## Example

**Input:** `admin/company`, `refresh()` → `fetchedCompanies = signal<BrowseCompanyDto[]>` → `BrowseCompanyDto` has: `id`, `companyCode`, `fullName`, `shortName`, `businessAdministrationNumber`, `companyType` (enum, pipe: `companyType`), `isActive`

**Output `src/app/features/admin/company/index/index.html`:**

```html
<!-- ── Page Header ─────────────────────────────────── -->
<div nz-flex nzJustify="space-between" nzAlign="center" class="feature-page-header">
  <h3 class="page-title">企業列表</h3>
  <button nz-button nzType="primary" (click)="isCreateModalVisible.set(true)">
    <nz-icon nzType="plus" />新增
  </button>
</div>

<!-- ── Table ───────────────────────────────────────── -->
<nz-skeleton [nzLoading]="isLoading()" [nzActive]="true" [nzParagraph]="{ rows: 5 }">
<nz-table
  #companyTable
  [nzData]="fetchedCompanies()"
  nzSize="default"
  [nzPageSize]="10"
  [nzShowSizeChanger]="true"
>
  <thead>
    <tr>
      <th nzWidth="120px">企業編碼</th>
      <th>全名</th>
      <th nzWidth="160px">簡稱</th>
      <th nzWidth="120px">統一編號</th>
      <th nzWidth="140px">類型</th>
      <th nzWidth="100px" nzAlign="center">狀態</th>
      <th nzWidth="150px" nzAlign="center">操作</th>
    </tr>
  </thead>
  <tbody>
    @for (company of companyTable.data; track company.id) {
      <tr>
        <td>{{ company.companyCode }}</td>
        <td>{{ company.fullName }}</td>
        <td>{{ company.shortName }}</td>
        <td>{{ company.businessAdministrationNumber }}</td>
        <td>{{ company.companyType | companyType }}</td>
        <td nzAlign="center">
          <nz-tag [nzColor]="company.isActive ? 'success' : 'default'">
            {{ company.isActive ? '啟用' : '停用' }}
          </nz-tag>
        </td>
        <td nzAlign="center">
          <button nz-button nzType="link" nzSize="small" (click)="openDetail(company)">詳細</button>
          <nz-divider nzType="vertical" />
          <button nz-button nzType="link" nzSize="small" nzDanger (click)="deleteCompany(company)">刪除</button>
        </td>
      </tr>
    }
  </tbody>
</nz-table>
</nz-skeleton>

<!-- ── Create Modal ────────────────────────────────── -->
<app-create-company-modal
  [isVisible]="isCreateModalVisible()"
  (visibleChange)="isCreateModalVisible.set($event)"
  (create)="onCompanyCreated($event)"
/>

<!-- ── Detail Modal ────────────────────────────────── -->
<app-detail-modal
  [companyId]="selectedCompanyId()"
  (closed)="selectedCompanyId.set(null)"
  (update)="onCompanyUpdated($event)"
/>
```

---

## Project Context

- Always use `nz-skeleton` wrapping `nz-table`, never put skeleton inside the table
- `@for` track by `.id` always
- Enum fields must use the corresponding pipe — check `index.ts` imports for pipe name
- Inline style only for one-off values; use `var(--margin-md)` etc. from global SCSS variables for spacing/color
- Modal component selectors use kebab-case matching the component folder name
- `nzDanger` on delete button, `nzType="link"` + `nzSize="small"` for all action buttons
- `nz-divider nzType="vertical"` between action buttons