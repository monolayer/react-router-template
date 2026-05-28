---
name: drizzle-postgres
description: Add Drizzle ORM and PostgreSQL to Express-backed React Router applications. Use when installing or configuring Drizzle, drizzle-kit, pg, Docker Compose PostgreSQL, migration scripts, database schema folders, request-scoped database context, seed runner with drizzle-seed, or database-backed React Router loaders/actions.
---

# Drizzle Postgres

Use this skill to implement a fully-featured, production-ready Drizzle ORM + PostgreSQL integration in a React Router app served by Express. This skill standardizes on the `pg` (node-postgres) driver, request-scoped context isolation via `AsyncLocalStorage`, deterministic seeding via `drizzle-seed`, and robust local hot-reload safety.

## Discovery First

Before editing, inspect the project for:

- Package manager and lockfiles (to detect whether to run `npm`, `pnpm`, `yarn`, or `bun`).
- TypeScript includes, configuration paths, and path aliases.
- Server entrypoint (`server.js`, `server/app.ts`), Express request handler placement, and route module conventions.

## Implementation Workflow

### 1. Bootstrap Phase (Dependencies & Infrastructure)

1. **Install Core Dependencies**: Auto-detect the package manager via lockfile markers (e.g. `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `bun.lockb`) and install:
   - Runtime dependencies: `drizzle-orm` and `pg`.
   - Development dependencies: `drizzle-kit`, `@types/pg`, `tsx`, `typescript`, `drizzle-seed`, and `drizzle-zod`.
2. **Add Local PostgreSQL Container**: Write or overwrite the root `docker-compose.yml` to run a lightweight, modern **`postgres:17-alpine`** container. Use default credentials: user `postgres`, password `postgres`, database `app_db`, host port `5432`, and a named volume `pgdata` with a healthy `pg_isready` check.
3. **Configure Environment Variables**: Set or overwrite `DATABASE_URL="postgresql://postgres:postgres@localhost:5432/app_db"` inside `.env.example` and `.env`. Preserve other unrelated variables.
4. **Create Drizzle Configuration**: Add a root-level `drizzle.config.ts` specifying the `postgresql` dialect, mapping `schema` to `./database/schema/index.ts`, and saving output migrations under `./drizzle`.

### 2. Database Layer Implementation

1. **Initialize Connection Pool with Vite HMR Safety**: Create `database/index.ts` utilizing `pg`'s `Pool`. Store the active connection pool on `globalThis` during development (`process.env.NODE_ENV !== "production"`) to prevent connection leak crashes during Vite module reloads.
2. **Add Modular Schema Directory**: Create the folder `database/schema/`.
   - Table definitions, relations (`relations(...)`), and infer type exports must be colocated inside separate domain files.
   - Collect and re-export all entities from `database/schema/index.ts`.
   - For initial bootstrapping, write `export {};` inside `index.ts` to keep the schema clean until real tables are declared.
3. **Add Modular Validation Directory**: Create `database/validation/` to hold Zod schemas generated via `drizzle-zod`'s `createInsertSchema` and `createSelectSchema`. Barrel-export all validations from `database/validation/index.ts`.
4. **Wire Request-Scoped Database Context**: Create `database/context.ts` using Node's `AsyncLocalStorage` and a typed `NodePgDatabase<typeof schema>` context store. Export the `DatabaseContext` and a mandatory `database()` getter.
5. **Implement Seed Runner**: Create `database/seed.ts` utilizing the official `drizzle-seed` runner. Ensure that the script performs a cascading `reset(db, schema)` before seeding. **Because `drizzle-seed` inserts explicit values for primary keys, you must always run a PostgreSQL sequence reset query (e.g. `SELECT setval(pg_get_serial_sequence('table', 'id'), COALESCE(MAX(id), 1))`) right after seeding to prevent primary key sequence desynchronization and unique constraint errors on subsequent app inserts.** The script must strictly end with `await pool.end()` to prevent hanging shell processes.

### 3. Application Integration

1. **Surgically Patch Express Server**: Modify `server/app.ts` to import `pool`, `db`, and `DatabaseContext`. Run the middleware `app.use((_, __, next) => DatabaseContext.run(db, next))` right before `createRequestHandler(...)`. Do not overwrite the entire server file; surgically inject the middleware to keep other custom logging, compression, or load context untouched.
2. **Inject Unified NPM Scripts**: Overwrite/inject standard scripting commands in `package.json` utilizing `dotenv-cli`:
   ```json
   "db:generate": "dotenv -- drizzle-kit generate",
   "db:migrate": "dotenv -- drizzle-kit migrate",
   "db:studio": "dotenv -- drizzle-kit studio",
   "db:seed": "dotenv -- tsx database/seed.ts",
   "db:create": "npm run db:migrate && npm run db:seed"
   ```
3. **Apply Clean Placeholder Strategy**: If no custom schema has been requested by the user, bootstrap with empty schema exports (`export {}`). Skip generating migrations or executing seeds, and instruct the user that migrations are ready once tables are defined.

## Defaults and Guardrails

- Standardize on `./database/` at the root folder.
- Ensure all column declarations specify their PG-native casing and mapping explicitly (e.g., `id: integer('id')`, `createdAt: timestamp('created_at')`).
- Use the **Relational Queries API (RQB)** for complex, nested data fetching, and the **Core SQL-like Select** builder for analytical summaries, aggregations, and standard modifications.
- Inside transactions (`db.transaction(async (tx) => { ... })`), strictly query against the transaction context client `tx`. Under no circumstances should the global `db` client be called within a transaction context.
- Never let connections hang; any direct script runners (such as `seed.ts`) must close the connection using `await pool.end()`.

## Verification

After executing, perform standard verification:

- Start the Docker Postgres container and confirm it reports a status of `(healthy)`.
- If custom tables were defined, run `npm run db:create` to verify migrating, and seeding completes cleanly.
- Run `npm run typecheck` to confirm the compilation passes without any TypeScript warnings or errors.
- Run the production asset build with `npm run build` to guarantee compilation safety.

## References

- Load `references/patterns.md` to access exact code templates for Docker Compose, `globalThis` connection pooling, AsyncLocalStorage middleware, and modular schema/validation/seeding architectures.
- Load `references/checklist.md` to execute the final compliance and quality assurance checks before finishing database tasks.
