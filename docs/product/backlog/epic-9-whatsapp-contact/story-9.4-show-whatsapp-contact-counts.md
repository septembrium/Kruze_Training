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
- [ ] With no recorded requests, all four counts show 0
- [ ] Counts are visible whether WhatsApp contact is currently on or off
- [ ] Counts cannot be edited or reset from this section
- [ ] Only admins can retrieve the counts (non-admin → 401/403)

**Technical Implementation:**
- Backend: extend `GET /api/settings/whatsapp` response with `counts: { footer: { last30Days, total }, contactPage: { last30Days, total } }`, computed with a single grouped SQL aggregate (no per-row loading)
- Frontend: read-only count display in the WhatsApp section of `client/src/pages/admin/Settings.tsx`
- Database: none (uses `whatsapp_contact_click` from Story 9.3)

**Dependencies:** Story 9.1.1, Story 9.3.1
