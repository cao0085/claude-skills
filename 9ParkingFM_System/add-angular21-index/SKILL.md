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
   - Action code constant name (e.g., `COMPANY_ACTIONS`) — imported from `@app/core/constants/action-codes`
   - Route group display name if group is new (e.g., `基礎系統`)

### Step 2: Analyze api.ts

Scan public methods and map to patterns:

| Method pattern | Generates |
|---|---|
| `getXxx(params)` | `queryParams` signal + `xxxs` signal + `totalCount` signal + `isLoading` signal + `filterForm` + `onSearch()` + `onReset()` + `onQueryParamsChange()` + `refresh()` |
| `createXxx()` | `isCreateXxxModalVisible` signal + `onXxxCreated()` |
| `xxxDetailResource()` | `selectedXxxId` signal + `openDetail()` + `openEdit()` + `openInEditMode` signal + `onXxxUpdated()` |
| `deleteXxx()` | `deleteXxx()` with `modal.confirm` |

### Step 3: Generate index.ts

**Fixed imports (always included):**
```typescript
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { withSkeletonDelay } from '@app/shared/utils/with-skeleton-delay';
import { CommonModule } from '@angular/common';
import { Component, inject, signal, OnInit, DestroyRef } from '@angular/core';
import { NonNullableFormBuilder, ReactiveFormsModule } from '@angular/forms';
import { HttpErrorResponse, HttpParams } from '@angular/common/http';
import { HasActionDirective } from '@app/shared/directives/has-action.directive';
import { PaginationResult } from '@app/core/models/pagination';
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
delete{Model}({camelCaseModel}: Browse{Model}Dto) {
    this.modal.confirm({
        nzTitle: `確定要刪除「 ${{camelCaseModel}.displayName} 」嗎？`,
        nzOnOk: () => {
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
        },
    });
}

// ==================== Private ====================
private refresh() {
    const q = this.queryParams();
    const fromObject = Object.fromEntries(
        Object.entries(q).filter(([, v]) => v !== undefined && v !== null)
    );
    const params = new HttpParams({ fromObject });

    this.{camelCaseModel}Api
        .get{Model}s(params)
        .pipe(
            withSkeletonDelay(v => this.isLoading.set(v)),
            takeUntilDestroyed(this.destroyRef),
        )
        .subscribe(result => {
            // TODO: 後端換新格式後直接使用 result.items / result.totalCount
            const paginatedResult: PaginationResult<Browse{Model}Dto> = {
                items: result,
                totalCount: result.length,
                totalPages: 1,
                pageNumber: q.pageNumber ?? 1,
                pageSize: q.pageSize ?? 10,
            };
            this.{camelCaseModel}s.set(paginatedResult.items);
            this.totalCount.set(paginatedResult.totalCount);
        });
}
```

> ⚠️ When the backend switches to `PaginationResult<Browse{Model}Dto>`, change `api.ts` return type and simplify `refresh()` to `this.{camelCaseModel}s.set(result.items); this.totalCount.set(result.totalCount);`

---

## Example

**Input:** `admin/company`, api.ts has `getCompanies(params)`, `createCompany`, `companyDetailResource`, `updateCompany`, `deleteCompany`. Action code: `COMPANY_ACTIONS`.

**Output `src/app/features/admin/company/index/index.ts`:**

```typescript
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { withSkeletonDelay } from '@app/shared/utils/with-skeleton-delay';
import { CommonModule } from '@angular/common';
import { Component, inject, signal, OnInit, DestroyRef } from '@angular/core';
import { NonNullableFormBuilder, ReactiveFormsModule } from '@angular/forms';
import { HttpErrorResponse, HttpParams } from '@angular/common/http';

import {
    BrowseCompanyDto,
    BrowseCompanyParams,
    CreateCompanyReqBody,
    UpdateCompanyReqBody,
    CompanyTypePipe,
    COMPANY_TYPE_OPTIONS,
    CompanyType,
} from '../models';
import { PaginationResult } from '@app/core/models/pagination';
import { COMPANY_ACTIONS } from '@app/core/constants/action-codes';
import { HasActionDirective } from '@app/shared/directives/has-action.directive';
import { CompanyManagementApi } from '../company-management.api';
import { CreateCompanyModal } from '../create-company-modal/create-company-modal';
import { DetailModal } from '../detail-modal/detail-modal';

import { NzButtonModule } from 'ng-zorro-antd/button';
import { NzDividerModule } from 'ng-zorro-antd/divider';
import { NzFlexModule } from 'ng-zorro-antd/flex';
import { NzFormModule } from 'ng-zorro-antd/form';
import { NzGridModule } from 'ng-zorro-antd/grid';
import { NzIconModule } from 'ng-zorro-antd/icon';
import { NzInputModule } from 'ng-zorro-antd/input';
import { NzMessageService } from 'ng-zorro-antd/message';
import { NzModalModule, NzModalService } from 'ng-zorro-antd/modal';
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
        NzFlexModule,
        NzFormModule,
        NzGridModule,
        NzIconModule,
        NzInputModule,
        NzModalModule,
        NzSelectModule,
        NzSkeletonModule,
        NzTableModule,
        NzTagModule,
        HasActionDirective,
    ],
    templateUrl: './index.html',
    styleUrl: './index.scss',
})
export class CompanyManagement implements OnInit {
    private fb = inject(NonNullableFormBuilder);
    private companyApi = inject(CompanyManagementApi);
    private destroyRef = inject(DestroyRef);
    private modal = inject(NzModalService);
    private message = inject(NzMessageService);

    readonly COMPANY_TYPE_OPTIONS = COMPANY_TYPE_OPTIONS;
    protected readonly COMPANY_ACTIONS = COMPANY_ACTIONS;

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
        this.modal.confirm({
            nzTitle: `確定要刪除「 ${company.fullName} 」嗎？`,
            nzOnOk: () => {
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
            },
        });
    }

    // ==================== Private ====================
    private refresh() {
        const q = this.queryParams();
        const fromObject = Object.fromEntries(
            Object.entries(q).filter(([, v]) => v !== undefined && v !== null)
        );
        const params = new HttpParams({ fromObject });

        this.companyApi
            .getCompanies(params)
            .pipe(
                withSkeletonDelay(v => this.isLoading.set(v)),
                takeUntilDestroyed(this.destroyRef),
            )
            .subscribe(result => {
                // TODO: 後端換新格式後移除此 mock
                const paginatedResult: PaginationResult<BrowseCompanyDto> = {
                    items: result,
                    totalCount: result.length,
                    totalPages: 1,
                    pageNumber: q.pageNumber ?? 1,
                    pageSize: q.pageSize ?? 10,
                };
                this.companies.set(paginatedResult.items);
                this.totalCount.set(paginatedResult.totalCount);
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
- `NzModalService` is injected directly (not in `providers`), but **`NzModalModule` must be in `imports`**
- Always use `takeUntilDestroyed(this.destroyRef)` for all subscriptions
- `withSkeletonDelay` wraps the list fetch only
- Delete error handling always checks `err.status === 422` for backend validation messages
- `export class {ClassName}` in `index.ts` must match the `loadComponent` `.then(m => m.{ClassName})` in `app.routes.ts`
- Action codes are `protected readonly` so the template can use `*appHasAction`
- Enum options (e.g. `COMPANY_TYPE_OPTIONS`) are `readonly` class properties for template binding
- Signal naming: list = `{camelCaseModel}s`, total = `totalCount`, selected = `selected{Model}Id`
- `queryParams` contains ALL request params (pagination + filters) as single SOT
- `refresh()` builds `HttpParams` by filtering out `undefined`/`null` values from `queryParams()`
