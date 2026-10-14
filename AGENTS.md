---
track: booqs
label: repo:nextjs-app
---

# Booqs — nextjs-app

The Next.js web app and GraphQL backend behind Booqs, and the API server the SwiftUI client talks to. One repo of the multi-repo Booqs track. Everything about Booqs that is useful without this checkout lives in xaxis (workspace `default`): `booqs/decisions` holds every decision in force, this repo's included, `booqs/strategy` the product strategy, and `booqs/conventions` is read before cross-repo work; `get-project` on `booqs` gives the track and lists the rest.

## Documentation layout

- `AGENTS.md` — this file: layout, map and conventions.
- Decisions: xaxis `booqs/decisions`, the areas Content model, Styles, Annotations, Auth and Deployment. A new decision lands there together with the code.
- `docs/` — when present, per-effort designs and plans, records and standing notes.
- `archive/` — finished efforts, one file or directory per effort.
- `SCRATCHPAD.md` — when present, Anton's untracked drafting space. Never open it without an explicit ask.

## Map

- "Booq" is the product's word for a book. A `BooqId` is `{library}-{id}`; the libraries (`backend/library.ts`) are `pg` (Project Gutenberg), `uu` (user uploads) and `lo` (local EPUBs from `public/epubs`, uncached, for development).
- Directories: `core/` (IR and domain types, tree and path utilities), `parser/` (EPUB → IR), `viewer/` (IR → React, selection, scroll), `common/`, `backend/` (database, cache, storage, libraries, auth), `graphql/` (schema and resolvers), `data/` (the server-side indirection layer frontend code calls), `app/api/` (route handlers), `application/` (client state, hooks and providers), `components/`, `reader/` (the reading interface), `app/` (App Router pages), `cli/` (`npm run cli`), `tests/`.
- Entry points: `proxy.ts` (token rotation, protected-route redirects, crawler 403s), `parser/process.ts` (the EPUB processing pipeline), `viewer/render.ts` (rendering), `core/model.ts` (the IR), `graphql/schema.graphql` (the API every client uses, served by graphql-yoga at `/api/graphql`).
- Storage: Postgres on Neon (`backend/schema.sql`), Redis on Upstash (a key-value cache, refresh tokens, reading history, sign-in secrets and passkey challenges), S3 (`backend/blob.ts`: Gutenberg and uploaded EPUBs, images, the parsed-book cache). Auth is email magic links plus WebAuthn passkeys; the token model is `booqs/decisions` → `token-model`.

### Layers

- Every directory except `@types`, `tests`, `public`, `coverage`, `certificates`, `archive` and dot-directories is a layer. A layer may access layers below it, never above.
- Order, bottom to top: `core` < `parser` < `viewer` < `common` < `backend` < `graphql` < `data` < `app/api` < `application` < `components` < `reader` < `app` < `cli`.
- `app/api` is its own layer between `data` and `application`, although it lives inside `app/`: Next.js requires route handlers there.
- Layers above `data` (except `cli`) never access `parser`, `backend` or `graphql` directly; they go through `data`. The one exception is `app/api/graphql/route.ts`, which mounts the yoga server. A layer violation is a significant architectural mistake.

## Commands

- `npm run dev` (`npm run https` for HTTPS, needs mkcert), `npm run build` (ESLint with zero warnings, then `next build`), `npm run start`, `npm run lint`, `npm run test` (Jest; `test:watch`, `test:coverage`), `npm run analyze` (bundle analyzer), `npm run cli`.
- The build pins `NODE_ENV=production`; a red build that fails prerender with a null `useContext` means the pin was bypassed (`booqs/decisions` → `build-node-env`).
- Run `npm run build` after completing a change and fix any errors before presenting it; work is not done without a passing build.

## Code conventions

- Prefer `function name(...) { ... }` to `const name = (...) => { ... }`. Private (non-exported) functions go at the bottom of the file, after all exports.
- Prefer pure functions. In `core/`, `parser/`, `viewer/` and `common/`, an impure function carries a short `// Impure: <reason>` comment above its declaration; impurity in `backend/`, `data/`, `application/` and `app/` needs none. The parser's diagnostic accumulator (`diags`, passed through and mutated) is deliberate and needs no per-function note.
- Exception: `parser/process.ts` and `parser/styles.ts` mutate documents in place. A parsed book's tree dominates memory, Vercel's serverless memory is better spent on caches, and the pipeline is linear, with no shared references to intermediate results.
- User-facing names (GraphQL fields and types, routes, UI labels) say "booq", never "book": `booqsWithNotes`, `/booq/[id]`. Internal and backend names may say "book" where it reads better.
- Nullability: `data/` exposes `undefined` for absent values, never `null`, converting with `?? undefined`. Backend `Db*` types may use `null` to match SQL; the conversion happens in the data layer's `*FromDb*` converters.
- Naming: `isLoading`, not `loading`. No `Action` suffix on data-layer server actions. Backend row types are `Db`-prefixed with snake_case fields matching the columns (`DbUser`, `DbNote`); data-layer types are camelCase. Converters are named `fooFromDbFoo` (`accountDataFromDbUser`).
- Types: inline prop and parameter types unless the type is referenced elsewhere. In a union whose members share some fields but not others, members lacking a discriminating field declare it as `field?: undefined` (as `BooqNode` does), so access needs no assertion.
- Every `as Type` cast has a comment on the line above saying why it is necessary and safe. Prefer type guards, narrowing or restructuring; a recurring cast is a sign the types want redesigning.
- All custom `data-*` attributes are defined in `core/attributes.ts`; check there before adding one.
- Mutually exclusive component state goes in one object with a discriminator (`{ state: 'loading' } | { state: 'error', error }`). Exception: SWR's own `isLoading`/`error` may be exposed directly.
- CSS and Tailwind use only the colours defined in `app/globals.css` (`--color-primary` is usable as `text-primary`); add a theme colour there if a new one is needed.
- Path alias `@/*` maps to the repo root. Tests live in `tests/`, mirroring the source tree, named `<Module>.test.ts(x)`. Add tests only when asked.

## Review posture, commits and push

- **Review posture: owned, at the commit.** Anton reviews each commit: he makes it, or approves it before an agent makes it (this repo's exception to the fleet default, kept for sentimental reasons). Without that approval, agents propose a commit message instead of committing.
- **Agents push on their own.** A push to `master` builds a staged production deployment on Vercel that serves no domain (*Auto-assign Custom Production Domains* is off), so a push ships nothing (`booqs/decisions` → `push-and-deploy`).
- **The deploy is the promote** of the staged build: the dashboard's Promote, or `vercel promote <url>`. It is Anton's call, or an agent's when he asks. Rollback is the same move, onto an older deployment.
- **A change that carries a schema change stays Anton's to push.** The repo has no migration tooling: the Postgres schema is `backend/schema.sql`, applied to Neon by hand, in step with the deploy. A change to it, or any change that needs the database changed before its code runs, waits for Anton.

## Hazards

- The Vercel firewall rule that lets native clients and CloudFront through bot protection lives in the dashboard, not here: a new API path those clients need must be added to it (`booqs/decisions` → `native-api-access`).
- `proxy.ts` 403s crawler user agents, link-preview fetchers included (`booqs/decisions` → `crawler-blocking`).
- The parser stores EPUB content as-is; renaming tags or camelCasing attributes belongs in `viewer/` at render time, because `BooqPath`s and annotations anchor to the stored tree (`booqs/decisions` → `ir-fidelity`). Content documents parse with `htmlparser2`; `fast-xml-parser` changes tree shapes (`content-parser`).
- Highlights are DOM spans, not the CSS Custom Highlight API, until WebKit can hit-test highlights (`booqs/decisions` → `highlight-rendering`).
- The repo is public, and a push publishes even though it no longer deploys: no secrets, tokens or private values in code, docs or commit messages.
