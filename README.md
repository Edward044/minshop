# minshop

[![Verify](https://github.com/ddyy/minshop/actions/workflows/verify.yml/badge.svg)](https://github.com/ddyy/minshop/actions/workflows/verify.yml)
[![npm](https://img.shields.io/npm/v/create-minshop?label=create-minshop)](https://www.npmjs.com/package/create-minshop)
[![npm downloads](https://img.shields.io/npm/dm/create-minshop)](https://www.npmjs.com/package/create-minshop)
[![License: MIT](https://img.shields.io/github/license/ddyy/minshop)](LICENSE)

An open-source store an agent can pay for itself. Agents read the catalog and pay a Lightning invoice with no human in the loop — over plain JSON or over MCP; people check out the normal way with Stripe. Merchants can run the store over MCP too.

It's a small, server-rendered store for Cloudflare Workers (D1 + R2) with a full admin, multiple payment rails, and an optional MCP server, within the free tier to start. No brand is baked in: store name and time zone are set during onboarding, so it clones cleanly as a template.

**[Live demo](https://demo.minshop.dev/)** (agent API at /api/products, /api/checkout). Uses test payments; do not enter real personal information.

![minshop storefront and checkout](docs/media/minshop.gif)

<table>
  <tr>
    <td><a href="docs/media/storefront-desktop.png"><img src="docs/media/storefront-desktop.png" alt="Storefront catalog"></a></td>
    <td><a href="docs/media/cart-payment-rails.png"><img src="docs/media/cart-payment-rails.png" alt="Cart with card, Lightning, and demo payment rails"></a></td>
  </tr>
  <tr>
    <td align="center">Storefront</td>
    <td align="center">Cart with three payment rails</td>
  </tr>
  <tr>
    <td><a href="docs/media/lightning-invoice.png"><img src="docs/media/lightning-invoice.png" alt="Lightning invoice checkout"></a></td>
    <td><a href="docs/media/admin-dashboard.png"><img src="docs/media/admin-dashboard.png" alt="Admin dashboard"></a></td>
  </tr>
  <tr>
    <td align="center">Lightning checkout</td>
    <td align="center">Admin</td>
  </tr>
</table>

It's intentionally lightweight and cheap to start. The default deployment uses Cloudflare Workers, D1, and R2 and can fit within their free-plan allowances; payment providers and optional services have their own pricing.

## Scaffold a store

> **Requires Node ≥ 22.12 and Git.**

```sh
npm create minshop@latest my-store
cd my-store
npm run provision:local -- --seed
npm run dev
```

The initializer creates a clean repository and installs both the storefront and
MCP dependencies. Use `--no-install` to only scaffold the files, or
`--ref <branch-or-tag>` to start from a specific revision.

## Features

- **Storefront** — server-rendered product list + product detail, near-zero client JS
- **Cart** — HttpOnly cookie-based, with a progressively enhanced drawer plus no-JS form fallbacks
- **Search** — FTS5 full-text (bm25, prefix match, typo-correct), no JS — or optional **semantic search** via Workers AI + Vectorize (see [Search](#search))
- **Categories** — nested (arbitrary-depth tree), many-to-many with products; storefront category pages with breadcrumbs + sub-category drill-down (recursive descendant queries)
- **Admin** — products, categories, pages, media, orders, customers, fulfillment, CSV export, and runtime store settings
- **Payments** — Stripe Checkout, self-hosted Bitcoin Lightning (phoenixd or LNbits), hosted OpenNode, or a built-in demo rail; configure enabled methods and the default in Admin (see [Payments](#payments))
- **Shipping** — merchant-managed zones, flat **or weight-banded** rates, and free-over-threshold rules, edited in Admin with no redeploy; supported checkout flows capture the address, chosen service, and shipping cost on the order
- **Discount codes** — promo-code field enabled from Admin; codes are created/managed in the Stripe Dashboard, and the applied discount is captured onto the order
- **Tax** — sales tax / VAT via Stripe Tax (off by default — **activate Stripe Tax in the Dashboard first**); computed from the customer address and captured onto the order
- **Order email** — confirmation email behind an `EmailProvider` seam with two adapters: **Resend** (HTTPS API, works on the Workers free plan) or **Cloudflare Email** (binding, paid plan); configured in Admin and a safe no-op until a provider is ready
- **Pages** — merchant-authored Markdown pages at `/pages/<slug>`, with a split-view editor, live preview, and layout presets (see [Pages](#pages))
- **Media library** — one place that owns every uploaded file; products, pages, and the logo record how each is used (see [Media library](#media-library))
- **Images** — uploaded to R2, served through the app (no public bucket required)
- **Swappable seams** — payments and storage sit behind interfaces (see [Architecture](#architecture))

## Engineering highlights

A few decisions in here are worth reading even if you never deploy a store:

- **Payments sit behind a port.** Checkout and webhook routes depend on a `PaymentProvider` interface rather than any one vendor. Stripe, Lightning (phoenixd or LNbits), OpenNode, and the demo rail are adapters, so adding a rail is one adapter plus its webhook route. Storage works the same way (`StorageProvider`, with R2 as the adapter).
- **HTML first, JS as enhancement.** Pages render on the server and ship near-zero client JavaScript. Cart, search, and checkout all work with JS disabled; the cart drawer and live search are progressive enhancements layered on plain forms.
- **Search without a search service.** Full-text search runs on SQLite FTS5 inside D1, with bm25 ranking, prefix matching, and typo correction. Semantic search (Workers AI + Vectorize) is an optional flag, not a dependency.
- **Cache invalidation follows what shoppers can see.** Cached pages carry `shell`, `catalog`, and per-product tags. Admin writes purge only the affected tags, and stock changes purge a product only when it crosses In stock / Low stock / Sold out, not on every checkout reservation. The page shell stays shared even for cookied shoppers because the one personal detail, the cart count, loads from a small private fragment.
- **Provider keys are write-only.** Stripe, Lightning, and email credentials are entered in Admin, encrypted under a KEK before they reach D1, and never displayed again. The Worker itself needs exactly two secrets.
- **Refactors are gated by equivalence tests.** The Vitest suite (70 test files) renders `.astro` components through AstroContainer and includes contract tests plus rendering baselines, so template extractions have to prove they didn't change the output a customer sees.
- **The same store serves people and agents.** Catalog and checkout are exposed as HTML, as JSON (`/api/products`, `/api/checkout`), and over MCP — all three reach the same checkout, so an agent can browse the catalog and settle a Lightning invoice with no human in the loop.

## Stack

| Piece | Choice |
|---|---|
| Framework | [Astro](https://astro.build) (SSR) on Cloudflare Workers via `@astrojs/cloudflare` |
| Styling | Tailwind CSS v4 (`@tailwindcss/vite`) |
| Data | Cloudflare D1 (SQLite) — products, orders |
| Images | Cloudflare R2 — zero egress |
| Payments | Stripe Checkout, Bitcoin Lightning (phoenixd / LNbits), OpenNode, or demo |

## Quick start (local)

> **Requires Node ≥ 22.12** — use Node 22, the tested/supported release line.

```sh
npm install

# One shot: build, migrate + seed the local DB, and generate the two local
# secrets (SECRETS_KEK + AUTH_SECRET) into .dev.vars
npm run provision:local -- --seed

npm run dev                        # http://localhost:4321
```

Storefront is at `/`, admin at `/admin` — first visit lands on the **setup wizard**. There are no provider secrets to fill in by hand: payment/email/Turnstile keys are pasted in **Admin → Settings** and stored encrypted in D1 (see [Settings](#settings)). Granular scripts also exist (`db:migrate`, `db:seed`), `npm run preview` runs the built worker in prod-mode (admin gate active), and `npm run destroy:local` resets the local store.

### Tests

```sh
npm test          # vitest run (unit tests for pure logic)
npm run test:watch
npm run test:d1   # fresh migrations + seed + built Worker against isolated D1
```

Covers the pure functions — `slugify`, the FTS search sanitizer + edit-distance, `parseProductForm`, image validation, cart counting, reservation target aggregation, the order-number scheme, the Access-JWT verifier, pagination clamping, and the whitelisted `orderByClause` sort builders (the SQL-injection boundary for sortable tables). `npm run test:d1` adds clean-room D1 gates for reservation concurrency/release/settlement/legacy compatibility, then boots the production Worker against an isolated database and runs a demo checkout through paid-order settlement and confirmation. `npm run verify` runs both suites, full Astro diagnostics, the production build, and the MCP typecheck/deployment dry run.

### Testing payments locally

Paste your `sk_test_…` key in **Admin → Settings → Payments → Card (Stripe)**, then forward Stripe events to your dev server with the [Stripe CLI](https://docs.stripe.com/stripe-cli):

```sh
stripe listen --forward-to localhost:4321/api/webhook
# paste the printed whsec_… into Settings → Payments → Card (Stripe) → webhook signing secret
stripe trigger checkout.session.completed
```

The order shows up in `/admin` (and in D1: `npx wrangler d1 execute minshop-db --local --command "SELECT * FROM orders"`).

## Deploy

### One-click (Deploy to Cloudflare)

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/ddyy/minshop)

Forks the repo, provisions D1 plus separate R2 buckets for public images (`minshop-images`) and private deliverables (`minshop-files`), applies migrations, and deploys the free-plan default. The files bucket must stay private: never enable r2.dev or attach a custom domain. Then finish onboarding at `/admin/setup` (below). Cloudflare Images, semantic search, and Cloudflare Email are opt-in; their bindings are commented in `wrangler.jsonc`.

**Fields the deploy form shows:**

- **`SECRETS_KEK`, `AUTH_SECRET`** (masked, required) — the only two values you must supply. Paste a fresh random value into *each* (they must differ): run `openssl rand -base64 32` twice. The button can't generate them, and masked-field hints are hidden, so a leading label field named **`USE_A_LONG_RANDOM_STRING_FOR_SECRETS_KEK_AND_AUTH_SECRET`** carries the instruction above them; its value is unused.
- **Location hint** — Cloudflare's own field for where to place the D1 database; pick the region nearest your shoppers.
- **No store name / time zone / search fields** — those are runtime settings you configure in the setup wizard and **Admin → Settings** (stored in D1), not at deploy time. Defaults until then: `My Shop` / `UTC` / keyword (FTS) search.

### CLI (one shot)

**One shot** — provisions a fresh, fully-independent instance (its own D1, R2, Vectorize index, Worker) and sets both Worker secrets:

```sh
npm create minshop@latest my-store
cd my-store
npx wrangler login
npm run provision:cf my-store      # scripts/provision-cf.sh <slug>
```

Or **manually**:

```sh
npx wrangler login

# Provision real resources. The committed wrangler.jsonc declares D1/R2 by NAME
# with no ids (so a one-click / Workers Builds deploy auto-provisions them); for a
# manual deploy, add the printed database_id to the "DB" entry — or let
# `wrangler deploy` create it.
npx wrangler d1 create minshop-db
npx wrangler r2 bucket create minshop-images
npx wrangler r2 bucket create minshop-files   # FILES binding; keep private
npm run db:migrate:remote          # applies migrations/ to the production DB

# The only two REQUIRED Worker secrets. The optional cache-purge secret is
# covered under Caching below. Everything else (Stripe, OpenNode, Lightning,
# Resend, Turnstile keys + their config) is entered in Admin → Settings and
# stored encrypted in D1 under SECRETS_KEK.
openssl rand -base64 32 | npx wrangler secret put AUTH_SECRET   # signs sessions
openssl rand -base64 32 | npx wrangler secret put SECRETS_KEK   # encrypts the key vault

npm run deploy                     # migrate, build, deploy, then purge shared cache if enabled
```

Then open the site — it funnels to the **setup wizard**: set the admin password (required to finish; until then `/admin/setup` is open, so do it right away or front `/admin` with Access), then paste your payment keys in **Settings → Payments**. Finally point a **production Stripe webhook** at `https://<your-host>/api/webhook/stripe` (events: `checkout.session.completed`, `checkout.session.async_payment_succeeded`, `checkout.session.async_payment_failed`, `checkout.session.expired`, and `charge.refunded`; async success fulfils delayed methods, failure/expiry releases held inventory, and `charge.refunded` keeps refunds made in the Stripe Dashboard — including partial ones — in sync with your order totals) and paste its `whsec_…` signing secret in the same card. Tear an instance down with `npm run destroy:cf <slug>`; reset its data in place with `npm run reset:remote`.

## Admin auth

Both `/admin` **and** `/api/admin/*` are gated by [`src/middleware.ts`](src/middleware.ts). You authenticate one of two ways:

- **Setup-wizard password (built-in).** On a fresh store the setup wizard at **`/admin/setup`** is reachable so you can create an admin password; it's stored as a PBKDF2 hash in D1 (there is **no `ADMIN_PASSWORD` env var**). The moment you save it the gate activates: `/admin` + `/api/admin` require sign-in at **`/admin/login`** (a form issuing an HMAC-signed, HttpOnly session cookie; `/api/admin` returns 401 instead of redirecting), and saving logs you in immediately. **First-run is open** — until a password exists, `/admin/setup` is publicly reachable, so set it promptly on a public deploy (or front `/admin` with Access, below).
  - **Optional Turnstile** on the login form: enable it in **Admin → Settings → Bot protection** (toggle + sitekey, with the secret stored in the encrypted vault) to add Cloudflare's bot challenge before the password is checked. The same toggle also gates the customer account sign-in. Cloudflare's documented always-pass test keys work for local trials.
- **Cloudflare Access (recommended for production).** Edge SSO/MFA, no app passwords, and it protects first-run setup too. In the Zero Trust dashboard add a **self-hosted application** covering **both** paths `your-host/admin` *and* `your-host/api/admin` (the API is a separate path — gating only `/admin` leaves the mutation endpoints open), add an identity provider (One-time PIN is enough), and an Allow policy for your email. Set both `CF_ACCESS_TEAM_DOMAIN` and `CF_ACCESS_AUD` in `wrangler.jsonc` vars; they are **required for Access mode** so the middleware can verify every JWT signature (zero-dependency, Web Crypto — see [`src/features/auth/access.ts`](src/features/auth/access.ts)). With either variable present, requests fail closed unless both are set and a valid assertion is supplied—including direct `*.workers.dev` requests with no header. (Use Access *or* the wizard password; if a password is set, password login takes precedence.)

**Forgot the admin password?** The hash lives in your D1 database, so the store owner can always reset it — clearing it drops the store back into the open setup wizard:

```bash
npm run admin:reset:remote   # deployed store  (or: npm run admin:reset for local)
```

Then reload `/admin/setup` and set a new one. (Same effect as running `DELETE FROM settings WHERE key='admin_password_hash';` in the Cloudflare dashboard → D1 console.) It requires Cloudflare account access, so it isn't a public reset — and if you run Cloudflare Access you're never locked out in the first place.

Local dev (`astro dev`) bypasses the gate so you're never locked out. Don't make `/admin` a "secret" path — that's security-through-obscurity; the gate is what protects it.

The Worker also applies native edge rate limits to anonymous login POSTs (10/minute per store, route, and connecting client), checkout/invoice POSTs (20/minute), and cache-missing public searches (60/minute). Search input is normalized and capped at 200 characters before FTS or Workers AI sees it; instances that enable Workers Caching keep public catalog/search responses for up to 10 minutes. Webhooks and authenticated admin APIs are deliberately excluded; provider signatures and the admin gate protect those paths without disrupting legitimate retries or bulk administration. The limits are declared as `AUTH_RATE_LIMITER`, `CHECKOUT_RATE_LIMITER`, and `SEARCH_RATE_LIMITER` bindings in the provisioning template; existing manually maintained Wrangler configs need the same `ratelimits` block.

## Caching

Workers Caching uses its native entrypoint + path/query key, so tracking parameters
create separate entries; keep internal links canonical and measure fragmentation
before adding a gateway solely for key normalization. The shell remains shared
even when a shopper has cookies. Its only customer-specific header value, the
cart count, loads from a tiny private fragment; the complete cart stays on
private, `no-store` routes and is fetched only when opened.

Caching is deployment-controlled and disabled in the generic config. It is not
an Admin setting because cache lookup occurs before the Worker runs. Enable it
only for an instance with one public hostname: move the instance to a custom
domain, set `CANONICAL_ORIGIN` to its exact HTTPS origin, disable the
`workers.dev` ingress, and enable Workers Caching in the same rendered
`wrangler.jsonc`:

```jsonc
{
  "workers_dev": false,
  "vars": {
    "CANONICAL_ORIGIN": "https://shop.example.com"
  },
  "cache": {
    "enabled": true
  }
}
```

`CANONICAL_ORIGIN` stabilizes absolute URLs in shared HTML, catalog JSON,
sitemap, robots, and `llms.txt`; internal storefront links remain
root-relative.

### Invalidation

Cacheable responses carry `shell`, `catalog`, and public
`product:prod_…` tags. Low-frequency Admin changes purge the affected tags after
their D1 write; a failed tag purge falls back to purging the entrypoint cache.
Checkout reservations and releases purge only when the customer-visible stock
state changes: products cross `In stock` / `Low stock` / `Sold out`, or a variant
crosses available / sold out. Exact stock counts appear only on private cart and
checkout pages. Stock purges target the affected `product:prod_…` tags, are
best-effort, and never fall back to purging the whole cache; the bounded
10-minute shared TTL remains the safety net for rate limits or transient purge
failures.

### Cross-version cache

By default, a deployment starts with a cold, version-keyed cache. An instance
that deploys frequently may add `"cross_version_cache": true` beside
`cache.enabled`, but only after setting the same `CACHE_PURGE_SECRET` in the
Worker and its deploy environment (an uncommitted `.dev.vars` is supported).
Use a dedicated random value so deploy access never requires distributing or
rotating `AUTH_SECRET`; the latter remains a compatibility fallback.
`npm run deploy` preflights that secret plus the single canonical hostname,
deploys, then calls a short-lived HMAC-authenticated Worker endpoint to purge
the shared cache. It retries transient purge failures. If all attempts fail,
the new Worker is already live: retry the deploy, then temporarily set
`cross_version_cache` to `false` and deploy again to recover with a cold
version-keyed cache.

Generate one random value, enter it at Wrangler's secure prompt, and put the
same value in the deploy machine's uncommitted `.dev.vars`:

```bash
openssl rand -base64 32
npx wrangler secret put CACHE_PURGE_SECRET
```

```dotenv
# .dev.vars — do not commit
CACHE_PURGE_SECRET=<the same random value>
```

Then enable the shared cache:

```jsonc
"cache": {
  "enabled": true,
  "cross_version_cache": true
}
```

### Smart Placement (