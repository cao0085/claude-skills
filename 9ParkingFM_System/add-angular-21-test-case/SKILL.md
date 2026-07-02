---
name: add-angular-21-test-case
description: "Scaffold test cases for an Angular 21 feature. Use this skill whenever: user runs /add-angular21-test-case, asks to write tests for an Angular feature, or needs to cover an api.ts service and index.ts component with Vitest + Angular Testing Library patterns."
---

# Add Angular Test Case

Generate `index/index.spec.ts` covering the API service and the index component.

---

## Workflow

### Step 1: Collect Info

1. Read the following files before generating:
   - `{featureName}.api.ts` — extract all public method signatures
   - `index/index.ts` — extract signals, filterForm fields, CRUD methods, action codes
   - `models.ts` — extract DTO types and enum types

2. Confirm before generating:
   - Feature path (e.g., `src/app/features/admin/company/`)
   - Which methods exist (getXxx, createXxx, updateXxx, deleteXxx, xxxDetailResource)
   - Which fields are in the filter form

---

## Test Structure

Two `describe` blocks in one file:

1. **`{ServiceName}` API tests** — verify HTTP calls are routed to the correct endpoint
2. **`{ClassName}` Component tests** — verify signal state and user interactions

---

## Section 1: API Tests

Tests verify the correct API endpoint and method is called. Since there is **no cache pattern**, each `getXxx()` call always hits the backend — test this explicitly.

```typescript
describe('{ServiceName}', () => {
    let api: {ServiceName};
    let mockApiService: Pick<ApiService, 'get' | 'post' | 'put' | 'delete' | 'createGetResource'>;

    beforeEach(() => {
        mockApiService = {
            get: vi.fn().mockReturnValue(of([])),
            post: vi.fn().mockReturnValue(of({})),
            put: vi.fn().mockReturnValue(of({})),
            delete: vi.fn().mockReturnValue(of({})),
            createGetResource: vi.fn().mockReturnValue({ value: signal(undefined), isLoading: signal(false) }),
        };

        TestBed.configureTestingModule({
            providers: [
                {ServiceName},
                { provide: ApiService, useValue: mockApiService },
            ],
        });

        api = TestBed.inject({ServiceName});
    });

    it('get{Model}s 呼叫 GET /{endpoint} 並傳入 params', () => {
        const params = new HttpParams({ fromObject: { pageNumber: '1' } });
        api.get{Model}s(params).subscribe();
        expect(mockApiService.get).toHaveBeenCalledWith('/{endpoint}', params, true);
    });

    it('get{Model}s 每次呼叫皆送出請求（無快取）', () => {
        const params = new HttpParams();
        api.get{Model}s(params).subscribe();
        api.get{Model}s(params).subscribe();
        expect(mockApiService.get).toHaveBeenCalledTimes(2);
    });

    it('create{Model} 呼叫 POST /{endpoint}', () => {
        api.create{Model}(body).subscribe();
        expect(mockApiService.post).toHaveBeenCalledWith('/{endpoint}', body);
    });

    it('update{Model} 呼叫 PUT /{endpoint}', () => {
        api.update{Model}(body).subscribe();
        expect(mockApiService.put).toHaveBeenCalledWith('/{endpoint}', body);
    });

    it('delete{Model} 呼叫 DELETE /{endpoint}/{id}', () => {
        api.delete{Model}('{id}').subscribe();
        expect(mockApiService.delete).toHaveBeenCalledWith('/{endpoint}/{id}', false, { body: { id: '{id}' } });
    });
});
```

---

## Section 2: Component Tests

### TestBed Setup

Key rules:
- `overrideComponent` is required because `CompanyManagementApi` and `NzMessageService` are in the component's own `providers`
- `NzModalService` must also go in `overrideComponent.set.providers` — even though it's not in the component's `providers` list, this ensures the mock takes priority over the service registered via `NzModalModule` in the component's `imports`
- `provideNoopAnimations()` at the TestBed root level

```typescript
describe('{ClassName}', () => {
    let component: {ClassName};
    let mockApi: Partial<{ServiceName}>;
    let mockMessage: Partial<NzMessageService>;
    let mockModal: Partial<NzModalService>;

    const mock{Model}: Browse{Model}Dto = {
        id: '{id}',
        // ... all required fields
    };

    beforeEach(async () => {
        mockApi = {
            get{Model}s: vi.fn().mockReturnValue(of([mock{Model}])),
            {camelCaseModel}DetailResource: vi.fn().mockReturnValue({ value: signal(undefined), isLoading: signal(false) }),
            create{Model}: vi.fn().mockReturnValue(of(void 0)),
            update{Model}: vi.fn().mockReturnValue(of(void 0)),
            delete{Model}: vi.fn().mockReturnValue(of(void 0)),
        };
        mockMessage = { success: vi.fn(), error: vi.fn() };
        mockModal = { confirm: vi.fn() };

        await TestBed.configureTestingModule({
            imports: [{ClassName}],
            providers: [provideNoopAnimations()],
        })
            .overrideComponent({ClassName}, {
                set: {
                    providers: [
                        { provide: {ServiceName}, useValue: mockApi },
                        { provide: NzMessageService, useValue: mockMessage },
                        { provide: NzModalService, useValue: mockModal },
                    ],
                },
            })
            .compileComponents();

        const fixture = TestBed.createComponent({ClassName});
        component = fixture.componentInstance;
        fixture.detectChanges();
    });
```

### What to Test

| Behavior | Test |
|---|---|
| 元件建立 | `expect(component).toBeTruthy()` |
| `ngOnInit` 載入列表 | `get{Model}s` 被呼叫，`{camelCaseModel}s()` 有資料 |
| `onSearch` 重設 pageNumber | `queryParams().pageNumber === 1`；篩選值寫入 queryParams |
| `onSearch` 空字串 → undefined | `queryParams().{field} === undefined` |
| `onReset` 清除篩選 | form reset；queryParams 篩選欄位清除；pageNumber=1 |
| `onQueryParamsChange` 分頁更新 | `queryParams().pageNumber/pageSize` 更新 |
| `openDetail` | `selectedId = id`；`openInEditMode = false` |
| `openEdit` | `selectedId = id`；`openInEditMode = true` |
| `onXxxCreated` 成功 | modal 關閉；`message.success` |
| `onXxxCreated` 失敗 | `message.error` |
| `onXxxUpdated` 成功 | `selectedId = null`；`message.success` |
| `onXxxUpdated` 失敗 | `message.error` |
| `deleteXxx` 確認成功 | `deleteXxx` 呼叫；`message.success` |
| `deleteXxx` 422 錯誤 | `message.error` 顯示 `err.error.detail` |
| `deleteXxx` 非 422 錯誤 | `message.error('刪除失敗')` |

### CRUD Test Templates

**Search / Reset:**
```typescript
describe('onSearch', () => {
    it('以篩選條件搜尋並將 pageNumber 重設為 1', () => {
        component.queryParams.update(q => ({ ...q, pageNumber: 3 }));
        component.filterForm.patchValue({ {field}: 'value' });
        (mockApi.get{Model}s as ReturnType<typeof vi.fn>).mockClear();

        component.onSearch();

        expect(component.queryParams().pageNumber).toBe(1);
        expect(component.queryParams().{field}).toBe('value');
        expect(mockApi.get{Model}s).toHaveBeenCalled();
    });

    it('空字串篩選條件轉為 undefined', () => {
        component.filterForm.patchValue({ {field}: '' });
        component.onSearch();
        expect(component.queryParams().{field}).toBeUndefined();
    });

    it('null 的下拉選項轉為 undefined', () => {
        component.filterForm.patchValue({ {enumField}: null, isActive: null });
        component.onSearch();
        expect(component.queryParams().{enumField}).toBeUndefined();
        expect(component.queryParams().isActive).toBeUndefined();
    });
});

describe('onReset', () => {
    it('清除篩選條件並重新載入，pageNumber 回到 1', () => {
        component.filterForm.patchValue({ {field}: 'value' });
        component.queryParams.update(q => ({ ...q, {field}: 'value', pageNumber: 3 }));
        (mockApi.get{Model}s as ReturnType<typeof vi.fn>).mockClear();

        component.onReset();

        expect(component.filterForm.value.{field}).toBe('');
        expect(component.queryParams().{field}).toBeUndefined();
        expect(component.queryParams().pageNumber).toBe(1);
        expect(mockApi.get{Model}s).toHaveBeenCalled();
    });
});
```

**Pagination:**
```typescript
describe('onQueryParamsChange', () => {
    it('更新 pageNumber / pageSize 並重新載入', () => {
        (mockApi.get{Model}s as ReturnType<typeof vi.fn>).mockClear();
        const params = { pageIndex: 2, pageSize: 20, sort: [], filter: [] } as unknown as NzTableQueryParams;

        component.onQueryParamsChange(params);

        expect(component.queryParams().pageNumber).toBe(2);
        expect(component.queryParams().pageSize).toBe(20);
        expect(mockApi.get{Model}s).toHaveBeenCalled();
    });
});
```

**Detail / Edit:**
```typescript
describe('openDetail / openEdit', () => {
    it('openDetail 設定 selectedId 且 openInEditMode 為 false', () => {
        component.openDetail(mock{Model});
        expect(component.selected{Model}Id()).toBe('{id}');
        expect(component.openInEditMode()).toBe(false);
    });

    it('openEdit 設定 selectedId 且 openInEditMode 為 true', () => {
        component.openEdit(mock{Model});
        expect(component.selected{Model}Id()).toBe('{id}');
        expect(component.openInEditMode()).toBe(true);
    });
});
```

**Create:**
```typescript
describe('on{Model}Created', () => {
    it('建立成功後關閉 Modal 並顯示成功訊息', () => {
        component.isCreate{Model}ModalVisible.set(true);
        component.on{Model}Created(body);
        expect(mockApi.create{Model}).toHaveBeenCalledWith(body);
        expect(component.isCreate{Model}ModalVisible()).toBe(false);
        expect(mockMessage.success).toHaveBeenCalledWith('新增成功');
    });

    it('建立失敗時顯示失敗訊息', () => {
        (mockApi.create{Model} as ReturnType<typeof vi.fn>).mockReturnValue(throwError(() => new Error()));
        component.on{Model}Created(body);
        expect(mockMessage.error).toHaveBeenCalledWith('新增失敗');
    });
});
```

**Update:**
```typescript
describe('on{Model}Updated', () => {
    it('更新成功後清除 selectedId 並顯示成功訊息', () => {
        component.selected{Model}Id.set('{id}');
        component.on{Model}Updated(body);
        expect(mockApi.update{Model}).toHaveBeenCalledWith(body);
        expect(component.selected{Model}Id()).toBeNull();
        expect(mockMessage.success).toHaveBeenCalledWith('更新成功');
    });

    it('更新失敗時顯示失敗訊息', () => {
        (mockApi.update{Model} as ReturnType<typeof vi.fn>).mockReturnValue(throwError(() => new Error()));
        component.on{Model}Updated(body);
        expect(mockMessage.error).toHaveBeenCalledWith('更新失敗');
    });
});
```

**Delete (with modal confirm):**
```typescript
describe('delete{Model}', () => {
    it('確認後呼叫 delete{Model} 並顯示成功訊息', () => {
        (mockModal.confirm as ReturnType<typeof vi.fn>).mockImplementation(({ nzOnOk }) => { nzOnOk(); });

        component.delete{Model}(mock{Model});

        expect(mockApi.delete{Model}).toHaveBeenCalledWith('{id}');
        expect(mockMessage.success).toHaveBeenCalledWith('刪除成功');
    });

    it('422 錯誤顯示後端提供的 detail 訊息', () => {
        (mockModal.confirm as ReturnType<typeof vi.fn>).mockImplementation(({ nzOnOk }) => { nzOnOk(); });
        (mockApi.delete{Model} as ReturnType<typeof vi.fn>).mockReturnValue(
            throwError(() => ({ status: 422, error: { detail: '有關聯資料，無法刪除' } }))
        );

        component.delete{Model}(mock{Model});

        expect(mockMessage.error).toHaveBeenCalledWith('有關聯資料，無法刪除');
    });

    it('非 422 錯誤顯示通用失敗訊息', () => {
        (mockModal.confirm as ReturnType<typeof vi.fn>).mockImplementation(({ nzOnOk }) => { nzOnOk(); });
        (mockApi.delete{Model} as ReturnType<typeof vi.fn>).mockReturnValue(
            throwError(() => ({ status: 500 }))
        );

        component.delete{Model}(mock{Model});

        expect(mockMessage.error).toHaveBeenCalledWith('刪除失敗');
    });
});
```

---

## Validator Tests

If the feature has custom validators in `validators.ts`:

```typescript
describe('{validatorName}', () => {
    let control: FormControl;

    beforeEach(() => {
        control = new FormControl();
    });

    it.each(['valid-value-1', 'valid-value-2'])('允許值: %s', (value) => {
        control.setValue(value);
        expect({validatorName}(control)).toBeNull();
    });

    it.each(['', null, undefined, 'invalid-value'])('拒絕值: %s', (value) => {
        control.setValue(value);
        expect({validatorName}(control)).toEqual({ {errorKey}: true });
    });
});
```

---

## Example Output

See `src/app/features/basic-system/company-management/index/index.spec.ts` as the canonical reference for this pattern.

---

## Project Context

- Test runner: **Vitest** — use `vi.fn()`, `vi.mocked()`, `throwError()` from `rxjs`
- `NzModalService.confirm` mock: `mockImplementation(({ nzOnOk }) => { nzOnOk(); })` — immediately triggers confirm callback
- `NzModalService` goes in `overrideComponent.set.providers` (not TestBed root) because it would otherwise resolve to the real service registered via `NzModalModule` in the component's `imports`
- `NzMessageService` and the feature API service go in `overrideComponent.set.providers` because they're in the component's own `providers` array
- `provideNoopAnimations()` goes in `TestBed.configureTestingModule.providers` (root level)
- `totalCount()` during tests reflects the temporary mock calculation in `refresh()` — test with `toBeGreaterThan(0)` rather than an exact value if the real backend format is not yet wired
- All signals are synchronously updated because mock observables use `of()` (synchronous emission)
