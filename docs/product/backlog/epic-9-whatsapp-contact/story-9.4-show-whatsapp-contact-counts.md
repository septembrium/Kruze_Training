# Story 9.4 — Show WhatsApp Contact Counts to Admin

## User Story 9.4.1: View WhatsApp Contact Counts

**As an** admin
**I want to** see how many WhatsApp contact requests visitors made, per source
**So that** I can see whether visitors use the WhatsApp channel and where they find it

**References:** UC-015 (AF-6, read-only counts), UC-043 (postcondition 2)
**Priority:** Should
**Estimate:** 1 story point

**Acceptance Criteria:**
- [ ] The WhatsApp contact section in System Settings shows four read-only numbers: footer — last 30 days, footer — total, Contact page — last 30 days, Contact page — total
- [ ] "Last 30 days" counts requests from the past 30 × 24 hours up to now; a request 31 days old is in the total but not in the 30-day count
- [ ] A request exactly 30×24 hours old is included in the 30-day count and always in the total
- [ ] With no recorded requests, all four counts show 0
- [ ] Counts are visible whether WhatsApp contact is currently on or off
- [ ] Counts cannot be edited or reset from this section
- [ ] If the counts cannot be retrieved, the admin can still view and change the WhatsApp settings, and the counts area states they are temporarily unavailable
- [ ] The counts are labelled as contact requests (clicks), so they are not read as delivered messages
- [ ] Only admins can retrieve the counts (non-admin → 401/403)

**Technical Implementation:**
- Backend: aggregate query in a storage method (single grouped SQL aggregate, no per-row loading); the service merges it into the `GET /api/settings/whatsapp` response, extending it with `counts: { footer: { last30Days, total }, contactPage: { last30Days, total } }`
- Frontend: read-only count display in the WhatsApp section of `client/src/pages/admin/Settings.tsx`
- Database: none (uses `whatsapp_contact_click` from Story 9.3)

**Definition of Done:** Update UC-015 Implemented By.

**Dependencies:** Story 9.1.1, Story 9.3.1
