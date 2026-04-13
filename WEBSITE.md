# Website Technical Reference

## Overview

Evennia custom web frontend for **fcmud.world**. All visitors see the same content worldwide — there are no jurisdictional restrictions or geo-variant page serving under the current compliance model.

## Geo-Detection Infrastructure (Retained, Inactive)

The geo-detection middleware is retained for potential future use but is currently inactive:

- **Middleware**: `web/middleware/geo.py` sets `request.geo_variant` and `request.geo_country` on every request.
- **Context processor**: Injects `geo_variant`, `geo_country`, and `discord_url` into all templates.
- **`_RESTRICTED_PATHS`**: Currently empty — no paths are geo-restricted.
- **`GEO_ELIGIBLE_COUNTRIES`**: Country set defined in `settings.py` — determines Variant A vs B classification, but no template currently branches on it.
- **Fail-closed**: Unknown or missing country code defaults to Variant A.

To re-enable geo-restrictions in the future: add paths to `_RESTRICTED_PATHS` and add `{% if geo_variant == 'B' %}` conditionals to templates.

## Login Geo-Logging (Disabled)

Login geo-logging (IP hash + country code per session) is gated by `settings.LOG_PLAYER_GEO_DATA` (currently `False`). The infrastructure exists in `accounts.py` `at_post_login` and can be re-enabled by setting the flag to `True`.

## Page Inventory

### Homepage

| Field | Value |
|---|---|
| URL | `/` |
| Template | `index.html` |
| Status | Built, compliant |

Pre-alpha notice, feature cards, game description. No financial product language.

### Vision

| Field | Value |
|---|---|
| URL | `/vision/` |
| Template | `vision.html` |
| Status | Built, compliant |

Founder's vision and game philosophy. No financial product language.

### About

| Field | Value |
|---|---|
| URL | `/about/` |
| Template | `about.html` |
| Status | Built, compliant |

Design pillars, world description, technical foundation. No financial product language.

### Costs

| Field | Value |
|---|---|
| URL | `/costs/` |
| Template | `costs.html` |
| Status | Built, compliant |

Subscription pricing (2 RLUSD alpha, 20 RLUSD go-live), game economy overview, running costs breakdown.

### Terms of Service

| Field | Value |
|---|---|
| URL | `/terms/` |
| Template | `terms.html` |
| Status | Draft — pending legal review |

Covers: nature of in-game assets, no redemption, trading on external protocols, game economy management, alpha/testnet phases. See `design/COMPLIANCE.md` § Terms of Service.

### Privacy Policy

| Field | Value |
|---|---|
| URL | `/privacy/` |
| Template | `privacy.html` |
| Status | Draft — pending legal review |

Standard privacy policy. Account data, session records, game activity. No KYC/AML data collection.

### Redemption (Stub)

| Field | Value |
|---|---|
| URL | `/redemption/` |
| Template | `redemption.html` |
| Status | Stub — "Page Not Available" |

Placeholder page stating FullCircleMUD does not offer gold redemption. URL route retained to avoid broken links.

### Eligible Jurisdictions (Stub)

| Field | Value |
|---|---|
| URL | `/eligible-jurisdictions/` |
| Template | `eligible_jurisdictions.html` |
| Status | Stub — "Page Not Available" |

Placeholder page. No jurisdictional eligibility framework exists. URL route retained to avoid broken links.

### Xaman Wallet Setup

| Field | Value |
|---|---|
| URL | `/xaman/` |
| Template | `xaman.html` |
| Status | Built |

Wallet setup guide: install Xaman, create wallet, link to game account. No financial product language.

### Documentation

| Field | Value |
|---|---|
| URL | `/docs/` |
| Templates | `docs.html`, `docs_gameplay.html`, `docs_blockchain.html`, `docs_client_api.html` |
| Status | Built, compliant |

Blockchain mechanics (XRPL ownership, wallets, import/export, AMM pricing, trustlines). No financial product language.

### Markets

| Field | Value |
|---|---|
| URL | `/markets/` |
| Template | `markets.html` |
| Status | Built |

Live AMM price dashboard from ResourceSnapshot data.

### Navigation

| Field | Value |
|---|---|
| Template | `_menu.html` |
| Status | Built, compliant |

Home, Vision, About, Costs, Markets, Docs, Discord. No geo-variant conditional links.

## Pages Not Built (No Longer Required)

The following pages were planned under the previous compliance model and are no longer needed:

- ~~`/kyc/`~~ — KYC verification (no redemption = no KYC)
- ~~`/location-review/`~~ — Geo-classification disputes (no jurisdictional framework)
- ~~`/reserve/`~~ — Reserve transparency (no reserve disclosed)

## Implementation Files

| Purpose | Path |
|---|---|
| Geo middleware | `web/middleware/geo.py` |
| URL routing | `web/website/urls.py` |
| Views | `web/website/views/` |
| Base template | `web/templates/website/base.html` |
| Navigation | `web/templates/website/_menu.html` |
| Homepage | `web/templates/website/index.html` |
| Vision | `web/templates/website/vision.html` |
| About | `web/templates/website/about.html` |
| Costs | `web/templates/website/costs.html` |
| Terms | `web/templates/website/terms.html` |
| Privacy | `web/templates/website/privacy.html` |
| Eligible Jurisdictions | `web/templates/website/eligible_jurisdictions.html` |
| Redemption | `web/templates/website/redemption.html` |
| Xaman | `web/templates/website/xaman.html` |
| Docs (index) | `web/templates/website/docs.html` |
| Docs (gameplay) | `web/templates/website/docs_gameplay.html` |
| Docs (blockchain) | `web/templates/website/docs_blockchain.html` |
| Docs (client API) | `web/templates/website/docs_client_api.html` |
| Markets | `web/templates/website/markets.html` |
