---
name: add-angular21-api
description: "Scaffold an Angular 21 API service class and its related DTOs for a feature. Use this skill whenever: user runs /add-angular21-api, provides endpoint definitions or DTO definitions, or asks to create/generate an api.ts file — especially when models.ts needs to be updated with new DTOs, or when aligning to backend endpoints."
---

# Add Angular API Service

Generate `{featureName}.api.ts` and update `models.ts` with the required DTOs.

---

## Workflow

### Step 1: Collect Info

1. If the user hasn't provided any info, ask for:
   - Feature group and feature name (e.g., group: `admin`, feature: `company`)
   - Endpoint definitions (HTTP method + path + request/response shape)
   - Are DTOs already defined? If yes, ask for DTO definitions
2. Determine which situation applies:
   - **Situation A — DTOs confirmed**: user provides endpoint + DTO definitions → generate models.ts DTOs + api.ts
   - **Situation B — DTOs not yet confirmed**: user only has endpoint or feature description → generate api.ts with placeholder types, mark TODOs

3. Confirm the following before generating:
   - Service class name (e.g., `CompanyManagementApi`)
   - Feature path (e.g., `src/app/features/admin/company/`)

### Step 2: Update models.ts DTOs

> ⚠️ Skip this step if in Situation B

Append DTO types to `src/app/features/{group}/{feature-name}/models.ts`.

Common DTO patterns:

```typescript
// Browse list (對應後端 Browse{Model}Dto)
export type Browse{Model}Dto = Pick<
    {Model},
    'field1' | 'field2' | 'field3'
>;

// Detail (對應後端 Get{Model}Dto)
export interface Get{Model}Dto extends {Model} {
    createdAt: string;
    updatedAt: string;
}

// Create request body (對應後端 Create{Model}Command)
export type Create{Model}ReqBody = Omit<{Model}, 'id' | 'isActive'>;

// Update request body (對應後端 Update{Model}Command)
export type Update{Model}ReqBody = Omit<{Model}, 'readonlyField'>;
```

### Step 3: Generate api.ts

- Path: `src/app/features/{group}/{feature-name}/{featureName}.api.ts`

**Common method patterns:**

| Endpoint | Pattern | Notes |
|---|---|---|
| GET list | `getXxx()` with cache + `shareReplay` | invalidate on mutation |
| GET detail | `xxxDetailResource(id: Signal)` with `createGetResource` | reactive, signal-based |
| POST | `createXxx(body)` with `tap(() => invalidate)` | |
| PUT | `updateXxx(body)` with `tap(() => invalidate)` | |
| DELETE | `deleteXxx(id)` with `tap(() => invalidate)` | |

Output format:

```typescript
import { computed, inject, Injectable, Signal } from '@angular/core';
import { ApiService } from '@app/core/services/api.service';
import {
    Browse{Model}Dto,
    Get{Model}Dto,
    Create{Model}ReqBody,
    Update{Model}ReqBody,
} from './models';
import { Observable, shareReplay, tap } from 'rxjs';

@Injectable()
export class {ServiceName} {
    private api = inject(ApiService);
    private {camelCaseModel}sCache$?: Observable<Browse{Model}Dto[]>;

    get{Model}s() {
        if (!this.{camelCaseModel}sCache$) {
            this.{camelCaseModel}sCache$ = this.api.get<Browse{Model}Dto[]>('/{endpoint}', undefined, true).pipe(
                shareReplay({ bufferSize: 1, refCount: false })
            );
        }
        return this.{camelCaseModel}sCache$;
    }

    private invalidate{Model}s() {
        this.{camelCaseModel}sCache$ = undefined;
    }

    {camelCaseModel}DetailResource({camelCaseModel}Id: Signal<string | null>) {
        return this.api.createGetResource<Get{Model}Dto>(
            computed(() => {
                const id = {camelCaseModel}Id();
                return id ? `/{endpoint}/${id}` : undefined;
            }),
        );
    }

    create{Model}(body: Create{Model}ReqBody) {
        return this.api.post('/{endpoint}', body).pipe(
            tap(() => this.invalidate{Model}s())
        );
    }

    update{Model}(body: Update{Model}ReqBody) {
        return this.api.put('/{endpoint}', body).pipe(
            tap(() => this.invalidate{Model}s())
        );
    }

    delete{Model}({camelCaseModel}Id: string) {
        return this.api.delete(`/{endpoint}/${camelCaseModel}Id}`, false, { body: { id: {camelCaseModel}Id } }).pipe(
            tap(() => this.invalidate{Model}s())
        );
    }
}
```

**Situation B — placeholder format:**

If DTOs are not yet confirmed, use the core model type directly as a placeholder — never use `any`.
Import from `./models` as usual (core model should already be re-exported there).

```typescript
import { {Model} } from './models'; // core model as placeholder

// TODO: replace with actual DTO when backend confirms the shape
get{Model}s(): Observable<{Model}[]> {
    return this.api.get<{Model}[]>('/{endpoint}', undefined, true).pipe(
        shareReplay({ bufferSize: 1, refCount: false })
    );
}

// TODO: replace with Get{Model}Dto when confirmed
{camelCaseModel}DetailResource({camelCaseModel}Id: Signal<string | null>) {
    return this.api.createGetResource<{Model}>(
        computed(() => {
            const id = {camelCaseModel}Id();
            return id ? `/{endpoint}/${id}` : undefined;
        }),
    );
}
```

---

## Example

**Input:**
```
feature: admin/company
endpoints:
  GET    /companies           → BrowseCompanyDto[]
  GET    /companies/:id       → GetCompanyDto
  POST   /companies           → CreateCompanyReqBody
  PUT    /companies           → UpdateCompanyReqBody
  DELETE /companies/:id
```

**Output `src/app/features/admin/company/company.api.ts`:**
```typescript
import { computed, inject, Injectable, Signal } from '@angular/core';
import { ApiService } from '@app/core/services/api.service';
import {
    BrowseCompanyDto,
    GetCompanyDto,
    CreateCompanyReqBody,
    UpdateCompanyReqBody,
} from './models';
import { Observable, shareReplay, tap } from 'rxjs';

@Injectable()
export class CompanyManagementApi {
    private api = inject(ApiService);
    private companiesCache$?: Observable<BrowseCompanyDto[]>;

    getCompanies() {
        if (!this.companiesCache$) {
            this.companiesCache$ = this.api.get<BrowseCompanyDto[]>('/companies', undefined, true).pipe(
                shareReplay({ bufferSize: 1, refCount: false })
            );
        }
        return this.companiesCache$;
    }

    private invalidateCompanies() {
        this.companiesCache$ = undefined;
    }

    companyDetailResource(companyId: Signal<string | null>) {
        return this.api.createGetResource<GetCompanyDto>(
            computed(() => {
                const id = companyId();
                return id ? `/companies/${id}` : undefined;
            }),
        );
    }

    createCompany(body: CreateCompanyReqBody) {
        return this.api.post('/companies', body).pipe(
            tap(() => this.invalidateCompanies())
        );
    }

    updateCompany(body: UpdateCompanyReqBody) {
        return this.api.put('/companies', body).pipe(
            tap(() => this.invalidateCompanies())
        );
    }

    deleteCompany(companyId: string) {
        return this.api.delete(`/companies/${companyId}`, false, { body: { id: companyId } }).pipe(
            tap(() => this.invalidateCompanies())
        );
    }
}
```

---

## Project Context

- API service is `@Injectable()` with no `providedIn` — provided at feature level
- All imports from `./models`, never directly from `@app/core/models/`
- Cache pattern: list endpoints use `shareReplay`, detail endpoints use `createGetResource` (signal-based)
- Always invalidate cache on create / update / delete