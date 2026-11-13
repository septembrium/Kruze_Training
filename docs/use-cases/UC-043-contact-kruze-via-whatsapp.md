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
4. The system records an anonymous contact request, together with the date and time and that the request came from the footer.
5. The system opens WhatsApp addressed to KRUZE's WhatsApp number, with the default message already filled in, also when the contact request could not be recorded.
6. The webshop visitor adjusts the message if desired and sends it to KRUZE.

## Alternative Flows
- **AF-1: Start from the Contact Page** — At step 1, the webshop visitor opens the Contact page instead of the footer. The system shows the address, phone number, email address, opening hours (existing opening hours, unchanged) and the invitation to chat via WhatsApp with the WhatsApp number. The flow continues at step 3; at step 4 the system records that the request came from the Contact page.

## Exception Flows
None. The system cannot detect what happens inside WhatsApp after step 5.

## Postconditions
- WhatsApp is open on the visitor's device, addressed to KRUZE, with the default message ready to send.
- One anonymous contact request is counted for the admin.

## Business Rules
- **BR-1:** A recorded contact request contains only the date and time and whether it came from the footer or the Contact page. Nothing that identifies the visitor is kept.
- **BR-2:** No more than 5 contact requests from the same visitor are counted within 10 minutes; further requests in that window are not counted, but WhatsApp still opens.

## Presentation Guidelines
- **PG-1:** The WhatsApp invitation is only shown when WhatsApp contact is switched on and a number is registered; otherwise the contact details appear exactly as before, without any WhatsApp mention.
- **PG-2:** The invitation is a clearly recognisable call to action (WhatsApp icon plus Dutch label, e.g., "Chat met ons via WhatsApp"), placed alongside the phone number and email address in both the footer and the Contact page.
- **PG-3:** WhatsApp opens in a new browser tab or the WhatsApp app, so the visitor does not lose their place in the webshop.

## Implementation Notes
- Configuration lives in the existing `parameter_table` (see UC-015, AF-6): `whatsapp_enabled`, `whatsapp_number`, `whatsapp_default_message`.
- A public, unauthenticated endpoint exposes the configuration only when enabled and a number is set; otherwise it returns a disabled state. Never expose any other parameter through it.
- The public configuration endpoint reads exactly the three whatsapp_* parameters by name via parameterService.getParameter; never use getAllParameters (it would expose e.g. accounting_email).
- parameter_value is NOT NULL text: store the toggle as 'true'/'false' and empty values as ''. Parameter names are matched case-insensitively.
- Link format: `https://wa.me/<digits-only international number>?text=<URL-encoded default message>`, with `target="_blank"` and `rel="noopener noreferrer"`.
- Click recording uses a public endpoint with a new table (e.g., `whatsapp_contact_click`: id, clicked_at, source ∈ {`footer`, `contact_page`}). Any other source value is rejected.
- Failing to record a contact request must never prevent the visitor from reaching WhatsApp. Send the request fire-and-forget (e.g., `navigator.sendBeacon`) so navigation is never blocked.
- Rate limiting is 5 requests per 10 minutes per IP (BR-2). Rate limiting uses the IP address in memory only and never persists it.
- Privacy decision (accepted by the owner, 2026-11-13): because nothing identifying is stored or placed on the visitor's device, visitor consent (UC-037) is not requested. Rate limiting uses the IP address in memory only and never persists it.
- On devices without the WhatsApp app, wa.me falls back to WhatsApp Web; no extra handling needed.
- The count measures contact intent (clicks), not delivered messages; a visitor without a WhatsApp account is still counted.
- Admin view of click totals is part of the WhatsApp section in System Settings (UC-015).
- The existing phone number and email in the footer and Contact page are hardcoded and stay out of scope for this use case.
- Dependency: requires UC-015 AF-6 (WhatsApp configuration). Suggested slice order: (1) UC-015 AF-6 configuration without counts; (2) footer link with default message, no counting; (3) anonymous counting with rate limiting; (4) admin counts view in UC-015; (5) Contact page (AF-1).

## Implemented By
- Routes: Not yet implemented
- Services: Not yet implemented
- Components: Not yet implemented (`client/src/components/storefront/StorefrontLayout.tsx` footer and `client/src/pages/storefront/Contact.tsx` to be extended)
- Storage: Not yet implemented
