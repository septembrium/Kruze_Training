### Story 9.1.7: Earn Points When an Admin Makes a Member's Sale Paid
**As a** Storefront member
**I want** to earn points when an admin makes my sale paid
**So that** a payment confirmed by hand is rewarded like any other

**References:** UC-032 (AF-4), UC-003 (main flow, AF-1), UC-004 (AF-3)
**Priority:** Should
**Estimate:** ~2 story points

**Acceptance Criteria:**
- [ ] When an admin changes a member's sale to PAID on the Admin dashboard, one points entry is recorded, using the basis, rate, rounding and stored rate of Story 9.1.3
- [ ] A sale an admin adds as paid for a qualifying member earns points once
- [ ] A not-yet-paid sale an admin edits to paid for a qualifying member earns points once
- [ ] A sale of a B2B client earns nothing (qualification rule of Story 9.1.4)
- [ ] A webshop sale that an admin makes paid and that the payment provider later confirms as paid (or the reverse) earns points once
- [ ] A failed award is retried automatically (Story 9.1.5)

**Technical Implementation:**
- Backend: call the award in `server/salesService.ts` from `updateSaleStatus` (~456), from `createSale` when `request.salesStatus` is PAID (4) (~84-112), and from `updateSale` (~203-233) when the sale becomes PAID; reuses `server/loyaltyService.ts`
- Frontend: none (the admin NewSaleDialog already has a status selector)
- Database: none

**Dependencies:** Story 9.1.3, Story 9.1.4, Story 9.1.5
