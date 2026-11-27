# Ember & Brine — Hot Sauce Storefront

A single-file React storefront for a hot sauce brand, built to run inside a **GoHighLevel (GHL) custom HTML block** with no build step required. Everything ships in one `index.html`.

## Overview

The app renders seven views via React Router's `HashRouter`, so the URL never changes from GHL's perspective — only the `#` fragment does. This means refreshes and shared links always work on a single GHL page.

| Route | View |
|---|---|
| `#/` | Home — hero, best sellers, hottest, heat picker, process, reviews, FAQ |
| `#/sauces` | All sauces with real-time search and heat filter (shareable URL) |
| `#/sauce/:id` | Single sauce — full details, quantity stepper, total price, "you may also like" |
| `#/checkout` | Order summary + Stripe card form |
| `#/orders` | Look up an order by its UUID reference key |
| `#/orders/:orderKey` | Order detail — shareable tracking link |
| `#/contact` | Contact page — form with API submission and "Get in touch" info card |
| `#/about` | About page — project info and creator card |

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

  CONTACT_URL: "https://…/api/ghl/add-contact",  // POST { name, email, message }; expects { status: true }
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

## Page Headers

The sauces and orders pages have a full-width mustard header with a layered background — a mustard-to-transparent gradient over a photo — so text stays readable on any image.

| Section | CSS ID | Image hides at |
|---|---|---|
| All sauces | `#eb-sauces-header` | ≤ 768 px (`max-width: 767.98px`) |
| Track your order | `#eb-orders-header` | ≤ 992 px (`max-width: 991.98px`) |

Background images are served from Google Drive via `https://lh3.googleusercontent.com/d/{file_id}`. To swap an image, replace the file ID in the relevant `#eb-sauces-header` or `#eb-orders-header` CSS rule.

## SEO — Sauce Detail Page

When a visitor lands on `#/sauce/:id`, the app dynamically updates `<head>` with:

| Tag | Value |
|---|---|
| `<title>` | `{Sauce name} — Ember & Brine` |
| `meta[name="description"]` | Sauce description |
| `og:type` | `product` |
| `og:title` / `og:description` | Sauce name / description |
| `og:image` | Sauce photo URL |
| `twitter:card` | `summary_large_image` |
| `twitter:title` / `twitter:description` / `twitter:image` | Sauce name / description / photo |

Tags are restored to their previous values (or removed if created) when the visitor navigates away, so other pages are not affected.

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
| `--eb-mustard` | `#F2C14E` | Hero bg, navbar bg, page section headers |
| `--eb-plum` | `#2E1A2B` | Primary text, buttons, footer bg |
| `--eb-chili` | `#C8321F` | Accent, hover states, heat indicators |
| `--eb-lime` | `#E3F0B7` | Accent backgrounds, cart drawer banner |
| `--eb-paper` | `#FFFCF4` | Page background, form inputs |
| `--eb-soft` | `#5B4558` | Secondary text |

## Contact Page

The `/contact` route renders a two-column layout:

- **Left — contact form:** Name, email, and message fields with client-side validation. On submit, POSTs `{ name, email, message }` to `CONFIG.CONTACT_URL` and shows a success or error message based on `data.status === true`. A test-demo disclaimer is shown above the submit button.
- **Right — "Get in touch" card:** Mustard background card with fake demo address, phone, and email. A warning pill at the bottom reads "⚠️ Not real information".

### `POST CONTACT_URL` — Submit contact form

Request body:
```json
{ "name": "…", "email": "…", "message": "…" }
```
Expected response:
```json
{ "status": true }
```
Any other response (or a network error) shows the error state.

## Footer

The footer contains the brand logo and two navigation columns:

| Column | Links |
|---|---|
| Shop | All sauces, Home, About |
| Help | FAQ, Shipping & returns, Track your order, Contact us |

## Accessibility

- Heat marks include an `aria-label` ("Heat N of 5").
- Loading skeletons use `aria-busy` and `aria-hidden`.
- Cart total and quantity react to changes via `aria-live`.
- All buttons have accessible labels.
- Respects `prefers-reduced-motion` — all animations are disabled when set.
