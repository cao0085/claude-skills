---
name: 9pfms:add-angular21-create-component
description: "Scaffold a new Angular 17+ standalone feature component: run ng g, create models.ts, and register route in app.routes.ts. Trigger when user says /add-angular21-create-component, asks to create a new page, scaffold a new feature, or add a new component to the system."
---

# Add Angular Component (Feature Page)

Scaffold a new feature page component with `models.ts` and route registration.

---

## Step 1: Gather info from user

Ask (or infer from context):

| 欄位 | 範例 | 說明 |
|---|---|---|
| `{group}` | `basic-system` | 功能群組資料夾名稱 |
| `{kebab-name}` | `company-management` | feature 名稱（kebab-case） |
| `{ClassName}` | `CompanyManagement` | component class 名稱（PascalCase） |
| `{routePath}` | `basic/companymanagement` | app.routes.ts 中的路徑 |

---

## Step 2: Run `ng g`

在專案根目錄執行（`YP.FinancialManagementSystem.Web/`）：

```bash
ng g c features/{group}/{kebab-name}/{kebab-name} \
  --standalone \
  --flat \
  --type="" \
  --skip-tests
```

- `--flat` — 不額外建子資料夾（已在路徑中指定）
- `--type=""` — 產出 `{kebab-name}.ts`，不加 `.component` 後綴
- `--skip-tests` — 不產生 spec 檔（spec 另外手動建立）

產出：
```
features/{group}/{kebab-name}/
├── {kebab-name}.ts
├── {kebab-name}.html
└── {kebab-name}.scss
```

---

## Step 3: Create `models.ts`

在同資料夾建立 `models.ts`，內容依業務需求填入，空白範本：

```typescript
// ==================== {Domain} ====================

export interface {Entity} {
  {entityId}: string;
  // ...
}

export interface {Entity}Detail extends {Entity} {
  // ...
}

export interface Create{Entity}ReqBody {
  // ...
}

export interface Update{Entity}ReqBody {
  {entityId}: string;
  // ...
}
```

若有對應的 core model，先從 `@app/core/models/` re-export：
```typescript
import { {CoreModel} } from '@app/core/models/{coreModel}';
export type { {CoreModel} } from '@app/core/models/{coreModel}';
```

---

## Step 4: Add route to `app.routes.ts`

路徑：`src/app/app.routes.ts`

在對應的 section 下（e.g., `// 基礎系統`）加入：

```typescript
{
  path: '{routePath}',
  canActivate: [menuRouteGuard],
  children: [
    {
      path: '',
      loadComponent: () => import('@app/features/{group}/{kebab-name}/{kebab-name}')
        .then(m => m.{ClassName}),
      data: { reuseRoute: true }
    }
  ]
},
```

### 現有 section 對應

| 功能群組 | Section 註解 | 路由前綴 |
|---|---|---|
| `basic-system` | `// 基礎系統` | `basic/` |

---

## 完成後確認

- [ ] `{kebab-name}.ts` 已產生且 selector 為 `app-{kebab-name}`
- [ ] `models.ts` 已建立
- [ ] `app.routes.ts` 已加入新路由
- [ ] 路由 path 與後端 MenuManager 設定一致

---

## Project context

- Web 專案根目錄：`YP.FinancialManagementSystem.Web/`
- Feature 路徑：`src/app/features/{group}/{kebab-name}/`
- 路由檔：`src/app/app.routes.ts`
- 路由使用 `menuRouteGuard` + `loadComponent` lazy loading
- 所有 feature component 皆 `standalone: true`，無 NgModule
