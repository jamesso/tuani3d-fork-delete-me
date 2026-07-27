# Domain Model — Unification & Simplification

The core of the merge: collapse both apps' concept sprawl into ~35 tables without losing robustness.

## 1. Materials (worst overlap — 11+ tables → 4)

Materials = **all BOM inputs**, not just filament: plastic, magnets, keyrings, RFID tags, hardware, packaging.

| Concept | Definition |
|---|---|
| **`material`** | Universal catalog entry. Brand, SKU, supplier, buy-cost, sell-markup, `unit` (g, pcs, m, ml), `kind = filament \| component \| packaging \| consumable`. Filament-specific attributes (polymer type, color hex, diameter, temps, drying) in `filament_props` (JSON or child table), present only when `kind = filament`. Absorbs old `material`/`material_type`/`material_subtype` (tuani3d-app) and `spool_catalog`/`filaments`/`color_catalog` (Bambuddy). |
| **`material_lot`** | Physical purchased instance: qty remaining, location, cost-at-purchase, received date. For filament, lot = **spool** with extra nullable columns (AMS slot, RFID tag, remain %, k-profile). Spool is a *role* of lot, not a separate entity. Bambuddy's AMS tracking / tag matching / usage tracker all point at filament-kind lots. For 500 magnets, lot = the bag. |
| **`consumption`** | Unified ledger: lot → print_job or order_item, quantity, `source = measured \| estimated \| manual`. Filament measured (AMS remain% delta, 3MF estimate fallback — Bambuddy `usage_tracker` logic). Components auto-posted `estimated` from BOM × qty on job completion, manually correctable. One ledger drives stock decrement AND actual COGS for every kind. |
| **`variant_material`** (BOM) | Variant → material + qty-per-unit. Scheduler reads only `kind = filament` rows for `required_filaments`; COGS and stock checks read ALL rows — so an order correctly blocks when you have plastic but no magnets. |

Material groups ("any black PLA") die as tables → filter query (`kind=filament AND type=PLA AND color≈black`); blended cost = weighted avg over matching lots, as a view.

## 2. Print job (4 tables → 1 lifecycle)

Old: Bambuddy `print_queue` → `print_archives` handoff; tuani3d-app `print_job` + `print_run`.

New: single **`print_job`**: `draft → queued → dispatched → printing → done | failed`.
- Keeps Bambuddy late binding (`target_model`, `target_location`, `required_filaments`), gates (`manual_start`, `scheduled_time`, `require_previous_success`, filament-sufficiency), SJF + starvation guard.
- Keeps archive fields on the same row: 3MF path, thumbnail, timelapse, measured grams/seconds/energy.
- `order_item_id` nullable FK — job fulfills an order or is ad-hoc.
- "Archive" = filter `status IN (done, failed)`. No table handoff.

## 3. Products & variants (the linchpin)

**Variant = sellable SKU = printable unit.** Price + BOM + `library_file_id` (3MF) + print profile on one entity. This join powers: `order_item → variant → {3MF for scheduler, BOM for cost, price for invoice}`.

- Keep tuani3d-app's `product` / `product_variant` model (ported to SQLAlchemy). Delete Bambuddy `projects`/`project_bom_items` (proto-product, superseded).
- Variant conditions (new, b_stock, sample…) kept.
- MakerWorld import kept as import action writing products/variants (merge best of both importers; source URL stored on product for re-sync — no collection tables needed for sync).

## 4. Bundles (4 tables → 1)

**`variant.kind = single | bundle`** + **`bundle_component (parent_variant_id, child_variant_id, qty)`**.

A bundle is a sellable SKU whose "BOM" is other variants. Orders, invoices, discounts, pricing treat it as any variant — no `order_item_type` branching, no parallel dialogs/views. Fulfillment explodes into components: per-component stock check, shortfall → one print job per missing component, ships when all present. Availability = min over components.

## 5. Collections

Cut as core entity. Merchandising → product **tags** (reuse Bambuddy library tag pattern). MakerWorld collection sync → import action keyed off stored source URL.

Returns ONLY as storefront presentation: `storefront_collection` (+ membership/sort) in the storefront module, plus per-product `published`, `slug`, `seo_description`.

## 6. Files

One library: Bambuddy's `library_files`/`library_folders` wins (hashing, dedupe, external mounts, trash, tags). Variants reference `library_file_id`. tuani3d-app's asset table + signed-URL upload dance deleted (no Vercel limit anymore).

## 7. Commerce (keep from tuani3d-app, rebuilt clean)

- **Orders**: order/order_item/order_log, payments, shipments, returns/RMA — best schema in either repo. Rename `order` → **`sales_order`** (kills reserved-word quoting forever). Add `channel = storefront | manual`.
- **Parties**: one `party` table, `kind = customer | supplier` + profile-versioning/address/note children (old schemas were mirror images).
- **Multi-currency**: keep fully — base currency, historical rates, per-order FX locking, live fetch.
- **Invoices**: keep model; PDF server-side (reportlab already in stack; kill html2pdf.js). Multi-language invoice output via translation JSON on product/variant.
- **Discounts (simplified)**: % or fixed at order + item level, optional code. No targeting engine until needed.
- **COGS**: `variant_cogs` = BOM estimate (pricing); `job_actual_cost` = measured grams × lot cost-at-purchase + energy_cost + time × labor rate (truth). Labor rate is a setting, no hardcoded fallback.

## 8. Inherited unchanged from Bambuddy

Machines (MQTT, printer manager, HMS, firmware, maintenance, cameras + fanout + Cam Wall + OBS overlay), virtual printer + proxy mode, smart plugs + energy tracking, notifications (providers/templates/digests), identity (JWT, RBAC `*_own`/`*_all`, MFA TOTP+email, OIDC, scoped API keys), settings, i18n (11 locales, file-based), 3MF tools, gcode viewer.

## Table budget (~35)

catalog: material, material_lot, consumption, product, product_variant, variant_material, bundle_component, library_file, library_folder, library_tag (+M2M)
production: printer, virtual_printer, print_job, maintenance_type, maintenance_event, sensor_history, smart_plug, energy_snapshot
commerce: party, party_profile, party_address, party_note, sales_order, order_item, order_log, payment, shipment, shipment_item, return, return_item, discount, invoice? (or derived), shipping_carrier/service
finance: currency, exchange_rate
platform: user, group (+M2M), api_key, setting, notification_provider/template/log, location
storefront: storefront_collection (+membership)
