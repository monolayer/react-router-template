---
name: app-development
description: Build full-stack React applications using React Router's framework mode. Use when building, scaffolding, or refactoring full-stack React Router applications or features, configuring routes, working with loaders and actions, handling forms, handling navigation, pending/optimistic UI, error boundaries, or working with react-router.config.ts or other react router conventions.
---

# Unified Application Development Handbook

This is the primary handbook and meta-skill for building, scaffolding, and refactoring full-stack React Router applications. It synthesizes full-stack routing and data flow with robust component composition patterns and industry-standard performance optimizations.

---

## 1. Bootstrapping & The Database Decision Matrix

Before implementing any data persistence layers or starting Drizzle migrations, evaluate the application requirements using this matrix to avoid overengineering:

| App Type / Characteristics                                                                                                 | Database Requirement                  | Recommended Action                                                                                                                                  |
| :------------------------------------------------------------------------------------------------------------------------- | :------------------------------------ | :-------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Static / Ephemeral** (Landing pages, marketing brochures, client-side only calculators, portfolios)                      | **❌ Forbidden (Overengineering)**    | Leverage static rendering, local client-only state, or hardcoded realistic placeholders. **Do not** write Docker Compose configs or configure ORMs. |
| **Client-Stored / User-Preference** (Dark mode toggle, local-only search history, personal settings)                       | **❌ No Database (Local Only)**       | Use `localStorage` or `cookies` with robust schema versioning and hydration mismatch guards.                                                        |
| **Stateful / Interactive Prototype** (Interactive e-commerce checkout flow, task board, dashboard prototype)               | **⚠️ Optional (Ask / Mock First)**    | Use local/in-memory mock-data persistence first unless the user explicitly requests persistent backend storage.                                     |
| **Persistent / Multi-user / Auth-Required** (SaaS apps, user dashboards, custom blogs with CMS, shared real-time canvases) | **✅ Mandatory (Load Drizzle Skill)** | Fully implement Drizzle ORM + PostgreSQL by invoking and following the `@.agents/skills/drizzle-postgres/` skill guidelines.                        |

> **💡 Directing Data Persistence Tasks:**
> When the matrix requires a database, the developer **must load and execute the `drizzle-postgres` skill** to handle Docker containers, initial migration bootstrapping, the pool setup on globalThis, and setting up AsyncLocalStorage for request-scoped DB connection contexts.

---

## 2. Procedural Full-Stack Workflow

When building or refactoring any application feature, adhere strictly to this sequential, four-phase procedural workflow:

### Phase 1: Route Setup & Configuration

1. **Define Routes**: Declare route paths explicitly inside `app/routes.ts`. Follow file-based layout nests or explicit path configurations.
2. **Setup Route Modules**: Create the route module file (e.g., `app/routes/home.tsx`).
3. **Type Safety**: Ensure the file utilizes the auto-generated types (`Route.LoaderArgs`, `Route.ComponentProps`, etc.) from `react-router typegen`.

### Phase 2: Server-Side Data Flow (Loaders & Actions)

1. **Request Isolation**: Ensure no mutable module-level state is used to share user request details. Reference: `rules/server-no-shared-module-state.md`.
2. **Parallel Fetching**: Start independent promises immediately, and run concurrent fetches via `Promise.all()` to prevent waterfalls. Reference: `rules/async-parallel.md`.
3. **Auth & Validation**: Validate form and search parameters using strict schema validations (e.g., Zod). Authenticate server-side operations directly inside each action endpoint.
4. **Database Context & Persistence**: If the application meets the criteria for persistent storage in the Database Decision Matrix, **use the skill `drizzle-postgres`**. Under its guidelines, use the request-scoped connection pools (`AsyncLocalStorage`) via `database()` getter inside loaders and actions. If inside a transaction, run queries strictly against the local transaction instance `tx`.

### Phase 3: Component Design & Composition

1. **Avoid Boolean Props**: Never expand a component's prop API with boolean flags to handle conditional design states. Represent design variants with distinct, composable child entities. Reference: `rules/architecture-avoid-boolean-props.md`.
2. **Compound Components**: Build complex UI blocks (such as modular dialogs, wizards, or inputs) using a shared context provider to allow flexible layout assembly. Reference: `rules/architecture-compound-components.md`.

### Phase 4: Client-Side Optimization & Polish

1. **Lazy State Initialization**: Always pass functions to `useState` for state values computed via expensive operations (such as parsing storage, building indexes). Reference: `rules/rerender-lazy-state-init.md`.
2. **Hydration Hygiene**: Eliminate visual flashing or hydration mismatches of storage-dependent values (like themes) by executing synchronized pre-hydration inline scripts. Reference: `rules/rendering-hydration-no-flicker.md`.

---

## 3. Reference Material Index

- **Full Routing, Sessions & Middleware**: When configuring nested routing, error boundaries, cookie sessions, or advanced middleware, read the reference [React Router Framework Mode](references/react-router-framework-mode.md).
- **Advanced Reusable Component Design**: When designing large component libraries, generic context interfaces, or refactoring boolean prop-heavy code, **use the skill `vercel-composition-patterns`**.
- **Deep Performance Optimization**: For exhaustive client/server-side performance optimization rules (bundle size, re-renders, JS speed), **use the skill `vercel-react-best-practices`**.
- **Database & Persistence Layer**: For configuring connection pooling, migrations, Zod schema validation, or relational queries API, **use the skill `drizzle-postgres`**.
