# Story 9.2 — Show WhatsApp Chat Link in Footer and on Contact Page

## User Story 9.2.1: Start a WhatsApp Chat from the Storefront

**As a** webshop visitor
**I want to** start a WhatsApp conversation with KRUZE from the contact details, with an opening message already filled in
**So that** I can ask a question in the app I already use, without calling or writing an email

**References:** UC-043 (main flow steps 1–3, 5 (opening WhatsApp); AF-1; PG-1–PG-3)
**Priority:** Should
**Estimate:** 3 story points

**Acceptance Criteria:**
- [ ] When WhatsApp contact is on and a number is set, the footer contact details on every storefront page offer a WhatsApp chat invitation showing the WhatsApp number, next to the phone number and email address
- [ ] Under the same conditions, the Contact page (`/nl/contact`) offers the same invitation with the number, alongside address, phone, email and the unchanged opening hours
- [ ] The invitation links to `https://wa.me/<number>?text=<message>`, with the default message URL-encoded; a message containing spaces, é, ë and & produces a correctly encoded link (verified on the link, not in WhatsApp)
- [ ] A link built from a 500-character message (including emoji) still opens WhatsApp with the message; if it does not, lower the limit and update UC-015 BR-9 and Story 9.1
- [ ] Following the invitation opens WhatsApp in a new browser tab or the app; the visitor's webshop page stays where it was
- [ ] The invitation has an accessible name and can be activated by keyboard
- [ ] When WhatsApp contact is off, or no number is set, neither the footer nor the Contact page shows any WhatsApp mention, and the existing contact details look as before
- [ ] If the configuration cannot be loaded, the invitation is not shown and the rest of the page renders normally
- [ ] After the admin changes the number or message (Story 9.1), a full page load of the storefront shows the new values
- [ ] (API contract) The public configuration endpoint returns only `enabled`, `number` and `defaultMessage` — no other parameter is exposed, and it returns `{ enabled: false }` when off or no number is set

**Technical Implementation:**
- Backend: public `GET /api/webshop/whatsapp-config` (no auth; consistent with the existing `/api/webshop/*` public routes); reads exactly the three `whatsapp_*` parameters via `parameterService.getParameter` — never `getAllParameters`
- Frontend: shared `WhatsAppContactLink` component (e.g. `client/src/components/storefront/WhatsAppContactLink.tsx`) using React Query with a short stale time so changes appear on the next full page load; used in the footer of `client/src/components/storefront/StorefrontLayout.tsx` and in `client/src/pages/storefront/Contact.tsx`; follow existing `COLORS` styling and `data-testid` conventions (`footer-link-whatsapp`, `contact-whatsapp`)
- Presentation (UC-043 PG-2/PG-3): label text "Chat met ons via WhatsApp" plus WhatsApp icon, followed by the number; link uses `target="_blank"` and `rel="noopener noreferrer"`
- Database: none

**Definition of Done:** Update UC-043 Implemented By.

**Dependencies:** Story 9.1.1
