---
name: add-angular21-template-table
description: "Scaffold the index.html table list template for an Angular 21 feature. Use this skill whenever: user runs /add-angular21-template-table, asks to generate a table layout template, or wants to create a list page with header, filter bar, skeleton, nz-table (server-side pagination), and action buttons using ng-zorro components."
---

# Add Angular Table List Template

Generate `index/index.html` for a feature with a standard table list layout: filter bar and table are both **card panels** (global `.app-panel .app-raised-surface`), filter cols are responsive (`appRwdCol`), and row actions use a "詳細 + 更多 ⋯" pattern.

---

## Global CSS building blocks (from `theme.less`)

| Class / directive | Where | Purpose |
|---|---|---|
| `feature-page-header` | page header row | title + primary button, bottom divider |
| `filter-bar app-panel app-raised-surface` | filter container | card surface (bg + radius + shadow + responsive padding) |
| `appRwdCol="quarter"` | each filter `nz-col` | responsive span (xs=24 / sm=12 / lg=6) — replaces static `nzSpan` |
| `filter-actions` | actions row | flex-end; buttons go full-width on mobile |
| `app-panel app-raised-surface` | table `<section>` wrapper | same card surface as filter bar |
| `app-table-skeleton` | `nz-skeleton` wrapping the table | lets the fixed-width table scroll horizontally inside flex |

> **Permission codes are hardcoded string literals** — `*appHasAction="'FEATURE:ACTION'"`. No `XXX_ACTIONS` constant.

---

## Workflow

### Step 1: Collect Info

1. Read the following files before generating:
   - `index/index.ts` — extract: signals, filter form fields, CRUD methods, pipe imports, `queryParams` shape
   - `models.ts` — find `Browse{Model}Dto` field list and any enums with options
2. If files are not provided, ask for:
   - Feature name and display title (e.g., `company`, `企業列表`)
   - The type used in the list signal (e.g., `BrowseCompanyDto`)
   - Available signals, methods, and filter fields from `index.ts`
   - Backend permission feature code (e.g., `COMPANY`)
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

**Column width rule (important):** give **every** `<th>` an explicit `nzWidth` and set `[nzScroll]="{ x: '<sum>px' }"` to (at least) the sum of the widths. Do **not** leave columns auto-width when using `nzScroll` — mixing auto + fixed columns causes a spurious horizontal scrollbar after container/sidebar resize.

- ID / code fields → `120px`
- Name / long text → `200px`
- Short label → `100px`
- Status / boolean → `80px` + `nzAlign="center"`
- Action column → `100px` + `nzAlign="center"`

### Step 3: Generate index.html

#### Section 1 — Page Header

```html
<!-- ── Page Header ─────────────────────────────────── -->
<div nz-flex nzJustify="space-between" nzAlign="center" class="feature-page-header">
  <h3 class="page-title">{PageTitle}</h3>
  <button *appHasAction="'{FEATURE}:WRITE'" nz-button nzType="primary" (click)="isCreate{Model}ModalVisible.set(true)">
    <nz-icon nzType="plus" />新增
  </button>
</div>
```

#### Section 2 — Filter Bar (card panel + responsive cols)

Each filter field is an `<div nz-col appRwdCol="quarter">` (4 per row on desktop, 2 on tablet, 1 on mobile). First row visible; extra fields use `[hidden]="isCollapse()"`.

```html
<!-- ── Filter Bar ──────────────────────────────────── -->
<div class="filter-bar app-panel app-raised-surface">
  <form nz-form [formGroup]="filterForm" nzLayout="vertical">
    <div nz-row [nzGutter]="24">

      <!-- enum select field -->
      <div nz-col appRwdCol="quarter">
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

      <!-- text field -->
      <div nz-col appRwdCol="quarter">
        <nz-form-item>
          <nz-form-label>{FieldLabel}</nz-form-label>
          <nz-form-control>
            <input nz-input formControlName="{field}" placeholder="{placeholder}" />
          </nz-form-control>
        </nz-form-item>
      </div>

      <!-- collapsible field -->
      <div nz-col appRwdCol="quarter" [hidden]="isCollapse()">
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
        <button nz-button (click)="onReset()">清除篩選</button>
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

#### Section 3 — Table (card panel + skeleton + server pagination)

Wrap the table in a `<section class="app-panel app-raised-surface">`; the skeleton carries `class="app-table-skeleton"`; the table has `[nzScroll]`.

```html
<!-- ── Table ───────────────────────────────────────── -->
<section class="app-panel app-raised-surface">
  <nz-skeleton class="app-table-skeleton" [nzLoading]="isLoading()" [nzActive]="true" [nzParagraph]="{ rows: 5 }">
    <nz-table
      #tableRef
      nzSize="default"
      nzShowSizeChanger
      [nzData]="{camelCaseModel}s()"
      [nzFrontPagination]="false"
      [nzTotal]="totalCount()"
      [nzPageIndex]="queryParams().pageNumber!"
      [nzPageSize]="queryParams().pageSize!"
      [nzScroll]="{ x: '{sumPx}px' }"
      [nzShowTotal]="totalTpl"
      (nzQueryParams)="onQueryParamsChange($event)"
    >
      <ng-template #totalTpl let-total let-range="range">
        共 {{ total }} 筆，目前第 {{ range[0] }}-{{ range[1] }} 筆
      </ng-template>

      <thead>
        <tr>
          {columns}
          <th nzWidth="80px" nzAlign="center">狀態</th>
          <th nzWidth="100px" nzAlign="center">操作</th>
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
              <!-- 更多操作：至少有一項權限才顯示 ⋯ -->
              <a
                *appHasAction="['{FEATURE}:MODIFY', '{FEATURE}:DELETE']"
                nz-dropdown
                [nzDropdownMenu]="rowActions"
                [nzPlacement]="'bottomRight'"
              >
                <nz-icon nzType="ellipsis" nzTheme="outline" />
              </a>
              <nz-dropdown-menu #rowActions="nzDropdownMenu">
                <ul nz-menu>
                  <li *appHasAction="'{FEATURE}:MODIFY'" nz-menu-item (click)="openEdit({camelCaseModel})">編輯</li>
                  <li *appHasAction="'{FEATURE}:DELETE'" nz-menu-item nzDanger (click)="delete{Model}({camelCaseModel})">刪除</li>
                </ul>
              </nz-dropdown-menu>
            </td>
          </tr>
        }
      </tbody>
    </nz-table>
  </nz-skeleton>
</section>
```

> The `⋯` dropdown requires `NzDropdownModule` in `index.ts` imports.
> Row-action rule: keep the most-used, non-destructive action (`詳細`) inline; put the rest (`編輯` / `刪除`) in the `⋯` dropdown. The `⋯` trigger uses the **array form** `*appHasAction="['{FEATURE}:MODIFY','{FEATURE}:DELETE']"` (OR semantics) so it hides entirely when the user has none of them. Destructive item (`刪除`) is last with `nzDanger`.
> If there are ≤ 2 actions total, you may keep them inline with `nz-divider nzType="vertical"` between them instead of a dropdown.

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
> ⚠️ If no `openEdit()` / `delete{Model}()`, drop the corresponding dropdown item (and the whole `⋯` block if none remain)
> ⚠️ Collapse toggle only appears when there are more filter fields than fit in one row (> 4 fields)

---

## Example

**Input:** `admin/company`, filter fields: companyType (enum), companyCode, fullName, shortName (visible), businessAdministrationNumber + isActive (collapsible). Permission feature: `COMPANY`.

**Output `src/app/features/admin/company/index/index.html`:**

```html
<!-- ── Page Header ─────────────────────────────────── -->
<div nz-flex nzJustify="space-between" nzAlign="center" class="feature-page-header">
  <h3 class="page-title">企業列表</h3>
  <button *appHasAction="'COMPANY:WRITE'" nz-button nzType="primary" (click)="isCreateModalVisible.set(true)">
    <nz-icon nzType="plus" />新增
  </button>
</div>

<!-- ── Filter Bar ──────────────────────────────────── -->
<div class="filter-bar app-panel app-raised-surface">
  <form nz-form [formGroup]="filterForm" nzLayout="vertical">
    <div nz-row [nzGutter]="24">
      <div nz-col appRwdCol="quarter">
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
      <div nz-col appRwdCol="quarter">
        <nz-form-item>
          <nz-form-label>企業編碼</nz-form-label>
          <nz-form-control>
            <input nz-input formControlName="companyCode" placeholder="企業編碼" />
          </nz-form-control>
        </nz-form-item>
      </div>
      <div nz-col appRwdCol="quarter">
        <nz-form-item>
          <nz-form-label>全名</nz-form-label>
          <nz-form-control>
            <input nz-input formControlName="fullName" placeholder="企業全名" />
          </nz-form-control>
        </nz-form-item>
      </div>
      <div nz-col appRwdCol="quarter">
        <nz-form-item>
          <nz-form-label>簡稱</nz-form-label>
          <nz-form-control>
            <input nz-input formControlName="shortName" placeholder="企業簡稱" />
          </nz-form-control>
        </nz-form-item>
      </div>
      <div nz-col appRwdCol="quarter" [hidden]="isCollapse()">
        <nz-form-item>
          <nz-form-label>統一編號</nz-form-label>
          <nz-form-control>
            <input nz-input formControlName="businessAdministrationNumber" placeholder="統一編號" />
          </nz-form-control>
        </nz-form-item>
      </div>
      <div nz-col appRwdCol="quarter" [hidden]="isCollapse()">
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
        <button nz-button (click)="onReset()">清除篩選</button>
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
<section class="app-panel app-raised-surface">
  <nz-skeleton class="app-table-skeleton" [nzLoading]="isLoading()" [nzActive]="true" [nzParagraph]="{ rows: 5 }">
    <nz-table
      #tableRef
      nzSize="default"
      nzShowSizeChanger
      [nzData]="companies()"
      [nzFrontPagination]="false"
      [nzTotal]="totalCount()"
      [nzPageIndex]="queryParams().pageNumber!"
      [nzPageSize]="queryParams().pageSize!"
      [nzScroll]="{ x: '860px' }"
      [nzShowTotal]="totalTpl"
      (nzQueryParams)="onQueryParamsChange($event)"
    >
      <ng-template #totalTpl let-total let-range="range">
        共 {{ total }} 筆，目前第 {{ range[0] }}-{{ range[1] }} 筆
      </ng-template>

      <thead>
        <tr>
          <th nzWidth="120px">企業編碼</th>
          <th nzWidth="200px">全名</th>
          <th nzWidth="100px">簡稱</th>
          <th nzWidth="120px">統一編號</th>
          <th nzWidth="140px">類型</th>
          <th nzWidth="80px" nzAlign="center">狀態</th>
          <th nzWidth="100px" nzAlign="center">操作</th>
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
              <!-- 更多操作：至少有一項權限才顯示 ⋯ -->
              <a
                *appHasAction="['COMPANY:MODIFY', 'COMPANY:DELETE']"
                nz-dropdown
                [nzDropdownMenu]="rowActions"
                [nzPlacement]="'bottomRight'"
              >
                <nz-icon nzType="ellipsis" nzTheme="outline" />
              </a>
              <nz-dropdown-menu #rowActions="nzDropdownMenu">
                <ul nz-menu>
                  <li *appHasAction="'COMPANY:MODIFY'" nz-menu-item (click)="openEdit(company)">編輯</li>
                  <li *appHasAction="'COMPANY:DELETE'" nz-menu-item nzDanger (click)="deleteCompany(company)">刪除</li>
                </ul>
              </nz-dropdown-menu>
            </td>
          </tr>
        }
      </tbody>
    </nz-table>
  </nz-skeleton>
</section>

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

- Filter bar and table are **both card panels** — `filter-bar app-panel app-raised-surface` and a `<section class="app-panel app-raised-surface">` around the skeleton
- Filter cols use `appRwdCol="quarter"` (not static `nzSpan`); requires `RwdColDirective` in `index.ts`
- `nz-skeleton` wrapping `nz-table` carries `class="app-table-skeleton"` and the table always has `[nzScroll]`; **every column has an explicit `nzWidth`** (avoids the resize scrollbar bug)
- Permission checks are **hardcoded strings**: `*appHasAction="'FEATURE:ACTION'"`; the `⋯` trigger uses the array form for OR semantics
- Row actions: `詳細` inline + `編輯`/`刪除` in a `⋯` dropdown (`NzDropdownModule`); `刪除` is `nzDanger`, last
- `@for` track by `.id` always
- Server-side pagination: `[nzFrontPagination]="false"` + bind `nzTotal`, `nzPageIndex`, `nzPageSize`, `(nzQueryParams)`
- Enum fields must use the corresponding pipe — check `index.ts` imports for pipe name
- Filter bar: first row always visible, extra fields use `[hidden]="isCollapse()"`; actions row shows 清除篩選 / 搜尋 / 展開收起
- Modal component selectors use kebab-case matching the component folder name; `[editMode]` toggles detail modal edit/read-only
