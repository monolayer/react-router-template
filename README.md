# React Router, Node, and PostgreSQL Template

This template provides a modern, production-ready stack for building full-stack applications with React Router v7, Express, PostgreSQL, and Drizzle ORM.

## About the Template

Use this template to scaffold application features with a production-grade containerized database and type-safe schema modeling. It handles server-side rendering (SSR), Hot Module Replacement (HMR), static asset compilation, and clean server-to-client context passing out of the box.

### Key Capabilities

- **Server-Side Rendering:** Renders your React components on the server for fast initial loading times.
- **Type-Safe Database Modeling:** Integrates Drizzle ORM to keep your TypeScript definitions and database schema fully in sync.
- **Request-Scoped Database Context:** Uses Node's `AsyncLocalStorage` to securely distribute your database client to application loaders and actions without global variables.
- **Asset Optimization:** Bundles client-side assets and Tailwind CSS with Vite.

---

## Setting Up Your Environment

Before starting the application, you must configure your local environment and spin up the database container.

### Prerequisites

To complete the setup, you must install the following software on your system:

- **Node.js** version 24 or later.
- **Docker** and **Docker Compose**.

### Initializing the Configuration

1. **Install the dependencies** by running the install command in your terminal:

   ```bash
   npm install
   ```

2. **Copy the environment template** to create your local environment file:

   ```bash
   cp .env.example .env
   ```

3. **Verify the connection settings** in your `.env` file. By default, the template specifies:

   ```env
   NODE_ENV="development"
   DATABASE_URL="postgresql://postgres:postgres@localhost:5432/app_db"
   ```

**IMPORTANT:** You must specify `NODE_ENV="development"` for the Express server to launch Vite in middleware development mode. If `NODE_ENV` is omitted or not set to `"development"`, the server starts in production mode and will look for a compiled client build.

---

## Running the Application

Follow these steps to start your PostgreSQL container, apply migrations, and run the development server.

### Starting Your Database Container

1. **Launch the database** in the background using Docker Compose:

   ```bash
   docker compose up -d
   ```

2. **Verify the container is active and healthy** by listing your running containers:

   ```bash
   docker ps
   ```

   **NOTE:** The database service includes an automatic healthcheck. You can confirm the status by verifying that the output contains `(healthy)` under the `STATUS` column.

### Running Your Database Migrations

You must apply Drizzle migrations to configure the required tables in your database before the application can load data.

1. **Generate the migration files** if you made schema changes:

   ```bash
   npm run db:generate
   ```

2. **Apply the migrations** to your active database instance:

   ```bash
   npm run db:migrate
   ```

### Starting the Development Server

1. **Launch the server** with hot module replacement:

   ```bash
   npm run dev
   ```

2. **Open your browser** and navigate to `http://localhost:3000` to interact with your running application.

---

## Managing Your Database

This template uses Drizzle ORM for type-safe database queries.

### Modifying the Database Schema

1. **Update the schema** definitions inside the `database/schema.ts` file.
2. **Generate a new migration** SQL file using the generate command:

   ```bash
   npm run db:generate
   ```

3. **Execute the migration** to update your database table layout:

   ```bash
   npm run db:migrate
   ```

### Request-Scoped Client Architecture

To prevent socket leaks and secure client instances, the application initiates a connection pool in `server/app.ts` and runs each Express request within an isolated transaction context:

```typescript
app.use((_, __, next) => DatabaseContext.run(db, next));
```

Inside your React Router actions or loaders, you can retrieve the active database client by invoking the context helper:

```typescript
import { database } from "~/database/context";

export async function loader() {
  const db = database();
  const records = await db.query.guestBook.findMany();
  return { records };
}
```

---

## Preparing for Production

When you are ready to compile and deploy your application, use the production build commands.

### Compiling Your Application

1. **Generate the production build** for both client and server targets:

   ```bash
   npm run build
   ```

2. **Launch the production server**:

   ```bash
   npm run start
   ```

### Packaging with Docker

You can package and deploy your compiled application using the provided Dockerfile.

1. **Build the container image** with your tag name:

   ```bash
   docker build -t my-react-router-app .
   ```

2. **Run the container** on your local machine or container hosting service:

   ```bash
   docker run -p 3000:3000 --env-file .env my-react-router-app
   ```
