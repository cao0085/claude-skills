---
name: 9pfms:add-angular21-api-service
description: "Extract HTTP API calls from an Angular 17+ standalone component into a dedicated *.api.ts service. Trigger when user says /add-angular21-api-service, asks to extract API from component, or wants to move HTTP calls out of a component."
---

# Add Angular API Service

Extract all `apiService.get/post/put/delete(...)` calls from a standalone component into a dedicated `{feature}.api.ts` file.

---

## Step 1: Read the component

Read the target component file to identify:
- All `apiService` calls and their URL strings
- ReqBody / response types used per call
- Which other files (modals, etc.) also import from `models.ts`

---

## Step 2: Create `{feature}.api.ts`

```typescript
import { inject, Injectable } from '@angular/core';
import { ApiService } from '@app/core/services/api.service';
import { /* only types used by API methods */ } from './models';

@Injectable()
export class {Feature}Api {
  private api = inject(ApiService);

  // Group methods by domain concept
  getXxx()                          { return this.api.get<XxxDto[]>('/xxx'); }
  getXxxById(id: string)            { return this.api.get<XxxDetail>(`/xxx/${id}`, undefined, true); }
  createXxx(body: CreateXxxReqBody) { return this.api.post('/xxx', body); }
  updateXxx(body: UpdateXxxReqBody) { return this.api.put('/xxx', body, true); }
  deleteXxx(id: string)             { return this.api.delete(`/xxx/${id}`); }
}
```

### Rules
- `@Injectable()` with **no `providedIn`** — must be provided via component
- **One method per endpoint**, no logic inside
- Import ReqBody types from `./models` (keep them there — modals may also import them)
- Export any inline anonymous body types that need naming (e.g. `ReorderXxxBody`)

---

## Step 3: Update the component

**Add to `@Component`:**
```typescript
@Component({
  providers: [{Feature}Api],   // ← component scope
  ...
})
```

**Replace injection:**
```typescript
// Remove
private apiService = inject(ApiService);

// Add
private {feature}Api = inject({Feature}Api);
```

**Replace all call sites:**
```typescript
// Before
this.apiService.post('/xxx', body)

// After
this.{feature}Api.createXxx(body)
```

**Clean up imports:** remove `ApiService` import if no longer used.

---

## Project context

- `ApiService` is at `@app/core/services/api.service`
- ReqBody types live in `models.ts` — **do not move them**, modals import from there
- Feature folder: `src/app/features/{feature-name}/`
