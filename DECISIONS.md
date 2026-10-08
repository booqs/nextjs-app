# Decisions

The decisions scoped to nextjs-app. The ones that span Booqs repos are in `../hub/DECISIONS.md`.

## Content model

### ir-fidelity

**Parsed EPUB content is stored faithfully — no tag renaming, attribute camelCasing or element mapping — and the React-specific transforms (attribute normalization, element mapping) happen at render time in `viewer/`.** 2026-04-27. Several renderers consume the IR (the web `viewer/`, and the native app's WKWebView and Quire paths), and stable tree shapes are what `BooqPath` addresses and annotations anchor to. Server-side processing (`parser/process.ts`, `backend/parse.ts`) adds only what every renderer needs: scoped IDs and in-book hrefs, `data-booqs-*` markers, image URLs and dimensions, processed CSS. *Rejected:* the pre-migration IR that renamed `<body>`, stripped `<html>` and camelCased attributes at parse time. *See:* `archive/model-migration/ir-design-migration.md`, `archive/model-migration/ir-design.md`.

### content-parser

**Content documents are parsed with `htmlparser2`, not `fast-xml-parser`.** 2026-05-05. `htmlparser2` preserves comments, whitespace text nodes and processing instructions, so tree shapes (and therefore paths) stay stable; a switch was tried and reverted over tree-shape parity. `fast-xml-parser` stays in `booqs-epub` for metadata and manifest extraction, where shape does not matter. *Rejected:* `fast-xml-parser` for content. *See:* `archive/model-migration/phase-3-handoff.md`.

## Styles

### epub-css-containment

**EPUB CSS is contained with native `@scope`, one scope per spine document (`[data-booqs-doc="N"]`), applied at render time; the server only rewrites root selectors and strips global colours.** 2026-04-29. Text selection across boundaries is a hard requirement, which rules out iframes and Shadow DOM; `BooqStyles` stores each stylesheet once, shared by every document that links it, so the per-document scope cannot be baked in server-side. *Rejected:* iframe per fragment, Shadow DOM, `postcss-prefix-selector` rewriting, server-side `@scope`. *See:* `archive/model-migration/css-handling.md`.

### epub-color-stripping

**`color`, `background` and `background-color` declarations are stripped only from rules with a global selector (`*`, `html`, `body`, `:root`, `:scope`, qualified and descendant forms included; one global selector in the list strips the whole rule's colours); inline `style` attributes are not sanitized.** 2026-04-29. Global colours are what fight the app theme and dark mode; colours on specific selectors and inline styles are usually deliberate author styling, and inline styles are rare. *Rejected:* stripping theme properties from inline styles. *See:* `archive/model-migration/css-handling.md`, `booqs/epub-css-sanitization-list`.

## Annotations

### locator-model

**Annotations anchor with a `BooqLocator` (paths plus up to 30 chars of `prefix`/`suffix` context around the selected `text`) that the client builds at selection time and the server stores as-is in flat columns; plain `BooqPath` stays for same-render uses (history, TOC, links, bookmarks).** 2026-04-30. The context exists so annotations can heal when a book's tree changes; the client is the authority on what was selected, server-side reconstruction stores wrong context when trees diverge, and optimistic/offline native clients need the full locator without a round trip; nothing queries by locator fields, so no JSONB. *Rejected:* EPUB CFI, the Web Annotation model as the internal format, one unified path type, server-side context construction, a JSONB locator. *See:* `archive/model-migration/booqs-locator-design.md`, `booqs/annotation-healing`, `booqs/bookmark-locator-migration`.

### highlight-rendering

**Highlights render by wrapping text in DOM spans, not with the CSS Custom Highlight API.** 2026-04-22. Spans give click targeting through `event.target.dataset`, touch slack and accessibility for free; the API has no hit-testing on WebKit until `highlightsFromPoint` ships, and the fallbacks erase its simplicity. Revisit when WebKit ships it. *Rejected:* CSS Custom Highlight API. *See:* `archive/model-migration/css-custom-highlights.md`, `booqs/css-highlight-api-annotations`.

## Auth

### token-model

**Auth is a 15-minute JWT access token plus a 30-day opaque, single-use refresh token stored in Redis as a SHA-256 hash (hashed since 2026-05-25); web clients rotate implicitly in `proxy.ts`, native clients send `X-Access-Token` and call `refreshTokens` explicitly.** 2026-04-15. Short access tokens bound a leak's window, which the old non-expiring JWT did not; rotation sits in the proxy because Server Components cannot write cookies, so fresh tokens must be set before the request reaches a page. *Rejected:* a single long-lived JWT; long-lived bearer tokens for native clients. *See:* `archive/refresh-tokens-tasks.md`, `booqs/token-rotation-shortcomings`.

## Deployment

### native-api-access

**The Vercel firewall rule `native-app-access` bypasses bot protection for `/api/graphql`, `/api/images`, `/api/upload` and the AASA path, with no secret; it lives in the dashboard, not in the repo (`vercel.json` cannot express a bypass).** 2026-07-29. Bot protection is on deliberately, and neither the native app nor CloudFront's image fallback can pass a browser challenge; the open paths concede only scripted reads of public-domain content, upload stays auth-gated, and platform DDoS mitigation still applies. On scripted-looking GraphQL volume, escalate to option A: tighten the same rule on the dormant `x-booqs-client` header and add a rate-limit rule above it (a bypass skips every later rule); the Hobby plan allows three custom rules, and this one plus the disabled `Deny all access` kill switch leave one for A's rate limit. *Rejected:* a secret header now (A, held as the escalation), Vercel's automation-bypass secret, a separate API project (deferred until Vercel's protection model changes), App Attest alone, user-agent or JA4 keys. *See:* `archive/native-api-access/`.

### crawler-blocking

**Crawlers are blocked at three layers: `app/robots.txt` disallows everything, `proxy.ts` returns 403 to crawler user agents (including `facebookexternalhit`, so no link previews), and a Vercel AI-bots ruleset denies them.** 2026-05-25. Crawlers were persistent; layers were added until one worked, and which of them are actually needed is unverified (Anton, 2026-10-08). *See:* `booqs/crawler-blocking-layers`.

### build-node-env

**`npm run build` pins `NODE_ENV=production`.** 2026-07-28. An inherited `NODE_ENV=development` (agent harness shells export it) makes `next build` prerender development React against production chunks, failing every client page with a null `useContext`; on Vercel the pin is a no-op. *See:* commit `e830e68`.
