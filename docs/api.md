# API

This file summarizes how the dashboard app exposes and consumes server-side GitHub APIs.

## Overview

- The UI never calls GitHub directly. Client code calls server functions (implemented with `createServerFn`) which run in the server runtime (Cloudflare Worker in production, local dev server in `pnpm dev`).
- Server functions implement caching, conditional requests, installation-token reuse, and Octokit usage. They are defined primarily in `apps/dashboard/src/lib/github.functions.ts`.

## Where to find implementations

- Server functions: `apps/dashboard/src/lib/github.functions.ts`
- Cache and split-cache logic: `apps/dashboard/src/lib/github-cache.ts`
- Octokit client creation and installation token cache: `apps/dashboard/src/lib/github.server.ts`
- Revalidation signal and namespace helpers: `apps/dashboard/src/lib/github-revalidation.ts`
- Client query bindings: `apps/dashboard/src/lib/github.query.ts` (maps UI queries to server functions)

## How the client calls the API

- UI uses TanStack Query + helper functions in `github.query.ts`. Each query key calls a server function (e.g. `getIssuesFromRepo`) through the query's `queryFn`.
- Example (conceptual):
  - `github.query` defines `githubIssuesFromRepoQueryOptions(...)` which uses `getIssuesFromRepo` under the hood.
  - The client calls `useQuery({...githubIssuesFromRepoQueryOptions(scope, {owner, repo})})`.

- Server functions are defined with `createServerFn({ method: 'GET' })` and perform the Octokit REST/GraphQL calls and return JSON to the client.

## REST vs GraphQL

- The codebase mixes both:
  - GraphQL: used for composed, nested reads (pull/issue page, cross-references, repo overview, discussions) to reduce round-trips and fetch rich shapes.
  - REST: used for stable, paginated, or endpoint-specific operations (lists, comments, commits, some app/install APIs).
- See `executeGitHubGraphQL` usages for GraphQL and `context.octokit.rest.*` for REST calls in `github.functions.ts`.

## Cache behaviour

- Server functions wrap GitHub calls with `getOrRevalidateGitHubResource` (in `github-cache.ts`) which implements the split-cache flow:
  - Resolve namespace versions from D1.
  - Build a KV storage key and try KV for the payload.
  - Fall back to legacy D1 entry on miss.
  - Use conditional requests to GitHub when the entry is stale.
- Token reuse and installation token caching are handled in `github.server.ts`.

## Local development notes

- Run `pnpm dev` at the repo root to start the local dev server. The server functions run locally so client queries call your local server which then performs Octokit requests.
- Apply D1 migrations locally with `pnpm --filter @diffkit/dashboard migrate` (requires `wrangler`).
- `GITHUB_CACHE_KV` is optional for local dev; when missing the code falls back to legacy D1 cache behavior.

## Tracing an endpoint (example)

1. Client calls `useQuery(...githubIssuesFromRepoQueryOptions(scope, input))`.
2. That query option calls `getIssuesFromRepo` (server fn in `github.functions.ts`).
3. `getIssuesFromRepo` delegates to `getCachedGitHubRequest` which calls `getOrRevalidateGitHubResource`.
4. `getOrRevalidateGitHubResource` checks KV (if configured), D1, and either returns cached payload or calls the provided `fetcher`.
5. The `fetcher` uses Octokit (`context.octokit.rest.issues.listForRepo` or GraphQL variant) to fetch GitHub and returns data which is persisted via the split-cache writes.

---

For more details, open the code files listed above and search for `createServerFn`, `getOrRevalidateGitHubResource`, and `executeGitHubGraphQL`.
