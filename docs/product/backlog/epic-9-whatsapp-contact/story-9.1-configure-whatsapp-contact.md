# Story 9.1 — Configure WhatsApp Contact in System Settings

## User Story 9.1.1: Configure WhatsApp Contact

**As an** admin
**I want to** switch WhatsApp contact on or off and set the WhatsApp number and default message
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
- [ ] A number with fewer than 8 or more than 15 digits, or containing letters, is rejected with a message naming the number field
- [ ] Switching on without a number is rejected with a message naming the number field
- [ ] Switching on without a default message is rejected with a message naming the message field
- [ ] A default message of 501+ characters is rejected; exactly 500 is accepted
- [ ] On any rejection, the previously saved settings remain unchanged
- [ ] Switching off is allowed with an empty number and message
- [ ] Only admins can read or change these settings (non-admin request → 401/403)

**Technical Implementation:**
- Backend: `GET /api/settings/whatsapp` and `POST /api/settings/whatsapp` (`isAdmin`), following the `/api/settings/shipping-params` pattern; validation + normalisation in a service (e.g. `server/whatsappService.ts`) using a Zod schema
- Storage: `parameter_table` rows `whatsapp_enabled` (`'true'`/`'false'`), `whatsapp_number`, `whatsapp_default_message` (empty → `''`, column is NOT NULL); write all three atomically
- Frontend: new WhatsApp tab/section in `client/src/pages/admin/Settings.tsx` — tab bar currently `lg:grid-cols-12`, needs a layout adjustment for a 13th tab
- Database: no schema change

**Dependencies:** None
