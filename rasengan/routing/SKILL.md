---
name: rasengan-routing
description: Router definition and routing patterns for Rasengan.js. Covers config-based and file-based routing, RouterComponent, defineRouter, defineRoutesGroup, dynamic routes, route groups, and file-based routing conventions. Use when creating or modifying app.router.ts, defining route structures, or setting up dynamic segments.
license: MIT
metadata:
  author: dilane3
  framework: rasengan
  version: "2.0.0"
---

# Rasengan.js Routing Patterns

## When to Activate

- Creating or editing `src/app/app.router.{js,ts}`
- Choosing between config-based and file-based routing
- Defining dynamic route segments `[param]`, `[_param]`, route groups `(group)`
- Setting up nested routers or route groups with `defineRoutesGroup`
- Customizing 404 / loading fallbacks at the router level

## RouterComponent Properties

| Property | Type | Description |
|----------|------|-------------|
| `layout` | `LayoutComponent \| RouteNode` | Layout component for the router |
| `routers` | `RouterComponent[]` | Nested sub-routers |
| `pages` | `PageComponent[] \| RouteNode[]` | Page components |
| `loaderComponent` | `FunctionComponent` | Loading fallback |
| `notFoundComponent` | `FunctionComponent` | 404 component |
| `useParentLayout` | `boolean` | Inherit parent layout |

## Config-based Routing

```ts
import { RouterComponent, defineRouter } from 'rasengan';
import Home from './home.page';
import About from './about.page';
import AppLayout from './layout';

class AppRouter extends RouterComponent {}

export default defineRouter({
  layout: AppLayout,
  pages: [Home, About],
  notFoundComponent: NotFound,
})(AppRouter);
```

## File-based Routing

```ts
import { RouterComponent, defineRouter } from 'rasengan';
import Router from 'virtual:rasengan/router';

class AppRouter extends RouterComponent {}

export default defineRouter({ imports: [Router] })(AppRouter);
```

## Route Groups with defineRoutesGroup

```ts
import { defineRoutesGroup } from 'rasengan';

export default defineRoutesGroup({
  path: '/dashboard',
  pages: [Settings, Profile],
  routers: [AdminRouter],
});
```

## Dynamic Route Conventions

| File pattern | Route segment | Example URL |
|-------------|---------------|-------------|
| `[id].page.tsx` | `:id` | `/posts/123` |
| `[_locale]/index.page.tsx` | `:locale?` | `/en` or `/` |
| `(group)/index.page.tsx` | (ignored in path) | `/` |
| `_optional/index.page.tsx` | `optional?` | `/optional` or `` |

Access params:

```tsx
import { useParams } from 'rasengan';

const Post = () => {
  const { id } = useParams();
  return <div>Post {id}</div>;
};
```

## Rules

- Always create a class extending `RouterComponent` and wrap it with `defineRouter()`
- Config-based: list all pages in `pages` array, set `layout` explicitly, provide `notFoundComponent`
- File-based: use `import Router from 'virtual:rasengan/router'` with `{ imports: [Router] }`
- Use `routers` for nested sub-routers when splitting large route trees
- Set `useParentLayout: true` on child routers to inherit the parent's layout
- Use `defineRoutesGroup` to group routes under a shared path prefix without a separate router class
- Optional segments `[_param]` generate both with and without the segment
- Route groups `(group)` do not add a path segment — use them for organization only
- Underscore-prefixed directories `_name/` become optional path segments
- Do NOT import directly from `react-router` — import routing APIs from `rasengan`
