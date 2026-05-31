# React Router Framework Mode

Framework mode is React Router's full-stack development experience with file-based routing, server-side, client-side, and static rendering strategies, data loading and mutations, and type-safe route module API.

## When to Apply

- Configuring new routes (`app/routes.ts`)
- Loading data with `loader` or `clientLoader`
- Handling mutations with `action` or `clientAction`
- Navigating with `<Link>`, `<NavLink>`, `<Form>`, `redirect`, and `useNavigate`
- Implementing pending/loading UI states
- Configuring SSR, SPA mode, or pre-rendering (`react-router.config.ts`)
- Implementing authentication

## References

Load the relevant reference for detailed guidance on the specific API/concept:

| Reference                                                                      | Use When                                              |
| ------------------------------------------------------------------------------ | ----------------------------------------------------- |
| [routing](react-router-framework-mode/routing.md)                              | Configuring routes, nested routes, dynamic segments   |
| [route modules](react-router-framework-mode/route-modules.md)                  | Understanding all route module exports                |
| [Special Files](react-router-framework-mode/special-files.md)                  | Customizing root.tsx, adding global nav/footer, fonts |
| [Data Loading](react-router-framework-mode/data-loading.md)                    | Loading data with loaders, streaming, caching         |
| [Actions and Form Handling](react-router-framework-mode/actions.md)            | Handling forms, mutations, validation                 |
| [Navigation](react-router-framework-mode/navigation.md)                        | Links, programmatic navigation, redirects             |
| [Pending UI and Optimistic Updates](react-router-framework-mode/pending-ui.md) | Loading states, optimistic UI                         |
| [Error Handling](react-router-framework-mode/error-handling.md)                | Error boundaries, error reporting                     |
| [Rendering Strategies](react-router-framework-mode/rendering-strategies.md)    | SSR vs SPA vs pre-rendering configuration             |
| [Middleware & Context API](react-router-framework-mode/middleware.md)          | Adding middleware (requires v7.9.0+)                  |
| [Sessions & Cookies](react-router-framework-mode/sessions.md)                  | Cookie sessions, authentication, protected routes     |
| [Route Module Types](react-router-framework-mode/type-safety.md)               | Auto-generated route types, type imports, type safety |

## Version Compatibility

Some features require specific React Router versions. **Always verify before implementing:**

```bash
npm list react-router
```

| Feature                 | Minimum Version | Notes                         |
| ----------------------- | --------------- | ----------------------------- |
| Middleware              | 7.9.0+          | Requires `v8_middleware` flag |
| Core framework features | 7.0.0+          | loaders, actions, Form, etc.  |

## Critical Patterns

These are the most important patterns to follow. Load the relevant reference for full details.

### Forms & Mutations

**Search forms** - use `<Form method="get">`, NOT `onSubmit` with `setSearchParams`:

```tsx
// ✅ Correct
<Form method="get">
  <input name="q" />
</Form>

// ❌ Wrong - don't manually handle search params
<form onSubmit={(e) => { e.preventDefault(); setSearchParams(...) }}>
```

**Inline mutations** - use `useFetcher`, NOT `<Form>` (which causes page navigation):

```tsx
const fetcher = useFetcher();
const optimistic = fetcher.formData?.get("favorite") === "true" ?? isFavorite;

<fetcher.Form method="post" action={`/favorites/${id}`}>
  <button>{optimistic ? "★" : "☆"}</button>
</fetcher.Form>;
```

See `references/actions.md` for complete patterns.

### Layouts

**Global UI belongs in `root.tsx`** - don't create separate layout files for nav/footer:

```tsx
// app/root.tsx - add navigation, footer, providers here
export default function App() {
  return (
    <div>
      <nav>...</nav>
      <Outlet />
      <footer>...</footer>
    </div>
  );
}
```

**Use nested routes** for section-specific layouts. See `references/routing.md`.

### Route Module Exports

**`meta` uses `loaderData`**, not deprecated `data`:

```tsx
// ✅ Correct
export function meta({ loaderData }: Route.MetaArgs) { ... }

// ❌ Wrong - `data` is deprecated
export function meta({ data }: Route.MetaArgs) { ... }
```

See `references/route-modules.md` for all exports.

## Further Documentation

If anything related to React Router is not covered in these references, you can search the official documentation:

https://reactrouter.com/docs
