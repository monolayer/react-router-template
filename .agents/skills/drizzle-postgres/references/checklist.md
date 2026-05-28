# Drizzle Postgres Compliance Checklist

Use this checklist to perform final Quality Assurance checks before concluding the database implementation.

---

## 1. Bootstrapping & Infrastructure

- [ ] **Docker Configuration**: `docker-compose.yml` specifies `postgres:17-alpine` with correct port mapping `5432:5432` and healthy `pg_isready` checks.
- [ ] **Config Standard**: `drizzle.config.ts` dialect is strictly set to `"postgresql"`.
- [ ] **TypeScript Path Mapping**: Schema configuration in `drizzle.config.ts` maps directly to `./database/schema/index.ts`.
- [ ] **Clean `.env` Handling**: Both `.env.example` and `.env` contain valid, matching database URLs.
- [ ] **Auto-Detected Package Manager**: Dependencies are installed using the correct local package manager (`npm`, `pnpm`, `yarn`, or `bun`) based on workspace lockfiles.
- [ ] **Dependency Coverage**: All required dependencies (`drizzle-orm`, `pg`, `drizzle-kit`, `@types/pg`, `tsx`, `drizzle-seed`, `drizzle-zod`) are present.

---

## 2. Directory Layout & Module Design

- [ ] **Folder Structure**: All code resides in the root-level `./database/` folder under modular subdirectories (`schema/`, `validation/`).
- [ ] **Initial Bootstrap Safety**: If no custom domain model was requested, `database/schema/index.ts` re-exports a clean placeholder (`export {}`), and migration/seeding routines are gracefully postponed.
- [ ] **Column Explicit Mapping**: Every column in modular schema files explicitly sets its database-native column name representation (e.g. `createdAt: timestamp("created_at")`).
- [ ] **Relation Colocation**: All table relation configurations (`relations(...)`) are defined in the same file as their parent table definition.
- [ ] **Lazy Import Resolution**: Related table imports inside `relations(...)` definitions are wrapped in lazy arrow functions to eliminate circular reference errors.
- [ ] **Type Inference Standard**: Entity types are inferred using modern, standard properties (`$inferSelect` and `$inferInsert`).
- [ ] **Zod Separation**: All Zod validation schemas reside strictly under `database/validation/` (never in `database/schema/`), mapping schemas cleanly via `drizzle-zod`.
- [ ] **Index Barrel Exports**: Both `database/schema/` and `database/validation/` contain functional `index.ts` barrel files re-exporting modular domain structures.

---

## 3. connection Lifecycle & Context

- [ ] **Vite HMR Safety**: `database/index.ts` caches the pg `Pool` instance on `globalThis` in development mode to prevent connection exhaustion.
- [ ] **Express Server Mounting**: `DatabaseContext` middleware is safely wired into `server/app.ts` using `app.use(...)`.
- [ ] **Surgical Middleware Insertion**: Express middleware is inserted directly before the React Router request handler to preserve pre-existing server logic.
- [ ] **Strict Context Fetching**: Loaders, actions, and custom server controllers strictly retrieve the database client through the `database()` utility function.

---

## 4. Seeding & Scripts

- [ ] **Cascading Reset State**: The `database/seed.ts` script invokes `await reset(db, schema)` before populating mock records.
- [ ] **Deterministic Seeding**: `drizzle-seed` runner options include a fixed random number seed (e.g., `seed: 42`).
- [ ] **Hanging Process Guard**: The seeding script explicitly invokes `await pool.end()` upon successful completion or catch-block failure to let terminal scripts terminate.
- [ ] **Sequence Synchronization**: The seeding script resets the auto-increment identity sequence of seeded tables (e.g. using `SELECT setval(pg_get_serial_sequence('table', 'id'), COALESCE(MAX(id), 1))`) to prevent `duplicate key value violates unique constraint` errors on subsequent application writes.
- [ ] **Unified NPM Scripts**: `package.json` contains unified script definitions for generating, migrating, studio, seeding, and `db:create`.

---

## 5. Transactions & Queries

- [ ] **API Paradigm Consistency**: Standard SELECT builders are used for aggregations, mutations, and groupings; RQB is used for multi-nested entity loads.
- [ ] **Strict Transaction client (`tx`)**: Inside any transaction context (`db.transaction(async (tx) => { ... })`), all operations execute strictly against the local context runner client `tx`. Under no circumstances should the outer global/module `db` client be called.

---

## 6. Compilation & Lint Verification

- [ ] **Types Compile Cleanly**: The project successfully completes TypeScript typechecking without warnings or errors (`npm run typecheck`).
- [ ] **Standard Production Build**: Executing the production bundler succeeds without issue (`npm run build`).
- [ ] **Linter Conformity**: Source additions comply with workspace linter rules without generating error overrides.
