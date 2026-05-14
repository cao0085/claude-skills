---
name: add-angular21-index
description: "Scaffold the index.ts component for an Angular 21 feature. Use this skill whenever: user runs /add-angular21-index, asks to initialize or fill in an index component, or wants to wire up an api.ts into a component with signals, CRUD methods, and optional mock mode."
---

# Add Angular Index Component

Fill in `index/index.ts` by reading the feature's `api.ts` and `models.ts`.

---

## Workflow

### Step 1: Collect Info

1. Read the following files before generating:
   - `{featureName}.api.ts` — extract all public methods and their signatures
   - `{featureName}.api.mock.ts` — check if mock exists
   - `models.ts` — extract DTO types used in the methods
2. If files are not provided, ask for:
   - Feature group and feature name (e.g., group: `admin`, feature: `company`)
   - List of API methods available
3. Confirm the following before generating:
   - Component class name for `index.ts` (e.g., `CompanyManagement`) — must match `app.routes.ts` loadComponent
   - Feature path (e.g., `src/app/features/admin/company/`)
   - Route path (e.g., `basic/company`) — format: `{group}/{featureName}`
   - Route group display name if group is new (e.g., `基礎系統`)

### Step 2: Analyze api.ts

Scan public methods and map to patterns:

| Method pattern | Generates |
|---|---|
| `getXxx()` | `fetchedXxx` signal + `isLoading` signal + `refresh()` |
| `createXxx()` | `isCreateModalVisible` signal + `onXxxCreated()` |
| `xxxDetailResource()` | `selectedXxxId` signal + `openDetail()` + `closeDetail()` + `onXxxUpdated()` |
| `deleteXxx()` | `deleteXxx()` with `modal.confirm` |

### Step 3: Generate index.ts

**Fixed imports (always included):**
```typescript
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { withSkeletonDelay } from '@app/shared/utils/with-skeleton-delay';
import { Component, inject, signal, OnInit, DestroyRef } from '@angular/core';
import { HttpErrorResponse } from '@angular/common/http';
```

**Mock imports (only if api.mock.ts exists):**
```typescript
import { {ServiceName}Mock } from '../{featureName}.api.mock';
```

**Mock flag (only if api.mock.ts exists) — declare above @Component:**
```typescript
const USE_MOCK = true;
```

**providers — always include the real api class regardless of mock mode:**
```typescript
providers: [{ServiceName}, NzMessageService]
```

**api injection:**
```typescript
// no mock
private api = inject({ServiceName});

// with mock
private api = USE_MOCK ? new {ServiceName}Mock() : inject({ServiceName});
```

**Signal groups:**

```typescript
/**
 * SOT
 */
readonly fetched{Model}s = signal<Browse{Model}Dto[]>([]);
readonly selected{Model}Id = signal<string | null>(null); // only if detail exists

/**
 * UI State
 */
readonly isLoading = signal(true);
readonly isCreate{Model}ModalVisible = signal(false); // only if create exists
```

**CRUD method template:**

```typescript
// ==================== Create ====================
on{Model}Created(body: Create{Model}ReqBody) {
    this.api
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
    this.selectedId.set({camelCaseModel}.id);
}

closeDetail() {
    this.selected{Model}Id.set(null);
}

on{Model}Updated(body: Update{Model}ReqBody) {
    this.api
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
        nzTitle: `確定要刪除「 ${{camelCaseModel}.name} 」嗎？`,
        nzOnOk: () => {
            this.api
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
    this.api
        .get{Model}s()
        .pipe(
            withSkeletonDelay(v => this.isLoading.set(v)),
            takeUntilDestroyed(this.destroyRef),
        )
        .subscribe({Model}s => this.fetched{Model}s.set({Model}s));
}
```

---

## Example

**Input:** `admin/company`, api.ts has `getCompanies`, `createCompany`, `companyDetailResource`, `updateCompany`, `deleteCompany`. Mock exists.

**Output `src/app/features/admin/company/index/index.ts`:**

```typescript
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { withSkeletonDelay } from '@app/shared/utils/with-skeleton-delay';
import { Component, inject, signal, OnInit, DestroyRef } from '@angular/core';
import { HttpErrorResponse } from '@angular/common/http';

import { BrowseCompanyDto, CreateCompanyReqBody, UpdateCompanyReqBody, CompanyTypePipe } from '../models';
import { CompanyManagementApi } from '../company-management.api';
import { CompanyManagementApiMock } from '../company-management.api.mock';

import { NzButtonModule } from 'ng-zorro-antd/button';
import { NzDividerModule } from 'ng-zorro-antd/divider';
import { NzFlexModule } from 'ng-zorro-antd/flex';
import { NzIconModule } from 'ng-zorro-antd/icon';
import { NzMessageService } from 'ng-zorro-antd/message';
import { NzModalModule, NzModalService } from 'ng-zorro-antd/modal';
import { NzSkeletonModule } from 'ng-zorro-antd/skeleton';
import { NzTableModule } from 'ng-zorro-antd/table';
import { NzTagModule } from 'ng-zorro-antd/tag';

const USE_MOCK = true;

@Component({
    selector: 'app-index',
    standalone: true,
    providers: [CompanyManagementApi, NzMessageService],
    imports: [
        CompanyTypePipe,
        NzButtonModule,
        NzDividerModule,
        NzFlexModule,
        NzIconModule,
        NzModalModule,
        NzSkeletonModule,
        NzTableModule,
        NzTagModule,
    ],
    templateUrl: './index.html',
    styleUrl: './index.scss',
})
export class CompanyManagement implements OnInit {
    private api = USE_MOCK ? new CompanyManagementApiMock() : inject(CompanyManagementApi);
    private destroyRef = inject(DestroyRef);
    private modal = inject(NzModalService);
    private message = inject(NzMessageService);

    /**
     * SOT
     */
    readonly fetchedCompanies = signal<BrowseCompanyDto[]>([]);
    readonly selectedCompanyId = signal<string | null>(null);

    /**
     * UI State
     */
    readonly isLoading = signal(true);
    readonly isCreateModalVisible = signal(false);

    ngOnInit(): void {
        this.refresh();
    }

    // ==================== Create ====================
    onCompanyCreated(body: CreateCompanyReqBody) {
        this.api
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
        this.selectedCompanyId.set(company.id);
    }

    closeDetail() {
        this.selectedCompanyId.set(null);
    }

    onCompanyUpdated(body: UpdateCompanyReqBody) {
        this.api
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
                this.api
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
        this.api
            .getCompanies()
            .pipe(
                withSkeletonDelay(v => this.isLoading.set(v)),
                takeUntilDestroyed(this.destroyRef),
            )
            .subscribe(companies => this.fetchedCompanies.set(companies));
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

- `imports` in `@Component` — only include what the template actually uses; leave a comment `// TODO: add template imports` if template is not yet written
- Pipe classes (e.g. `CompanyTypePipe`) go in `imports`, not `providers`
- `NzMessageService` goes in `providers`
- `NzModalService` is injected directly (not in `providers`), but **`NzModalModule` must be in `imports`** so the service is resolvable
- Always use `takeUntilDestroyed(this.destroyRef)` for all subscriptions
- `withSkeletonDelay` wraps the list fetch only
- Delete error handling always checks `err.status === 422` for backend validation messages
- `export class {ClassName}` in `index.ts` must match the `loadComponent` `.then(m => m.{ClassName})` in `app.routes.ts`