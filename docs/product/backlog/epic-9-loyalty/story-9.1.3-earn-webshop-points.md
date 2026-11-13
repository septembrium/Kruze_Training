### Story 9.1.3: Earn Points on a Paid Webshop Order
**As a** Storefront member
**I want** to earn loyalty points when my webshop order is paid
**So that** my purchases are rewarded

**References:** UC-032 (main flow, BR-1 to BR-4, BR-9, BR-10)
**Priority:** Should
**Estimate:** ~3 story points

**Context:**
Happy path only. This story creates the append-only points ledger and the earning logic reused by Stories 9.1.7 and 9.2.2. Who qualifies is in Story 9.1.4; payment delays and failures are in Story 9.1.5. The legacy `loyalty` table is unused and is replaced.

**Acceptance Criteria:**
- [ ] When the Mollie webhook sets the sale of a member's order to PAID, one points entry is recorded, linked to the sale
- [ ] The earn basis is the amount for products including VAT, after line discounts and after any sale-level discount, excluding the shipping line
- [ ] Points = basis x earn rate, rounded down to whole points per sale (EUR 42,50 products plus EUR 5,00 shipping at 1 point per EUR 1 earns 42 points)
- [ ] At a rate of 0.5, EUR 42,50 earns 21 points
- [ ] The earn rate in force is stored on the entry
- [ ] After a rate change, earlier entries keep their stored rate and only sales paid afterwards use the new rate
- [ ] A basket worth less than 1 point records no entry
- [ ] An entry is never changed or deleted, and every entry references a sale
- [ ] A sale paid before go-live has no entry

**Technical Implementation:**
- Backend: new `server/loyaltyService.ts` (new) with the award logic; called from `handleWebhook` in `server/checkoutService.ts` (~303-357) when the sale becomes PAID (status 4, `server/salesService.ts`); rate read via `parameterService`
- Frontend: none (visible in Story 9.1.6)
- Database: new points ledger table in `shared/schema.ts` (new; signed points so reversals fit later; unique per sale for earn entries, so a sale awards at most once); retire the unused `loyalty` table at `shared/schema.ts:392`; migration via drizzle-kit

**Dependencies:** Story 9.1.1. Must ship in the same release as Stories 9.1.4 and 9.1.6.
