# UC-043: Contact KRUZE via WhatsApp

A webshop visitor starts a WhatsApp conversation with KRUZE straight from the storefront contact details, with a ready-made opening message. This gives visitors a low-threshold, familiar way to ask questions, and gives the business owner an anonymous count of how often the channel is used.

## Metadata
- **Domain:** Storefront / Customer Contact
- **Primary Actor:** Webshop Visitor
- **Secondary Actors:** WhatsApp (messaging service)
- **Priority:** Should
- **Status:** Draft
- **Interface:** Storefront

## Preconditions
- The storefront is publicly accessible.
- The admin has switched on WhatsApp contact and registered a WhatsApp number and default message (UC-015).

## Main Flow
1. The webshop visitor wants to ask KRUZE a question and looks up the contact details in the footer of a storefront page.
2. The system shows the contact details: shop name, address, phone number, email address, and an invitation to chat via WhatsApp together with the WhatsApp number.
3. The webshop visitor indicates they want to start a WhatsApp conversation.
4. The system records an anonymous contact request with the date and time and the place it was made from (footer).
5. The system opens WhatsApp addressed to KRUZE's WhatsApp number, with the default message already filled in.
6. The webshop visitor adjusts the message if desired and sends it to KRUZE.

## Alternative Flows
- **AF-1: Start from the Contact Page** — At step 1, the webshop visitor opens the Contact page instead of the footer. The system shows the address, phone number, email address, opening hours and the invitation to chat via WhatsApp with the WhatsApp number. The flow continues at step 3; at step 4 the place is recorded as the Contact page.
- **AF-2: Device Without the WhatsApp App** — At step 5, the webshop visitor's device does not have WhatsApp installed (e.g., a desktop computer). WhatsApp offers to continue in WhatsApp Web, with the same number and message filled in. The webshop visitor continues there and the flow continues at step 6.

## Exception Flows
- **EF-1: Visitor Has No WhatsApp Account** — At step 5 (or during AF-2), the webshop visitor cannot use WhatsApp because they have no account. No conversation is started; the visitor can still use the phone number or email address shown at step 2. The contact request recorded at step 4 remains counted.

## Postconditions
- A WhatsApp conversation with KRUZE is open on the visitor's device, containing the default message, and the visitor has sent it.
- One anonymous contact request is counted for the business owner.

## Business Rules
- **BR-1:** A recorded contact request contains only the date and time and the place it was made from (footer or Contact page). No personal data, device identifiers or cookies are stored, so no visitor consent (UC-037) is required.
- **BR-2:** The place a contact request was made from must be either the footer or the Contact page; any other value is rejected.
- **BR-3:** Failing to record a contact request must never prevent the visitor from reaching WhatsApp.
- **BR-4:** The number of contact requests accepted from a single source in a short period is limited, to keep the count meaningful and prevent abuse.

## Presentation Guidelines
- **PG-1:** The WhatsApp invitation is only shown when WhatsApp contact is switched on and a number is registered; otherwise the contact details appear exactly as before, without any WhatsApp mention.
- **PG-2:** The invitation is a clearly recognisable call to action (WhatsApp icon plus Dutch label, e.g., "Chat met ons via WhatsApp"), placed alongside the phone number and email address in both the footer and the Contact page.
- **PG-3:** WhatsApp opens in a new browser tab or the WhatsApp app, so the visitor does not lose their place in the webshop.

## Implementation Notes
- Configuration lives in the existing `parameter_table` (see UC-015, AF-6): `whatsapp_enabled`, `whatsapp_number`, `whatsapp_default_message`.
- A public, unauthenticated endpoint exposes the configuration only when enabled and a number is set; otherwise it returns a disabled state. Never expose any other parameter through it.
- Link format: `https://wa.me/<digits-only international number>?text=<URL-encoded default message>`, with `target="_blank"` and `rel="noopener noreferrer"`.
- Click recording uses a public endpoint with a new table (e.g., `whatsapp_contact_click`: id, clicked_at, source ∈ {`footer`, `contact_page`}). Send it fire-and-forget (e.g., `navigator.sendBeacon`) so navigation is never blocked (BR-3). Apply rate limiting per IP (BR-4); do not persist the IP.
- Admin view of click totals is part of the WhatsApp section in System Settings (UC-015).
- The existing phone number and email in the footer and Contact page are hardcoded and stay out of scope for this use case.

## Implemented By
- Routes: Not yet implemented
- Services: Not yet implemented
- Components: Not yet implemented (`client/src/components/storefront/StorefrontLayout.tsx` footer and `client/src/pages/storefront/Contact.tsx` to be extended)
- Storage: Not yet implemented
