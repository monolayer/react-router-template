# Drizzle Postgres Code Patterns

Use these patterns to guide your implementation. Always adapt paths, imports, aliases, and server structures to the project context.

---

## 1. Package Installation

Auto-detect the appropriate package manager from local lockfile signatures (`package-lock.json` -> `npm`, `pnpm-lock.yaml` -> `pnpm`, `yarn.lock` -> `yarn`, `bun.lockb` -> `bun`) and run the equivalent installation commands:

### npm

```bash
npm install drizzle-orm pg
npm install -D drizzle-kit @types/pg tsx typescript drizzle-seed drizzle-zod
```

### pnpm

```bash
pnpm add drizzle-orm pg
pnpm add -D drizzle-kit @types/pg tsx typescript drizzle-seed drizzle-zod
```

---

## 2. Docker Infrastructure (`docker-compose.yml`)

Always run the lightweight and secure **`postgres:17-alpine`** image with automatic healthchecks:

```yaml
services:
  db:
    image: postgres:17-alpine
    container_name: local_postgres
    restart: always
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: app_db
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d app_db"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

---

## 3. Environment Setup (`.env` and `.env.example`)

Inject the canonical database credentials, overwriting existing variables:

```env
NODE_ENV="development"
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/app_db"
```

---

## 4. Drizzle Configuration (`drizzle.config.ts`)

```typescript
import { defineConfig } from "drizzle-kit";

if (!process.env.DATABASE_URL) {
  throw new Error("DATABASE_URL is required");
}

export default defineConfig({
  out: "./drizzle",
  schema: "./database/schema/index.ts",
  dialect: "postgresql",
  dbCredentials: {
    url: process.env.DATABASE_URL,
  },
  strict: true,
  verbose: true,
});
```

---

## 5. Hot-Reload-Safe Connection Pool (`database/index.ts`)

Store the pool on `globalThis` to prevent Vite from spawning multiple database pools during hot-module reloads:

```typescript
import { Pool } from "pg";
import { drizzle } from "drizzle-orm/node-postgres";
import * as schema from "./schema";

const globalForDb = globalThis as unknown as {
  pool: Pool | undefined;
};

export const pool =
  globalForDb.pool ??
  new Pool({
    connectionString: process.env.DATABASE_URL,
  });

if (process.env.NODE_ENV !== "production") {
  globalForDb.pool = pool;
}

export const db = drizzle(pool, { schema });
```

---

## 6. Modular Database Schemas (`database/schema/`)

Colocate table structure and relations in dedicated domain files, and re-export them from a central index.

### Initial Clean Placeholder (`database/schema/index.ts`)

```typescript
// Initial bootstrap placeholder. Add custom domain exports here once requested.
export {};
```

### Table & Relation Colocation Example (`database/schema/users.ts`)

```typescript
import { pgTable, integer, varchar, timestamp } from "drizzle-orm/pg-core";
import { relations } from "drizzle-orm";
import { posts } from "./posts";

export const users = pgTable("users", {
  id: integer("id").primaryKey().generatedAlwaysAsIdentity(),
  name: varchar("name", { length: 255 }).notNull(),
  email: varchar("email", { length: 255 }).notNull().unique(),
  createdAt: timestamp("created_at").defaultNow().notNull(),
});

export const usersRelations = relations(users, ({ many }) => ({
  posts: many(posts),
}));

export type User = typeof users.$inferSelect;
export type NewUser = typeof users.$inferInsert;
```

### Table & Relation Colocation Example (`database/schema/posts.ts`)

```typescript
import { pgTable, integer, text, varchar, timestamp } from "drizzle-orm/pg-core";
import { relations } from "drizzle-orm";
import { users } from "./users";

export const posts = pgTable("posts", {
  id: integer("id").primaryKey().generatedAlwaysAsIdentity(),
  title: varchar("title", { length: 255 }).notNull(),
  content: text("content").notNull(),
  authorId: integer("author_id")
    .references(() => users.id, { onDelete: "cascade" })
    .notNull(),
  createdAt: timestamp("created_at").defaultNow().notNull(),
});

export const postsRelations = relations(posts, ({ one }) => ({
  author: one(users, {
    fields: [posts.authorId],
    references: [users.id],
  }),
}));

export type Post = typeof posts.$inferSelect;
export type NewPost = typeof posts.$inferInsert;
```

### Central Schema Index (`database/schema/index.ts`)

```typescript
export * from "./users";
export * from "./posts";
```

---

## 7. Modular Zod Validations (`database/validation/`)

Always define validation logic inside modular files separated from core database schema definitions:

### Modular Validation Example (`database/validation/users.ts`)

```typescript
import { createInsertSchema, createSelectSchema } from "drizzle-zod";
import { users } from "../schema/users";
import { z } from "zod";

export const insertUserSchema = createInsertSchema(users, {
  name: (schema) => schema.min(2).max(100),
  email: (schema) => schema.email("Invalid email address"),
});

export const selectUserSchema = createSelectSchema(users);
```

### Central Validation Index (`database/validation/index.ts`)

```typescript
export * from "./users";
export * from "./posts";
```

---

## 8. Request-Scoped Context Isolation (`database/context.ts`)

Enforce request-scoped connections to ensure request isolation and protect loaders and actions against database context contamination:

```typescript
import { AsyncLocalStorage } from "node:async_hooks";
import type { NodePgDatabase } from "drizzle-orm/node-postgres";
import * as schema from "./schema";

export const DatabaseContext = new AsyncLocalStorage<NodePgDatabase<typeof schema>>();

export function database() {
  const db = DatabaseContext.getStore();
  if (!db) {
    throw new Error(
      "DatabaseContext is not initialized. Make sure your request is running within the Express middleware boundary.",
    );
  }
  return db;
}
```

---

## 9. Seed Script Runner (`database/seed.ts`)

Use the official `drizzle-seed` library with a fixed random seed. The PG pool **must** be explicitly terminated to ensure script completion in CI/CD and script environments. **You must always synchronize any auto-incrementing primary key identity sequences right after seeding to prevent subsequent application inserts from running into unique key constraint violations.**

```typescript
import { drizzle } from "drizzle-orm/node-postgres";
import { seed, reset } from "drizzle-seed";
import { sql } from "drizzle-orm";
import { pool } from "./index";
import * as schema from "./schema";

async function main() {
  const db = drizzle(pool);

  console.log("Resetting database tables...");
  await reset(db, schema);

  console.log("Seeding database with deterministic mock data...");
  await seed(db, schema, { seed: 42 }).refine((funcs) => ({
    users: {
      count: 20,
      columns: {
        name: funcs.fullName(),
        email: funcs.email(),
      },
    },
  }));

  console.log("Resetting auto-increment sequence to prevent unique constraint conflicts...");
  await db.execute(
    sql`SELECT setval(pg_get_serial_sequence('users', 'id'), COALESCE(MAX(id), 1)) FROM users;`,
  );

  console.log("Database seeding completed successfully.");
  await pool.end();
}

main().catch(async (err) => {
  console.error("Database seeding failed:", err);
  await pool.end();
  process.exit(1);
});
```

---

## 10. Express Server Wiring (`server/app.ts`)

Surgically patch the `server/app.ts` file to import the database connections and run the `DatabaseContext` middleware right before mounting the React Router request handler:

```typescript
import { createRequestHandler } from "@react-router/express";
import express from "express";
import "react-router";

// Surgical Additions
import { db } from "~/database/index";
import { DatabaseContext } from "~/database/context";

declare module "react-router" {
  interface AppLoadContext {
    VALUE_FROM_EXPRESS: string;
  }
}

export const app = express();

// Wire Request-Scoped Database Context Middleware
app.use((_, __, next) => DatabaseContext.run(db, next));

app.use(
  createRequestHandler({
    build: () => import("virtual:react-router/server-build"),
    getLoadContext() {
      return {
        VALUE_FROM_EXPRESS: "Hello from Express",
      };
    },
  }),
);
```

---

## 11. Database NPM Scripts (`package.json`)

```json
"scripts": {
  "db:generate": "dotenv -- drizzle-kit generate",
  "db:migrate": "dotenv -- drizzle-kit migrate",
  "db:studio": "dotenv -- drizzle-kit studio",
  "db:seed": "dotenv -- tsx database/seed.ts",
  "db:create": "npm run db:migrate && npm run db:seed"
}
```

---

## 12. Query and Transaction Patterns inside React Router

All queries, mutations, and transactions inside route module actions and loaders must be run against the request-scoped database context client.

### Standard Loader Query

```typescript
import { database } from "~/database/context";
import { users } from "~/database/schema";

export async function loader() {
  const db = database();
  // Fetch users utilizing the Relational Queries API
  const activeUsers = await db.query.users.findMany();
  return { users: activeUsers };
}
```

### Standard Action Mutation

```typescript
import { database } from "~/database/context";
import { users } from "~/database/schema";

export async function action({ request }: { request: Request }) {
  const db = database();
  const formData = await request.formData();
  const name = formData.get("name") as string;
  const email = formData.get("email") as string;

  await db.insert(users).values({ name, email });
  return { success: true };
}
```

### Safe Database Transactions

Inside `db.transaction(...)`, all database actions **must** execute against the scoped `tx` runner, never the outer or global `db` client.

```typescript
import { database } from "~/database/context";
import { users, posts } from "~/database/schema";

export async function action() {
  const db = database();

  await db.transaction(async (tx) => {
    const [newUser] = await tx
      .insert(users)
      .values({ name: "Jane Doe", email: "jane@example.com" })
      .returning();

    await tx.insert(posts).values({
      title: "First Post",
      content: "Hello World!",
      authorId: newUser.id,
    });
  });

  return { success: true };
}
```
