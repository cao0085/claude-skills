---
name: angular-erp-auth-routing
description: Angular v21 ERP authentication, route guarding, and permission control patterns. Use this skill whenever the user is building an Angular application that involves login authentication with token-based API responses, dynamic route access control based on backend-provided allowed route lists, UI-level permission gating (show/hide/enable/disable), or any combination of guards (canMatch, canActivate) with a signal-based auth store. Also trigger when the user asks about Angular route guards, functional guards, canMatch vs canActivate, permission directives, or ERP-style two-layer access control (route-level + action-level). Even if the user doesn't say "ERP", trigger when the pattern is: backend returns token + allowed routes + permissions after login.
---

# Angular v21 ERP Auth + Routing + Permission Control

This skill covers the complete authentication and authorization pattern for an ERP-style Angular v21 application where the backend drives access control.

## Architecture overview

The flow is:

1. Frontend calls login API
2. Backend responds with `token` + `allowedRoutes[]` + `permissions[]`
3. Frontend stores these in a signal-based `AuthStore`
4. **Route-level control**: `canMatch` / `canActivate` guards filter which pages the user can access, based on `allowedRoutes[]`
5. **UI-level control**: structural directives and direct signal checks control which buttons, menus, and actions are visible, based on `permissions[]`

Two layers, two arrays, clean separation:
- `allowedRoutes[]` → coarse-grained, "can you see this module at all"
- `permissions[]` → fine-grained, "can you perform this action"

---

## 1. AuthStore — signal-based state

Use a single `Injectable` with `signal` and `computed`. No RxJS needed for auth state in v21.

```ts
// auth.store.ts
import { Injectable, signal, computed } from '@angular/core';

export interface AuthState {
  token: string | null;
  allowedRoutes: string[];   // e.g. ['/dashboard', '/orders', '/inventory']
  permissions: string[];      // e.g. ['order:create', 'order:read', 'inventory:edit']
}

@Injectable({ providedIn: 'root' })
export class AuthStore {
  private state = signal<AuthState>({
    token: null,
    allowedRoutes: [],
    permissions: [],
  });

  token = computed(() => this.state().token);
  allowedRoutes = computed(() => this.state().allowedRoutes);
  permissions = computed(() => this.state().permissions);
  isLoggedIn = computed(() => !!this.state().token);

  setAuth(data: AuthState) {
    this.state.set(data);
  }

  hasRoute(route: string): boolean {
    return this.state().allowedRoutes.includes(route);
  }

  hasPermission(perm: string): boolean {
    return this.state().permissions.includes(perm);
  }

  hasAnyPermission(...perms: string[]): boolean {
    return perms.some(p => this.state().permissions.includes(p));
  }

  hasAllPermissions(...perms: string[]): boolean {
    return perms.every(p => this.state().permissions.includes(p));
  }

  clear() {
    this.state.set({ token: null, allowedRoutes: [], permissions: [] });
  }
}
```

Key points:
- One signal holds the entire auth state; `computed` derives slices.
- Methods like `hasRoute` and `hasPermission` are plain synchronous checks — no need for observables.
- Call `setAuth()` after login, `clear()` on logout.

---

## 2. Route guards — functional style

Angular v21 recommends **functional guards** over class-based guards. Use `CanMatchFn` and `CanActivateFn`.

### canMatch vs canActivate

- `canMatch` runs during route **matching**. If it returns false, the Router skips this route definition and tries the next one. Best for route-level access — prevents lazy chunk downloads entirely.
- `canActivate` runs **after** matching succeeds. If it returns false, navigation is blocked. Best for finer permission checks within an already-matched route.

Use `canMatch` for "can this user access this module" and `canActivate` for "can this user perform this specific action within the module."

### Guard implementations

```ts
// guards/auth.guard.ts
import { inject } from '@angular/core';
import { Router, type CanActivateFn, type CanMatchFn } from '@angular/router';
import { AuthStore } from '../auth.store';

/** Basic login check — is there a token? */
export const authGuard: CanActivateFn = () => {
  const auth = inject(AuthStore);
  const router = inject(Router);
  if (auth.isLoggedIn()) return true;
  return router.createUrlTree(['/login']);
};

/**
 * Route-level access check — does allowedRoutes[] contain this route?
 * Reads route.data['routeKey'] to know which key to check.
 * Use as canMatch so the route won't even match if not allowed.
 */
export const routeAccessGuard: CanMatchFn = (route) => {
  const auth = inject(AuthStore);
  const router = inject(Router);
  const requiredRoute = route.data?.['routeKey'] as string;
  if (!requiredRoute || auth.hasRoute(requiredRoute)) return true;
  return router.createUrlTree(['/forbidden']);
};

/**
 * Action-level permission check — factory that returns a CanActivateFn.
 * Pass one or more permission strings; user needs at least one to pass.
 */
export const permissionGuard = (...requiredPerms: string[]): CanActivateFn => {
  return () => {
    const auth = inject(AuthStore);
    const router = inject(Router);
    if (auth.hasAnyPermission(...requiredPerms)) return true;
    return router.createUrlTree(['/forbidden']);
  };
};
```

---

## 3. Route configuration

The pattern: a top-level layout route with `canActivate: [authGuard]`, and each child module uses `canMatch: [routeAccessGuard]` with a `data.routeKey`. Sub-routes that need finer control use `canActivate: [permissionGuard(...)]`.

```ts
// app.routes.ts
import { Routes } from '@angular/router';
import { authGuard, routeAccessGuard, permissionGuard } from './guards/auth.guard';

export const routes: Routes = [
  {
    path: 'login',
    loadComponent: () => import('./login/login.component'),
  },
  {
    path: 'forbidden',
    loadComponent: () => import('./forbidden/forbidden.component'),
  },

  // --- Protected area ---
  {
    path: '',
    canActivate: [authGuard],
    loadComponent: () => import('./layout/layout.component'),
    children: [
      {
        path: 'dashboard',
        data: { routeKey: '/dashboard' },
        canMatch: [routeAccessGuard],
        loadComponent: () => import('./dashboard/dashboard.component'),
      },
      {
        path: 'orders',
        data: { routeKey: '/orders' },
        canMatch: [routeAccessGuard],
        children: [
          {
            path: '',
            loadComponent: () => import('./orders/order-list.component'),
          },
          {
            path: 'create',
            canActivate: [permissionGuard('order:create')],
            loadComponent: () => import('./orders/order-create.component'),
          },
          {
            path: ':id',
            loadComponent: () => import('./orders/order-detail.component'),
          },
          {
            path: ':id/edit',
            canActivate: [permissionGuard('order:edit')],
            loadComponent: () => import('./orders/order-edit.component'),
          },
        ],
      },
      {
        path: 'inventory',
        data: { routeKey: '/inventory' },
        canMatch: [routeAccessGuard],
        loadComponent: () => import('./inventory/inventory.component'),
      },
    ],
  },

  { path: '**', redirectTo: '/forbidden' },
];
```

Key points:
- Every lazy-loaded module uses `loadComponent` (standalone components, no NgModule).
- `canMatch` with `routeAccessGuard` means a user without `/orders` access won't even trigger the lazy chunk download.
- `permissionGuard('order:create')` on child routes provides granular action-level checks.

---

## 4. UI-level permission control

### Structural directive — `*hasPermission`

Controls whether a DOM element is rendered at all.

```ts
// directives/has-permission.directive.ts
import {
  Directive, input, effect, inject,
  TemplateRef, ViewContainerRef,
} from '@angular/core';
import { AuthStore } from '../auth.store';

@Directive({ selector: '[hasPermission]' })
export class HasPermissionDirective {
  private auth = inject(AuthStore);
  private templateRef = inject(TemplateRef);
  private vcr = inject(ViewContainerRef);
  private rendered = false;

  hasPermission = input.required<string | string[]>();

  constructor() {
    effect(() => {
      const perms = this.hasPermission();
      const list = Array.isArray(perms) ? perms : [perms];
      const allowed = this.auth.hasAnyPermission(...list);

      if (allowed && !this.rendered) {
        this.vcr.createEmbeddedView(this.templateRef);
        this.rendered = true;
      } else if (!allowed && this.rendered) {
        this.vcr.clear();
        this.rendered = false;
      }
    });
  }
}
```

### Template usage examples

```html
<!-- Single permission -->
<button *hasPermission="'order:create'" routerLink="/orders/create">
  新增訂單
</button>

<!-- Multiple permissions (any match = show) -->
<div *hasPermission="['inventory:edit', 'inventory:delete']">
  庫存管理工具列
</div>
```

### Direct signal check in templates

For simpler cases, call `AuthStore` methods directly:

```html
<!-- Dynamic sidebar menu -->
@for (item of menuItems(); track item.route) {
  @if (auth.hasRoute(item.route)) {
    <a [routerLink]="item.route" routerLinkActive="active">
      {{ item.label }}
    </a>
  }
}

<!-- Inline disable instead of hide -->
<button
  [disabled]="!auth.hasPermission('order:delete')"
  (click)="deleteOrder()">
  刪除訂單
</button>
```

---

## 5. Login flow example

```ts
// login/login.component.ts
import { Component, inject } from '@angular/core';
import { Router } from '@angular/router';
import { AuthService } from '../services/auth.service';
import { AuthStore } from '../auth.store';

@Component({ /* ... */ })
export default class LoginComponent {
  private authService = inject(AuthService);
  private authStore = inject(AuthStore);
  private router = inject(Router);

  async onLogin(credentials: { account: string; password: string }) {
    const res = await this.authService.login(credentials);

    // Backend response shape:
    // { token: string, allowedRoutes: string[], permissions: string[] }
    this.authStore.setAuth({
      token: res.token,
      allowedRoutes: res.allowedRoutes,
      permissions: res.permissions,
    });

    this.router.navigate(['/dashboard']);
  }
}
```

---

## 6. HTTP interceptor — attach token

```ts
// interceptors/auth.interceptor.ts
import { inject } from '@angular/core';
import { type HttpInterceptorFn } from '@angular/common/http';
import { AuthStore } from '../auth.store';
import { Router } from '@angular/router';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const auth = inject(AuthStore);
  const token = auth.token();

  if (token) {
    req = req.clone({
      setHeaders: { Authorization: `Bearer ${token}` },
    });
  }

  return next(req);
};
```

Register in `app.config.ts`:

```ts
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { authInterceptor } from './interceptors/auth.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(withInterceptors([authInterceptor])),
  ],
};
```

---

## 7. Permission naming convention

Use a consistent `resource:action` pattern for the permissions array:

| Permission string   | Meaning              |
|----------------------|----------------------|
| `order:create`       | Can create orders    |
| `order:read`         | Can view orders      |
| `order:edit`         | Can edit orders      |
| `order:delete`       | Can delete orders    |
| `inventory:read`     | Can view inventory   |
| `inventory:edit`     | Can edit inventory   |
| `report:export`      | Can export reports   |
| `user:manage`        | Can manage users     |

This maps cleanly to both backend RBAC and frontend guard/directive checks.

---

## Quick reference — when to use what

| Scenario                          | Mechanism                              |
|-----------------------------------|----------------------------------------|
| Is the user logged in?            | `canActivate: [authGuard]`             |
| Can the user access this module?  | `canMatch: [routeAccessGuard]`         |
| Can the user perform this action? | `canActivate: [permissionGuard(...)]`  |
| Show/hide a button or section     | `*hasPermission` directive             |
| Enable/disable a control          | `[disabled]="!auth.hasPermission()"` |
| Build dynamic sidebar menu        | `@if (auth.hasRoute(item.route))`      |