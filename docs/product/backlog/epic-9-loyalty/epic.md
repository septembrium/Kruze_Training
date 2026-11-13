# Epic 9 — Loyalty Points (Earning)

**Goal:** Registered B2C customers earn loyalty points on every paid purchase, in the webshop and at the POS, and can see their balance, history and member code in their Storefront account.

**Priority:** Should (UC-032 Priority: Should)

**Total Estimated Points:** ~26 points

**Source:** UC-032 (Reviewed). Related: UC-015 (AF-6, BR-7), UC-021 (AF-11), UC-025 (AF-5), UC-026 (AF-5, AF-6, AF-7).

## Features and Stories

### Feature 9.1 — Earn points in the webshop

| Story | Title | Points | Status |
|-------|-------|--------|--------|
| [Story 9.1.1](story-9.1.1-link-account-customer.md) | Link Storefront Account to a Customer Record on First Sign-In | ~3 | Not Started |
| [Story 9.1.2](story-9.1.2-earn-rate-setting.md) | Set the Loyalty Earn Rate | ~2 | Not Started |
| [Story 9.1.3](story-9.1.3-earn-webshop-points.md) | Earn Points on a Paid Webshop Order | ~8 | Not Started |
| [Story 9.1.4](story-9.1.4-points-balance-history.md) | See My Points Balance and History | ~3 | Not Started |

### Feature 9.2 — Earn points at the POS

| Story | Title | Points | Status |
|-------|-------|--------|--------|
| [Story 9.2.1](story-9.2.1-member-code.md) | See and Regenerate My Member Code | ~5 | Not Started |
| [Story 9.2.2](story-9.2.2-pos-member-identification.md) | Identify a Member at the POS and Earn Points | ~5 | Not Started |

## Dependency Order

```
9.1.1  ->  9.1.3 + 9.1.4 (same release)  ->  9.2.1  ->  9.2.2
9.1.2  (any time; the rate is seeded at 1 point per EUR 1, so 9.1.3 does not need it)
```

- 9.1.3 and 9.1.4 must ship together: points that cannot be seen, or a page with nothing to show, have no value on their own.
- 9.2.2 reuses the earning logic and ledger built in 9.1.3.

## Key Decisions (from UC-032)
- One global earn rate; a new rate applies only to later sales; the rate in force is stored on each entry.
- Earn basis: products incl. VAT after discounts (including sale-level discount), excl. shipping; rounded down per sale.
- Points are awarded when the sale becomes PAID (webshop: Mollie webhook; POS: at creation). At most once per sale.
- B2B clients, guests, non-member customers and the default Kruze company customer earn nothing.
- An append-only ledger replaces the unused legacy `loyalty` table. Awarded points are final in v1 (paid sales cannot be changed).

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
- QR code rendering (9.2.1) and QR scanning (9.2.2): no library is in `package.json` today; run `npm audit` before adding.
