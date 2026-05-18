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
   - Which fields are filterable (used in BrowseXxxParams)?
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
import { PaginationParams } from '@app/core/models/pagination';
export type { PaginationResult, PaginationParams } from '@app/core/models/pagination';

// 瀏覽篩選參數
export interface Browse{Model}Params extends PaginationParams, Partial<Pick<{Model},
    'field1' | 'field2' | 'isActive'
>> {}

// 瀏覽列表（對應後端 Browse{Model}Dto）
export type Browse{Model}Dto = Pick<
    {Model},
    'id' | 'field1' | 'field2' | 'isActive'
>;

// 詳細資料（對應後端 Get{Model}Dto）
export interface Get{Model}Dto extends {Model} {
    createdAt: string;
    updatedAt: string;
}

// 建立（對應後端 Create{Model}Command）
export type Create{Model}ReqBody = Omit<{Model}, 'id' | 'isActive'>;

// 更新（對應後端 Update{Model}Command）
export type Update{Model}ReqBody = Omit<{Model}, 'readonlyField'>;
```

### Step 3: Generate api.ts

- Path: `src/app/features/{group}/{feature-name}/{featureName}.api.ts`
- **No cache pattern** — the component owns pagination/query state and calls the API directly each time
- Mutations return the observable directly; no `tap` / invalidation needed

**Common method patterns:**

| Endpoint | Pattern | Notes |
|---|---|---|
| GET list | `getXxx(params: HttpParams)` | returns array now; comment out PaginationResult version |
| GET detail | `xxxDetailResource(id: Signal)` with `createGetResource` | reactive, signal-based |
| POST | `createXxx(body)` | direct observable |
| PUT | `updateXxx(body)` | direct observable |
| DELETE | `deleteXxx(id)` | direct observable |

Output format:

```typescript
import { computed, inject, Injectable, Signal } from '@angular/core';
import { HttpParams } from '@angular/common/http';
import { ApiService } from '@app/core/services/api.service';
import {
    Browse{Model}Dto,
    Get{Model}Dto,
    Create{Model}ReqBody,
    Update{Model}ReqBody,
} from './models';

@Injectable()
export class {ServiceName} {
    private api = inject(ApiService);

    get{Model}s(params: HttpParams) {
        // return this.api.get<PaginationResult<Browse{Model}Dto>>('/{endpoint}', params, true);
        return this.api.get<Browse{Model}Dto[]>('/{endpoint}', params, true);
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
        return this.api.post('/{endpoint}', body);
    }

    update{Model}(body: Update{Model}ReqBody) {
        return this.api.put('/{endpoint}', body);
    }

    delete{Model}({camelCaseModel}Id: string) {
        return this.api.delete(`/{endpoint}/${'{camelCaseModel}Id'}`, false, { body: { id: {camelCaseModel}Id } });
    }
}
```

**Situation B — placeholder format:**

If DTOs are not yet confirmed, use the core model type directly as a placeholder — never use `any`.

```typescript
import { {Model} } from './models'; // core model as placeholder

// TODO: replace with actual DTO when backend confirms the shape
get{Model}s(params: HttpParams) {
    return this.api.get<{Model}[]>('/{endpoint}', params, true);
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
filterable fields: companyCode, fullName, shortName, businessAdministrationNumber, companyType, isActive
endpoints:
  GET    /companies           → BrowseCompanyDto[]
  GET    /companies/:id       → GetCompanyDto
  POST   /companies           → CreateCompanyReqBody
  PUT    /companies           → UpdateCompanyReqBody
  DELETE /companies/:id
```

**models.ts additions:**
```typescript
import { PaginationParams } from '@app/core/models/pagination';
export type { PaginationResult, PaginationParams } from '@app/core/models/pagination';

export interface BrowseCompanyParams extends PaginationParams, Partial<Pick<Company,
    'companyCode' | 'fullName' | 'shortName' | 'businessAdministrationNumber' | 'companyType' | 'isActive'
>> {}

export type BrowseCompanyDto = Pick<
    Company,
    'id' | 'companyCode' | 'fullName' | 'shortName' | 'businessAdministrationNumber' | 'companyType' | 'isActive'
>;

export interface GetCompanyDto extends Company {
    createdAt: string;
    updatedAt: string;
}

export type CreateCompanyReqBody = Omit<Company, 'id' | 'isActive'>;

export type UpdateCompanyReqBody = Omit<Company, 'companyCode' | 'businessAdministrationNumber'>;
```

**Output `src/app/features/admin/company/company.api.ts`:**
```typescript
import { computed, inject, Injectable, Signal } from '@angular/core';
import { HttpParams } from '@angular/common/http';
import { ApiService } from '@app/core/services/api.service';
import {
    BrowseCompanyDto,
    GetCompanyDto,
    CreateCompanyReqBody,
    UpdateCompanyReqBody,
} from './models';

@Injectable()
export class CompanyManagementApi {
    private api = inject(ApiService);

    getCompanies(params: HttpParams) {
        // return this.api.get<PaginationResult<BrowseCompanyDto>>('/companies', params, true);
        return this.api.get<BrowseCompanyDto[]>('/companies', params, true);
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
        return this.api.post('/companies', body);
    }

    updateCompany(body: UpdateCompanyReqBody) {
        return this.api.put('/companies', body);
    }

    deleteCompany(companyId: string) {
        return this.api.delete(`/companies/${companyId}`, false, { body: { id: companyId } });
    }
}
```

---

## Project Context

- API service is `@Injectable()` with no `providedIn` — provided at feature level
- All imports from `./models`, never directly from `@app/core/models/`
- **No cache pattern** — list endpoints accept `HttpParams` and return a fresh observable each call
- The commented-out `PaginationResult<...>` line shows the intended backend response type once the backend is updated
- Mutations return the observable directly — no `tap` / no cache invalidation needed
