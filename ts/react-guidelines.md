# React Guidelines

Generic frontend React tech stack and conventions. Project-specific code, data models, and visualizations belong in the project repo, not here.

## Setup

Run these in order to scaffold a new project from scratch.

```bash
# 1. Scaffold a Vite + React + TS project
pnpm create vite my-app --template react-ts
cd my-app

# 2. Core runtime deps
pnpm add react-router @tanstack/react-query zustand zod
pnpm add clsx tailwind-merge   # for the cn() helper

# 3. Styling (Tailwind v4 ships its own Vite plugin)
pnpm add -D tailwindcss @tailwindcss/vite

# 4. Testing
pnpm add -D vitest @testing-library/react @testing-library/jest-dom jsdom
pnpm add -D @playwright/test

# 5. Lint + format
pnpm add -D eslint prettier eslint-config-prettier

# 6. shadcn/ui — copies component sources into src/components/ui/
pnpm dlx shadcn@latest init

# 7. Verify
pnpm dev
```

After scaffolding, write the configs (`vite.config.ts`, `tsconfig.json`, `eslint.config.js`, `.prettierrc`, `playwright.config.ts`) and commit.

## Tech Stack

| Category | Tool | npm package | Source |
|----------|------|-------------|--------|
| Build / dev server | Vite | `vite` | https://github.com/vitejs/vite |
| Language | TypeScript (strict) | `typescript` | https://github.com/microsoft/TypeScript |
| Routing | React Router | `react-router` | https://github.com/remix-run/react-router |
| Server state | TanStack Query | `@tanstack/react-query` | https://github.com/TanStack/query |
| Client state | Zustand | `zustand` | https://github.com/pmndrs/zustand |
| Validation | Zod | `zod` | https://github.com/colinhacks/zod |
| UI components | shadcn/ui | `shadcn` (CLI) | https://github.com/shadcn-ui/ui |
| Styling | Tailwind CSS | `tailwindcss` | https://github.com/tailwindlabs/tailwindcss |
| Unit / component testing | Vitest + React Testing Library | `vitest`, `@testing-library/react` | https://github.com/vitest-dev/vitest |
| End-to-end testing | Playwright | `@playwright/test` | https://github.com/microsoft/playwright |
| Linter | ESLint | `eslint` | https://github.com/eslint/eslint |
| Formatter | Prettier | `prettier` | https://github.com/prettier/prettier |
| Package manager | pnpm | `pnpm` | https://github.com/pnpm/pnpm |

### shadcn/ui Is Not A Runtime Dependency

shadcn/ui is a CLI that **copies component source files into `src/components/ui/`**. You own and edit those files directly — there is no `shadcn` package to import from at runtime. Add new components with:

```bash
pnpm dlx shadcn@latest add button
pnpm dlx shadcn@latest add dialog
```

Then import from your own tree, e.g. `import { Button } from "@/components/ui/button"`.

## Project Structure

Generic skeleton; adjust per project but keep the top-level split.

```
src/
├── components/        # Reusable components (incl. shadcn/ui in src/components/ui/)
├── hooks/             # Custom React hooks (incl. TanStack Query hooks)
├── lib/               # Utilities (queryClient, cn, env, constants)
├── routes/            # React Router route components
├── stores/            # Zustand stores
└── types/             # Shared TypeScript types + Zod schemas
```

## Quick Start

```bash
pnpm install
pnpm dev
```

## Development Commands

```bash
pnpm install      # Install dependencies
pnpm dev          # Start Vite dev server with HMR
pnpm build        # tsc && vite build → dist/
pnpm preview      # Serve production build locally
pnpm lint         # ESLint
pnpm format       # Prettier --write
pnpm test         # Vitest (unit + component)
pnpm test:e2e     # Playwright
```

### Dev vs Build vs Preview

| Command | What it does | Default URL |
|---------|--------------|-------------|
| `pnpm dev` | Vite dev server with HMR | `localhost:5173` |
| `pnpm build` | Type-check + bundle to `dist/` | — |
| `pnpm preview` | Serve `dist/` to verify the production build | `localhost:4173` |

## State Separation

| State type | Tool | Examples |
|------------|------|----------|
| Server state | TanStack Query | API responses, cached resources |
| Client state (cross-component) | Zustand | Auth user, theme, multi-step wizard |
| URL state | React Router search params | Filters, pagination, selected tab |
| Ephemeral component state | `useState` | Toggle, hover, focus |

**Never** put server data in Zustand. Use TanStack Query so caching, refetching, and invalidation are handled in one place.

## Styling

Use Tailwind utilities. Combine classes with the `cn()` helper (typically `clsx` + `tailwind-merge`):

```tsx
import { cn } from "@/lib/utils";

<div className={cn("rounded-md p-4", isActive && "bg-blue-500 text-white")} />
```

Keep component exports separate from constant exports so Vite HMR stays fast:

```ts
// lib/constants.ts — non-component exports live here
export const COLORS = { /* ... */ };

// Component.tsx — component exports only
export function Component() { /* ... */ }
```

## Code Standards

- TypeScript `strict` mode enabled (`tsconfig.json`).
- No `any`. Use `unknown` plus type guards or Zod parsing.
- Validate every external boundary with Zod: network responses, `localStorage`, env vars, file reads.
- Path alias: `@/` maps to `src/`.
- ESLint owns code rules; Prettier owns formatting. Add `eslint-config-prettier` last in the ESLint extends array to disable formatting rules that would conflict with Prettier.
- ESLint + Prettier run in CI; fix locally before pushing.
- Test new behavior with Vitest; cover critical user flows with Playwright.
