---
name: triplelift-tlx-supply-integration
description: >-
  Stand up a server-to-server OpenRTB supply integration with the TripleLift
  Exchange (TLX) — the single geo-balanced auction endpoint, the user-sync
  handshake, privacy signalling, and the impression-counting rules that get supply
  blocked when they are wrong.
api: TripleLift Exchange (TLX) API
endpoint: https://tlx.3lift.com/s2s/auction
protocol: openrtb
operations:
  - POST /s2s/auction
  - GET /xuid
  - GET /getuid
  - GET /sync
generated: '2026-08-12'
method: generated
source: https://docs.triplelift.com/docs/supply-partners
grounding: >-
  Endpoint paths, query parameters, datacenter list, sync flow and impression-counting
  rules are read verbatim from TripleLift's supply-partner and user-sync
  documentation. TripleLift publishes no OpenAPI for this surface; the contract is
  IAB OpenRTB 2.x plus the TripleLift-specific notes documented per object.
---

# Integrate supply with the TripleLift Exchange

TLX is an OpenRTB 2.x exchange. There is no SDK, no OpenAPI document and no bearer token — you POST OpenRTB bid requests to a single endpoint and are identified by a query parameter.

## Step 1 — get your supplier_id

TripleLift issues a `supplier_id` during onboarding, through the Publisher Client Services Team. There is no self-serve path. This is the only credential in the integration.

## Step 2 — call the one endpoint

```
POST https://tlx.3lift.com/s2s/auction?supplier_id=123
Content-Type: application/json
```

Body is a standard OpenRTB 2.x `BidRequest`. TripleLift documents its supported subset object by object — BidRequest, Impression, Regs, Video, App, Content — and only the TripleLift-specific deviations, so treat the IAB specification as the base contract and their pages as the overlay.

**Do not call a region-specific endpoint.** `tlx.3lift.com` is already geo-load balanced across four datacenters:

| Region | Location | AWS region |
|---|---|---|
| US East | Northern Virginia | us-east-1 |
| US West | Northern California | us-west-1 |
| Europe | Frankfurt | eu-central-1 |
| Asia South | Singapore | ap-southeast-1 |

TripleLift explicitly advises against pinning a region — the balanced endpoint routes traffic and provides failover if a region goes down. Pinning removes your failover.

## Step 3 — set the timeout correctly

Use the OpenRTB `tmax` field to declare the maximum milliseconds you will wait for bids. TripleLift documents `tmax` as optional, and publishes no fixed exchange-side timeout value — the budget is negotiated per request rather than fixed in the docs. Set it from your own auction budget, not from a number you read somewhere.

## Step 4 — set up user sync

This is a three-party handshake, not a single call:

1. You give TripleLift your user-sync redirect domain.
2. TripleLift adds it to an internally-maintained allowlist. Syncs from an un-allowlisted domain will not work.
3. TripleLift gives you a sync pixel to drop on the page.

Three endpoints, all on `eb2.3lift.com`, each for a different direction:

| Endpoint | Purpose | Key parameters |
|---|---|---|
| `/xuid` | store **your** user ID in TripleLift's match table so it comes back on bid requests | `mid` (your TripleLift member ID), `xuid` (your user ID), `dongle` (constant code TripleLift issues) |
| `/getuid` | retrieve the TripleLift user ID (tluid) via browser redirect | `redir` (URL-encoded; the `$UID` macro is replaced with the user's ID) |
| `/sync` | retrieve the tluid **and** have TripleLift fan out syncs to its own partners, improving match rate — **must be called in an iframe** | — |

You host the match table. The **tluid you receive back must be set in `user.buyeruid`** on subsequent bid requests. If you skip this, you are sending unmatched traffic and your fill will reflect it.

## Step 5 — pass privacy signals on every sync

All three sync endpoints accept:

- `gdpr` and `gdpr_consent` — IAB Europe TCF
- `us_privacy` — IAB CCPA Compliance Framework
- `gpp` — IAB Global Privacy Platform

Pass every signal you hold, on every call, and mirror them into the OpenRTB `Regs` object on bid requests. TripleLift documents `Regs` as a supported object specifically for this.

## Step 6 — count impressions the way buyers require

This is the rule that gets supply blocked, and it differs by environment:

- **In-app: 1px-in-view.** Industry standard requires an in-app impression be counted only once 1px of the ad is in view. TripleLift is explicit that failing to use this methodology in-app "will often/eventually result in Buyers blocking the supply." If you need ORTB customisations from TripleLift to comply, tell the integration team during onboarding.
- **Web display: on render.**
- **Video: on video start.**

## Step 7 — publish sellers.json alignment

TripleLift publishes its own `sellers.json` at `https://triplelift.com/sellers.json` and uses the sellers.json identifier as the `PUBLISHER_ID` dimension in reporting. Make sure the publisher IDs you send match the identity you expect to see in reports — reporting identity and supply-chain identity are the same key here.

## What you do not get

- No sandbox. The previously-referenced `sand-bidder.triplelift.net` host no longer resolves and no test environment, test bid request, or fixture set is documented.
- No machine-readable contract. There is no OpenAPI and no JSON Schema for the TripleLift-supported OpenRTB subset — object support is documented as prose tables only.
- No changelog. The dated changelog covers the Reporting API only; exchange-side changes are not published.

## Header bidding instead

If you are integrating client-side rather than server-to-server, use the Prebid.js `tripleliftBidAdapter` — TripleLift maintains it upstream in Prebid.org's `prebid.js` package (actively released), not in TripleLift's own GitHub fork (last pushed 2020).
