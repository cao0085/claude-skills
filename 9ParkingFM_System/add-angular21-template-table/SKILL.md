---
name: add-angular21-template-table
description: "Scaffold the index.html table list template for an Angular 21 feature. Use this skill whenever: user runs /add-angular21-template-table, asks to generate a table layout template, or wants to create a list page with header, filter bar, skeleton, nz-table (server-side pagination), and action buttons using ng-zorro components."
---

# Add Angular Table List Template

Generate `index/index.html` for a feature with a standard table list layout including filter bar and server-side pagination.

---

## Workflow

### Step 1: Collect Info

1. Read the following files before generating:
   - `index/index.ts` — extract: signals, filter form fields, CRUD methods, action code constants, pipe imports, `queryParams` shape
   - `models.ts` — find `Browse{Model}Dto` field list and any enums with options
2. If files are not provided, ask for:
   - Feature name and display title (e.g., `company`, `企業列表`)
   - The type used in the list signal (e.g., `BrowseCompanyDto`)
   - Available signals, methods, and filter fields from `index.ts`
   - Action code constant name (e.g., `COMPANY_ACTIONS`)
3. Confirm before generating:
   - Page title (e.g., `企業列表`)
   - Primary entity variable name used in `@for` (e.g., `company`)
   - Which fields to show in table
   - Which fields appear in the filter bar (and their input type: text / select / enum-select)

### Step 2: Determine Table Data Type and Columns

1. Read `index.ts` → find `{camelCaseModel}s` signal type
2. Go to `models.ts` and find `Browse{Model}Dto` fields
3. Apply rendering rules:

| Field type | Render as |
|---|---|
| `boolean` (e.g. `isActive`) | `<nz-tag>` with success/default color |
| Enum type (e.g. `companyType`) | `{{ value \| enumPipe }}` |
| `string` / `number` / `Date` | `{{ value }}` |
| Nullable (`string \| null`) | `{{ value ?? '-' }}` |

**Column width guidelines:**
- ID / code fields → `nzWidth="120px"`
- Short name / label → `nzWidth="160px"`
- Status / boolean → `nzWidth="100px"` + `nzAlign="center"`
- Action column → `nzWidth="150px"` + `nzAlign="center"`
- Long text / name → no width (auto expand)

### Step 3: Generate index.html

**Fixed structure (3 sections):**

---

#### Section 1 — Page Header

```html
<!-- ── Page Header ─────────────────────────────────── -->
<div nz-flex nzJustify="space-between" nzAlign="center" class="feature-page-header">
  <h3 class="page-title">{PageTitle}</h3>
  <button *appHasAction="{MODEL}_ACTIONS.WRITE" nz-button nzType="primary" (click)="isCreate{Model}ModalVisible.set(true)">
    <nz-icon nzType="plus" />新增
  </button>
</div>
```

---

#### Section 2 — Filter Bar

Place filter fields in `nz-col nzSpan="6"` cells (4 per row). The first row is always visible; extra fields use `[hidden]="isCollapse()"`.

```html
<!-- ── Filter Bar ──────────────────────────────────── -->
<div class="filter-bar">
  <form nz-form [formGroup]="filterForm" nzLayout="vertical">
    <div nz-row [nzGutter]="24">

      <!-- visible fields (first row) -->
      <div nz-col nzSpan="6">
        <nz-form-item>
          <nz-form-label>{FieldLabel}</nz-form-label>
          <nz-form-control>
            <input nz-input formControlName="{field}" placeholder="{placeholder}" />
          </nz-form-control>
        </nz-form-item>
      </div>

      <!-- enum select field -->
      <div nz-col nzSpan="6">
        <nz-form-item>
          <nz-form-label>{EnumFieldLabel}</nz-form-label>
          <nz-form-control>
            <nz-select formControlName="{enumField}" nzPlaceHolder="全部" nzAllowClear>
              @for (opt of {ENUM}_OPTIONS; track opt.value) {
                <nz-option [nzValue]="opt.value" [nzLabel]="opt.label" />
              }
            </nz-select>
          </nz-form-control>
        </nz-form-item>
      </div>

      <!-- collapsible fields (extra rows) -->
      <div nz-col nzSpan="6" [hidden]="isCollapse()">
        <nz-form-item>
          <nz-form-label>{FieldLabel}</nz-form-label>
          <nz-form-control>
            <input nz-input formControlName="{field}" placeholder="{placeholder}" />
          </nz-form-control>
        </nz-form-item>
      </div>

      <!-- isActive field (boolean select, always collapsible unless it's a key filter) -->
      <div nz-col nzSpan="6" [hidden]="isCollapse()">
        <nz-form-item>
          <nz-form-label>狀態</nz-form-label>
          <nz-form-control>
            <nz-select formControlName="isActive" nzPlaceHolder="全部" nzAllowClear>
              <nz-option [nzValue]="true" nzLabel="啟用" />
              <nz-option [nzValue]="false" nzLabel="停用" />
            </nz-select>
          </nz-form-control>
        </nz-form-item>
      </div>

    </div>
    <div nz-row>
      <div nz-col nzSpan="24" class="filter-actions">
        <button nz-button (click)="onReset()">清除</button>
        <button nz-button nzType="primary" (click)="onSearch()">搜尋</button>
        <a class="filter-toggle" (click)="isCollapse.update(v => !v)">
          {{ isCollapse() ? '展開' : '收起' }}
          <nz-icon [nzType]="isCollapse() ? 'down' : 'up'" />
        </a>
      </div>
    </div>
  </form>
</div>
```

---

#### Section 3 — Table (server-side pagination)

```html
<!-- ── Table ───────────────────────────────────────── -->
<nz-skeleton [nzLoading]="isLoading()" [nzActive]="true" [nzParagraph]="{ rows: 5 }">
  <nz-table
    #tableRef
    nzSize="default"
    nzShowSizeChanger
    [nzData]="{camelCaseModel}s()"
    [nzFrontPagination]="false"
    [nzTotal]="totalCount()"
    [nzPageIndex]="queryParams().pageNumber!"
    [nzPageSize]="queryParams().pageSize!"
    [nzShowTotal]="totalTpl"
    (nzQueryParams)="onQueryParamsChange($event)"
  >
    <ng-template #totalTpl let-total let-range="range">
      共 {{ total }} 筆，目前第 {{ range[0] }}-{{ range[1] }} 筆
    </ng-template>

    <thead>
      <tr>
        {columns}
        <th nzWidth="100px" nzAlign="center">狀態</th>
        <th nzWidth="150px" nzAlign="center">操作</th>
      </tr>
    </thead>

    <tbody>
      @for ({camelCaseModel} of tableRef.data; track {camelCaseModel}.id) {
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
            <button *appHasAction="{MODEL}_ACTIONS.MODIFY" nz-button nzType="link" nzSize="small" (click)="openEdit({camelCaseModel})">編輯</button>
            <nz-divider nzType="vertical" />
            <button *appHasAction="{MODEL}_ACTIONS.DELETE" nz-button nzType="link" nzSize="small" nzDanger (click)="delete{Model}({camelCaseModel})">刪除</button>
          </td>
        </tr>
      }
    </tbody>
  </nz-table>
</nz-skeleton>
```

---

#### Section 4 — Modals

```html
<!-- Create Modal -->
<app-create-{kebab-model}-modal
  [isVisible]="isCreate{Model}ModalVisible()"
  (visibleChange)="isCreate{Model}ModalVisible.set($event)"
  (create)="on{Model}Created($event)"
/>

<!-- Detail Modal -->
<app-detail-modal
  [{camelCaseModel}Id]="selected{Model}Id()"
  [editMode]="openInEditMode()"
  (closed)="selected{Model}Id.set(null)"
  (update)="on{Model}Updated($event)"
/>
```

> ⚠️ If the model has no `isActive` field, remove the status column entirely
> ⚠️ If no `openDetail()` method in index.ts, remove the detail modal block and detail button
> ⚠️ If no `openEdit()` method in index.ts, remove the edit button
> ⚠️ Collapse toggle only appears when there are more filter fields than fit in one row (> 4 fields)

---

## Example

**Input:** `admin/company`, filter fields: companyType (enum), companyCode, fullName, shortName (visible), businessAdministrationNumber + isActive (collapsible). Action code: `COMPANY_ACTIONS`.

**Output `src/app/features/admin/company/index/index.html`:**

```html
<!-- ── Page Header ─────────────────────────────────── -->
<div nz-flex nzJustify="space-between" nzAlign="center" class="feature-page-header">
  <h3 class="page-title">企業列表</h3>
  <button *appHasAction="COMPANY_ACTIONS.WRITE" nz-button nzType="primary" (click)="isCreateModalVisible.set(true)">
    <nz-icon nzType="plus" />新增
  </button>
</div>

<!-- ── Filter Bar ──────────────────────────────────── -->
<div class="filter-bar">
  <form nz-form [formGroup]="filterForm" nzLayout="vertical">
    <div nz-row [nzGutter]="24">
      <div nz-col nzSpan="6">
        <nz-form-item>
          <nz-form-label>企業類型</nz-form-label>
          <nz-form-control>
            <nz-select formControlName="companyType" nzPlaceHolder="全部" nzAllowClear>
              @for (opt of COMPANY_TYPE_OPTIONS; track opt.value) {
                <nz-option [nzValue]="opt.value" [nzLabel]="opt.label" />
              }
            </nz-select>
          </nz-form-control>
        </nz-form-item>
      </div>
      <div nz-col nzSpan="6">
        <nz-form-item>
          <nz-form-label>企業編碼</nz-form-label>
          <nz-form-control>
            <input nz-input formControlName="companyCode" placeholder="企業編碼" />
          </nz-form-control>
        </nz-form-item>
      </div>
      <div nz-col nzSpan="6">
        <nz-form-item>
          <nz-form-label>全名</nz-form-label>
          <nz-form-control>
            <input nz-input formControlName="fullName" placeholder="企業全名" />
          </nz-form-control>
        </nz-form-item>
      </div>
      <div nz-col nzSpan="6">
        <nz-form-item>
          <nz-form-label>企業簡稱</nz-form-label>
          <nz-form-control>
            <input nz-input formControlName="shortName" placeholder="企業簡稱" />
          </nz-form-control>
        </nz-form-item>
      </div>
      <div nz-col nzSpan="6" [hidden]="isCollapse()">
        <nz-form-item>
          <nz-form-label>統一編號</nz-form-label>
          <nz-form-control>
            <input nz-input formControlName="businessAdministrationNumber" placeholder="統一編號" />
          </nz-form-control>
        </nz-form-item>
      </div>
      <div nz-col nzSpan="6" [hidden]="isCollapse()">
        <nz-form-item>
          <nz-form-label>狀態</nz-form-label>
          <nz-form-control>
            <nz-select formControlName="isActive" nzPlaceHolder="全部" nzAllowClear>
              <nz-option [nzValue]="true" nzLabel="啟用" />
              <nz-option [nzValue]="false" nzLabel="停用" />
            </nz-select>
          </nz-form-control>
        </nz-form-item>
      </div>
    </div>
    <div nz-row>
      <div nz-col nzSpan="24" class="filter-actions">
        <button nz-button (click)="onReset()">清除</button>
        <button nz-button nzType="primary" (click)="onSearch()">搜尋</button>
        <a class="filter-toggle" (click)="isCollapse.update(v => !v)">
          {{ isCollapse() ? '展開' : '收起' }}
          <nz-icon [nzType]="isCollapse() ? 'down' : 'up'" />
        </a>
      </div>
    </div>
  </form>
</div>

<!-- ── Table ───────────────────────────────────────── -->
<nz-skeleton [nzLoading]="isLoading()" [nzActive]="true" [nzParagraph]="{ rows: 5 }">
  <nz-table
    #tableRef
    nzSize="default"
    nzShowSizeChanger
    [nzData]="companies()"
    [nzFrontPagination]="false"
    [nzTotal]="totalCount()"
    [nzPageIndex]="queryParams().pageNumber!"
    [nzPageSize]="queryParams().pageSize!"
    [nzShowTotal]="totalTpl"
    (nzQueryParams)="onQueryParamsChange($event)"
  >
    <ng-template #totalTpl let-total let-range="range">
      共 {{ total }} 筆，目前第 {{ range[0] }}-{{ range[1] }} 筆
    </ng-template>

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
      @for (company of tableRef.data; track company.id) {
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
            <button *appHasAction="COMPANY_ACTIONS.MODIFY" nz-button nzType="link" nzSize="small" (click)="openEdit(company)">編輯</button>
            <nz-divider nzType="vertical" />
            <button *appHasAction="COMPANY_ACTIONS.DELETE" nz-button nzType="link" nzSize="small" nzDanger (click)="deleteCompany(company)">刪除</button>
          </td>
        </tr>
      }
    </tbody>
  </nz-table>
</nz-skeleton>

<!-- Create Modal -->
<app-create-company-modal
  [isVisible]="isCreateModalVisible()"
  (visibleChange)="isCreateModalVisible.set($event)"
  (create)="onCompanyCreated($event)"
/>

<!-- Detail Modal -->
<app-detail-modal
  [companyId]="selectedCompanyId()"
  [editMode]="openInEditMode()"
  (closed)="selectedCompanyId.set(null)"
  (update)="onCompanyUpdated($event)"
/>
```

---

## Project Context

- Always use `nz-skeleton` wrapping `nz-table`, never put skeleton inside the table
- `@for` track by `.id` always
- Server-side pagination: `[nzFrontPagination]="false"` + bind `nzTotal`, `nzPageIndex`, `nzPageSize`, `(nzQueryParams)`
- `*appHasAction` on WRITE (新增 button), MODIFY (編輯 button), DELETE (刪除 button)
- Action code bindings use `protected readonly` constants from `index.ts`
- Enum fields must use the corresponding pipe — check `index.ts` imports for pipe name
- Filter bar: first row always visible, extra fields use `[hidden]="isCollapse()"` toggle
- Filter actions row always shows: 清除 / 搜尋 / 展開收起 link
- `nz-divider nzType="vertical"` between action buttons
- `nzDanger` on delete button, `nzType="link"` + `nzSize="small"` for all action buttons
- Modal component selectors use kebab-case matching the component folder name
- `[editMode]` binding on detail modal for edit/read-only mode toggle