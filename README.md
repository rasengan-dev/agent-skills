# Rasengan.js Agent Skills

Agent skills for [Rasengan.js](https://rasengan.dev) — the modern React meta-framework built on Vite and react-router.

## Skills

Skills are organized under per-package parent folders (`rasengan/`, `futon/`, `rasengan-server/`). Each subdirectory is a separate installable skill.

### Frontend framework (`rasengan/`)

| Skill | Description | Install |
|-------|-------------|---------|
| `rasengan-pages` | Page/layout components, metadata, MDX pages, entry point, navigation, 404 | `npx skills add rasengan-dev/agent-skills@rasengan-pages` |
| `rasengan-routing` | Router definition, config-based/file-based routing, dynamic routes, navigation hooks | `npx skills add rasengan-dev/agent-skills@rasengan-routing` |
| `rasengan-config` | Project configuration, SSR/SSG/SPA modes, CLI commands, TypeScript, env vars | `npx skills add rasengan-dev/agent-skills@rasengan-config` |
| `rasengan-data-fetching` | Loader functions, SSG static path generation, loading states | `npx skills add rasengan-dev/agent-skills@rasengan-data-fetching` |
| `rasengan-styling` | CSS Modules, Tailwind CSS, Sass/Less/Stylus preprocessors | `npx skills add rasengan-dev/agent-skills@rasengan-styling` |
| `rasengan-project-setup` | Project scaffolding, create-rasengan CLI, file structure, TypeScript setup | `npx skills add rasengan-dev/agent-skills@rasengan-project-setup` |
| `rasengan-optimizing` | Static assets (public/), metadata/SEO, Sage Mode / React Compiler | `npx skills add rasengan-dev/agent-skills@rasengan-optimizing` |
| `rasengan-deployment` | Vercel adapter, Node.js self-hosting, build output | `npx skills add rasengan-dev/agent-skills@rasengan-deployment` |
| `rasengan-ecosystem` | Kurama, Image, Theme, i18n, Kage Demo, MDX | `npx skills add rasengan-dev/agent-skills@rasengan-ecosystem` |

### HTTP runtime (`futon/`)

`@rasenganjs/futon` is the zero-dependency, WinterCG-compatible HTTP middleware/router library underneath `@rasenganjs/server` — usable standalone too.

| Skill | Description | Install |
|-------|-------------|---------|
| `rasengan-futon-core` | Futon app class, middleware onion model, built-in middleware, Context, response helpers, cookies, hooks, HttpError | `npx skills add rasengan-dev/agent-skills@rasengan-futon-core` |
| `rasengan-futon-routing` | Router class, route groups, radix-tree dispatch, path param types, body parsing | `npx skills add rasengan-dev/agent-skills@rasengan-futon-routing` |
| `rasengan-futon-adapters` | RuntimeAdapter interface, Node/Bun/Workerd adapters, Express and WinterCG bridges | `npx skills add rasengan-dev/agent-skills@rasengan-futon-adapters` |

### Backend framework (`rasengan-server/`)

`@rasenganjs/server` is the modular, controller-based backend framework built on `@rasenganjs/futon`, plus its closely-related ecosystem packages (`@rasenganjs/ws`, `@rasenganjs/queue`, `@rasenganjs/validators`, `@rasenganjs/drizzle`).

| Skill | Description | Install |
|-------|-------------|---------|
| `rasengan-server-core` | bootstrap, ServerApp, modules, controllers, DI/Container, config, CLI | `npx skills add rasengan-dev/agent-skills@rasengan-server-core` |
| `rasengan-server-validation` | Zod schema validation, per-route and controller-level schemas | `npx skills add rasengan-dev/agent-skills@rasengan-server-validation` |
| `rasengan-server-websockets` | Raw app.websocket(), @rasenganjs/ws Gateways, rooms, broadcasting | `npx skills add rasengan-dev/agent-skills@rasengan-server-websockets` |
| `rasengan-server-queues` | @rasenganjs/queue background jobs, retries/backoff, recurring jobs | `npx skills add rasengan-dev/agent-skills@rasengan-server-queues` |
| `rasengan-server-uploads` | Multipart file uploads via futon's fileUpload() + diskStorage() | `npx skills add rasengan-dev/agent-skills@rasengan-server-uploads` |
| `rasengan-server-drizzle` | Drizzle ORM integration — DrizzleModule.forRoot(), DataSource provider, driver adapters, migrations | `npx skills add rasengan-dev/agent-skills@rasengan-server-drizzle` |

## Usage

### Install all skills (recommended)

```bash
npx skills add rasengan-dev/agent-skills --all
```

### Install individual skills

```bash
npx skills add rasengan-dev/agent-skills@rasengan-<skill-name> -g -y
```

### Install multiple specific skills

```bash
npx skills add rasengan-dev/agent-skills --skill rasengan-pages --skill rasengan-routing
```

Skills are auto-loaded by your coding agent when working on Rasengan.js projects.
