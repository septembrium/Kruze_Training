### Story 9.1.5: Points Survive Payment Delays and Failures
**As a** Storefront member
**I want** my points awarded exactly once, even when payment is delayed or recording fails
**So that** I get my points once the order is really paid, never twice and never lost

**References:** UC-032 (EF-2, EF-3, BR-1, BR-10)
**Priority:** Should
**Estimate:** ~3 story points

**Acceptance Criteria:**
- [ ] A pending or failed payment awards nothing
- [ ] When a pending payment (e.g., bank transfer) is later confirmed, the points are awarded then
- [ ] A repeated Mollie webhook for the same sale awards once
- [ ] Two concurrent webhooks for the same sale award once
- [ ] When recording fails, the sale stays PAID and the customer sees no error
- [ ] A failed award is retried later and recorded once
- [ ] The retry only covers awards that were attempted and failed; PAID sales that were never attempted (e.g., paid before go-live) are not awarded
- [ ] The same retry rule applies to POS sales (Story 9.2.2) and admin-marked sales (Story 9.1.7)

**Technical Implementation:**
- Backend: idempotent award in `server/loyaltyService.ts` (Story 9.1.3) with a uniqueness guarantee per sale; a failure marker or outbox written when an award fails, and a retry that processes only those markers; no sweep over all PAID sales without an entry
- Frontend: none
- Database: uniqueness per sale on the ledger; failure marker or outbox storage (shape in the spec)

**Dependencies:** Story 9.1.3
