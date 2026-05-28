# Rasengan.js Agent Skills

Agent skills for [Rasengan.js](https://rasengan.dev) — the modern React meta-framework built on Vite and react-router.

## Skills

Skills are organized under the `rasengan/` parent folder. Each subdirectory is a separate installable skill:

| Skill | Description | Install |
|-------|-------------|---------|
| `rasengan/pages` | Page/layout components, metadata, MDX pages, entry point, navigation, 404 | `npx skills add rasengan-dev/agent-skills@rasengan/pages` |
| `rasengan/routing` | Router definition, config-based/file-based routing, dynamic routes, navigation hooks | `npx skills add rasengan-dev/agent-skills@rasengan/routing` |
| `rasengan/config` | Project configuration, SSR/SSG/SPA modes, CLI commands, TypeScript, env vars | `npx skills add rasengan-dev/agent-skills@rasengan/config` |
| `rasengan/data-fetching` | Loader functions, SSG static path generation, loading states | `npx skills add rasengan-dev/agent-skills@rasengan/data-fetching` |
| `rasengan/styling` | CSS Modules, Tailwind CSS, Sass/Less/Stylus preprocessors | `npx skills add rasengan-dev/agent-skills@rasengan/styling` |
| `rasengan/project-setup` | Project scaffolding, create-rasengan CLI, file structure, TypeScript setup | `npx skills add rasengan-dev/agent-skills@rasengan/project-setup` |
| `rasengan/optimizing` | Static assets (public/), metadata/SEO, Sage Mode / React Compiler | `npx skills add rasengan-dev/agent-skills@rasengan/optimizing` |
| `rasengan/deployment` | Vercel adapter, Node.js self-hosting, build output | `npx skills add rasengan-dev/agent-skills@rasengan/deployment` |
| `rasengan/ecosystem` | Kurama, Image, Theme, i18n, Kage Demo, MDX | `npx skills add rasengan-dev/agent-skills@rasengan/ecosystem` |

## Usage

```bash
npx skills add <owner/repo>@rasengan/<skill-name> -g -y
```

Skills are auto-loaded by your coding agent when working on Rasengan.js projects.
