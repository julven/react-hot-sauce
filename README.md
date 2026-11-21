# Ember & Brine — Hot Sauce Storefront

A single-file React storefront for a hot sauce brand, built to run inside a **GoHighLevel (GHL) custom HTML block** with no build step required. Everything ships in one `index.html`.

## Overview

The app renders five views via React Router's `HashRouter`, so the URL never changes from GHL's perspective — only the `#` fragment does. This means refreshes and shared links always work on a single GHL page.

| Route | View |
|---|---|
| `#/` | Home — hero, best sellers, hottest, heat picker, process, bundle, reviews, FAQ |
| `#/sauces` | All sauces with real-time search and heat filter (shareable URL) |
| `#/sauce/:id` | Single sauce — full details, quantity stepper, total price, "you may also like" |
| `#/checkout` | Order summary + Stripe card form |
| `#/orders` | Look up an order by its UUID reference key |
| `#/orders/:orderKey` | Order detail — shareable tracking link |

## Tech Stack

| Library | Version | How loaded |
|---|---|---|
| React | 18.3.1 | jsDelivr CDN |
| React Router DOM | 6.26.2 | jsDelivr CDN |
| Babel Standalone | 7.24.7 | jsDelivr CDN (transpiles JSX in-browser) |
| Bootstrap | 5.3.3 | jsDelivr CDN (CSS only) |
| Stripe.js | latest | Loaded on demand on the checkout page |
| Google Fonts — Bricolage Grotesque | — | Google Fonts CDN |

No `npm install`, no bundler, no build step.

## Configuration

All settings live in the `CONFIG` object near the top of the `<script>` block:

```js
const CONFIG = {
  API_URL:   "https://…/api/hotsauce/top",      // best-sellers + hottest (loads on page open)
  ALL_URL:   "https://…/api/hotsauce",           // full list (loads when /sauces opens)
  ITEM_URL:  "https://…/api/hotsauce/id/:id",    // single sauce; :id is replaced at runtime
  MAX_QTY:   99,                                  // max bottles per sauce per order
  SHIPPING_FLAT: 6.95,                            // shipping fee below FREE_SHIP_AT
  FREE_SHIP_AT:  35,                              // cart total that unlocks free shipping
  CURRENCY:  "$",

  STRIPE_PUBLISHABLE_KEY: "pk_test_…",           // safe to keep in page code (never sk_…)
  ORDER_URL:         "https://…/api/hotsauce-order",           // POST to save + charge
  ORDER_CONFIRM_URL: "",                          // optional; for 3D Secure post-auth callback
  ORDER_LOOKUP_URL:  "https://…/api/hotsauce-order/:order_key",// GET for order tracking
  DEBUG_CHECKOUT: true,                           // logs checkout steps to browser console

  BUNDLE: { id: "trio", name: "The Trio", price: 31, … },
  CART_KEY:       "eb-cart",                      // localStorage key for quantities
  CART_ITEMS_KEY: "eb-cart-items",                // localStorage key for item objects
};
```

Set `DEBUG_CHECKOUT: false` before going live.

## How to Deploy in GoHighLevel

1. Open your GHL funnel/website page.
2. Add a full-width section and drop in a **Custom JS/HTML** (code) element.
3. Paste the entire contents of `index.html`.
4. Update the `CONFIG` values (API URLs, Stripe key, shipping).
5. Publish.

> **Bootstrap caveat:** Bootstrap's CSS resets apply to the whole GHL page. For best results, build the page from this block alone and don't mix other GHL section types.

## API Contract

### `GET API_URL` — Top sauces
Returns:
```json
{ "best_selling": [ …sauce objects… ], "hottest": [ …sauce objects… ] }
```

### `GET ALL_URL` — All sauces
Returns a plain array `[…]` or `{ "data": […] }`.

### `GET ITEM_URL` (`:id` replaced) — Single sauce
Returns `{ …sauce object… }` or `{ "data": { …sauce object… } }`. A `404`/`400` renders the "not found" page.

### Sauce object fields used by the app

| Field | Type | Notes |
|---|---|---|
| `id` | number/string | Used as `dbId` for API calls and routing |
| `slug` | string | Used as cart key if present |
| `name` | string | |
| `description` | string | |
| `heat_level` | 1–5 | Clamped to range |
| `price_cents` | number | Divided by 100 |
| `color_hex` | string | Bottle SVG fill |
| `bg_hex` | string | Card background |
| `image_url` | string | Falls back to generated SVG bottle |
| `buy_url` | string | External link, skips cart |
| `stock_qty` | number/null | `null` = unlimited |
| `size_oz` | number | Optional |
| `main_peppers` | string[] | Optional |
| `ingredients` | string | Optional |
| `ferment_days` | number | Optional |
| `is_vegan` | boolean | |
| `is_gluten_free` | boolean | |
| `is_active` | boolean | `false` rows are filtered out |

## Database Schema

Table: `hotsauce` (PostgreSQL)

| Column | Type | Constraints |
|---|---|---|
| `id` | `BIGINT` | `PRIMARY KEY`, `GENERATED ALWAYS AS IDENTITY` |
| `slug` | `TEXT` | `UNIQUE NOT NULL` |
| `name` | `TEXT` | `NOT NULL` |
| `description` | `TEXT` | |
| `main_peppers` | `TEXT[]` | `NOT NULL DEFAULT '{}'` |
| `ingredients` | `TEXT` | |
| `heat_level` | `SMALLINT` | `NOT NULL` |
| `price_cents` | `INTEGER` | `NOT NULL` |
| `currency` | `CHAR(3)` | `NOT NULL DEFAULT 'USD'` |
| `size_oz` | `NUMERIC(5,2)` | `NOT NULL DEFAULT 5.00` |
| `stock_qty` | `INTEGER` | `NOT NULL DEFAULT 0` |
| `is_vegan` | `BOOLEAN` | `NOT NULL DEFAULT true` |
| `is_gluten_free` | `BOOLEAN` | `NOT NULL DEFAULT true` |
| `ferment_days` | `SMALLINT` | |
| `color_hex` | `CHAR(7)` | |
| `bg_hex` | `CHAR(7)` | |
| `image_url` | `TEXT` | |
| `buy_url` | `TEXT` | |
| `is_active` | `BOOLEAN` | `NOT NULL DEFAULT true` |
| `created_at` | `TIMESTAMP WITH TIME ZONE` | `NOT NULL DEFAULT now()` |
| `updated_at` | `TIMESTAMP WITH TIME ZONE` | `NOT NULL DEFAULT now()` |

### `POST ORDER_URL` — Place order
See the `handleSubmit` comment in the `CheckoutPage` component for the exact request shape and expected responses (`{ success, order_key, client_secret, … }`).

## Cart Persistence

Cart quantities and item objects are saved to `localStorage` under `CART_KEY` and `CART_ITEMS_KEY`. They survive page refreshes and navigation.

## Brand Tokens

Custom CSS variables defined in `:root`:

| Token | Value | Usage |
|---|---|---|
| `--eb-mustard` | `#F2C14E` | Hero bg, navbar bg |
| `--eb-plum` | `#2E1A2B` | Primary text, buttons |
| `--eb-chili` | `#C8321F` | Accent, hover states, heat indicators |
| `--eb-lime` | `#E3F0B7` | Accent backgrounds |
| `--eb-paper` | `#FFFCF4` | Page background |
| `--eb-soft` | `#5B4558` | Secondary text |

## Accessibility

- Heat marks include an `aria-label` ("Heat N of 5").
- Loading skeletons use `aria-busy` and `aria-hidden`.
- Cart total and quantity react to changes via `aria-live`.
- All buttons have accessible labels.
- Respects `prefers-reduced-motion` — all animations are disabled when set.
