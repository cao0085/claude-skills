---
name: add-angular21-mock
description: "Scaffold a mock API file for local development when the backend is not ready. Use this skill whenever: user runs /add-angular21-mock, asks to create mock/fake data for an API service, or needs to develop a feature before the backend or DTOs are confirmed."
---

# Add Angular API Mock

Generate `{featureName}.api.mock.ts` for local development.
This file is for personal use only — do not commit to the repository.

---

## Workflow

### Step 1: Collect Info

1. If the user hasn't provided any info, ask for:
   - Feature group and feature name (e.g., group: `admin`, feature: `company`)
   - Does `{featureName}.api.ts` already exist?
     - **Yes** → read the existing api.ts to extract method signatures
     - **No** → ask for the methods needed and their return shapes
   - Are DTOs already defined in `models.ts`?
     - **Yes** → use them
     - **No** → use placeholder types with `TODO` comments

2. Confirm the following before generating:
   - Mock class name (e.g., `CompanyManagementApiMock`)
   - Feature path (e.g., `src/app/features/admin/company/`)

### Step 2: Generate api.mock.ts

- Path: `src/app/features/{group}/{feature-name}/{featureName}.api.mock.ts`

> ⚠️ Add a comment at the top: `// DEV ONLY — do not commit`

**Rules:**
- Mirror every public method from the original api.ts
- `get{Model}s(params: HttpParams)` — accept `HttpParams` but ignore it; return `of(MOCK_{MODEL}S)`
- Replace all API calls with `of(mockData)`
- Keep the same return types
- Declare mock data as constants above the class
- Use `signal()` for resource-like methods (those that use `createGetResource`)

Output format:

```typescript
// DEV ONLY — do not commit
import { computed, signal, Signal } from '@angular/core';
import { HttpParams } from '@angular/common/http';
import { of } from 'rxjs';
import {
    Browse{Model}Dto,
    Get{Model}Dto,
    Create{Model}ReqBody,
    Update{Model}ReqBody,
} from './models';

const MOCK_{MODEL}S: Browse{Model}Dto[] = [
    // TODO: fill in mock data
];

const MOCK_{MODEL}_DETAIL: Get{Model}Dto = {
    // TODO: fill in mock data
};

export class {ServiceName}Mock {

    get{Model}s(_params: HttpParams) {
        return of(MOCK_{MODEL}S);
    }

    {camelCaseModel}DetailResource({camelCaseModel}Id: Signal<string | null>) {
        // simplified mock — returns fixed data regardless of id
        return {
            value: signal(MOCK_{MODEL}_DETAIL),
            isLoading: signal(false),
            error: signal(undefined),
        };
    }

    create{Model}(body: Create{Model}ReqBody) {
        console.log('[MOCK] create{Model}', body);
        return of(void 0);
    }

    update{Model}(body: Update{Model}ReqBody) {
        console.log('[MOCK] update{Model}', body);
        return of(void 0);
    }

    delete{Model}({camelCaseModel}Id: string) {
        console.log('[MOCK] delete{Model}', {camelCaseModel}Id);
        return of(void 0);
    }
}
```

**Situation — DTOs not yet confirmed:**

Use the core model type directly as a placeholder — never use `any`.

```typescript
// DEV ONLY — do not commit
import { signal, Signal } from '@angular/core';
import { HttpParams } from '@angular/common/http';
import { of } from 'rxjs';
import { {Model} } from './models'; // core model as placeholder

// TODO: replace with actual Browse{Model}Dto when confirmed
const MOCK_{MODEL}S: {Model}[] = [
    // TODO: fill in mock data
];

// TODO: replace with actual Get{Model}Dto when confirmed
const MOCK_{MODEL}_DETAIL: {Model} = {
    // TODO: fill in mock data
};

export class {ServiceName}Mock {

    get{Model}s(_params: HttpParams) {
        return of(MOCK_{MODEL}S);
    }

    {camelCaseModel}DetailResource({camelCaseModel}Id: Signal<string | null>) {
        return {
            value: signal(MOCK_{MODEL}_DETAIL),
            isLoading: signal(false),
            error: signal(undefined),
        };
    }

    // mutations — fill in as needed
}
```

---

## Example

**Input:** `admin/company`, api.ts already exists

**Output `src/app/features/admin/company/company.api.mock.ts`:**
```typescript
// DEV ONLY — do not commit
import { signal, Signal } from '@angular/core';
import { HttpParams } from '@angular/common/http';
import { of } from 'rxjs';
import {
    BrowseCompanyDto,
    GetCompanyDto,
    CreateCompanyReqBody,
    UpdateCompanyReqBody,
} from './models';

const MOCK_COMPANIES: BrowseCompanyDto[] = [
    // TODO: fill in mock data
];

const MOCK_COMPANY_DETAIL: GetCompanyDto = {
    // TODO: fill in mock data
};

export class CompanyManagementApiMock {

    getCompanies(_params: HttpParams) {
        return of(MOCK_COMPANIES);
    }

    companyDetailResource(companyId: Signal<string | null>) {
        return {
            value: signal(MOCK_COMPANY_DETAIL),
            isLoading: signal(false),
            error: signal(undefined),
        };
    }

    createCompany(body: CreateCompanyReqBody) {
        console.log('[MOCK] createCompany', body);
        return of(void 0);
    }

    updateCompany(body: UpdateCompanyReqBody) {
        console.log('[MOCK] updateCompany', body);
        return of(void 0);
    }

    deleteCompany(companyId: string) {
        console.log('[MOCK] deleteCompany', companyId);
        return of(void 0);
    }
}
```

---

## Project Context

- Mock class does NOT extend or implement the real api class — just mirrors public methods
- `getXxx` accepts `HttpParams` but ignores it (`_params` convention)
- `createGetResource` is replaced with a plain object using `signal()`
- All mutations log to console for easy debugging
- Mock data constants are declared at the top of the file for easy editing