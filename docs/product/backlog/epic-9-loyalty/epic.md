# Epic 9 — Loyalty Points (Earning)

**Goal:** Registered B2C customers earn loyalty points on every paid purchase, in the webshop and at the POS, and see their balance, history and member code in their Storefront account.

**Priority:** Should (UC-032 Priority: Should)

**Total Estimated Points:** ~30 points

**Source:** UC-032. Related: UC-004, UC-015 (AF-6, BR-7), UC-021 (AF-11), UC-025 (AF-5), UC-026 (AF-5, AF-6, AF-7).

## Features and Stories

### Feature 9.1 — Earn points in the webshop

| Story | Title | Points | Status |
|-------|-------|--------|--------|
| [Story 9.1.1](story-9.1.1-link-account-customer.md) | Link Storefront Account to a Customer Record | ~3 | Not Started |
| [Story 9.1.2](story-9.1.2-earn-rate-setting.md) | Set the Loyalty Earn Rate | ~2 | Not Started |
| [Story 9.1.3](story-9.1.3-earn-webshop-points.md) | Earn Points on a Paid Webshop Order | ~3 | Not Started |
| [Story 9.1.4](story-9.1.4-checkout-member-attribution.md) | Only the Logged-In Member Earns at Checkout | ~2 | Not Started |
| [Story 9.1.5](story-9.1.5-points-survive-payment-delays.md) | Points Survive Payment Delays and Failures | ~3 | Not Started |
| [Story 9.1.6](story-9.1.6-points-balance-history.md) | See My Points Balance and History | ~3 | Not Started |
| [Story 9.1.7](story-9.1.7-admin-makes-sale-paid.md) | Earn Points When an Admin Makes a Member's Sale Paid | ~2 | Not Started |

### Feature 9.2 — Earn points at the POS

| Story | Title | Points | Status |
|-------|-------|--------|--------|
| [Story 9.2.1](story-9.2.1-member-code.md) | See and Regenerate My Member Code | ~5 | Not Started |
| [Story 9.2.2](story-9.2.2-pos-sale-earns-points.md) | A POS Sale for a Selected Member Earns Points | ~3 | Not Started |
| [Story 9.2.3](story-9.2.3-pos-identify-member-by-code.md) | Identify a Member by Typing Their Member Code | ~2 | Not Started |
| [Story 9.2.4](story-9.2.4-pos-identify-member-by-qr.md) | Identify a Member by Scanning Their QR Code | ~2 | Not Started |

## Dependency Order

```
9.1.1 -> [9.1.3 + 9.1.4 + 9.1.6, same release]
[release] -> 9.1.5 -> 9.1.7, 9.2.2
9.1.4 -> 9.1.7, 9.2.2
9.1.1 + 9.1.6 -> 9.2.1
9.2.1 + 9.2.2 -> 9.2.3 -> 9.2.4
9.1.2 any time (rate seeded at 1; 9.1.3 does not need it)
```

- 9.1.3, 9.1.4 and 9.1.6 ship together: points nobody can see have no value, and without 9.1.4 guests and B2B clients would earn points.
- 9.1.4 holds the single qualification rule used by all earning paths; 9.1.7 and 9.2.2 only prove they use it.
- 9.1.5's retry is reused by 9.1.7 and 9.2.2.
- 9.2.1 needs 9.1.1 and 9.1.6; 9.2.3 needs 9.2.1 and 9.2.2.

## Key Decisions
- One global earn rate; a new rate applies only to later sales; the rate in force is stored on each entry.
- Earn basis: products incl. VAT after discounts (including sale-level discount), excl. shipping; rounded down per sale.
- Points are awarded when the sale becomes PAID: webshop (Mollie webhook), POS (created PAID), or an admin making a sale paid (status change, creating a sale directly as paid, or editing a sale to paid). At most once per sale.
- Only a logged-in B2C member earns. B2B clients, guests, customers without a Storefront account, the guest customer (id 673, used by `posService.getGuestCustomer` and the webshop guest checkout) never earns. The qualification rule lives once, in 9.1.4.
- An account is linked to an existing customer record only when its email is confirmed by the auth service; an unconfirmed account is linked at the first sign-in after confirmation.
- Retry only re-attempts awards that were attempted and failed. There is never a sweep of all PAID sales without an entry, so there is no backfill.
- An append-only ledger replaces the unused legacy `loyalty` table. Awarded points are final in v1.

## Out of Scope
- Redeeming points (UC-034)
- Badges (UC-033)
- Challenges and bonus points (UC-035)
- Separate loyalty login, email verification, multiple emails, full signup (UC-031)
- Reversal of points, refunds and credit notes
- Admin viewing or adjusting a member's points
- Backfilling points for purchases made before go-live
- Linking a sale to a member after it is completed
- Promotional multipliers and notifications

## New Dependencies to Check
- QR code rendering (9.2.1) and camera QR scanning (9.2.4): no library in `package.json` today; run `npm audit` first. A keyboard-wedge USB scanner may need none.
