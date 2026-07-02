---
name: add-angular21-index
description: "Scaffold the index.ts component for an Angular 21 feature. Use this skill whenever: user runs /add-angular21-index, asks to initialize or fill in an index component, or wants to wire up an api.ts into a component with signals, CRUD methods, search filter form, and server-side pagination."
---

# Add Angular Index Component

Fill in `index/index.ts` by reading the feature's `api.ts` and `models.ts`.

---

## Workflow

### Step 1: Collect Info

1. Read the following files before generating:
   - `{featureName}.api.ts` — extract all public methods and their signatures
   - `models.ts` — extract DTO types, `Browse{Model}Params`, enum options used in filter form
2. If files are not provided, ask for:
   - Feature group and feature name (e.g., group: `admin`, feature: `company`)
   - List of API methods available
   - Which fields are filterable
3. Confirm the following before generating:
   - Component class name for `index.ts` (e.g., `CompanyManagement`) — must match `app.routes.ts` loadComponent
   - Feature path (e.g., `src/app/features/admin/company/`)
   - Route path (e.g., `basic/company`) — format: `{group}/{featureName}`
   - The backend permission feature code (e.g., `COMPANY`) — used as hardcoded `'COMPANY:WRITE'` strings **in the template only**
   - Route group display name if group is new (e.g., `基礎系統`)

> **Permission codes are hardcoded string literals in the HTML** (e.g. `*appHasAction="'COMPANY:WRITE'"`). There is **no** `action-codes.ts` constant and **no** `protected readonly XXX_ACTIONS` on the component. `index.ts` does not import or expose any action constant.

### Step 2: Analyze api.ts

Scan public methods and map to patterns:

| Method pattern | Generates |
|---|---|
| `getXxx(params)` | `queryParams` signal + `xxxs` signal + `totalCount` signal + `isLoading` signal + `filterForm` + `onSearch()` + `onReset()` + `onQueryParamsChange()` + `refresh()` |
| `createXxx()` | `isCreateXxxModalVisible` signal + `onXxxCreated()` |
| `xxxDetailResource()` | `selectedXxxId` signal + `openDetail()` + `openEdit()` + `openInEditMode` signal + `onXxxUpdated()` |
| `deleteXxx()` | `deleteXxx()` using injected `ConfirmDeleteService` |

### Step 3: Generate index.ts

**Fixed imports (always included):**
```typescript
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { withSkeletonDelay } from '@app/shared/utils/with-skeleton-delay';
import { CommonModule } from '@angular/common';
import { Component, inject, signal, OnInit, DestroyRef } from '@angular/core';
import { NonNullableFormBuilder, ReactiveFormsModule } from '@angular/forms';
import { HttpErrorResponse } from '@angular/common/http';
import { toHttpParams } from '@app/core/utils/http-params.utils';
import { HasActionDirective } from '@app/shared/directives/has-action.directive';
import { RwdColDirective } from '@app/shared/directives/rwd-col.directive';
import { ConfirmDeleteService } from '@app/shared/components/confirm-delete/confirm-delete.service';
```

> `HasActionDirective` + `RwdColDirective` are always in the component `imports` array (template uses `*appHasAction` and `appRwdCol`). If the table has a "更多操作 ⋯" dropdown, also add `NzDropdownModule`. **Do not** import `NzModalModule` / `NzModalService` just for delete — deletion goes through the injected `ConfirmDeleteService`.

**Injected members:**
```typescript
private fb = inject(NonNullableFormBuilder);
private {camelCaseModel}Api = inject({ServiceName});
private destroyRef = inject(DestroyRef);
private confirmDelete = inject(ConfirmDeleteService); // only if deleteXxx exists
private message = inject(NzMessageService);
```

**Signal groups:**

```typescript
/**
 * SOT - Request
 */
readonly queryParams = signal<Browse{Model}Params>({
    pageNumber: 1,
    pageSize: 10,
    sortOrder: undefined,
    orderColumn: undefined,
    // ...filterable fields: undefined
});

/**
 * SOT - Response
 */
readonly {camelCaseModel}s = signal<Browse{Model}Dto[]>([]);
readonly totalCount = signal(0);
readonly selected{Model}Id = signal<string | null>(null); // only if detail exists

/**
 * UI State
 */
readonly isLoading = signal(true);
readonly isCreate{Model}ModalVisible = signal(false); // only if create exists
readonly isCollapse = signal(true);
readonly openInEditMode = signal(false); // only if detail exists
```

**Filter Form:**

```typescript
/**
 * Filter Form
 */
readonly filterForm = this.fb.group({
    // string fields
    {field}: [''],
    // enum fields
    {enumField}: [null as {EnumType} | null],
    // boolean fields
    isActive: [null as boolean | null],
});
```

**Search / Reset / Pagination methods:**

```typescript
onSearch(): void {
    const v = this.filterForm.value;
    this.queryParams.update(q => ({
        ...q,
        pageNumber: 1,
        {field}: v.{field} || undefined,
        {enumField}: v.{enumField} ?? undefined,
        isActive: v.isActive ?? undefined,
    }));
    this.refresh();
}

onReset(): void {
    this.filterForm.reset();
    this.queryParams.update(q => ({ pageNumber: 1, pageSize: q.pageSize }));
    this.refresh();
}

onQueryParamsChange(params: NzTableQueryParams): void {
    this.queryParams.update(q => ({
        ...q,
        pageNumber: params.pageIndex,
        pageSize: params.pageSize,
    }));
    this.refresh();
}
```

**CRUD method template:**

```typescript
// ==================== Create ====================
on{Model}Created(body: Create{Model}ReqBody) {
    this.{camelCaseModel}Api
        .create{Model}(body)
        .pipe(takeUntilDestroyed(this.destroyRef))
        .subscribe({
            next: () => {
                this.isCreate{Model}ModalVisible.set(false);
                this.message.success('新增成功');
                this.refresh();
            },
            error: () => this.message.error('新增失敗'),
        });
}

// ==================== Detail ====================
openDetail({camelCaseModel}: Browse{Model}Dto) {
    this.openInEditMode.set(false);
    this.selected{Model}Id.set({camelCaseModel}.id);
}

openEdit({camelCaseModel}: Browse{Model}Dto) {
    this.openInEditMode.set(true);
    this.selected{Model}Id.set({camelCaseModel}.id);
}

on{Model}Updated(body: Update{Model}ReqBody) {
    this.{camelCaseModel}Api
        .update{Model}(body)
        .pipe(takeUntilDestroyed(this.destroyRef))
        .subscribe({
            next: () => {
                this.selected{Model}Id.set(null);
                this.message.success('更新成功');
                this.refresh();
            },
            error: () => this.message.error('更新失敗'),
        });
}

// ==================== Delete ====================
// 用共用 ConfirmDeleteService；重要資料傳 keyword → 需重打該值才能確認
delete{Model}({camelCaseModel}: Browse{Model}Dto) {
    this.confirmDelete
        .open({
            title: '刪除{中文名}',
            message: `確定要刪除「${{camelCaseModel}.{labelField}}」嗎？此操作無法復原。`,
            keyword: {camelCaseModel}.{keyField},   // 一般刪除可省略此兩行
            keywordLabel: '{keyField 中文}',
        })
        .pipe(takeUntilDestroyed(this.destroyRef))
        .subscribe((ok) => {
            if (!ok) return;
            this.{camelCaseModel}Api
                .delete{Model}({camelCaseModel}.id)
                .pipe(takeUntilDestroyed(this.destroyRef))
                .subscribe({
                    next: () => {
                        this.message.success('刪除成功');
                        this.refresh();
                    },
                    error: (err: HttpErrorResponse) => {
                        const detail =
                            err.status === 422
                                ? (err.error?.detail ?? '刪除失敗')
                                : '刪除失敗';
                        this.message.error(detail);
                    },
                });
        });
}

// ==================== Private ====================
private refresh() {
    const q = this.queryParams();

    this.{camelCaseModel}Api
        .get{Model}s(toHttpParams(q))
        .pipe(
            withSkeletonDelay(v => this.isLoading.set(v)),
            takeUntilDestroyed(this.destroyRef),
        )
        .subscribe(result => {
            this.{camelCaseModel}s.set(result.items);
            this.totalCount.set(result.totalCount);
            // 後端有修正分頁時才同步回 SOT，避免無謂 re-fetch
            if (result.pageNumber !== q.pageNumber || result.pageSize !== q.pageSize) {
                this.queryParams.update(p => ({ ...p, pageNumber: result.pageNumber, pageSize: result.pageSize }));
            }
        });
}
```

> ⚠️ `refresh()` assumes `get{Model}s()` returns `PaginationResult<Browse{Model}Dto>`. If the backend still returns a bare array, temporarily map it: `this.{camelCaseModel}s.set(result); this.totalCount.set(result.length);` and drop the page-sync block until the backend is updated.

---

## Example

**Input:** `admin/company`, api.ts has `getCompanies(params)`, `createCompany`, `companyDetailResource`, `updateCompany`, `deleteCompany`. Permission feature: `COMPANY`.

**Output `src/app/features/admin/company/index/index.ts`:**

```typescript
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { withSkeletonDelay } from '@app/shared/utils/with-skeleton-delay';
import { CommonModule } from '@angular/common';
import { Component, inject, signal, OnInit, DestroyRef } from '@angular/core';
import { NonNullableFormBuilder, ReactiveFormsModule } from '@angular/forms';
import { HttpErrorResponse } from '@angular/common/http';
import { toHttpParams } from '@app/core/utils/http-params.utils';

import {
    BrowseCompanyDto,
    BrowseCompanyParams,
    CreateCompanyReqBody,
    UpdateCompanyReqBody,
    CompanyTypePipe,
    COMPANY_TYPE_OPTIONS,
    CompanyType,
} from '../models';
import { HasActionDirective } from '@app/shared/directives/has-action.directive';
import { RwdColDirective } from '@app/shared/directives/rwd-col.directive';
import { ConfirmDeleteService } from '@app/shared/components/confirm-delete/confirm-delete.service';
import { CompanyManagementApi } from '../company-management.api';
import { CreateCompanyModal } from '../create-company-modal/create-company-modal';
import { DetailModal } from '../detail-modal/detail-modal';

import { NzButtonModule } from 'ng-zorro-antd/button';
import { NzDividerModule } from 'ng-zorro-antd/divider';
import { NzDropdownModule } from 'ng-zorro-antd/dropdown';
import { NzFlexModule } from 'ng-zorro-antd/flex';
import { NzFormModule } from 'ng-zorro-antd/form';
import { NzGridModule } from 'ng-zorro-antd/grid';
import { NzIconModule } from 'ng-zorro-antd/icon';
import { NzInputModule } from 'ng-zorro-antd/input';
import { NzMessageService } from 'ng-zorro-antd/message';
import { NzSelectModule } from 'ng-zorro-antd/select';
import { NzSkeletonModule } from 'ng-zorro-antd/skeleton';
import { NzTableModule, NzTableQueryParams } from 'ng-zorro-antd/table';
import { NzTagModule } from 'ng-zorro-antd/tag';

@Component({
    selector: 'app-index',
    standalone: true,
    providers: [CompanyManagementApi, NzMessageService],
    imports: [
        CommonModule,
        ReactiveFormsModule,
        CreateCompanyModal,
        DetailModal,
        CompanyTypePipe,
        NzButtonModule,
        NzDividerModule,
        NzDropdownModule,
        NzFlexModule,
        NzFormModule,
        NzGridModule,
        NzIconModule,
        NzInputModule,
        NzSelectModule,
        NzSkeletonModule,
        NzTableModule,
        NzTagModule,
        HasActionDirective,
        RwdColDirective,
    ],
    templateUrl: './index.html',
    styleUrl: './index.scss',
})
export class CompanyManagement implements OnInit {
    private fb = inject(NonNullableFormBuilder);
    private companyApi = inject(CompanyManagementApi);
    private destroyRef = inject(DestroyRef);
    private confirmDelete = inject(ConfirmDeleteService);
    private message = inject(NzMessageService);

    readonly COMPANY_TYPE_OPTIONS = COMPANY_TYPE_OPTIONS;

    /**
     * SOT - Request
     */
    readonly queryParams = signal<BrowseCompanyParams>({
        pageNumber: 1,
        pageSize: 10,
        sortOrder: undefined,
        orderColumn: undefined,
        companyCode: undefined,
        fullName: undefined,
        shortName: undefined,
        businessAdministrationNumber: undefined,
        companyType: undefined,
        isActive: undefined,
    });

    /**
     * SOT - Response
     */
    readonly companies = signal<BrowseCompanyDto[]>([]);
    readonly totalCount = signal(0);
    readonly selectedCompanyId = signal<string | null>(null);

    /**
     * UI State
     */
    readonly isLoading = signal(true);
    readonly isCreateModalVisible = signal(false);
    readonly isCollapse = signal(true);
    readonly openInEditMode = signal(false);

    /**
     * Filter Form
     */
    readonly filterForm = this.fb.group({
        companyCode: [''],
        fullName: [''],
        shortName: [''],
        businessAdministrationNumber: [''],
        companyType: [null as CompanyType | null],
        isActive: [null as boolean | null],
    });

    ngOnInit(): void {
        this.refresh();
    }

    onSearch(): void {
        const v = this.filterForm.value;
        this.queryParams.update(q => ({
            ...q,
            pageNumber: 1,
            companyCode: v.companyCode || undefined,
            fullName: v.fullName || undefined,
            shortName: v.shortName || undefined,
            businessAdministrationNumber: v.businessAdministrationNumber || undefined,
            companyType: v.companyType ?? undefined,
            isActive: v.isActive ?? undefined,
        }));
        this.refresh();
    }

    onReset(): void {
        this.filterForm.reset();
        this.queryParams.update(q => ({ pageNumber: 1, pageSize: q.pageSize }));
        this.refresh();
    }

    onQueryParamsChange(params: NzTableQueryParams): void {
        this.queryParams.update(q => ({
            ...q,
            pageNumber: params.pageIndex,
            pageSize: params.pageSize,
        }));
        this.refresh();
    }

    // ==================== Create ====================
    onCompanyCreated(body: CreateCompanyReqBody) {
        this.companyApi
            .createCompany(body)
            .pipe(takeUntilDestroyed(this.destroyRef))
            .subscribe({
                next: () => {
                    this.isCreateModalVisible.set(false);
                    this.message.success('新增成功');
                    this.refresh();
                },
                error: () => this.message.error('新增失敗'),
            });
    }

    // ==================== Detail ====================
    openDetail(company: BrowseCompanyDto) {
        this.openInEditMode.set(false);
        this.selectedCompanyId.set(company.id);
    }

    openEdit(company: BrowseCompanyDto) {
        this.openInEditMode.set(true);
        this.selectedCompanyId.set(company.id);
    }

    onCompanyUpdated(body: UpdateCompanyReqBody) {
        this.companyApi
            .updateCompany(body)
            .pipe(takeUntilDestroyed(this.destroyRef))
            .subscribe({
                next: () => {
                    this.selectedCompanyId.set(null);
                    this.message.success('更新成功');
                    this.refresh();
                },
                error: () => this.message.error('更新失敗'),
            });
    }

    // ==================== Delete ====================
    deleteCompany(company: BrowseCompanyDto) {
        this.confirmDelete
            .open({
                title: '刪除企業',
                message: `確定要刪除「${company.fullName}」嗎？此操作無法復原。`,
                keyword: company.companyCode,
                keywordLabel: '企業編碼',
            })
            .pipe(takeUntilDestroyed(this.destroyRef))
            .subscribe((ok) => {
                if (!ok) return;
                this.companyApi
                    .deleteCompany(company.id)
                    .pipe(takeUntilDestroyed(this.destroyRef))
                    .subscribe({
                        next: () => {
                            this.message.success('刪除成功');
                            this.refresh();
                        },
                        error: (err: HttpErrorResponse) => {
                            const detail =
                                err.status === 422
                                    ? (err.error?.detail ?? '刪除失敗')
                                    : '刪除失敗';
                            this.message.error(detail);
                        },
                    });
            });
    }

    // ==================== Private ====================
    private refresh() {
        const q = this.queryParams();

        this.companyApi
            .getCompanies(toHttpParams(q))
            .pipe(
                withSkeletonDelay(v => this.isLoading.set(v)),
                takeUntilDestroyed(this.destroyRef),
            )
            .subscribe(result => {
                this.companies.set(result.items);
                this.totalCount.set(result.totalCount);
                // 後端有修正分頁時才同步回 SOT，避免無謂 re-fetch
                if (result.pageNumber !== q.pageNumber || result.pageSize !== q.pageSize) {
                    this.queryParams.update(p => ({ ...p, pageNumber: result.pageNumber, pageSize: result.pageSize }));
                }
            });
    }
}
```

### Step 4: Register Route in app.routes.ts

File: `src/app/app.routes.ts`

1. Find the `APP_FEATURE_ROUTES` array
2. Look for an existing group comment block matching `{group}` (e.g., `// ========== 基礎系統 ==========`)
   - **Found** → append the new route inside that block
   - **Not found** → create a new block at the end of `APP_FEATURE_ROUTES`
3. Insert the following route entry:

```typescript
{
    path: '{group}/{featureName}',
    canActivate: [menuRouteGuard],
    children: [
        {
            path: '',
            loadComponent: () =>
                import('@app/features/{group}/{feature-folder}/index').then(
                    (m) => m.{ClassName},
                ),
            data: { reuseRoute: true },
        },
    ],
},
```

> ⚠️ `{ClassName}` must match the `export class` name in `index/index.ts`

**New group block format (if group doesn't exist):**

```typescript
// ========================================
// {GroupDisplayName}
// ========================================
{
    path: '{group}/{featureName}',
    canActivate: [menuRouteGuard],
    ...
},
```

---

## Project Context

- `imports` in `@Component` — only include what the template actually uses
- Pipe classes (e.g. `CompanyTypePipe`) go in `imports`, not `providers`
- `NzMessageService` goes in `providers`
- **Permission codes are hardcoded strings in the HTML** (`*appHasAction="'COMPANY:WRITE'"`); `index.ts` neither imports nor exposes any action-code constant
- **Delete uses the shared `ConfirmDeleteService`** (injected, `providedIn: 'root'`); do not use `NzModalService.confirm` and do not import `NzModalModule` for it
- `RwdColDirective` is always imported (template filter cols use `appRwdCol`); `NzDropdownModule` when the action column has a "更多 ⋯" dropdown
- Always use `takeUntilDestroyed(this.destroyRef)` for all subscriptions
- `withSkeletonDelay` wraps the list fetch only
- Delete error handling checks `err.status === 422` for backend validation messages
- `export class {ClassName}` in `index.ts` must match the `loadComponent` `.then(m => m.{ClassName})` in `app.routes.ts`
- Enum options (e.g. `COMPANY_TYPE_OPTIONS`) are `readonly` class properties for template binding
- Signal naming: list = `{camelCaseModel}s`, total = `totalCount`, selected = `selected{Model}Id`
- `queryParams` contains ALL request params (pagination + filters) as single SOT
- `refresh()` sends `toHttpParams(queryParams())`, sets `result.items` / `result.totalCount`, and only writes pagination back to `queryParams` when the backend clamped it
