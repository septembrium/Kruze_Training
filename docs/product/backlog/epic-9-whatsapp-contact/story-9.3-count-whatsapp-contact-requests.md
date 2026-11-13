# Story 9.3 — Count WhatsApp Contact Requests Anonymously

## User Story 9.3.1: Record Anonymous WhatsApp Contact Requests

**As an** admin
**I want** every WhatsApp chat request made by a storefront visitor to be counted anonymously
**So that** I can judge whether the WhatsApp channel is worth offering, without tracking visitors

**References:** UC-043 (step 4, step 5 "also when not recorded", BR-1, BR-2)
**Priority:** Should
**Estimate:** 3 story points

**Acceptance Criteria:**
- [ ] Following the WhatsApp invitation in the footer results in one stored contact request with date/time and source `footer`; following it on the Contact page results in one with source `contact_page` (verified end to end in the browser, including the beacon request)
- [ ] The recording request is sent as `application/json` and parsed by the server (a stored request exists after a real click)
- [ ] A recorded contact request contains only id, date/time and source — no IP address, user agent, customer id, session or other identifier
- [ ] No cookie or local/session storage entry is set by this feature
- [ ] A request with a source other than `footer` or `contact_page`, or without a source, is rejected (400) and not recorded
- [ ] From the same IP address, the first 5 requests in a fixed 10-minute window are recorded; further requests in that window receive the same success status as a recorded request (204) and are not recorded
- [ ] When the next 10-minute window starts, requests from that IP are recorded again
- [ ] The link opens WhatsApp immediately without waiting for the recording response; a failed, slow or rate-limited recording does not change that
- [ ] Recording a request needs no login
- [ ] When WhatsApp contact is switched off, the endpoint answers 204 and records nothing (story-level hardening decision, not in UC-043)

**Technical Implementation:**
- Database: new table `whatsapp_contact_click` (`id` serial, `clicked_at` timestamptz default now, `source` text with check `IN ('footer','contact_page')`), Drizzle schema + insert type in `shared/schema.ts`, migration via `/add-table`
- Backend: public `POST /api/webshop/whatsapp-clicks` with Zod-validated `{ source }`; success returns 204
- Rate limiting: use the already-installed `express-rate-limit` (fixed 10-minute window, max 5) with a custom handler returning 204 (not the default 429), keyed on `req.ip` (trust proxy is already set in `server/index.ts`); IP never persisted or logged. Note the global `apiLimiter` in `server/index.ts` also applies — tests must not trip it
- Frontend: `WhatsAppContactLink` sends the click via `navigator.sendBeacon` with a `Blob` of type `application/json` (fallback `fetch` with `keepalive`), fire-and-forget, without delaying navigation
- Note: rate-limit state is in memory, per server instance; table growth is unbounded and accepted for expected traffic

**Definition of Done:** Story 9.2 acceptance criteria still pass; update UC-043 Implemented By.

**Dependencies:** Story 9.2.1
