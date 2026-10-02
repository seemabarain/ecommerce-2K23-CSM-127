# Sprint 2: Catalog Data Foundation

**Project:** CaseVault — Mobile Accessories Store
**Course:** E-Commerce
**Team:** `[Team name]` — `[Member 1, Member 2, Member 3, Member 4]`
**Repository:** `/docs/SPRINT_2.md`
**Sprint length:** 15 days (checkpoints on Days 5, 10, 15)
**Depends on:** [`SPRINT_1.md`](./SPRINT_1.md), Week 3 (Product → Variant → SKU matrix)

> **Note for reviewers.** Items marked `TEAM TODO` are evidence that can only be produced by running the code (captured API output, test run output, screenshots). Everything else in this document is the design the implementation must match.

---

## Table of Contents

1. [Sprint Goal & Scope Boundary](#1-sprint-goal--scope-boundary)
2. [Link to Sprint 1 Decisions (Reused / Changed)](#2-link-to-sprint-1-decisions-reused--changed)
3. [Updated ERD & Data Dictionary](#3-updated-erd--data-dictionary)
4. [Administration Route Table with Examples](#4-administration-route-table-with-examples)
5. [Data Integrity & Authorization Decisions](#5-data-integrity--authorization-decisions)
6. [Seed Data & Demonstration Instructions](#6-seed-data--demonstration-instructions)
7. [Test Strategy, Command & Result](#7-test-strategy-command--result)
8. [Known Limitations & Sprint 3 Backlog](#8-known-limitations--sprint-3-backlog)
- [Appendix A — Migration SQL](#appendix-a--migration-sql)
- [Appendix B — Seed SQL](#appendix-b--seed-sql)
- [Appendix C — README Section (Local Setup)](#appendix-c--readme-section-local-setup)
- [Appendix D — Repository Layout](#appendix-d--repository-layout)

---

## 1. Sprint Goal & Scope Boundary

**Sprint goal.** Given a product catalog administrator, the system must persist **categories, products, variants, and SKUs** without losing identity, relationship, price, or inventory meaning.

### In scope (implemented in Sprint 2)

| Area | Delivered |
|---|---|
| Categories | Tree with stable ids and unique slugs, optional parent, cycle prevention, deactivate/list |
| Products | Create/edit with name, slug, description, status, canonical category |
| Variants | Option-combination records that only exist for valid combinations |
| SKUs | Unique code, own price (integer minor units), own stock, active flag |
| Administration | Authenticated, role-checked admin API (create/read/update/delete) |
| Integrity | Constraints, triggers, migrations, seed data, automated tests |

### Out of scope (explicitly *not* claimed)

| Item | Where it lands |
|---|---|
| Dynamic specifications **behaviour** (validation rules, filtering) | Sprint 3 |
| Asset **upload**, storage, resizing | Sprint 3 |
| Public catalog read/search, device-compatibility filter | Sprint 3 |
| Publication workflows (scheduling, approvals) | Sprint 3 |
| Cart, checkout, Stripe, orders, wishlist | Sprint 3 and later |

> The `assets`, `device_models`, and `product_compatibility` tables and the `products.specs` column exist in the schema so the model is complete and Sprint 3 can build on it. They have **no API and no behaviour** in Sprint 2.

---

## 2. Link to Sprint 1 Decisions (Reused / Changed)

| Sprint 1 decision | Sprint 2 status | Notes |
|---|---|---|
| Single-vendor storefront (Section 1) | **Reused** | No seller/vendor entity. |
| Vanilla HTML/CSS/JS frontend (Section 3) | **Reused** | Admin is exercised via API (curl/tests); a minimal admin UI is optional. |
| Supabase (PostgreSQL + Auth + RLS) (Section 3) | **Reused** | Postgres is the system of record; Supabase Auth issues JWTs; RLS is defense-in-depth. |
| Supabase auto-generated REST for standard CRUD | **Changed** | Sprint 2 requires versioned admin routes (`/api/v1/admin/...`) with consistent validation and error shapes, so a small **Node.js + Express** service sits in front of Postgres for admin writes. RLS still blocks direct writes through the auto-generated endpoints. |
| `USERS` table with `password_hash` | **Changed** | Passwords/JWTs are owned by Supabase Auth (`auth.users`). The app keeps a `PROFILES` table (`user_id`, `role`) for authorization. No password hash is stored by the application. |
| Flat `CATEGORIES (id, name, slug)` | **Extended** | Added `parent_id`, `sort_order`, `is_active`, timestamps; tree with cycle prevention. |
| `PRODUCTS.base_price` | **Removed** | Price is meaningful only on a sellable unit. Price now lives on `SKUS` (single source of truth). A "from" price is *derived* at read time. |
| `PRODUCT_VARIANTS (color, material, price_override, stock_quantity, sku)` | **Split** | Sprint 1 mixed *configuration* and *sellable unit* in one table. Sprint 2 follows Week 3: `VARIANTS` = configuration (option values); `SKUS` = sellable unit (code, price, stock). |
| `CART_ITEMS.variant_id`, `ORDER_ITEMS.variant_id`, `WISHLIST_ITEMS.variant_id` | **Re-pointed (planned)** | These will reference `SKUS.id` in Sprint 3, because price and stock now live on the SKU. |
| `DEVICE_MODELS`, `PRODUCT_COMPATIBILITY` (N:M junction) | **Reused (schema only)** | Kept unchanged except `release_year` → `smallint`. Admin CRUD and the compatibility filter are Sprint 3. |
| MVP priority: Admin Inventory Control = Medium | **Reused** | Sprint 2 delivers the catalog subset of that feature (products, variants, SKUs, stock). Compatibility mapping admin is Sprint 3. |
| Optional Redis/queue out of scope | **Reused** | No caching layer. Stock levels are plain columns; Realtime stays a Sprint 3+ concern. |

**Deliberate deviation from the sample diagram in the Sprint 2 manual:** the manual's sketch shows `PRODUCTS ||--o{ CART_ITEMS`. We connect `CART_ITEMS` to **`SKUS`** instead. A product such as the *Slim Case* has several SKUs with different prices and stock, so a cart line pointing at the product would be ambiguous; only a SKU identifies *exactly what is being bought, at what price, from which stock*. The product is still reachable via `SKUS.product_id`.

---

## 3. Updated ERD & Data Dictionary

### 3.1 Mermaid ER diagram

Solid lines (`--`) are implemented in Sprint 2. Dotted lines (`..`) are the **planned** Sprint 1 connections (Cart, Cart_Items, Orders, Order_Items, Wishlist_Items) that Sprint 3 will implement against `SKUS`.

```mermaid
erDiagram
    PROFILES {
        uuid user_id PK "FK auth.users.id"
        text role "customer or admin"
        timestamptz created_at
    }

    CATEGORIES {
        bigint id PK
        bigint parent_id FK "nullable, self reference"
        text name
        text slug UK
        integer sort_order
        boolean is_active
        timestamptz created_at
        timestamptz updated_at
    }

    PRODUCTS {
        bigint id PK
        bigint category_id FK "canonical category"
        text name
        text slug UK
        text description
        text status "draft, active, archived"
        jsonb specs "validated JSONB object"
        timestamptz created_at
        timestamptz updated_at
    }

    PRODUCT_OPTIONS {
        bigint id PK
        bigint product_id FK
        text name "e.g. color, material"
        smallint position
        text_array allowed_values "text[]"
    }

    VARIANTS {
        bigint id PK
        bigint product_id FK
        jsonb option_values "e.g. color Black, material Silicone"
        timestamptz created_at
        timestamptz updated_at
    }

    SKUS {
        bigint id PK
        bigint product_id FK
        bigint variant_id FK "nullable for no-variant products"
        text code UK
        smallint pack_size
        integer price_minor "money in minor units"
        integer compare_at_minor "nullable"
        text currency "ISO 4217, default PKR"
        integer stock_quantity "CHECK >= 0"
        boolean is_active
        timestamptz created_at
        timestamptz updated_at
    }

    ASSETS {
        bigint id PK
        bigint product_id FK
        bigint variant_id FK "nullable"
        text storage_key UK
        text role "hero, gallery, detail, swatch"
        text alt_text
        integer sort_order
        timestamptz created_at
    }

    DEVICE_MODELS {
        bigint id PK
        text brand
        text model_name
        smallint release_year
    }

    PRODUCT_COMPATIBILITY {
        bigint id PK
        bigint product_id FK
        bigint device_model_id FK
    }

    CART {
        bigint id PK "planned Sprint 3"
        uuid user_id FK
        timestamptz updated_at
    }

    CART_ITEMS {
        bigint id PK "planned Sprint 3"
        bigint cart_id FK
        bigint sku_id FK "was variant_id in Sprint 1"
        integer quantity
    }

    ORDERS {
        bigint id PK "planned Sprint 3"
        uuid user_id FK
        integer total_minor
        text status
        timestamptz created_at
    }

    ORDER_ITEMS {
        bigint id PK "planned Sprint 3"
        bigint order_id FK
        bigint sku_id FK "was variant_id in Sprint 1"
        text sku_code_snapshot
        text product_name_snapshot
        integer unit_price_minor
        integer quantity
    }

    WISHLIST_ITEMS {
        bigint id PK "planned Sprint 3"
        uuid user_id FK
        bigint sku_id FK
        timestamptz added_at
    }

    CATEGORIES |o--o{ CATEGORIES : "parent of"
    CATEGORIES ||--o{ PRODUCTS : "canonically contains"
    PRODUCTS ||--o{ PRODUCT_OPTIONS : "defines"
    PRODUCTS ||--o{ VARIANTS : "has"
    PRODUCTS ||--o{ SKUS : "sold as"
    VARIANTS |o--o{ SKUS : "materializes"
    PRODUCTS ||--o{ ASSETS : "displays"
    VARIANTS |o--o{ ASSETS : "optionally illustrated by"
    PRODUCTS ||--o{ PRODUCT_COMPATIBILITY : "fits"
    DEVICE_MODELS ||--o{ PRODUCT_COMPATIBILITY : "supported by"

    PROFILES ||..o| CART : "owns (planned)"
    CART ||..o{ CART_ITEMS : "contains (planned)"
    SKUS ||..o{ CART_ITEMS : "selected as (planned)"
    PROFILES ||..o{ ORDERS : "places (planned)"
    ORDERS ||..|{ ORDER_ITEMS : "contains (planned)"
    SKUS ||..o{ ORDER_ITEMS : "sold as (planned)"
    PROFILES ||..o{ WISHLIST_ITEMS : "saves (planned)"
    SKUS ||..o{ WISHLIST_ITEMS : "saved as (planned)"
```

### 3.2 Cardinality and delete/update policy

Every foreign key uses **`ON UPDATE RESTRICT`**: primary keys are identity columns and are never changed.

| Relationship | Cardinality | FK | `ON DELETE` | Why |
|---|---|---|---|---|
| Category → Category | 0..1 parent : 0..N children | `categories.parent_id` | `RESTRICT` | A category with children cannot be removed; admins **deactivate** instead. |
| Category → Product | 1 : 0..N | `products.category_id` | `RESTRICT` | A category that still owns products cannot disappear (ownership is never orphaned). |
| Product → Option | 1 : 0..N | `product_options.product_id` | `CASCADE` | Options are meaningless without their product. |
| Product → Variant | 1 : 0..N | `variants.product_id` | `CASCADE` | Variants are meaningless without their product. |
| Product → SKU | 1 : 0..N (≥1 once `active`) | `skus.product_id` | `CASCADE` | Hard delete is only possible for never-sold products (see below). |
| Variant → SKU | 0..1 : 0..N | `skus (variant_id, product_id)` composite | `CASCADE` | Composite FK guarantees the SKU's variant belongs to the **same product**. `variant_id` is `NULL` for products without variants. |
| Product → Asset | 1 : 0..N | `assets.product_id` | `CASCADE` | Schema only in Sprint 2. |
| Variant → Asset | 0..1 : 0..N | `assets (variant_id, product_id)` composite | `SET NULL (variant_id)` | Deleting a variant keeps the product-level image. Requires PostgreSQL 15+ (Supabase is ≥ 15). |
| Product ↔ Device model | N : M | `product_compatibility.*` | `CASCADE` (product) / `RESTRICT` (device) | Junction row dies with the product; a device model in use cannot be deleted. |
| Profile → auth user | 1 : 1 | `profiles.user_id` | `CASCADE` | Profile follows the account. |
| **Planned:** SKU → Cart item | 1 : 0..N | `cart_items.sku_id` | `CASCADE` | Carts are transient. |
| **Planned:** SKU → Order item | 1 : 0..N | `order_items.sku_id` | **`RESTRICT`** | A SKU that was ever sold can never be hard-deleted; history stays intact. |
| **Planned:** User → Order | 1 : 0..N | `orders.user_id` | `RESTRICT` | Orders are financial records. |

### 3.3 Data dictionary

#### `categories`

| Column | Type | Constraints / Notes |
|---|---|---|
| `id` | `bigint` identity | **PK** |
| `parent_id` | `bigint` | FK → `categories.id`, nullable (root), `CHECK (parent_id <> id)`, cycle-guard trigger |
| `name` | `text` | NOT NULL, 1–80 chars after trim |
| `slug` | `text` | NOT NULL, **UNIQUE**, `^[a-z0-9]+(-[a-z0-9]+)*$` |
| `sort_order` | `integer` | NOT NULL, default 0 |
| `is_active` | `boolean` | NOT NULL, default true |
| `created_at`, `updated_at` | `timestamptz` | NOT NULL, default `now()`; `updated_at` maintained by trigger |

#### `products`

| Column | Type | Constraints / Notes |
|---|---|---|
| `id` | `bigint` identity | **PK** |
| `category_id` | `bigint` | NOT NULL, FK → `categories.id` (canonical category) |
| `name` | `text` | NOT NULL, 1–160 chars |
| `slug` | `text` | NOT NULL, **UNIQUE**, slug format check |
| `description` | `text` | NOT NULL, default `''` |
| `status` | `text` | NOT NULL, default `draft`, `CHECK IN ('draft','active','archived')` |
| `specs` | `jsonb` | NOT NULL, default `{}`, `CHECK (jsonb_typeof(specs) = 'object')` |
| `created_at`, `updated_at` | `timestamptz` | as above |

**Specification validation rule (JSONB chosen over EAV).** `products.specs` is a flat-or-nested JSON **object** (database `CHECK`), never an array or scalar. Anything that needs joins, constraints, sorting, or frequent filtering — identity, price, stock, category, SKU code, colour/material options — is **not** in `specs`; it is relational (Week 3 rule of thumb). `specs` holds only genuinely dynamic, category-specific facts such as `{"camera_cutout": "iPhone 15", "magsafe": true, "drop_protection_m": 2}`. Rationale: CaseVault's dynamic attributes differ per accessory type (cases vs. chargers vs. screen protectors) but are *read* far more than they are filtered, so a single JSONB column avoids the join and validation cost of EAV. Key-level validation per category and GIN indexing for filters are Sprint 3 work.

#### `product_options`

| Column | Type | Constraints / Notes |
|---|---|---|
| `id` | `bigint` identity | **PK** |
| `product_id` | `bigint` | NOT NULL, FK → `products.id` |
| `name` | `text` | NOT NULL, `^[a-z][a-z0-9_]*$` (e.g. `color`, `material`), **UNIQUE** `(product_id, name)` |
| `position` | `smallint` | NOT NULL, default 0 |
| `allowed_values` | `text[]` | NOT NULL, at least one value |

#### `variants`

| Column | Type | Constraints / Notes |
|---|---|---|
| `id` | `bigint` identity | **PK**; also `UNIQUE (id, product_id)` (target of composite FKs) |
| `product_id` | `bigint` | NOT NULL, FK → `products.id` |
| `option_values` | `jsonb` | NOT NULL, object; e.g. `{"color":"Black","material":"Silicone"}`; trigger checks keys/values against `product_options` |
| `created_at`, `updated_at` | `timestamptz` | as above |
| — | — | **UNIQUE `(product_id, option_values)`**: the same combination cannot exist twice |

#### `skus`

| Column | Type | Constraints / Notes |
|---|---|---|
| `id` | `bigint` identity | **PK** |
| `product_id` | `bigint` | NOT NULL, FK → `products.id` |
| `variant_id` | `bigint` | nullable; composite FK `(variant_id, product_id)` → `variants (id, product_id)` |
| `code` | `text` | NOT NULL, **UNIQUE**, upper-case `^[A-Z0-9]+(-[A-Z0-9]+)*$` |
| `pack_size` | `smallint` | NOT NULL, default 1, `> 0`. Lets one variant be sold as several SKUs (e.g. 1-pack, 2-pack) |
| `price_minor` | `integer` | NOT NULL, `>= 0`. Money in minor units (1 PKR = 100 paisa); matches Stripe's integer amounts. **No floats.** |
| `compare_at_minor` | `integer` | nullable "was price"; `CHECK (compare_at_minor > price_minor)` |
| `currency` | `text` | NOT NULL, default `PKR`, `^[A-Z]{3}$` |
| `stock_quantity` | `integer` | NOT NULL, default 0, **`CHECK (stock_quantity >= 0)`** |
| `is_active` | `boolean` | NOT NULL, default true |
| `created_at`, `updated_at` | `timestamptz` | as above |
| — | — | Partial unique indexes: `(variant_id, pack_size)` where `variant_id IS NOT NULL`; `(product_id, pack_size)` where `variant_id IS NULL` |

#### `assets` *(schema only)*

| Column | Type | Constraints / Notes |
|---|---|---|
| `id` | `bigint` identity | **PK** |
| `product_id` | `bigint` | NOT NULL, FK → `products.id` |
| `variant_id` | `bigint` | nullable, composite FK to `variants (id, product_id)` |
| `storage_key` | `text` | NOT NULL, **UNIQUE** (object key or URL) |
| `role` | `text` | NOT NULL, `CHECK IN ('hero','gallery','detail','swatch')` |
| `alt_text` | `text` | NOT NULL, non-blank (accessibility) |
| `sort_order` | `integer` | NOT NULL, default 0 |
| `created_at` | `timestamptz` | NOT NULL |

#### `device_models`, `product_compatibility` *(retained from Sprint 1, schema only)*

| Table | Columns |
|---|---|
| `device_models` | `id` PK, `brand` text, `model_name` text, `release_year` smallint, **UNIQUE** `(brand, model_name)` |
| `product_compatibility` | `id` PK, `product_id` FK, `device_model_id` FK, **UNIQUE** `(product_id, device_model_id)` |

#### `profiles`

| Column | Type | Constraints / Notes |
|---|---|---|
| `user_id` | `uuid` | **PK**, FK → `auth.users.id` |
| `role` | `text` | NOT NULL, default `customer`, `CHECK IN ('customer','admin')` |
| `created_at` | `timestamptz` | NOT NULL |

### 3.4 Product vs. Variant vs. SKU in CaseVault (Week 3 vocabulary)

| Level | Meaning | CaseVault example |
|---|---|---|
| **Product** | Customer-facing concept; shared story, description, category | *Slim Case for iPhone 15* |
| **Variant** | A valid configuration of the product's options | *Black + Silicone* |
| **SKU** | Sellable stock unit with own code, price, and stock | `CV-SLM15-BLK-SIL` |

**SKU matrix for the Slim Case** (a missing combination is *absent*, never a fake or zero-stock SKU — requirement CAT04):

| | Silicone | Leather |
|---|---|---|
| **Black** | `CV-SLM15-BLK-SIL` — Rs. 1,499, stock 25 | `CV-SLM15-BLK-LEA` — Rs. 2,499, stock **0** (valid, sold out) |
| **Navy** | `CV-SLM15-NVY-SIL` — Rs. 1,499, stock 18 | **no variant row** (combination is not produced) |

Sold-out and non-existent are different facts: *Black/Leather* exists with `stock_quantity = 0`; *Navy/Leather* has no row at all.

---

## 4. Administration Route Table with Examples

**Base path:** `/api/v1/admin` · **Format:** JSON · **Money:** integer minor units (`price_minor`) plus `currency`.

### 4.1 Conventions

**Authentication.** Every route requires `Authorization: Bearer <ADMIN_JWT>` (a Supabase Auth access token) for a user whose `profiles.role = 'admin'`.

| Situation | Status | `error.code` |
|---|---|---|
| Missing, malformed, or expired token | `401` | `unauthenticated` |
| Valid token, role is not `admin` | `403` | `forbidden` |

**Success envelope.** `{ "data": <object | array> }`. Lists add `"meta": { "total", "limit", "offset" }`.

**Error envelope (the same everywhere).**

```json
{
  "error": {
    "code": "duplicate_sku_code",
    "message": "A SKU with code 'CV-SLM15-BLK-SIL' already exists.",
    "details": [{ "field": "code", "issue": "must be unique" }],
    "request_id": "7d1f0c2e"
  }
}
```

**Status-code policy.**

| Status | Meaning | Examples of `error.code` |
|---|---|---|
| `400` | Malformed or missing/invalid fields | `validation_failed` |
| `401` / `403` | Authentication / authorization | `unauthenticated`, `forbidden` |
| `404` | Referenced resource does not exist | `not_found` |
| `409` | Conflict with existing data | `duplicate_slug`, `duplicate_sku_code`, `duplicate_variant_combination`, `resource_in_use` |
| `422` | Well-formed but breaks a business rule | `category_cycle`, `inactive_parent`, `invalid_variant_options`, `negative_stock`, `product_needs_active_sku`, `last_active_sku` |

PostgreSQL errors are translated, never leaked: `23505` (unique) → `409`, `23514` (check) → `422`, `23503` (foreign key) → `409`/`404`, custom `CV001`–`CV005` → `422`. Clients never see a stack trace.

### 4.2 Route table

| Method | Route | Purpose | Required by manual |
|---|---|---|---|
| `POST` | `/api/v1/admin/categories` | Create a category | ✅ baseline |
| `GET` | `/api/v1/admin/categories` | Return the category tree | ✅ baseline |
| `PATCH` | `/api/v1/admin/categories/:id` | Rename, move, reorder, (de)activate | extra (CAT01) |
| `POST` | `/api/v1/admin/products` | Create a **draft** product | ✅ baseline |
| `GET` | `/api/v1/admin/products` | List products (filter, paginate) | ✅ baseline |
| `GET` | `/api/v1/admin/products/:id` | One product with options, variants, SKUs | extra |
| `PATCH` | `/api/v1/admin/products/:id` | Update content or status | ✅ baseline |
| `DELETE` | `/api/v1/admin/products/:id` | Hard-delete a never-sold product | extra (CRUD "D") |
| `POST` | `/api/v1/admin/products/:id/options` | Define an option (e.g. `color`) | extra (needed for variants) |
| `POST` | `/api/v1/admin/products/:id/variants` | Create a valid option combination | extra (CAT04) |
| `POST` | `/api/v1/admin/products/:id/skus` | Add a SKU | ✅ baseline |
| `PATCH` | `/api/v1/admin/skus/:id` | Update price, stock, or active status | ✅ baseline |
| `DELETE` | `/api/v1/admin/skus/:id` | Hard-delete a never-sold SKU | extra (CRUD "D") |

Examples below use `Authorization: Bearer <ADMIN_JWT>` (token redacted) and `$API` = `http://localhost:3000/api/v1/admin`.

---

### 4.3 `POST /categories`

| | |
|---|---|
| **Auth** | admin only |
| **Body** | `name` (string, required) · `slug` (string, optional; generated from `name`) · `parent_id` (integer, optional) · `sort_order` (integer, optional) · `is_active` (boolean, optional, default `true`) |
| **Success** | `201 Created` → `{ "data": Category }` |
| **Errors** | `400 validation_failed` · `401` · `403` · `404 not_found` (parent) · `409 duplicate_slug` · `422 inactive_parent` |

```bash
curl -X POST "$API/categories" \
  -H "Authorization: Bearer <ADMIN_JWT>" -H "Content-Type: application/json" \
  -d '{"name":"Charging","slug":"charging","parent_id":1}'
```
```json
{
  "data": {
    "id": 4, "parent_id": 1, "name": "Charging", "slug": "charging",
    "sort_order": 0, "is_active": true,
    "created_at": "2026-10-02T09:15:00Z", "updated_at": "2026-10-02T09:15:00Z"
  }
}
```

**Duplicate slug →** `409`
```json
{ "error": { "code": "duplicate_slug", "message": "A category with slug 'charging' already exists.",
             "details": [{ "field": "slug", "issue": "must be unique" }], "request_id": "a91c33be" } }
```

### 4.4 `GET /categories`

| | |
|---|---|
| **Auth** | admin only |
| **Query** | `include_inactive` (boolean, default `true` for admins) |
| **Success** | `200 OK` → nested tree ordered by `sort_order`, `name` |
| **Errors** | `401` · `403` |

```json
{
  "data": [
    { "id": 1, "name": "Accessories", "slug": "accessories", "is_active": true, "children": [
      { "id": 2, "name": "Cases", "slug": "cases", "is_active": true, "children": [] },
      { "id": 3, "name": "Screen Protectors", "slug": "screen-protectors", "is_active": true, "children": [] },
      { "id": 4, "name": "Charging", "slug": "charging", "is_active": true, "children": [
        { "id": 5, "name": "Wireless Chargers", "slug": "wireless-chargers", "is_active": true, "children": [] }
      ] }
    ] }
  ]
}
```

### 4.5 `PATCH /categories/:id`

| | |
|---|---|
| **Auth** | admin only |
| **Body** | any of `name`, `slug`, `parent_id` (or `null` to make root), `sort_order`, `is_active` |
| **Success** | `200 OK` → updated category |
| **Errors** | `400` · `401` · `403` · `404` · `409 duplicate_slug` · `422 category_cycle` · `422 inactive_parent` |

**Cycle attempt** (make *Accessories* a child of its own descendant *Wireless Chargers*):
```bash
curl -X PATCH "$API/categories/1" -H "Authorization: Bearer <ADMIN_JWT>" \
  -H "Content-Type: application/json" -d '{"parent_id":5}'
```
```json
{ "error": { "code": "category_cycle",
             "message": "Category 1 cannot become its own ancestor.",
             "details": [{ "field": "parent_id", "issue": "would create a cycle" }], "request_id": "c0d2e7f1" } }
```

**Deactivate** (also deactivates all descendants — see §5.3): `{"is_active": false}` → `200 OK`.

### 4.6 `POST /products`

| | |
|---|---|
| **Auth** | admin only |
| **Body** | `name` (required) · `category_id` (required) · `slug` (optional; generated) · `description` (optional) · `specs` (optional JSON object) · `status` (optional; only `draft` is accepted on create) |
| **Success** | `201 Created` → product in `draft` status |
| **Errors** | `400` · `401` · `403` · `404 not_found` (category) · `409 duplicate_slug` · `422 product_needs_active_sku` (if `status: "active"` is sent) |

```bash
curl -X POST "$API/products" -H "Authorization: Bearer <ADMIN_JWT>" \
  -H "Content-Type: application/json" \
  -d '{"name":"Slim Case for iPhone 15","category_id":2,
       "description":"Slim, grippy protection.",
       "specs":{"magsafe":false,"drop_protection_m":1.5}}'
```
```json
{
  "data": {
    "id": 1, "category_id": 2, "name": "Slim Case for iPhone 15",
    "slug": "slim-case-for-iphone-15", "description": "Slim, grippy protection.",
    "status": "draft", "specs": { "magsafe": false, "drop_protection_m": 1.5 },
    "created_at": "2026-10-02T09:20:00Z", "updated_at": "2026-10-02T09:20:00Z"
  }
}
```

### 4.7 `GET /products`

| | |
|---|---|
| **Auth** | admin only |
| **Query** | `status` (`draft`\|`active`\|`archived`) · `category_id` · `q` (name search) · `limit` (default 20, max 100) · `offset` |
| **Success** | `200 OK` → `{ data: [Product with options, variants, skus], meta }` |
| **Errors** | `400` (bad filter) · `401` · `403` |

Administrative records expose **raw** `stock_quantity` and inactive SKUs (unlike the public contract in §5.6).

```json
{
  "data": [{
    "id": 1, "name": "Slim Case for iPhone 15", "slug": "slim-case-for-iphone-15", "status": "active",
    "category": { "id": 2, "name": "Cases" },
    "options": [ { "name": "color", "allowed_values": ["Black","Navy"] },
                 { "name": "material", "allowed_values": ["Silicone","Leather"] } ],
    "variants": [
      { "id": 1, "option_values": { "color": "Black", "material": "Silicone" } },
      { "id": 2, "option_values": { "color": "Navy",  "material": "Silicone" } },
      { "id": 3, "option_values": { "color": "Black", "material": "Leather"  } }
    ],
    "skus": [
      { "id": 1, "variant_id": 1, "code": "CV-SLM15-BLK-SIL", "price_minor": 149900, "currency": "PKR", "stock_quantity": 25, "is_active": true },
      { "id": 2, "variant_id": 2, "code": "CV-SLM15-NVY-SIL", "price_minor": 149900, "currency": "PKR", "stock_quantity": 18, "is_active": true },
      { "id": 3, "variant_id": 3, "code": "CV-SLM15-BLK-LEA", "price_minor": 249900, "currency": "PKR", "stock_quantity": 0,  "is_active": true }
    ]
  }],
  "meta": { "total": 4, "limit": 20, "offset": 0 }
}
```

### 4.8 `GET /products/:id`

Same shape as one element of §4.7 `data`. Errors: `401` · `403` · `404 not_found`.

### 4.9 `PATCH /products/:id`

| | |
|---|---|
| **Auth** | admin only |
| **Body** | any of `name`, `slug`, `description`, `category_id`, `status`, `specs` |
| **Success** | `200 OK` → updated product |
| **Errors** | `400` · `401` · `403` · `404` · `409 duplicate_slug` · `422 product_needs_active_sku` |

**Publishing a product with no active SKU is rejected by the database** (not only the API):
```bash
curl -X PATCH "$API/products/4" -H "Authorization: Bearer <ADMIN_JWT>" \
  -H "Content-Type: application/json" -d '{"status":"active"}'
```
```json
{ "error": { "code": "product_needs_active_sku",
             "message": "Product 4 needs at least one active SKU before it can be set to 'active'.",
             "details": [{ "field": "status", "issue": "no active SKU" }], "request_id": "5be40a9d" } }
```

### 4.10 `DELETE /products/:id`

| | |
|---|---|
| **Auth** | admin only |
| **Success** | `204 No Content` |
| **Errors** | `401` · `403` · `404` · `409 resource_in_use` (product has a SKU referenced by an order — enforced by `ON DELETE RESTRICT` once Sprint 3 adds `order_items`; archive instead via `PATCH {"status":"archived"}`) |

### 4.11 `POST /products/:id/options`

| | |
|---|---|
| **Auth** | admin only |
| **Body** | `name` (required, lower_snake_case) · `allowed_values` (required, non-empty string array) · `position` (optional) |
| **Success** | `201 Created` |
| **Errors** | `400` · `401` · `403` · `404` · `409 duplicate_option` |

```bash
curl -X POST "$API/products/1/options" -H "Authorization: Bearer <ADMIN_JWT>" \
  -H "Content-Type: application/json" -d '{"name":"color","allowed_values":["Black","Navy"]}'
```
```json
{ "data": { "id": 1, "product_id": 1, "name": "color", "position": 0, "allowed_values": ["Black","Navy"] } }
```

### 4.12 `POST /products/:id/variants`

| | |
|---|---|
| **Auth** | admin only |
| **Body** | `option_values` (required object; must contain **every** option of the product, each with an allowed value) |
| **Success** | `201 Created` |
| **Errors** | `400` · `401` · `403` · `404` · `409 duplicate_variant_combination` · `422 invalid_variant_options` |

```bash
curl -X POST "$API/products/1/variants" -H "Authorization: Bearer <ADMIN_JWT>" \
  -H "Content-Type: application/json" -d '{"option_values":{"color":"Black","material":"Silicone"}}'
```
```json
{ "data": { "id": 1, "product_id": 1, "option_values": { "color": "Black", "material": "Silicone" } } }
```

**Invalid value →** `422`
```json
{ "error": { "code": "invalid_variant_options",
             "message": "Value 'Red' is not allowed for option 'color'.",
             "details": [{ "field": "option_values.color", "issue": "allowed: Black, Navy" }], "request_id": "f2a771c0" } }
```

### 4.13 `POST /products/:id/skus`

| | |
|---|---|
| **Auth** | admin only |
| **Body** | `code` (required, `A-Z0-9-`) · `price_minor` (required integer ≥ 0) · `stock_quantity` (optional integer ≥ 0, default 0) · `variant_id` (optional; must belong to this product) · `pack_size` (optional, default 1) · `compare_at_minor` (optional, must exceed `price_minor`) · `currency` (optional, default `PKR`) · `is_active` (optional) |
| **Success** | `201 Created` → SKU |
| **Errors** | `400` (e.g. float price, lowercase code) · `401` · `403` · `404` (product / variant not found for this product) · `409 duplicate_sku_code` · `409 duplicate_sku_pack` · `422 negative_stock` |

```bash
curl -X POST "$API/products/1/skus" -H "Authorization: Bearer <ADMIN_JWT>" \
  -H "Content-Type: application/json" \
  -d '{"code":"CV-SLM15-BLK-SIL","variant_id":1,"price_minor":149900,"stock_quantity":25}'
```
```json
{
  "data": {
    "id": 1, "product_id": 1, "variant_id": 1, "code": "CV-SLM15-BLK-SIL",
    "pack_size": 1, "price_minor": 149900, "compare_at_minor": null, "currency": "PKR",
    "stock_quantity": 25, "is_active": true,
    "created_at": "2026-10-02T09:40:00Z", "updated_at": "2026-10-02T09:40:00Z"
  }
}
```

**Duplicate SKU code →** `409 duplicate_sku_code` (see error envelope in §4.1). **Float price** (`"price_minor": 14.99`) → `400 validation_failed`, `details: [{"field":"price_minor","issue":"must be a non-negative integer"}]`.

### 4.14 `PATCH /skus/:id`

| | |
|---|---|
| **Auth** | admin only |
| **Body** | any of `price_minor`, `compare_at_minor`, `stock_quantity` (absolute set), `stock_delta` (signed change; mutually exclusive with `stock_quantity`), `is_active` |
| **Success** | `200 OK` → updated SKU |
| **Errors** | `400` · `401` · `403` · `404` · `422 negative_stock` · `422 last_active_sku` (deactivating the last active SKU of an `active` product) |

`stock_delta` is applied atomically in SQL (`stock_quantity = stock_quantity + $delta`), so two simultaneous admin updates cannot lose each other's change, and the `CHECK` rejects results below zero.

```bash
curl -X PATCH "$API/skus/1" -H "Authorization: Bearer <ADMIN_JWT>" \
  -H "Content-Type: application/json" -d '{"stock_delta":-30}'
```
```json
{ "error": { "code": "negative_stock",
             "message": "Stock for SKU 'CV-SLM15-BLK-SIL' cannot go below 0 (current 25, delta -30).",
             "details": [{ "field": "stock_delta", "issue": "result would be -5" }], "request_id": "9ab3dd02" } }
```

### 4.15 `DELETE /skus/:id`

| | |
|---|---|
| **Auth** | admin only |
| **Success** | `204 No Content` |
| **Errors** | `401` · `403` · `404` · `409 resource_in_use` (referenced by `order_items`, from Sprint 3) · `422 last_active_sku` |

### 4.16 Authorization failure examples

```bash
curl -i -X POST "$API/products" -H "Content-Type: application/json" -d '{"name":"x","category_id":2}'
# HTTP/1.1 401  {"error":{"code":"unauthenticated","message":"Missing or invalid access token."}}

curl -i -X POST "$API/products" -H "Authorization: Bearer <CUSTOMER_JWT>" \
  -H "Content-Type: application/json" -d '{"name":"x","category_id":2}'
# HTTP/1.1 403  {"error":{"code":"forbidden","message":"Administrator role required."}}
```

---

## 5. Data Integrity & Authorization Decisions

### 5.1 Principle: the database is the last line of defence

API validation gives friendly messages; **the database enforces the rule** so that no other client (a script, the Supabase REST endpoint, a future service) can bypass it (requirement CAT05).

| Rule | Enforced by (DB) | Surfaced by (API) |
|---|---|---|
| Unique category slug | `UNIQUE categories_slug_key` | `409 duplicate_slug` |
| Unique product slug | `UNIQUE products_slug_key` | `409 duplicate_slug` |
| Unique SKU code | `UNIQUE skus_code_key` + upper-case/format `CHECK` | `409 duplicate_sku_code` |
| No self-parent | `CHECK (parent_id <> id)` | `422 category_cycle` |
| No ancestor cycle | `BEFORE UPDATE` trigger, recursive ancestor walk (`CV001`) | `422 category_cycle` |
| Variant belongs to SKU's product | Composite FK `(variant_id, product_id)` | `404`/`409` |
| Variant uses only valid options/values | `BEFORE INSERT/UPDATE` trigger on `variants` (`CV003`) | `422 invalid_variant_options` |
| No duplicate combination | `UNIQUE (product_id, option_values)` | `409 duplicate_variant_combination` |
| No fake combinations | Absence of a row = unavailable (no zero-stock placeholder) | n/a |
| Money is exact | `price_minor integer CHECK (>= 0)`, `compare_at_minor > price_minor` | `400`/`422` |
| Stock never negative | `CHECK (stock_quantity >= 0)` | `422 negative_stock` |
| `active` product has ≥ 1 active SKU | Trigger on `products` (`CV004`) and `skus` (`CV005`) | `422 product_needs_active_sku`, `422 last_active_sku` |
| Sold SKUs cannot vanish | `order_items.sku_id ... ON DELETE RESTRICT` (Sprint 3) | `409 resource_in_use` |

The complete migration is in [Appendix A](#appendix-a--migration-sql).

### 5.2 Answers to the manual's business-rule questions

**1. Can a draft product have no SKU? Can a published product have no sellable SKU?**
A **draft may have zero SKUs** — admins build the product first (name, options, variants) and add SKUs after. A product **cannot be `active` without at least one active SKU**: a trigger rejects `status → 'active'` (and creating a product directly as `active`). The same rule is checked in reverse: deactivating or deleting the last active SKU of an `active` product is rejected (`last_active_sku`). Example from the seed: *Rugged Armor Case* is a `draft` with no SKUs; `PATCH {"status":"active"}` on it returns `422 product_needs_active_sku` (§4.9). Note "sellable" here means *an active SKU exists*, not *in stock* — an active product whose every SKU is out of stock is still `active` and displayed as sold out (see question 4).

**2. One canonical category, many categories, or both?**
**One canonical category now** (`products.category_id NOT NULL`), as suggested in Week 3. It gives every product exactly one owner for admin, breadcrumbs, and reporting. Alternate discovery paths (e.g. a MagSafe case also appearing under *Charging*) are a **many-to-many `product_categories` table added in Sprint 3**, together with public browsing, so no Sprint 2 data needs to be reshaped.

**3. What happens when a parent category is deactivated?**
**All descendants are deactivated** in the same transaction by an `AFTER UPDATE` trigger. Example: deactivating *Charging* (id 4) also deactivates *Wireless Chargers* (id 5). Products are **not** deleted or modified; they simply sit in inactive categories and will be excluded from public reads. Re-activation is deliberately *not* cascaded (an admin may have retired a child on purpose), and a child cannot be activated under an inactive parent (`CV002`, `422 inactive_parent`).

**4. How is an out-of-stock SKU represented in a public response?**
The SKU is **returned, marked unavailable**, rather than hidden: hiding it makes shoppers think the option never existed and hides demand signals (Week 3, slide 14). The public contract (built in Sprint 3 on these columns) never exposes the exact stock count, only a status:

```json
{ "sku_code": "CV-SLM15-BLK-LEA", "options": { "color": "Black", "material": "Leather" },
  "price_minor": 249900, "currency": "PKR",
  "availability": "out_of_stock", "purchasable": false }
```

`availability` is derived: `stock_quantity = 0` → `out_of_stock`; `1..5` → `low_stock` (supports the Sprint 1 "only 3 left" idea); otherwise `in_stock`. Inactive SKUs (`is_active = false`) are *omitted*. A **non-existent combination** (Navy/Leather) is also omitted — which is correct, because it is not a sold-out product, it is not a product.

**5. Can two SKUs share a price? Can a SKU have a price override?**
**Yes, SKUs may share a price** — no uniqueness on price (*Black/Silicone* and *Navy/Silicone* are both Rs. 1,499). Sprint 1's `price_override` on variants is **replaced**: every SKU carries its own explicit `price_minor` (there is no inherited product price to override), so the price of a sellable unit is never ambiguous. `compare_at_minor` supports a struck-through "was" price and must exceed the current price.

**6. What prevents negative stock and duplicate SKU codes?**
`CHECK (stock_quantity >= 0)` and `UNIQUE (code)` in the database. Stock changes via `stock_delta` are atomic `UPDATE ... SET stock_quantity = stock_quantity + $1`; a result below zero raises `23514`, which the API returns as `422 negative_stock`. Duplicate codes raise `23505`, returned as `409 duplicate_sku_code`. SKU codes are also constrained to upper-case `A-Z0-9-` so `cv-x` and `CV-X` cannot coexist as two codes. Both rules have DB-level tests that bypass the API (§7).

**7. What happens to a product referenced by a future cart or order after it is deactivated?**
- **Orders:** `order_items.sku_id` is `ON DELETE RESTRICT` and each order line stores **snapshots** (`sku_code_snapshot`, `product_name_snapshot`, `unit_price_minor`), so history is immutable even if the product is later renamed, repriced, or archived. A sold SKU can never be hard-deleted — admins **archive** (`status = 'archived'`) or deactivate instead.
- **Carts:** `cart_items` keep their `sku_id`. On cart read and at checkout the service re-checks `skus.is_active`, product `status = 'active'`, category active, and stock; offending lines are flagged `unavailable` and checkout rejects them. Nothing is silently deleted from a shopper's cart.

### 5.3 Authorization design

**Identity.** Supabase Auth issues the JWT. The admin API verifies the token with Supabase (`auth.getUser(token)`), then loads `profiles.role` for that `user_id`.

```
Request ──► requireAuth (401 if no/invalid token)
        ──► requireAdmin (403 if profiles.role <> 'admin')
        ──► validate body (400)
        ──► SQL via parameterised queries ──► DB constraints (409/422)
```

- Admin writes are only possible through this API using server-side credentials; the **service-role key never leaves the server** and is never committed.
- Every catalog table has **Row Level Security enabled**. Policies: anonymous/authenticated users may *read* only active rows (`categories.is_active`, `products.status = 'active'`, `skus.is_active`); only `is_admin()` may write. So even a direct call to Supabase's auto-generated REST endpoint with the public anon key **cannot** create or modify catalog data.
- The first admin is created by `npm run seed:admin` (reads `ADMIN_EMAIL` / `ADMIN_PASSWORD` from the environment; nothing is hard-coded).
- SQL is always parameterised; slugs and codes are format-validated twice (API + DB).

### 5.4 Status-code and PostgreSQL error mapping

| PostgreSQL | Meaning | HTTP | `error.code` (chosen by constraint name) |
|---|---|---|---|
| `23505` | unique violation | `409` | `duplicate_slug`, `duplicate_sku_code`, `duplicate_variant_combination`, `duplicate_option`, `duplicate_sku_pack` |
| `23514` | check violation | `422` | `negative_stock`, `invalid_price`, `invalid_slug`, `invalid_compare_at_price` |
| `23503` | foreign key violation | `409` / `404` | `resource_in_use` / `not_found` |
| `CV001` | category cycle | `422` | `category_cycle` |
| `CV002` | active child under inactive parent | `422` | `inactive_parent` |
| `CV003` | invalid variant option set | `422` | `invalid_variant_options` |
| `CV004` | product → `active` without active SKU | `422` | `product_needs_active_sku` |
| `CV005` | removing last active SKU of active product | `422` | `last_active_sku` |

### 5.5 What Sprint 3 may rely on

- A **SKU id** is the stable identity for cart lines, order lines, and wishlists.
- `skus.price_minor` is the only price; `skus.stock_quantity` the only stock. Sprint 3 must not duplicate either.
- Checkout decrements stock with `UPDATE skus SET stock_quantity = stock_quantity - $q WHERE id = $id AND stock_quantity >= $q` inside the order transaction; the `CHECK` guarantees correctness even under concurrent checkouts.

---

## 6. Seed Data & Demonstration Instructions

### 6.1 What the seed contains

**Category tree (3 levels):**

```
Accessories
├── Cases
├── Screen Protectors
└── Charging
    └── Wireless Chargers
```

**Products, variants, SKUs:**

| # | Product | Category | Status | Variants | SKUs |
|---|---|---|---|---|---|
| 1 | Slim Case for iPhone 15 | Cases | active | **3** (Black/Silicone, Navy/Silicone, Black/Leather) | `CV-SLM15-BLK-SIL` (25), `CV-SLM15-NVY-SIL` (18), `CV-SLM15-BLK-LEA` (**0**) |
| 2 | Tempered Glass for iPhone 15 | Screen Protectors | active | none | `CV-TG15-1PK` (60), `CV-TG15-2PK` (35) |
| 3 | MagSafe Charger 15W | Wireless Chargers | active | none | `CV-MSC15-STD` (12) |
| 4 | Rugged Armor Case for iPhone 15 | Cases | **draft** | none | none (shows the draft-without-SKU rule) |

- **Valid SKUs:** 6 (requirement: ≥ 4).
- **Intentionally unavailable combination:** *Navy + Leather* of the Slim Case — no variant row, no SKU row.
- **Out-of-stock but valid SKU:** `CV-SLM15-BLK-LEA`.
- Seed also inserts device model *Apple iPhone 15* and its compatibility rows (schema demonstration only).

The seed inserts products as `draft`, adds SKUs, **then** activates them — which proves the "active needs a SKU" trigger works on the real data path. Full script: [Appendix B](#appendix-b--seed-sql).

### 6.2 Reproduce on a clean database

```bash
supabase start                 # local Postgres + Auth
supabase db reset              # applies supabase/migrations/*.sql, then supabase/seed.sql
npm run seed:admin             # creates the admin auth user + profiles.role = 'admin'
npm run dev                    # starts the admin API on http://localhost:3000
```

`supabase db reset` drops and recreates the database, so the same demonstration is reproducible on any clean machine.

### 6.3 Demonstration script — administrator creates category → product → variant → SKU

> **TEAM TODO:** run the commands below on a clean database, paste your **captured** responses under each step (redact tokens and private URLs), and add a screenshot if desired. The expected shapes are shown so you can compare.

```bash
export API=http://localhost:3000/api/v1/admin
# Obtain an admin access token (redacted in the submission):
export TOKEN="<ADMIN_JWT>"
H=(-H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json")
```

**Step 1 — create a category**
```bash
curl -s -X POST "$API/categories" "${H[@]}" \
  -d '{"name":"Car Mounts","slug":"car-mounts","parent_id":1}'
```
Expected: `201`, `data.slug = "car-mounts"`, `data.parent_id = 1`.

**Step 2 — create a draft product**
```bash
curl -s -X POST "$API/products" "${H[@]}" \
  -d '{"name":"Vent Phone Mount","category_id":6,"description":"Clips to any air vent."}'
```
Expected: `201`, `data.status = "draft"`.

**Step 3 — define an option and a variant**
```bash
curl -s -X POST "$API/products/5/options"  "${H[@]}" -d '{"name":"color","allowed_values":["Black","Silver"]}'
curl -s -X POST "$API/products/5/variants" "${H[@]}" -d '{"option_values":{"color":"Black"}}'
```
Expected: `201`, `data.option_values = {"color":"Black"}`.

**Step 4 — add a SKU**
```bash
curl -s -X POST "$API/products/5/skus" "${H[@]}" \
  -d '{"code":"CV-VNT-BLK","variant_id":4,"price_minor":89900,"stock_quantity":30}'
```
Expected: `201`, `data.code = "CV-VNT-BLK"`, `data.price_minor = 89900`.

**Step 5 — publish, then retrieve through the admin API**
```bash
curl -s -X PATCH "$API/products/5" "${H[@]}" -d '{"status":"active"}'
curl -s "$API/products/5" -H "Authorization: Bearer $TOKEN"
curl -s "$API/products?status=active"  -H "Authorization: Bearer $TOKEN"
curl -s "$API/categories"              -H "Authorization: Bearer $TOKEN"
```
Expected: product `status = "active"` with its option, variant, and SKU nested; category tree shows *Car Mounts* under *Accessories*.

**Step 6 — rejection paths (for the demo)**
```bash
# duplicate SKU code -> 409
curl -s -X POST "$API/products/5/skus" "${H[@]}" -d '{"code":"CV-VNT-BLK","price_minor":89900}'
# negative stock -> 422
curl -s -X PATCH "$API/skus/7" "${H[@]}" -d '{"stock_delta":-999}'
# no token -> 401
curl -s -o /dev/null -w "%{http_code}\n" -X POST "$API/products" -d '{}'
```

**Captured evidence** — `TEAM TODO: paste responses / screenshots here`

```text
(step 1 response)
(step 2 response)
(step 3 responses)
(step 4 response)
(step 5 responses)
(step 6 responses)
```

---

## 7. Test Strategy, Command & Result

### 7.1 Strategy

- **Framework:** Jest + Supertest (API layer) and direct SQL through `pg` (database layer), against a local Supabase Postgres reset to the migration + seed state before the run.
- **Two layers on purpose.** *DB tests* insert/update with raw SQL to prove the rule holds **without the API** (CAT05). *API tests* prove the rule is translated into the documented status code and error shape.
- **Every business rule has at least one failure-path test**, as the sprint checklist requires.
- Screenshots are supplementary only; they do not replace any test below.

### 7.2 Test inventory

| ID | Layer | Rule / requirement | Type | Expected |
|---|---|---|---|---|
| T01 | API | Create category with required fields (CAT01) | success | `201` |
| T02 | API | Create child category (2-level tree) | success | `201`, `parent_id` set |
| T03 | API | Category tree endpoint returns nested children | success | `200`, nesting correct |
| T04 | API | Duplicate category slug | failure | `409 duplicate_slug` |
| T05 | DB | Duplicate category slug via raw SQL | failure | `23505` |
| T06 | DB | Category cannot be its own parent | failure | `23514` |
| T07 | API | Re-parent root under its descendant (cycle) | failure | `422 category_cycle` |
| T08 | DB | Cycle via raw `UPDATE` bypassing API | failure | `CV001` |
| T09 | API | Deactivating parent deactivates descendants | success | children `is_active = false` |
| T10 | DB | Activating child under inactive parent | failure | `CV002` |
| T11 | API | Create product with required fields → draft (CAT02) | success | `201`, `status = draft` |
| T12 | API | Create product missing `name`/`category_id` | failure | `400 validation_failed` |
| T13 | API | Duplicate product slug | failure | `409 duplicate_slug` |
| T14 | DB | Duplicate product slug via raw SQL | failure | `23505` |
| T15 | API | Create product with unknown category | failure | `404 not_found` |
| T16 | API | Draft product may exist with zero SKUs | success | `200`, `skus = []` |
| T17 | API | Publish product with no active SKU | failure | `422 product_needs_active_sku` |
| T18 | DB | Insert product directly as `active` | failure | `CV004` |
| T19 | API | Add SKU with required fields (CAT03) | success | `201` |
| T20 | API | Duplicate SKU code | failure | `409 duplicate_sku_code` |
| T21 | DB | Duplicate / case-variant SKU code via raw SQL | failure | `23505` / `23514` |
| T22 | API | SKU price as float or negative | failure | `400 validation_failed` |
| T23 | DB | `price_minor < 0`; `compare_at_minor <= price_minor` | failure | `23514` |
| T24 | API | Create valid variant combination (CAT04) | success | `201` |
| T25 | API | Variant with unallowed option value | failure | `422 invalid_variant_options` |
| T26 | API | Variant missing a required option | failure | `422 invalid_variant_options` |
| T27 | API | Duplicate variant combination | failure | `409 duplicate_variant_combination` |
| T28 | DB | SKU whose `variant_id` belongs to a different product | failure | `23503` |
| T29 | Seed | Navy/Leather has no variant and no SKU; Black/Leather exists with stock 0 | success | assertion on seed data |
| T30 | API | Update SKU price and stock | success | `200` |
| T31 | API | `stock_delta` leading below zero | failure | `422 negative_stock` |
| T32 | DB | `UPDATE skus SET stock_quantity = -1` | failure | `23514` |
| T33 | API | Deactivate last active SKU of active product | failure | `422 last_active_sku` |
| T34 | API | Archive product (soft delete) keeps data | success | `200`, SKUs intact |
| T35 | API | Unauthenticated request to every admin route | failure | `401` |
| T36 | API | Customer-role token on every admin write route | failure | `403` |
| T37 | API | Expired/garbage token | failure | `401` |
| T38 | DB | Anonymous role cannot `INSERT` into `products`/`skus` (RLS) | failure | permission/RLS error |
| T39 | API | Errors never expose stack traces (all error responses match envelope) | failure | envelope shape |
| T40 | Seed | Seed reproduces: ≥2 category levels, ≥3 products, ≥4 SKUs | success | counts match |

### 7.3 Command

```bash
npm test
# package.json:  "test": "supabase db reset && jest --runInBand --verbose"
```

### 7.4 Result

> **TEAM TODO:** run `npm test` and paste the **real** output here. Do not edit it.

```text
Test Suites: __ passed, __ total
Tests:       __ passed, __ total
Time:        __ s
```

Coverage of rules: categories (T01–T10), products (T11–T18), SKUs (T19–T23, T30–T34), variants/combinations (T24–T29), authorization (T35–T38), error hygiene (T39), seed (T29, T40).

---

## 8. Known Limitations & Sprint 3 Backlog

### 8.1 Known limitations

| Limitation | Impact / mitigation |
|---|---|
| Single canonical category per product | Alternate discovery paths deferred (Sprint 3 `product_categories`). |
| `specs` validated only as a JSON object | No per-category key validation or indexing yet. |
| `assets`, `device_models`, `product_compatibility` have no API | Schema only; seeded minimally. |
| Admin API supports no bulk import | Single-record operations only. |
| Cart/order tables are not created | Foreign keys to `skus` are documented in §3 and will be added in Sprint 3. |
| Option values compare by exact string (`"Black"` ≠ `"black"`) | Validated against `allowed_values`, so only the declared spelling is accepted. |
| Reactivating a category does not reactivate children | Intentional; documented in §5.2 Q3. |
| Stock history/audit log is not kept | Only the current quantity is stored. |

### 8.2 Sprint 3 backlog (in priority order)

1. **Public catalog reads** — active products/SKUs with derived `availability`, hiding inactive categories, products, and SKUs.
2. **Device-compatibility filter** — admin CRUD for `device_models` / `product_compatibility`, plus the "select your device" query (Sprint 1 domain feature).
3. **Dynamic specifications** — per-category key validation for `products.specs`, GIN index for filtering.
4. **Assets** — upload pipeline (validate type/size → store → link to product/variant), roles, alt text, ordering.
5. **Publication rules** — extend the `draft → active → archived` states.
6. **Many-to-many `product_categories`** for alternate discovery paths.
7. **Cart readiness** — `cart`, `cart_items(sku_id)`, `wishlist_items(sku_id)`; cart revalidation of price/stock/active flags.
8. **Checkout** — Stripe test mode, `orders`/`order_items(sku_id RESTRICT)` with snapshots, atomic stock decrement.
9. **Realtime stock** — Supabase Realtime subscription on `skus.stock_quantity`.

### 8.3 Sprint review checklist

- [x] A reviewer can distinguish product, variant, and SKU in the database and the admin API (§3.4, §4.7).
- [x] Money, stock, slugs, and SKU codes are protected by constraints (§5.1).
- [x] The seed command reproduces the demonstration on a clean database (§6.2).
- [x] Tests exist for at least one failure path of every major rule (§7.2).
- [x] The document explains what Sprint 3 can safely build on (§5.5, §8.2).

---

## Appendix A — Migration SQL

`supabase/migrations/0001_catalog_foundation.sql`

```sql
-- =====================================================================
-- Sprint 2: Catalog data foundation
-- Requires PostgreSQL 15+ (ON DELETE SET NULL (column)). Supabase: OK.
-- Custom SQLSTATEs: CV001 category_cycle, CV002 inactive_parent,
--   CV003 invalid_variant_options, CV004 product_needs_active_sku,
--   CV005 last_active_sku
-- =====================================================================

-- ---------- shared helpers -------------------------------------------
create or replace function public.set_updated_at() returns trigger
language plpgsql as $$
begin
  new.updated_at := now();
  return new;
end $$;

-- ---------- profiles (authorization) ---------------------------------
create table public.profiles (
  user_id    uuid primary key
             references auth.users(id) on delete cascade on update restrict,
  role       text not null default 'customer'
             check (role in ('customer', 'admin')),
  created_at timestamptz not null default now()
);

create or replace function public.is_admin() returns boolean
language sql stable security definer set search_path = public as $$
  select exists (
    select 1 from public.profiles
    where user_id = auth.uid() and role = 'admin'
  );
$$;

-- ---------- categories -----------------------------------------------
create table public.categories (
  id          bigint generated always as identity primary key,
  parent_id   bigint references public.categories(id)
              on delete restrict on update restrict,
  name        text not null check (char_length(btrim(name)) between 1 and 80),
  slug        text not null,
  sort_order  integer not null default 0,
  is_active   boolean not null default true,
  created_at  timestamptz not null default now(),
  updated_at  timestamptz not null default now(),
  constraint categories_slug_key unique (slug),
  constraint categories_slug_format
    check (slug ~ '^[a-z0-9]+(-[a-z0-9]+)*$'),
  constraint categories_not_own_parent
    check (parent_id is null or parent_id <> id)
);
create index categories_parent_id_idx on public.categories(parent_id);

create or replace function public.categories_guard() returns trigger
language plpgsql as $$
begin
  if new.parent_id is not null then
    -- CV001: the new parent must not be a descendant of this category
    if tg_op = 'UPDATE' and exists (
      with recursive ancestors(id, parent_id) as (
        select c.id, c.parent_id from public.categories c where c.id = new.parent_id
        union
        select c.id, c.parent_id
        from public.categories c join ancestors a on c.id = a.parent_id
      )
      select 1 from ancestors where id = new.id
    ) then
      raise exception 'category % cannot become its own ancestor', new.id
        using errcode = 'CV001';
    end if;

    -- CV002: an active category cannot sit under an inactive parent
    if new.is_active and exists (
      select 1 from public.categories p
      where p.id = new.parent_id and not p.is_active
    ) then
      raise exception 'category % cannot be active under an inactive parent', new.id
        using errcode = 'CV002';
    end if;
  end if;
  return new;
end $$;

create trigger categories_guard_trg
  before insert or update of parent_id, is_active on public.categories
  for each row execute function public.categories_guard();

create or replace function public.categories_cascade_deactivate() returns trigger
language plpgsql as $$
begin
  if old.is_active and not new.is_active then
    with recursive descendants(id) as (
      select c.id from public.categories c where c.parent_id = new.id
      union
      select c.id from public.categories c join descendants d on c.parent_id = d.id
    )
    update public.categories
       set is_active = false
     where id in (select id from descendants) and is_active;
  end if;
  return null;
end $$;

create trigger categories_cascade_deactivate_trg
  after update of is_active on public.categories
  for each row execute function public.categories_cascade_deactivate();

create trigger categories_updated_at before update on public.categories
  for each row execute function public.set_updated_at();

-- ---------- products -------------------------------------------------
create table public.products (
  id           bigint generated always as identity primary key,
  category_id  bigint not null references public.categories(id)
               on delete restrict on update restrict,
  name         text not null check (char_length(btrim(name)) between 1 and 160),
  slug         text not null,
  description  text not null default '',
  status       text not null default 'draft'
               check (status in ('draft', 'active', 'archived')),
  specs        jsonb not null default '{}'::jsonb,
  created_at   timestamptz not null default now(),
  updated_at   timestamptz not null default now(),
  constraint products_slug_key unique (slug),
  constraint products_slug_format
    check (slug ~ '^[a-z0-9]+(-[a-z0-9]+)*$'),
  constraint products_specs_is_object
    check (jsonb_typeof(specs) = 'object')
);
create index products_category_id_idx on public.products(category_id);
create index products_status_idx on public.products(status);

create trigger products_updated_at before update on public.products
  for each row execute function public.set_updated_at();

-- ---------- product_options & variants -------------------------------
create table public.product_options (
  id              bigint generated always as identity primary key,
  product_id      bigint not null references public.products(id)
                  on delete cascade on update restrict,
  name            text not null check (name ~ '^[a-z][a-z0-9_]*$'),
  position        smallint not null default 0,
  allowed_values  text[] not null check (cardinality(allowed_values) >= 1),
  constraint product_options_product_name_key unique (product_id, name)
);

create table public.variants (
  id             bigint generated always as identity primary key,
  product_id     bigint not null references public.products(id)
                 on delete cascade on update restrict,
  option_values  jsonb not null check (jsonb_typeof(option_values) = 'object'),
  created_at     timestamptz not null default now(),
  updated_at     timestamptz not null default now(),
  constraint variants_id_product_key unique (id, product_id),
  constraint variants_unique_combination unique (product_id, option_values)
);

-- CV003: a variant must set every option of its product, with allowed values only
create or replace function public.validate_variant_options() returns trigger
language plpgsql as $$
declare
  opt record;
  val text;
  opt_count integer;
  key_count integer;
begin
  select count(*) into opt_count
    from public.product_options where product_id = new.product_id;
  select count(*) into key_count from jsonb_object_keys(new.option_values);

  if key_count <> opt_count then
    raise exception 'variant must define exactly the % option(s) of product %',
      opt_count, new.product_id using errcode = 'CV003';
  end if;

  for opt in
    select name, allowed_values from public.product_options
    where product_id = new.product_id
  loop
    val := new.option_values ->> opt.name;
    if val is null or not (val = any (opt.allowed_values)) then
      raise exception 'value % is not allowed for option %', val, opt.name
        using errcode = 'CV003';
    end if;
  end loop;
  return new;
end $$;

create trigger variants_validate_options_trg
  before insert or update of option_values, product_id on public.variants
  for each row execute function public.validate_variant_options();

create trigger variants_updated_at before update on public.variants
  for each row execute function public.set_updated_at();

-- ---------- skus -----------------------------------------------------
create table public.skus (
  id                bigint generated always as identity primary key,
  product_id        bigint not null,
  variant_id        bigint,
  code              text not null,
  pack_size         smallint not null default 1 check (pack_size > 0),
  price_minor       integer not null check (price_minor >= 0),
  compare_at_minor  integer,
  currency          text not null default 'PKR' check (currency ~ '^[A-Z]{3}$'),
  stock_quantity    integer not null default 0,
  is_active         boolean not null default true,
  created_at        timestamptz not null default now(),
  updated_at        timestamptz not null default now(),
  constraint skus_code_key unique (code),
  constraint skus_code_format check (code ~ '^[A-Z0-9]+(-[A-Z0-9]+)*$'),
  constraint skus_stock_non_negative check (stock_quantity >= 0),
  constraint skus_compare_at_gt_price
    check (compare_at_minor is null or compare_at_minor > price_minor),
  constraint skus_product_fk foreign key (product_id)
    references public.products(id) on delete cascade on update restrict,
  -- composite FK: the variant must belong to the SAME product (skipped when variant_id is NULL)
  constraint skus_variant_same_product_fk foreign key (variant_id, product_id)
    references public.variants(id, product_id) on delete cascade on update restrict
);
create unique index skus_variant_pack_key
  on public.skus(variant_id, pack_size) where variant_id is not null;
create unique index skus_product_pack_no_variant_key
  on public.skus(product_id, pack_size) where variant_id is null;
create index skus_product_id_idx on public.skus(product_id);

create trigger skus_updated_at before update on public.skus
  for each row execute function public.set_updated_at();

-- CV004: a product can only be 'active' if it has at least one active SKU
create or replace function public.products_require_sku_to_activate() returns trigger
language plpgsql as $$
begin
  if new.status = 'active'
     and (tg_op = 'INSERT' or old.status is distinct from 'active') then
    if tg_op = 'INSERT' or not exists (
      select 1 from public.skus s where s.product_id = new.id and s.is_active
    ) then
      raise exception 'product % needs at least one active SKU to be active', new.id
        using errcode = 'CV004';
    end if;
  end if;
  return new;
end $$;

create trigger products_require_sku_trg
  before insert or update of status on public.products
  for each row execute function public.products_require_sku_to_activate();

-- CV005: the last active SKU of an active product cannot be removed/deactivated
create or replace function public.skus_guard_last_active() returns trigger
language plpgsql as $$
declare
  pid bigint := old.product_id;
begin
  if (tg_op = 'DELETE' and old.is_active)
     or (tg_op = 'UPDATE' and old.is_active and not new.is_active) then
    if exists (select 1 from public.products p where p.id = pid and p.status = 'active')
       and not exists (
         select 1 from public.skus s
         where s.product_id = pid and s.is_active and s.id <> old.id
       ) then
      raise exception 'SKU % is the last active SKU of an active product', old.id
        using errcode = 'CV005';
    end if;
  end if;
  return null;
end $$;

create trigger skus_guard_last_active_trg
  after update of is_active or delete on public.skus
  for each row execute function public.skus_guard_last_active();

-- ---------- assets (schema only in Sprint 2) -------------------------
create table public.assets (
  id           bigint generated always as identity primary key,
  product_id   bigint not null,
  variant_id   bigint,
  storage_key  text not null,
  role         text not null check (role in ('hero', 'gallery', 'detail', 'swatch')),
  alt_text     text not null check (char_length(btrim(alt_text)) > 0),
  sort_order   integer not null default 0,
  created_at   timestamptz not null default now(),
  constraint assets_storage_key_key unique (storage_key),
  constraint assets_product_fk foreign key (product_id)
    references public.products(id) on delete cascade on update restrict,
  constraint assets_variant_same_product_fk foreign key (variant_id, product_id)
    references public.variants(id, product_id)
    on delete set null (variant_id) on update restrict
);

-- ---------- device compatibility (retained from Sprint 1; schema only) ----
create table public.device_models (
  id            bigint generated always as identity primary key,
  brand         text not null,
  model_name    text not null,
  release_year  smallint,
  constraint device_models_brand_model_key unique (brand, model_name)
);

create table public.product_compatibility (
  id               bigint generated always as identity primary key,
  product_id       bigint not null references public.products(id)
                   on delete cascade on update restrict,
  device_model_id  bigint not null references public.device_models(id)
                   on delete restrict on update restrict,
  constraint product_compatibility_key unique (product_id, device_model_id)
);

-- ---------- Row Level Security (defense in depth) --------------------
alter table public.profiles enable row level security;
create policy profiles_read_own on public.profiles
  for select to authenticated using (user_id = auth.uid());

do $$
declare t text;
begin
  foreach t in array array[
    'categories','products','product_options','variants','skus',
    'assets','device_models','product_compatibility'
  ] loop
    execute format('alter table public.%I enable row level security', t);
    execute format(
      'create policy %I on public.%I for all to authenticated
         using (public.is_admin()) with check (public.is_admin())',
      t || '_admin_all', t);
  end loop;
end $$;

-- public (anon + authenticated) read access: active rows only
create policy categories_public_read on public.categories
  for select to anon, authenticated using (is_active);
create policy products_public_read on public.products
  for select to anon, authenticated using (status = 'active');
create policy product_options_public_read on public.product_options
  for select to anon, authenticated using (
    exists (select 1 from public.products p where p.id = product_id and p.status = 'active'));
create policy variants_public_read on public.variants
  for select to anon, authenticated using (
    exists (select 1 from public.products p where p.id = product_id and p.status = 'active'));
create policy skus_public_read on public.skus
  for select to anon, authenticated using (
    is_active and exists (select 1 from public.products p where p.id = product_id and p.status = 'active'));
create policy assets_public_read on public.assets
  for select to anon, authenticated using (
    exists (select 1 from public.products p where p.id = product_id and p.status = 'active'));
create policy device_models_public_read on public.device_models
  for select to anon, authenticated using (true);
create policy product_compatibility_public_read on public.product_compatibility
  for select to anon, authenticated using (
    exists (select 1 from public.products p where p.id = product_id and p.status = 'active'));

-- ---------- Planned for Sprint 3 (documented, NOT created here) ------
-- cart_items.sku_id       -> skus(id)  ON DELETE CASCADE  ON UPDATE RESTRICT
-- wishlist_items.sku_id   -> skus(id)  ON DELETE CASCADE  ON UPDATE RESTRICT
-- order_items.sku_id      -> skus(id)  ON DELETE RESTRICT ON UPDATE RESTRICT
-- orders.user_id          -> auth.users(id) ON DELETE RESTRICT ON UPDATE RESTRICT
```

---

## Appendix B — Seed SQL

`supabase/seed.sql` (runs automatically after migrations on `supabase db reset`; it is not meant to be re-run on a populated database)

```sql
-- ---------- category tree (3 levels) ---------------------------------
insert into categories (parent_id, name, slug, sort_order)
values (null, 'Accessories', 'accessories', 1);

insert into categories (parent_id, name, slug, sort_order)
select id, 'Cases', 'cases', 1 from categories where slug = 'accessories';
insert into categories (parent_id, name, slug, sort_order)
select id, 'Screen Protectors', 'screen-protectors', 2 from categories where slug = 'accessories';
insert into categories (parent_id, name, slug, sort_order)
select id, 'Charging', 'charging', 3 from categories where slug = 'accessories';
insert into categories (parent_id, name, slug, sort_order)
select id, 'Wireless Chargers', 'wireless-chargers', 1 from categories where slug = 'charging';

-- ---------- products (all start as draft) ----------------------------
insert into products (category_id, name, slug, description, specs)
select id, 'Slim Case for iPhone 15', 'slim-case-for-iphone-15',
       'Slim, grippy protection.', '{"magsafe": false, "drop_protection_m": 1.5}'
from categories where slug = 'cases';
insert into products (category_id, name, slug, description, specs)
select id, 'Tempered Glass for iPhone 15', 'tempered-glass-for-iphone-15',
       '9H hardness, oleophobic coating.', '{"hardness": "9H"}'
from categories where slug = 'screen-protectors';
insert into products (category_id, name, slug, description, specs)
select id, 'MagSafe Charger 15W', 'magsafe-charger-15w',
       'Magnetic 15W wireless charger.', '{"power_w": 15, "magsafe": true}'
from categories where slug = 'wireless-chargers';
insert into products (category_id, name, slug, description)
select id, 'Rugged Armor Case for iPhone 15', 'rugged-armor-case-for-iphone-15',
       'Heavy-duty protection (draft, SKUs pending).'
from categories where slug = 'cases';

-- ---------- options & variants for the multi-variant product ---------
insert into product_options (product_id, name, position, allowed_values)
select id, 'color', 1, array['Black', 'Navy'] from products where slug = 'slim-case-for-iphone-15';
insert into product_options (product_id, name, position, allowed_values)
select id, 'material', 2, array['Silicone', 'Leather'] from products where slug = 'slim-case-for-iphone-15';

-- Navy + Leather is intentionally NOT inserted: that combination is not produced.
insert into variants (product_id, option_values)
select id, v::jsonb from products,
  (values ('{"color":"Black","material":"Silicone"}'),
          ('{"color":"Navy","material":"Silicone"}'),
          ('{"color":"Black","material":"Leather"}')) as t(v)
where slug = 'slim-case-for-iphone-15';

-- ---------- SKUs ------------------------------------------------------
insert into skus (product_id, variant_id, code, price_minor, stock_quantity)
select p.id, v.id, x.code, x.price, x.stock
from products p
join variants v on v.product_id = p.id
join (values
  ('{"color":"Black","material":"Silicone"}', 'CV-SLM15-BLK-SIL', 149900, 25),
  ('{"color":"Navy","material":"Silicone"}',  'CV-SLM15-NVY-SIL', 149900, 18),
  ('{"color":"Black","material":"Leather"}',  'CV-SLM15-BLK-LEA', 249900, 0)   -- valid but sold out
) as x(opts, code, price, stock) on v.option_values = x.opts::jsonb
where p.slug = 'slim-case-for-iphone-15';

insert into skus (product_id, code, pack_size, price_minor, stock_quantity)
select id, 'CV-TG15-1PK', 1, 79900, 60 from products where slug = 'tempered-glass-for-iphone-15';
insert into skus (product_id, code, pack_size, price_minor, stock_quantity)
select id, 'CV-TG15-2PK', 2, 129900, 35 from products where slug = 'tempered-glass-for-iphone-15';
insert into skus (product_id, code, price_minor, compare_at_minor, stock_quantity)
select id, 'CV-MSC15-STD', 349900, 399900, 12 from products where slug = 'magsafe-charger-15w';

-- ---------- publish (allowed now: each has an active SKU) -----------
-- 'rugged-armor-case-for-iphone-15' stays a draft with no SKU on purpose.
update products set status = 'active'
where slug in ('slim-case-for-iphone-15', 'tempered-glass-for-iphone-15', 'magsafe-charger-15w');

-- ---------- device compatibility (schema demonstration) --------------
insert into device_models (brand, model_name, release_year) values ('Apple', 'iPhone 15', 2023);
insert into product_compatibility (product_id, device_model_id)
select p.id, d.id from products p, device_models d
where d.model_name = 'iPhone 15'
  and p.slug in ('slim-case-for-iphone-15','tempered-glass-for-iphone-15',
                 'magsafe-charger-15w','rugged-armor-case-for-iphone-15');
```

`npm run seed:admin` (a short Node script) creates the demo admin through the Supabase Admin API using `ADMIN_EMAIL` / `ADMIN_PASSWORD` from `.env`, then upserts `profiles(user_id, role = 'admin')`. Credentials are never stored in the repository.

---

## Appendix C — README Section (Local Setup)

Add this to the repository `README.md`:

````markdown
## Sprint 2 — Catalog Admin: Local Setup

**Prerequisites:** Node.js 20+, Docker, [Supabase CLI](https://supabase.com/docs/guides/cli).

```bash
git clone <repo-url> && cd casevault
cp .env.example .env        # fill in values; never commit .env
npm install
supabase start              # local Postgres + Auth (prints keys)
supabase db reset           # runs migrations + seed.sql
npm run seed:admin          # creates the admin user
npm run dev                 # http://localhost:3000
npm test                    # resets DB, runs Jest (DB + API tests)
```

### Environment variables (`.env.example`)

| Variable | Purpose |
|---|---|
| `PORT` | API port (default `3000`) |
| `SUPABASE_URL` | Supabase project/API URL (`http://127.0.0.1:54321` locally) |
| `SUPABASE_ANON_KEY` | Public key, used to verify user tokens |
| `SUPABASE_SERVICE_ROLE_KEY` | **Server only.** Admin operations. Never expose or commit |
| `DATABASE_URL` | Postgres connection string (`postgresql://postgres:postgres@127.0.0.1:54322/postgres` locally) |
| `ADMIN_EMAIL` / `ADMIN_PASSWORD` | Used only by `npm run seed:admin` |
| `TEST_CUSTOMER_EMAIL` / `TEST_CUSTOMER_PASSWORD` | Non-admin user used by authorization tests |

`.env` is listed in `.gitignore`; only `.env.example` (placeholders) is committed.
````

---

## Appendix D — Repository Layout

```
casevault/
├── docs/
│   ├── SPRINT_1.md
│   └── SPRINT_2.md                   <- this document
├── supabase/
│   ├── migrations/0001_catalog_foundation.sql
│   └── seed.sql
├── src/
│   ├── app.js                        # Express app, route mounting
│   ├── db.js                         # pg Pool (parameterised queries)
│   ├── middleware/
│   │   ├── requireAuth.js            # 401
│   │   ├── requireAdmin.js           # 403
│   │   └── errorHandler.js           # PG error -> envelope mapping (§5.4)
│   ├── validation/                   # request validators (400)
│   └── routes/admin/
│       ├── categories.js
│       ├── products.js
│       └── skus.js
├── scripts/seedAdmin.js
├── tests/
│   ├── db/        # raw-SQL constraint tests (T05, T06, T08, T10, ...)
│   └── api/       # Supertest route tests
├── .env.example
├── .gitignore                        # includes .env
└── README.md
```

Commit style: small, reviewable commits per checkpoint (e.g. `feat(db): categories table + cycle guard`, `feat(api): POST /admin/skus`, `test: duplicate SKU rejection`).
