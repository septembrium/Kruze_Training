### Story 9.1.5: Points Survive Payment Delays and Failures
**As a** Storefront member
**I want** my points awarded exactly once, even when payment is delayed or recording fails
**So that** my points arrive once the order is really paid, and never twice

**References:** UC-032 (EF-2, EF-3, BR-1, BR-10)
**Priority:** Should
**Estimate:** ~3 story points

**Acceptance Criteria:**
- [ ] A pending or failed payment awards nothing
- [ ] When a pending payment (e.g., bank transfer) is later confirmed, the points are awarded then
- [ ] A repeated Mollie webhook for the same sale awards once
- [ ] Two concurrent webhooks for the same sale award once
- [ ] When recording fails, the sale stays PAID and the customer sees no error
- [ ] A failed award is retried automatically at least every N minutes (N set in the spec) until it succeeds, and is recorded once
- [ ] The retry only covers awards that were attempted and failed; PAID sales that were never attempted (e.g., paid before go-live) are not awarded

**Technical Implementation:**
- Backend: failure marker or outbox written when an award in `server/loyaltyService.ts` fails, and a scheduled retry that processes only those markers; no sweep over all PAID sales without an entry
- Frontend: none
- Database: failure marker or outbox storage (shape in the spec); the per-sale uniqueness on the ledger comes from Story 9.1.3

**Dependencies:** Story 9.1.3. Stories 9.1.7 and 9.2.2 reuse the retry.
