### Story 9.1.7: Earn Points When an Admin Marks a Member's Sale Paid
**As a** Storefront member
**I want** to earn points when an admin marks my sale as paid
**So that** a payment confirmed by hand is rewarded like any other

**References:** UC-032 (AF "Sale Marked Paid by Admin"), UC-004
**Priority:** Should
**Estimate:** ~2 story points

**Acceptance Criteria:**
- [ ] When an admin changes a member's sale to PAID on the Admin dashboard, one points entry is recorded, using the basis, rate, rounding and stored rate of Story 9.1.3
- [ ] Only a sale of a customer who qualifies under Story 9.1.4 earns points
- [ ] A sale that already has an entry gets no second entry
- [ ] A failed award follows the retry rule of Story 9.1.5

**Technical Implementation:**
- Backend: call the award from `updateSaleStatus` in `server/salesService.ts` (~456) when the new status is PAID (4); reuses `server/loyaltyService.ts`
- Frontend: none
- Database: none

**Dependencies:** Story 9.1.3, Story 9.1.4
