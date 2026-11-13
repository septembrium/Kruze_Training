### Story 9.1.7: Earn Points When an Admin Makes a Member's Sale Paid
**As a** Storefront member
**I want** to earn points when an admin makes my sale paid
**So that** a payment confirmed by hand is rewarded like any other

**References:** UC-032 (AF "Sale Marked Paid by Admin"), UC-004
**Priority:** Should
**Estimate:** ~2 story points

**Acceptance Criteria:**
- [ ] When an admin changes a member's sale to PAID on the Admin dashboard, one points entry is recorded, using the basis, rate, rounding and stored rate of Story 9.1.3
- [ ] A sale an admin creates directly as paid, or edits to paid, for a qualifying member earns points once, like a status change
- [ ] A selected B2B client earns nothing (qualification rule of Story 9.1.4)
- [ ] A sale that already has an entry gets no second entry
- [ ] A failed award is retried automatically (Story 9.1.5)

**Technical Implementation:**
- Backend: call the award in `server/salesService.ts` from `updateSaleStatus` (~456), from `createSale` when `request.salesStatus` is PAID (4) (~84-112), and from `updateSale` (~203-233) when the sale becomes PAID; reuses `server/loyaltyService.ts`
- Frontend: none (the admin NewSaleDialog already has a status selector)
- Database: none

**Dependencies:** Story 9.1.3, Story 9.1.4, Story 9.1.5
