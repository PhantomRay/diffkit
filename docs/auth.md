# Auth

This document summarizes how DiffKit uses GitHub OAuth and GitHub App authentication,
how tokens are stored and reused, and how the server chooses which token to use.

## Overview

- Two GitHub auth modes are supported:
  - **OAuth App** — used for user authentication (login) and as an OAuth-token fallback when the GitHub App is not installed for a repository/owner.
  - **GitHub App** — used for installation-scoped reads/writes (e.g. issues, PRs, comments), webhooks, and optional per-user App authorization flows. This is the **preferred** token for data fetching as it typically has higher rate limits and repository-scoped access.

## OAuth App (classic)

- Purpose: authenticate DiffKit users and provide a user-level OAuth token with scopes such as \`repo\`, \`read:org\`, and \`user:email\`. This token is used as a fallback for data fetching when the GitHub App is not installed.
- Flow: Better Auth handles the OAuth login flow. See \`getAuth()\` in [apps/dashboard/src/lib/auth.server.ts](apps/dashboard/src/lib/auth.server.ts).
- Storage: OAuth account rows are stored in the \`account\` table with \`providerId = "github"\`. Helpers:
  - \`getGitHubOAuthAccountByUserId()\` — [apps/dashboard/src/lib/github-app.server.ts](apps/dashboard/src/lib/github-app.server.ts) (finds the OAuth account row).
  - \`getGitHubAccessTokenByUserId(userId)\` — returns the OAuth token used to create Octokit clients.
- Usage: \`getGitHubContext()\` in [apps/dashboard/src/lib/github.functions.ts](apps/dashboard/src/lib/github.functions.ts) creates a "base" Octokit client authenticated with this OAuth token.

## GitHub App (app + installation + optional app-user)

- Purpose: provide installation-scoped tokens for repository/organization operations, receive webhooks, and (optionally) support an "app user" OAuth flow for actions tied to the GitHub App identity.
- Credentials: app id and private key are configured in environment variables (\`GITHUB_APP_ID\`, \`GITHUB_APP_PRIVATE_KEY\`, etc.). See \`getGitHubAppAuthConfig()\` and helpers in [apps/dashboard/src/lib/github-app.server.ts](apps/dashboard/src/lib/github-app.server.ts).
- Installation tokens:
  - Minted via the App (JWT auth) and cached per \`installationId\`.
  - Cached in worker memory, and optionally persisted in \`GITHUB_CACHE_KV\`.
  - Creation and reuse logic: \`getGitHubInstallationClient()\` in [apps/dashboard/src/lib/github.server.ts](apps/dashboard/src/lib/github.server.ts) handles the fetching and caching of these tokens.
- App-user tokens (optional): supports an OAuth-like exchange for a per-user App token; stored separately in \`account\` rows with \`providerId = "github-app"\`. Helpers: \`exchangeGitHubAppUserCode()\`, \`getGitHubAppUserAccessTokenByUserId()\` in [apps/dashboard/src/lib/github-app.server.ts](apps/dashboard/src/lib/github-app.server.ts). These are primarily used for "write" operations or actions that must be attributed to the user via the App identity.

## Which token is used (selection logic)

The server uses a tiered strategy to select the most appropriate token for each operation. The logic is primarily located in [apps/dashboard/src/lib/github.functions.ts](apps/dashboard/src/lib/github.functions.ts):

1. **Base Context**: \`getGitHubContext()\` initializes a \`GitHubContext\` using the user's **OAuth token** (\`providerId = "github"\`).
2. **Owner/Repo Context**: \`getGitHubContextForOwner(owner)\` looks up if the user has access to a GitHub App **installation** for that owner via \`getGitHubAppUserInstallations()\`.
   - If an installation is found: It wraps the base context with an \`installationContext\` using an **installation token**.
   - If no installation is found: It falls back to the base **OAuth token** context.
3. **User/Write Context**: \`getGitHubUserContextForOwner(owner)\` is used specifically for operations that require user attribution (like posting comments).
   - It checks for a **"github-app"** user token (App-user OAuth flow).
   - If found AND an installation exists for the owner, it uses the App-user token.
   - Otherwise, it falls back to the standard **OAuth token**.

**Summary of Priority:**
- **Read Operations**: Installation Token > OAuth Token.
- **Write Operations**: App-user Token > OAuth Token (with fallback logic).

## Caching and reuse

- Installation tokens: cached in-memory per worker isolate and stored in KV (\`GITHUB_CACHE_KV\`) when available to reduce cross-isolate remints. See [apps/dashboard/src/lib/github.server.ts](apps/dashboard/src/lib/github.server.ts).
- OAuth tokens: stored in the \`account\` table (DB) and read when creating user Octokit clients.
- Payload cache & related metadata: split-cache (KV payloads + D1 control plane) sits on top of these tokens and is used by server functions. See [apps/dashboard/src/lib/github-cache.ts](apps/dashboard/src/lib/github-cache.ts) and [apps/dashboard/src/lib/github.functions.ts](apps/dashboard/src/lib/github.functions.ts).

## Webhooks and invalidation

- Webhooks from the App invalidate cached data and installation state. The webhook handler calls \`invalidateGitHubInstallationToken()\` and writes revalidation signals via \`markGitHubRevalidationSignals()\` and \`bumpGitHubCacheNamespaces()\`.
- Webhook route: \`apps/dashboard/src/routes/api/webhooks/github.ts\` and helpers in [apps/dashboard/src/lib/github-app.server.ts](apps/dashboard/src/lib/github-app.server.ts) and [apps/dashboard/src/lib/github-cache.ts](apps/dashboard/src/lib/github-cache.ts).

## DB model summary

- \`user\` table: DiffKit users.
- \`account\` table: external provider accounts. \`providerId\` values used:
  - \`github\` — OAuth App account row
  - \`github-app\` — App-user account row (optional)

See schema: [apps/dashboard/src/db/schema.ts](apps/dashboard/src/db/schema.ts).

## Local dev notes

- Local dev uses the same auth code paths but runs server functions in the local dev server. Create \`.dev.vars\` with the OAuth and App credentials described in the repo README.
- \`GITHUB_CACHE_KV\` is optional for local dev; installation token reuse will be limited to the worker process memory cache when KV is absent.

## Where to inspect relevant code

- OAuth & Better Auth: [apps/dashboard/src/lib/auth.server.ts](apps/dashboard/src/lib/auth.server.ts) and [apps/dashboard/src/lib/auth.functions.ts](apps/dashboard/src/lib/auth.functions.ts)
- Token selection & Octokit creation: [apps/dashboard/src/lib/auth-runtime.ts](apps/dashboard/src/lib/auth-runtime.ts), [apps/dashboard/src/lib/github.server.ts](apps/dashboard/src/lib/github.server.ts), [apps/dashboard/src/lib/github-app.server.ts](apps/dashboard/src/lib/github-app.server.ts)
- Server functions that use tokens: [apps/dashboard/src/lib/github.functions.ts](apps/dashboard/src/lib/github.functions.ts)
- Cache & invalidation: [apps/dashboard/src/lib/github-cache.ts](apps/dashboard/src/lib/github-cache.ts), [apps/dashboard/src/lib/github-revalidation.ts](apps/dashboard/src/lib/github-revalidation.ts)
- Webhooks: [apps/dashboard/src/routes/api/webhooks/github.ts](apps/dashboard/src/routes/api/webhooks/github.ts)
