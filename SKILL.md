---
name: new-feature-page
description: Scaffold a new feature page in Agent Studio UI - page folder, types, react-query service, lazy route with pageLayout handle, and optional sidebar link. Use when asked to add a new page, screen, route, or section to the studio UI (e.g. "add an X page", "create a new route for Y").
---

# New feature page

Adds a feature page that matches the conventions already used in `src/pages/*`.
Reference implementation: `src/pages/audit-logs/` (page + types + service + components).

## Inputs to settle first

Ask only if the request leaves them genuinely ambiguous; otherwise infer:

- **slug** - kebab-case, used for both the folder (`src/pages/<slug>/`) and the route path.
- **title / subtitle** - shown by the shared page layout.
- **backend** - which service the data comes from and whether it uses the default
  `src/lib/axios` instance (`{BASE_URL}/v3`) or another base URL from `src/lib/constants.ts`.
- **sidebar** - whether the page gets a nav entry in `src/data/sidelinks.tsx`.

## Steps

### 1. Create the page folder

```
src/pages/<slug>/
  index.tsx              # default-exported page component
  types.ts               # request/response + filter types
  <slug>.service.ts      # react-query hooks wrapping axios
  components/            # only if the page needs more than one component
```

`index.tsx` must have a **default export** - the router imports it as `.default`.

### 2. Types (`types.ts`)

Declare the API response shapes and any filter object. Do not use `any` in
exported types; the codebase is strictly typed via `src/lib/types.ts`.

### 3. Service (`<slug>.service.ts`)

One exported hook per endpoint, wrapping `@tanstack/react-query` over the shared
axios instance. Follow the house shape:

```ts
import { useQuery } from "@tanstack/react-query";
import axios from "@/lib/axios";
import { ThingListResponse, ThingFilters } from "./types";

export const useThings = (
  apiKey: string,
  filters: ThingFilters = {},
  enabled = true,
) => {
  const { data, isLoading, error, refetch, isRefetching } = useQuery({
    queryKey: ["things", filters],
    queryFn: () =>
      axios.get<ThingListResponse>("/things/", {
        headers: { "x-api-key": apiKey },
        params: { ...filters, limit: filters.limit ?? 100, offset: filters.offset ?? 0 },
      }),
    select: (res) => res.data,
    retry: false,
    refetchOnWindowFocus: false,
    enabled: !!apiKey && enabled,
  });

  return { data, isLoading, error, refetch, isRefetching };
};
```

Notes:
- The api key comes from the global store (`useStore(s => s.apiKeys)[0].api_key`);
  pass it into the hook rather than reading the store inside the service.
- Never add bespoke error toasts for ordinary failures - the axios interceptor in
  `src/lib/axios.ts` already surfaces them.
- Mutations use `useMutation` + `queryClient.invalidateQueries` on the same key.

### 4. Register the route (`src/router.tsx`)

Add a lazy child under the authenticated `AppShell` branch, next to related pages:

```tsx
{
  path: "<slug>",
  handle: {
    pageLayout: {
      title: "<Title>",
      subtitle: "<One-line description>",
      showBackButton: true,
    },
  },
  lazy: async () => ({
    Component: (await import("@/pages/<slug>")).default,
  }),
},
```

- Dev-only pages go behind the `isDevEnv` guard already used in the file.
- Enterprise-only pages go behind `IS_ENTERPRISE_DEPLOYMENT`.
- Public pages belong under the `/auth` branch instead, with `routeTitle(...)`.

### 5. Sidebar link (optional)

Add an entry to `src/data/sidelinks.tsx` only when the page should be navigable.
Match the existing permission/flag gating used by neighbouring links - do not
introduce a new gating mechanism.

### 6. UI

- Use ShadcnUI primitives from `src/components/ui/`. If a needed primitive is
  missing, pull it via the **shadcn MCP** rather than hand-writing it.
- Tailwind only, and keep dark-mode classes on anything that sets a colour.
- Loading and empty states are expected: skeletons from `ui/skeleton`, not spinners.

## Verify

```bash
npx tsc --noEmit
```

Type check only - do not run a production build. Fix every error introduced by
the new files before reporting done.
