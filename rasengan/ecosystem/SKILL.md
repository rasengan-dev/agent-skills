---
name: rasengan-ecosystem
description: Rasengan.js ecosystem package patterns. Covers @rasenganjs/kurama (state management), @rasenganjs/image (optimized images), @rasenganjs/theme (light/dark/system), @rasenganjs/i18n (internationalization), @rasenganjs/mdx (MDX content), @rasenganjs/vercel (deployment), and @rasenganjs/serve (self-hosting). Use when integrating ecosystem packages into a Rasengan.js project.
license: MIT
metadata:
  author: dilane3
  framework: rasengan
  version: "2.0.0"
---

# Rasengan.js Ecosystem Package Patterns

## When to Activate

- Adding state management with `@rasenganjs/kurama`
- Optimizing images with `@rasenganjs/image`
- Implementing light/dark/system theming with `@rasenganjs/theme`
- Adding internationalization with `@rasenganjs/i18n`
- Setting up MDX content with `@rasenganjs/mdx`
- Deploying to Vercel with `@rasenganjs/vercel`
- Self-hosting with `@rasenganjs/serve`

## State Management — @rasenganjs/kurama

```ts
import { createStore } from '@rasenganjs/kurama';

const useStore = createStore(({ set }) => ({
  count: 0,
  increment: () => set(state => ({ count: state.count + 1 })),
  reset: () => set({ count: 0 }),
}));

// In a component:
function Counter() {
  const count = useStore(state => state.count);
  const increment = useStore(state => state.increment);
  return <button onClick={increment}>{count}</button>;
}

// Subscribe to multiple values:
const { count, increment } = useStore();
```

Rules:
- Create stores outside components to avoid re-creation on re-render
- Use selectors (`useStore(state => state.count)`) for granular subscriptions and to avoid unnecessary re-renders
- Call `set()` with a partial state or updater function (Zustand-like API)

## Image Optimization — @rasenganjs/image

```tsx
import { Image } from '@rasenganjs/image';

<Image
  src="/hero.jpg"
  alt="Hero image"
  width={1200}
  height={600}
  placeholder="blur"    // or "wave"
  lazy                  // enable lazy loading
/>
```

Rules:
- Always provide `width` and `height` to prevent Cumulative Layout Shift (CLS)
- Use `placeholder="blur"` for production — generates a blurred placeholder
- Use `placeholder="wave"` for a lighter, animated wave placeholder
- Use `lazy` prop to defer offscreen images (recommended for most images)
- Import from `@rasenganjs/image`, not a generic `<img>` tag

## Theming — @rasenganjs/theme

```tsx
import { ThemeProvider, useTheme } from '@rasenganjs/theme';

// Wrap in root App component:
<ThemeProvider defaultTheme="system">
  <Component router={AppRouter}>{children}</Component>
</ThemeProvider>

// In any component:
function ThemeToggle() {
  const { theme, setTheme } = useTheme();
  return (
    <button onClick={() => setTheme(theme === 'dark' ? 'light' : 'dark')}>
      {theme === 'dark' ? '☀️' : '🌙'}
    </button>
  );
}
```

Rules:
- Wrap at the App root level (inside `src/main.tsx`)
- `defaultTheme="system"` respects OS preference (recommended)
- Theme is persisted to `localStorage` automatically
- `useTheme()` returns `{ theme, setTheme, toggleTheme }`

## Internationalization — @rasenganjs/i18n

```tsx
import { I18nProvider, useLocale, useTranslation } from '@rasenganjs/i18n';

// Wrap in root App component:
<I18nProvider defaultLocale="en" locales={['en', 'fr', 'de']}>
  <Component router={AppRouter}>{children}</Component>
</I18nProvider>

// Translate in components:
function Greeting() {
  const { t } = useTranslation();
  return <h1>{t('welcome')}</h1>;
}

// Switch locale:
function LocaleSwitcher() {
  const { locale, setLocale } = useLocale();
  return (
    <select value={locale} onChange={e => setLocale(e.target.value)}>
      <option value="en">English</option>
      <option value="fr">Français</option>
    </select>
  );
}
```

Rules:
- Locale is auto-detected from URL by default; `setLocale` updates the URL
- Use flat key structures for translation files (e.g., `{ "welcome": "Hello" }`)
- Wrap `I18nProvider` at the App root, inside `src/main.tsx`
- `useTranslation()` loads translations for the current locale automatically

## Deployment

### Vercel

```bash
npm i @rasenganjs/vercel
# Add build command: rasengan build
# Add output directory: dist
```

#### Configuration

```js
// rasengan.config.js
import { defineConfig } from "rasengan";
import { rasengan } from "rasengan/plugin";
import { configure } from "@rasenganjs/vercel";
 
export default defineConfig({
	vite: {
		plugins: [
			rasengan({
				adapter: configure(),
			}),
		],
	},
});
```

### Self-hosted (Node)

```bash
npm @rasenganjs/serve
rasengan build
```

#### Usage

```json
// package.json
{
  "scripts": {
    "serve": "rasengan-serve ./dist"
  }
}
```

Rules:
- `@rasenganjs/vercel` is a dev dependency — the Vercel adapter
- `@rasenganjs/serve` is a dev dependency — starts a production Node server
- Build output is `dist/` — configure deployment to point there
- All ecosystem packages require `"type": "module"` in `package.json`
