# Build Sequence

Ordering principle: keep Bambuddy's test suite green at every step; each phase ends runnable.

## Phase 0 — Fork hygiene (this repo)
- [x] Fork maziggy/bambuddy → jamesso/tuani3d, `upstream` remote for future merges
- [x] Archive jamesso/tuani3d-app (read-only, tag `archive/pre-bambuddy-rebuild`)
- [ ] Rebrand pass (name, icons, sponsor/bug-report relay removal)
- [ ] CI: keep lint/test/security workflows; point at fork

## Phase 1 — Foundation strip & re-org
- [ ] Postgres-only: remove SQLite paths, `db_dialect.py` conditionals
- [ ] **Alembic baseline** replacing hand-rolled migrations in `core/database.py`
- [ ] Cut list: Spoolman, Obico, LDAP, Orca Cloud, extra git-backup providers, sponsor plumbing
- [ ] Monorepo layout: `apps/server`, `apps/web` (move frontend), `apps/store` (empty shell)
- [ ] Break up `main.py` (6.9k LOC): lifespan, callbacks, middleware into modules
- [ ] OpenAPI-generated TS client replacing hand-written `client.ts`
- Exit: existing Bambuddy features work on Postgres, suite green

## Phase 2 — Materials unification
- [ ] `material` / `material_lot` / `consumption` schema (see 01-domain-model §1)
- [ ] Migrate Bambuddy spool/catalog/filament code onto filament-kind lots
- [ ] Retarget `usage_tracker` at consumption ledger
- [ ] Purchasing + non-filament components UI (ports tuani3d-app materials concepts)
- Exit: AMS tracking works against new schema; magnets/hardware trackable

## Phase 3 — Catalog
- [ ] `product` / `product_variant` / `variant_material` / `bundle_component`
- [ ] Variant ↔ `library_file_id` ↔ print profile join
- [ ] Delete `projects`; port MakerWorld product import (merge both importers)
- Exit: sellable SKU with BOM, 3MF, price exists

## Phase 4 — Unified print_job
- [ ] Merge `print_queue` + `print_archives` → single `print_job` lifecycle
- [ ] Scheduler reads `required_filaments` from variant BOM when job created from variant
- [ ] Ad-hoc (order-less) flow end-to-end on real printers
- Exit: production page = queue+timeline+archive on one model

## Phase 5 — Commerce module (tests as built; property-based on money math)
- [ ] `party` (customers/suppliers), `sales_order` / `order_item` / `order_log`
- [ ] Payments, shipments, returns/RMA
- [ ] Currency + FX locking, invoices (server-side PDF), simple discounts
- Exit: manual order → invoice → payment works

## Phase 6 — Order → production bridge (the point of the merge)
- [ ] Order item → stock check → shortfall auto-creates print_jobs from BOM
- [ ] Job completion → measured consumption → actual COGS on order
- [ ] Live print progress on order page via existing WebSocket
- Exit: buy-to-printer flow, estimated-vs-actual margin per SKU report

## Phase 7 — Ops frontend shell
- [ ] New nav (~9 items): Dashboard · Orders · Production · Printers · Products · Materials · Customers · Reports · Settings
- [ ] Route-level code splitting; port Bambuddy printer/production pages; new commerce pages
- [ ] shadcn/ui adoption + Bambuddy theme tokens

## Phase 8 — Storefront (`apps/store`)
- [ ] Next.js: marketing MDX pages, catalog SSG/ISR from `/api/public/*`
- [ ] `storefront_collection`, `published`/`slug` on products
- [ ] Guest checkout → `POST /api/public/orders` → Stripe Checkout hosted page
- Exit: public purchase lands in ops pipeline and can start a printer

## Phase 9 — Cutover
- [ ] One-shot import script from both old DBs (no back-compat maintained)
- [ ] Reports: sales, COGS actual-vs-est, energy, printer utilization
- [ ] Deployment: server+Postgres+MinIO on printer-LAN box; store on Vercel/edge; Tailscale between

## Standing rules
- Alembic for every schema change; no `_safe_execute`-style raw DDL
- All new endpoints behind existing RBAC; new permissions follow `resource:action_own/_all`
- Money math: Decimal only, property-based tests
- Upstream merges: `git fetch upstream && git merge upstream/main` — expect conflicts to concentrate in `main.py`/routes until Phase 1 re-org; prefer cherry-picking machine-layer fixes after that
