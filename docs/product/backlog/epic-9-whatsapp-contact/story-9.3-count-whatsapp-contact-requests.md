# Story 9.3 — Count WhatsApp Contact Requests Anonymously

## User Story 9.3.1: Record Anonymous WhatsApp Contact Requests

**As an** admin
**I want** every WhatsApp chat request from the storefront to be counted anonymously
**So that** I can judge whether the WhatsApp channel is worth offering, without tracking visitors

**References:** UC-043 (step 4, step 5 "also when not recorded", BR-1, BR-2)
**Priority:** Should
**Estimate:** 3 story points

**Acceptance Criteria:**
- [ ] Following the WhatsApp CTA in the footer records one contact request with the date/time and source `footer`
- [ ] Following the WhatsApp CTA on the Contact page records one contact request with source `contact_page`
- [ ] A recorded contact request contains only id, date/time and source — no IP address, user agent, customer id, session or other identifier
- [ ] No cookie or local/session storage entry is set by this feature
- [ ] A request with a source other than `footer` or `contact_page`, or without a source, is rejected (400) and not recorded
- [ ] From the same IP, requests 1–5 within 10 minutes are recorded; the 6th and later within that window are not recorded, and the response does not signal an error to the visitor
- [ ] After the 10-minute window passes, requests from that IP are recorded again
- [ ] WhatsApp still opens when the recording request fails (server down, network error, rate-limited) — navigation is never delayed or blocked by recording
- [ ] Recording a request needs no login
- [ ] When WhatsApp contact is switched off, the endpoint does not record requests

**Technical Implementation:**
- Database: new table `whatsapp_contact_click` (`id` serial, `clicked_at` timestamptz default now, `source` text with check `IN ('footer','contact_page')`), Drizzle schema + insert type in `shared/schema.ts`, migration via `/add-table`
- Backend: public `POST /api/storefront/whatsapp-clicks` with Zod-validated `{ source }`; in-memory per-IP rate limiter (5 / 10 min) — IP never persisted or logged
- Frontend: `WhatsAppContactLink` sends the click via `navigator.sendBeacon` (fallback `fetch` with `keepalive`) fire-and-forget, before the browser follows the link
- Note: in-memory rate limiting is per server instance; acceptable for the expected traffic

**Dependencies:** Story 9.2.1
