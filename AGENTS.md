# Agent Guide (AGENTS.md)

This guide provides automated agents with a fast, high-signal orientation of the codebase structure, build workflows, data flows, and critical development constraints.

---

## 1. Quick Orientation

Use this folder-level mapping to locate files relevant to your target modifications:

```
├── server.js               # Express application wrapper (manages development/production entrypoints)
├── server/
│   └── app.ts              # Express application configuration & server initialization
├── app/
│   ├── routes.ts           # Route matching manifest and indices
│   ├── root.tsx            # Main root application shell & document structure
│   └── routes/
│       └── home.tsx        # Homepage route displaying loader data & handling actions
├── package.json            # Scripts, project metadata, and dependency declarations
└── tsconfig.json           # Compiler rules for TypeScript compatibility
```

---

## 2. Command Reference

Execute these commands to manage dependencies and the compilation cycle.

### Application Development and Build Verification

- **Start local development server (with Vite middleware and HMR):**

  ```bash
  npm run dev
  ```

- **Run TypeScript type verification:**

  ```bash
  npm run typecheck
  ```

- **Build production assets:**

  ```bash
  npm run build
  ```

- **Start production server (requires compiled build files):**

  ```bash
  npm run start
  ```

---

## 3. Architecture and Data Flow

Understanding the pipeline of request handling is crucial for safe code changes.

### Request Pipeline and Middleware Context

```
[HTTP Request] ──> [server.js]
                        │
                        ▼ (Is NODE_ENV === "development"?)
             ├── Yes ──> [Load Vite Dev Middleware]
             └── No  ──> [Serve /build Static Assets]
                        │
                        ▼
                [server/app.ts]
                        │
                        ▼ (Render React Shell)
                [React Router SSR] ──> [HTTP Response]
```

### Key Design Patterns

- **Vite SSR Load Module:** In development mode, `server.js` dynamic-imports module `./server/app.ts` via Vite's Server-Side Rendering (SSR) module loader. This ensures live hot-reloading for server code changes without requiring manual Node process restarts.

---

## 4. Invariants and Safety Rules

When making changes to this codebase, you must preserve the following constraints:

### Environment and State Configuration

- **The NODE_ENV Development invariant:** You must configure `NODE_ENV="development"` in your `.env` file for local development. If omitted or set to any other value, `server.js` attempts to load compiled production files from `./build/server/index.js`, which results in an immediate startup crash when no build folder is present.

## 5. Application Data

- **Drizzle and PostgreSQL requirement:** Application data needs to be stored in a database, requiring a database integration. Use Drizzle ORM with PostgreSQL. Initialize, configure, and maintain this setup strictly by invoking and following the `drizzle-postgres` skill.

### TypeScript and Compilation Hygiene

- **Unused Type Suppressions:** Do not include redundant or unused `@ts-expect-error` or `@ts-ignore` comments in your server or routing code. All virtual imports (such as `virtual:react-router/server-build`) are automatically typed via `react-router typegen` and must compile cleanly during `npm run typecheck`.

---

## 5. Change Verification Checklist

Before finalizing any modifications, verify compliance with the following checklists:

### Before Committing Changes

1. **Verify environment variable values** in your `.env` file are correct.
2. **Run compiler verification** using the type-check script to confirm no broken interfaces:

   ```bash
   npm run typecheck
   ```

3. **Compile the production application** to verify there are no compilation errors:

   ```bash
   npm run build
   ```
