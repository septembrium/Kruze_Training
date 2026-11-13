### Story 9.1.3: Earn Points on a Paid Webshop Order
**As a** logged-in webshop member
**I want** to earn loyalty points when my webshop order is paid
**So that** my purchases are rewarded

**References:** UC-032 (main flow, EF-2, EF-3, BR-1 to BR-7, BR-9 to BR-11), UC-025 (AF-5)
**Priority:** Should
**Estimate:** ~8 story points

**Context:**
This story also creates the append-only points ledger and the shared earning logic that the POS (Story 9.2.2) reuses. Sale status IDs are NEW=1, SHIPPED=2, INVOICED=3, PAID=4 (`server/salesService.ts`). A webshop sale becomes PAID in `handleWebhook` in `server/checkoutService.ts`. The legacy `loyalty` table (`shared/schema.ts`) is unused and is replaced.

**Acceptance Criteria:**
- [ ] When the payment provider confirms payment and the sale becomes PAID, one points entry is recorded for the member, linked to the sale
- [ ] The earn basis is the amount paid for products including VAT, after line discounts and after any sale-level discount, excluding the shipping line
- [ ] Points = earn basis x earn rate, rounded down to whole points per sale (EUR 42,50 products plus EUR 5,00 shipping at 1 point per EUR 1 earns 42 points)
- [ ] A fractional rate is applied correctly (EUR 42,50 at 0.5 earns 21 points)
- [ ] A basket worth less than one point at the rate in force records no entry (or a zero entry is never shown as earned); the sale is unaffected
- [ ] The earn rate in force at the time of the award is stored on the entry
- [ ] A rate change affects only sales that become paid afterwards; earlier entries keep their stored rate
- [ ] A pending or failed payment (e.g., bank transfer not yet received) awards nothing; when the payment is confirmed later, the points are awarded then
- [ ] A repeated or concurrent Mollie webhook for the same sale never creates a second entry
- [ ] If recording the points fails, the sale is still marked PAID and the customer is not shown an error; the points are recorded once the problem is resolved (a retry or a later webhook), and never twice
- [ ] A logged-in member's sale is linked to the member's own customer record (from Story 9.1.1), whatever email is typed in the checkout order details
- [ ] A guest checkout earns nothing, even when its email matches a member's customer record, and a guest sale is not linked to that member
- [ ] A B2B client (`customers.b2bClient`) earns nothing, even with a Storefront account; B2B is decided by this flag, not by the B2B prices flag
- [ ] Entries are never updated or deleted; every entry references a sale
- [ ] Entries have a signed points value, so a reversal entry could be added later; none are written in v1
- [ ] No points are awarded for purchases made before go-live (every member starts at zero)
- [ ] The default payment method (bank transfer) and card/online payments follow the same rules

**Technical Implementation:**
- Backend: new `server/loyaltyService.ts` (new) with an idempotent `awardPointsForSale(saleId)` (earn basis from sales lines excluding the shipping line, rate from `parameterService`, floor per sale); call it from `handleWebhook` in `server/checkoutService.ts` where the sale moves to PAID (PAID = 4); change `findOrCreateCustomer` in `server/checkoutService.ts` (~line 570) so a logged-in checkout uses the linked customer instead of matching by email, and a guest checkout never uses a member's record as a member link; a recording failure is caught and logged without PII, and a retry path (e.g., award on the next webhook or a re-run for PAID sales without an entry); the checkout request must carry the authenticated user (`POST /api/checkout/create` in `server/routes.ts` is currently unauthenticated, so the user is read from the session when present)
- Frontend: none required (the member sees the result in Story 9.1.4)
- Database: new `loyalty_points_ledger` table in `shared/schema.ts` (new; columns: id, customerId, saleId, points, earnRate, createdAt; unique constraint on saleId for earn entries to guarantee at-most-once); drop or retire the unused `loyalty` table, `insertLoyaltySchema` and the `Loyalty` types in `shared/schema.ts`; seeded rate comes from Story 9.1.2 (default 1); migration via `npx drizzle-kit generate`; insert and select types exported; indexes on customerId and saleId

**Dependencies:** Story 9.1.1 (customer link). Story 9.1.2 is not required (rate is seeded at 1). Must ship in the same release as Story 9.1.4.
