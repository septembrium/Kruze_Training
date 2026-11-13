# Story 9.2 — Show WhatsApp Chat Link in Footer and on Contact Page

## User Story 9.2.1: Start a WhatsApp Chat from the Storefront

**As a** webshop visitor
**I want to** start a WhatsApp conversation with KRUZE from the contact details, with an opening message already filled in
**So that** I can ask a question in the app I already use, without calling or writing an email

**References:** UC-043 (main flow steps 1–3, 5–6; AF-1; PG-1–PG-3)
**Priority:** Should
**Estimate:** 3 story points

**Acceptance Criteria:**
- [ ] When WhatsApp contact is on and a number is set, the footer Contact column shows a WhatsApp CTA ("Chat met ons via WhatsApp" + WhatsApp icon) next to the phone number and email address, on every storefront page
- [ ] Under the same conditions, the `/nl/contact` page shows the WhatsApp CTA alongside address, phone, email and the (unchanged) opening hours
- [ ] The CTA links to `https://wa.me/<number>?text=<URL-encoded default message>`; a message with spaces, accents (é, ë) and `&` arrives intact in WhatsApp
- [ ] The CTA opens in a new tab/the WhatsApp app (`target="_blank"`, `rel="noopener noreferrer"`); the webshop tab stays on the current page
- [ ] When WhatsApp contact is off, or no number is set, neither the footer nor the Contact page shows any WhatsApp mention, and the existing contact details look as before
- [ ] If the configuration cannot be loaded, the CTA is not shown and the rest of the page renders normally
- [ ] After the admin changes the number or message (Story 9.1), the storefront shows the new values on the next page load
- [ ] The public configuration endpoint returns only `enabled`, `number` and `defaultMessage` — no other parameter is exposed, and it returns `{ enabled: false }` when off or no number is set

**Technical Implementation:**
- Backend: public `GET /api/storefront/whatsapp-config` (no auth); reads exactly the three `whatsapp_*` parameters via `parameterService.getParameter` — never `getAllParameters`
- Frontend: shared `WhatsAppContactLink` component (e.g. `client/src/components/storefront/WhatsAppContactLink.tsx`) using React Query; used in the footer of `client/src/components/storefront/StorefrontLayout.tsx` and in `client/src/pages/storefront/Contact.tsx`; follow existing `COLORS` styling and `data-testid` conventions (`footer-link-whatsapp`, `contact-whatsapp`)
- Database: none

**Dependencies:** Story 9.1.1
