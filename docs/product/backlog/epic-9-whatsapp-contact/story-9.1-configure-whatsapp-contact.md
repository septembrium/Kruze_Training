# Story 9.1 — Configure WhatsApp Contact in System Settings

## User Story 9.1.1: Configure WhatsApp Contact

**As an** admin
**I want to** configure WhatsApp contact (on/off, number, default message)
**So that** I control whether and how visitors can reach KRUZE via WhatsApp without a redeploy

**References:** UC-015 (AF-6, EF-4, BR-7, BR-8, BR-9), UC-043
**Priority:** Should
**Estimate:** 3 story points

**Acceptance Criteria:**
- [ ] System Settings has a WhatsApp contact section showing the current on/off setting, WhatsApp number and default message
- [ ] With no values configured yet, the section shows WhatsApp contact as off with an empty number and message
- [ ] The admin can change the on/off setting, number and default message and save them; saved values are shown after reloading the page
- [ ] A number entered as `+32 497 97 44 37` or `+32-497-974437` is saved as `32497974437`
- [ ] A number whose country code starts with 0 (e.g. `0497974437`) is rejected with a message naming the number field
- [ ] A number of exactly 8 digits and of exactly 15 digits is accepted; 7 and 16 digits are rejected with a message naming the number field
- [ ] A number containing letters or other non-digit characters (apart from the accepted `+`, spaces and `-`) is rejected with a message naming the number field
- [ ] Switching on without a number is rejected with a message naming the number field
- [ ] Switching on without a default message is rejected with a message naming the message field
- [ ] The default message may contain at most 500 characters as the admin sees them (an emoji or accented letter counts as one); 500 such characters are accepted, 501 are rejected
- [ ] On any rejection, the previously saved settings remain unchanged
- [ ] When switched off, an empty number and empty message are accepted
- [ ] A non-empty number that is invalid is rejected even when WhatsApp contact is switched off
- [ ] Only admins can read or change these settings (non-admin request → 401/403)

**Technical Implementation:**
- Backend: `GET /api/settings/whatsapp` and `POST /api/settings/whatsapp` (`isAdmin`). Routes stay thin (Zod parse → service → respond); do not copy the logic-in-route style of `/api/settings/shipping-params` literally. Validation + normalisation in a service (e.g. `server/whatsappService.ts`) using a Zod schema; message length counted in user-perceived characters (grapheme clusters, e.g. `Intl.Segmenter`), not UTF-16 code units
- GET response is an object `{ enabled, number, defaultMessage }`, designed to be extended by Story 9.4 (counts)
- Storage: `parameter_table` rows `whatsapp_enabled` (`'true'`/`'false'`), `whatsapp_number`, `whatsapp_default_message` (empty → `''`, column is NOT NULL). Write all three parameters in one transaction via a new service/storage method (`parameterService.setParameter` has no multi-row transaction)
- Frontend: new WhatsApp tab/section in `client/src/pages/admin/Settings.tsx` — tab bar currently `lg:grid-cols-12`, needs a layout adjustment for a 13th tab
- Database: no schema change

**Definition of Done:** Update UC-015 Implemented By (AF-6) and status.

**Dependencies:** None
