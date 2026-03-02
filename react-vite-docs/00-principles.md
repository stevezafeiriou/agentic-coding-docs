# Core Principles & Decision Checklist

> This is the root file. AI agents must internalize these before writing any code.

---

## Non-Negotiable Principles

**TypeScript strictness is mandatory.**
All files use strict TypeScript. No `any`. No `as unknown as X`.
If you can't type it, redesign it. Type inference is preferred over explicit
annotation where unambiguous.

**Co-location over abstraction.**
Keep files that change together physically close. A component's styles,
types, hooks, and tests live next to it — not in separate global folders —
until reuse is proven, not assumed.

**Explicit over implicit.**
Supabase RLS is the security layer — never rely on frontend filtering alone.
Every route, every mutation, every realtime subscription has an explicit
access check. If it's not in a policy, it doesn't exist.

**Server first, client second.**
Fetch data at the highest appropriate level. Avoid waterfalls. Use Supabase
server-side clients in loaders/actions where the framework supports it.
Client-side fetching is for real-time updates and user-triggered mutations only.

**Fail visibly, recover gracefully.**
Every async operation has three states: loading, success, error — and the UI
renders all three. Errors are shown to users with actionable messages, not
swallowed or console.logged.

**Optimistic UI for mutations.**
Perceived performance matters more than actual performance. Mutations update
local state immediately and revert on error. Never make users wait for a
network round-trip to see their action reflected.

**Never repeat Supabase queries.**
Every query is a typed function in `/lib/queries/`. Components call these
functions — they never write inline Supabase query chains.

---

## Stack Reference

| Concern       | Tool                              | Notes                             |
| ------------- | --------------------------------- | --------------------------------- |
| Framework     | React 18+ with TypeScript         | Strict mode always on             |
| Build         | Vite 5+                           | ESM-first, HMR                    |
| Styling       | Tailwind CSS v3                   | Design tokens in config           |
| UI Components | shadcn/ui (Radix primitives)      | Copy-into-repo model              |
| Backend       | Supabase                          | Postgres, Auth, Realtime, Storage |
| Data fetching | TanStack Query v5                 | Server state only                 |
| Forms         | React Hook Form + Zod             | Validation at schema level        |
| Routing       | React Router v6 / TanStack Router | Typed routes                      |
| Global state  | Zustand                           | UI state only, never server state |
| Icons         | Lucide React                      | Consistent, tree-shakeable        |

---

## New Feature Decision Checklist

Before writing any code for a new feature, answer these:

### Data

- [ ] Where does this data live? (Supabase table, edge function, external API)
- [ ] Who can read it? → Write the RLS SELECT policy first
- [ ] Who can write it? → Write the RLS INSERT/UPDATE/DELETE policies first
- [ ] Does it need real-time updates? → Supabase Realtime subscription
- [ ] Is it user-specific or organization-wide? → Add `user_id` or `org_id` FK

### Component

- [ ] Is this a page, a feature, or a shared primitive?
- [ ] What are its loading / error / empty states?
- [ ] Does it need optimistic updates?
- [ ] Is the data server state (TanStack Query) or UI state (Zustand/useState)?
- [ ] Does it need to be accessible? (keyboard nav, ARIA labels, focus traps)

### Types

- [ ] Is there a generated Supabase type for this table? (`Database['public']['Tables']['x']`)
- [ ] Are all props typed — no implicit `any`?
- [ ] Are all API responses validated with Zod before use?

### Performance

- [ ] Does this component need memoization? (Only add if profiled)
- [ ] Does this query need pagination?
- [ ] Is this image/asset optimized?

---

## Project Structure

```
src/
├── app/                    # Route components (pages)
│   ├── dashboard/
│   │   ├── page.tsx
│   │   ├── layout.tsx
│   │   └── components/     # Route-local components
├── components/             # Shared UI components
│   ├── ui/                 # shadcn/ui primitives (auto-generated)
│   └── [feature]/          # Feature-specific shared components
├── hooks/                  # Shared custom hooks
├── lib/
│   ├── supabase/
│   │   ├── client.ts       # Browser client (singleton)
│   │   ├── server.ts       # Server client (per-request)
│   │   └── types.ts        # Re-export generated Database types
│   ├── queries/            # ALL Supabase query functions
│   │   ├── users.ts
│   │   ├── organizations.ts
│   │   └── [table].ts
│   ├── mutations/          # ALL Supabase mutation functions
│   └── validations/        # Zod schemas
├── stores/                 # Zustand stores (UI state only)
├── types/                  # Global TypeScript types
└── utils/                  # Pure utility functions
```
