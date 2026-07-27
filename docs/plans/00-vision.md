# Tuani3D — Vision & Architecture Decision

*2026-07-27. Greenfield plan: single pane for a 3D printing business, combining Bambuddy (machine layer) with tuani3d-app (commerce layer).*

## Decision

**Fork Bambuddy as the foundation. Rebuild tuani3d-app's commerce as native modules inside it. One backend, one domain model, two frontends.**

- This repo is a fork of [maziggy/bambuddy](https://github.com/maziggy/bambuddy) (`upstream` remote) so machine-layer fixes can be merged from upstream when useful.
- The previous app is archived read-only at [jamesso/tuani3d-app](https://github.com/jamesso/tuani3d-app), tag `archive/pre-bambuddy-rebuild`. It is the reference for commerce schema/UX, not a code source.

## Why this direction

| Option | Verdict |
|---|---|
| Rebuild both from scratch | Bambuddy's MQTT protocol layer, scheduler, camera fanout, virtual printer (~100k LOC + 138k LOC tests) encode years of reverse-engineering. Rewriting = months of hardware regression for zero gain. |
| Extend tuani3d-app | Wrong base. Next.js/Supabase can't host persistent MQTT, FTPS, camera fanout, SSDP. |
| Two apps + sync layer | Single pane wants single domain model, auth, UI. Two DBs + sync = permanent tax. |
| **Fork Bambuddy, graft commerce** | Machine layer, tests, auth/RBAC, real-time all inherited free. Only build what's missing: money. |

Bambuddy has no commerce (customers/orders/invoices/COGS). tuani3d-app's printer layer was a UI-complete stub (connection credentials collected, never read). tuani3d-app's real value is ~3k lines of commerce schema thinking — cheap to rebuild cleanly, expensive to keep (0 tests, 0 API auth, 1900-LOC page monoliths).

## Target architecture

```
apps/
  server/       Python 3.13 · FastAPI · SQLAlchemy 2 async · Postgres ONLY · Alembic
    modules/
      machines/    Bambuddy verbatim: mqtt, printer_manager, camera, virtual_printer, hms, firmware
      scheduling/  Bambuddy scheduler, retargeted at unified print_job
      library/     Bambuddy files + 3MF tools
      catalog/     products, variants, BOM, materials
      inventory/   material lots (incl. spools), consumption ledger, locations
      commerce/    orders, payments, shipments, returns, invoices, discounts, parties
      finance/     currency, FX, COGS, reports
      identity/    Bambuddy auth: JWT, RBAC, MFA, OIDC, API keys
      platform/    settings, notifications, backups, websocket, i18n
      storefront/  public read-only API slice, storefront collections, checkout intake
  web/          Ops console — Vite SPA, auth'd, WebSocket-heavy (evolved from Bambuddy frontend)
  store/        Storefront + marketing — Next.js (SSG/ISR), public, guest checkout via Stripe Checkout
```

Deliberate choices:

- **Postgres only, Alembic from day one.** Replaces Bambuddy's hand-rolled `_safe_execute` migrations in `core/database.py` (its worst design flaw) and drops SQLite dual-dialect complexity.
- **No Supabase / no Makerkit.** Bambuddy's auth (150-perm RBAC, `*_own`/`*_all` split, MFA, OIDC, scoped API keys) is strictly superior, and deployment needs a real host anyway (MQTT). Storage: S3-compatible (MinIO) in the same compose.
- **OpenAPI-generated TS client** for both frontends. Kills the hand-maintained 7.5k-LOC `client.ts` drift and untyped fetches.
- **Real-time pattern everywhere:** one WebSocket, messages fan into TanStack Query cache (Bambuddy's existing pattern). Order status, payment received, print progress — same pipe.
- **UI:** shadcn/ui component kit + Bambuddy's CSS-var theme-token discipline. Route-level code splitting from day one (Bambuddy's 430KB PrintersPage is the cautionary tale).
- **Tests inherited, not written:** keep Bambuddy's 310-file suite green for machines/scheduling/auth; new commerce modules ship with tests. Money math (FX, COGS, invoice totals) gets property-based tests — non-negotiable.

## Two frontends, one backend

Ops console and storefront have opposite requirements; forcing them into one shell re-introduces complexity.

- **`web/` (ops):** auth'd SPA, WebSockets, RBAC, no SEO needs.
- **`store/`:** Next.js App Router. Marketing pages as MDX/static. Catalog SSG+ISR from `/api/public/*` (published variants only, display prices in visitor currency via FX module; never exposes cost, margin, or exact stock). Cart client-side; checkout `POST /api/public/orders` → lands directly in the ops order pipeline → auto print-job creation. Payments via Stripe Checkout hosted page. Guest checkout first; accounts later.

**The flow that justifies everything:** customer clicks buy → order created → items map to variants → stock check → shortfall auto-creates print jobs with required filaments from BOM → scheduler late-binds to any capable idle printer → completion writes measured grams/time/energy → actual COGS on the order → invoice → margin report knows the true number. Neither predecessor app could do this.

## Killer differentiator

Estimated-vs-actual margin per SKU: BOM-estimated COGS for pricing vs measured filament (AMS delta / 3MF estimate) + energy (smart plugs) + time × labor rate for truth.

## Navigation (single pane, ~9 items)

Dashboard · Orders · Production (queue+timeline+archive merged; Cam Wall as view mode) · Printers · Products (catalog, variants, BOM, MakerWorld import) · Materials (catalog + lots/spools + purchasing + shopping list) · Customers · Reports (sales, COGS actual-vs-est, energy, utilization) · Settings

## Cut list (future-proof ≠ carry everything)

From Bambuddy: Spoolman sync (we ARE the spool manager), Obico, LDAP, SpoolBuddy kiosk routes (keep device API), Orca Cloud, 4× git-backup providers (keep one), sponsor/bug-report plumbing, SQLite support.

From tuani3d-app: Supabase entirely, DB-driven UI translations (file-based i18n; per-entity translation only for invoice output as JSON column), Firecrawl, `discount_target` targeting engine (simple % / fixed at order+item level until needed), bank accounts as module (fold into business settings), `collection` as core entity (returns only as storefront presentation).

Result target: **~35 tables** replacing 63 (Bambuddy) + 52 (tuani3d-app).
