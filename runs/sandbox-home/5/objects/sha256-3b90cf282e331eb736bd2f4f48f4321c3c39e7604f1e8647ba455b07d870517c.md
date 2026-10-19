# Spec: Pay an order by card, end to end in a browser

## Intent

The Feature "Card payments" lets customers pay for their order by card at checkout, see whether the payment went through and why not, and lets the shop see each payment's outcome. The previous Story built both halves. The service (sandbox-api) takes and records payments. The web checkout (sandbox-web) asks for card details and shows the result. Each half was tested only on its own: the checkout against a fake API, the service without a browser. So nobody has yet seen a customer pay an order, which is the Feature's main goal.

This Story shows that a customer can pay from the real checkout page, in a browser, against the running service, with every outcome the service offers. It fixes whatever breaks between the two. It also makes the checkout's own field messages read the same as the service's messages for the same mistake, so the customer never sees two different texts for one problem.

Two teams take part:
- **service** (sandbox-api): fixes anything on the service side that stops the browser checkout from working (for example cross-origin headers), and publishes its field messages so the checkout can use the same texts.
- **web** (sandbox-web): runs the checkout in a browser against the running service, fixes anything on the checkout side, uses the service's wording for its field messages, and records the run.

## Scope

In:
- Running the checkout page in a real browser (driven by hand or by a browser automation tool) against the service, started from its own repository with its documented command. No fake API and no stand-in page.
- Every test card in the service's `docs/test-cards.md`, each giving the result that document promises.
- Invalid card details, retrying after a decline, an order that is already paid, and a double click on Pay.
- Checking that the shop's payment list and per-order lookup show each attempt made during the run, and nothing more.
- Fixing every fault found between the checkout and the service, on whichever side it lies. This includes cross-origin calls and preflight, and response headers the checkout needs to read, such as `Idempotency-Replayed`.
- Making the checkout's own field messages read the same as the service's messages for the same field and the same mistake.
- A written record of the run that a person can repeat: how both were started, which cards and inputs were used, and what was seen each time.

Out:
- New payment behaviour: new outcomes, decline reasons, payment methods, refunds, or 3-D Secure.
- A shop back-office screen. The shop still sees outcomes through List payments and Order payment status.
- Authentication and access control, checking the amount against an order record, and keeping idempotency keys across a service restart. These stay open from the previous Story.
- Changing the card rules themselves (what counts as valid). See Open questions for differences in rules found while comparing messages.
- Running the end-to-end check in CI, deploying to a shared environment, and testing several browsers. One current desktop browser is enough.
- Changing the texts of the decline messages. The checkout already shows them as the service sends them.

## Done when

D1: The checkout page, served as its README says and opened in a browser, loads the order's payment status from the running service and shows the card form for an unpaid order. No call is blocked by the browser (no cross-origin error in the browser console).
D2: For each card in `docs/test-cards.md` that gives `succeeded`, paying a new unpaid order shows the customer that the order is paid, with the amount and currency charged equal to the order's, and the card brand and last four digits matching the card used.
D3: For each card in `docs/test-cards.md` that gives `declined`, paying a new unpaid order shows the customer that the payment did not go through, with the message that document lists for that card's reason, word for word. All five reasons are covered: `card_declined`, `insufficient_funds`, `expired_card`, `incorrect_card_details`, `processing_error`.
D4: Entering invalid card details (at least: a missing field, a number that fails the check digit, an expiry date in the past, a security code of the wrong length) shows a message under the wrong field, and no payment attempt for that order appears in the service's lookup.
D5: For every field and mistake that both the checkout and the service check, the checkout's own message is the same text as the service's message. This includes the security code message, which depends on the card brand. A customer who triggers that mistake sees one wording, whichever side catches it.
D6: After a decline in the browser, the customer pays again on the same page, without reopening the order, using a card that succeeds. The page then shows the order as paid. The order's lookup shows the declined attempt and the succeeded attempt.
D7: Opening the checkout for an order that is already paid shows it as paid, with no card form. If the order becomes paid while a second checkout for it is still open (for example in another tab), paying from that second checkout shows that the order is already paid, with the service's message. No second succeeded attempt is recorded for the order.
D8: Double-clicking Pay (and pressing Enter twice quickly) records exactly one payment attempt for that submit in the service's lookup, and the page shows one result.
D9: When the service answers a Pay order request as a replay of an earlier request (`Idempotency-Replayed`), the checkout can read that header from its own origin and shows the same result as the first answer. No second attempt is recorded.
D10: After the run, the service's List payments shows every attempt made in the browser, newest first, each with the right outcome, decline reason, amount, currency, brand and last four digits. Narrowing by `succeeded` and by `declined` gives the matching attempts. The per-order lookup gives each order's attempts and whether it is paid. Nothing else was recorded, and no full card number or security code shows up anywhere in these answers.
D11: A written record of the run is kept with the checkout. It says how the service and the checkout were started, lists each case in D1 to D10 with the card or input used, and gives the result seen for each. A person can repeat the run from it.
D12: Every fault found during the run is fixed in the repository where it lies, and the case that showed it passes when run again. Each team's existing test suite still passes.

## Interfaces

No new calls. The checkout keeps using **Pay order**, **Order payment status** and **List payments** as the service's contract describes them. Any fix for the browser, such as cross-origin headers or exposing response headers, is a change to that contract, written by the service team.

**Cross-origin access**: offered by the service to the checkout.
- A checkout served from a different origin (another host or port) can call Pay order and Order payment status from the browser, including the preflight for a request with a JSON body and an `Idempotency-Key` header.
- The checkout can read every response header it relies on, including `Idempotency-Replayed`.

**Field messages**: offered by the service to the checkout.
- The list of card-check mistakes the service reports, by field (`cardholder_name`, `number`, `expiry`, `security_code`) and by code (for example `required`, `invalid_number`, `invalid_format`, `invalid_length`, `expired`), with the customer message for each. Where the message depends on the card brand, each variant is given.
- It is published where the web team can read it, in the contract or in the service's documentation, and it is the reference wording. The checkout shows these same texts for the same mistakes it catches itself.

**Test cards**: the service's `docs/test-cards.md`, as it is now. The run uses it as the list of cases for D2 and D3. If the run shows the document is wrong, the service fixes the document or the behaviour so that they match.

## Open questions

1. **Whose wording wins.** The Story says the two sets of messages must read the same, but not which one to keep. My reading: the service's messages are the reference. They are already documented, and the service's refusal is what a customer sees when the checkout lets a mistake through. The checkout adopts them. If the web team finds a service message unclear, the service changes it, and both sides use the new text.
2. **Different rules, not just different words.** While comparing messages, the teams may find a mistake that one side checks and the other doesn't. For example, the service refuses control characters in the name and limits it to 100 characters. My reading: such differences are listed in the run record. They are fixed in this Story only when the checkout refuses details the service would accept, because that stops a valid payment. Otherwise they are left for a later Story.
3. **Who runs the browser.** The run needs both sides at once. My reading: the web team runs it, with the service started from the service repository's current main branch. The service team makes its fixes on its side and checks them with a browser call from another origin. If the run finds a service fault that the service team has not already fixed, the run record names it, and the Story is not done until it is fixed.
4. **Repeatable automation.** The Story asks to show the result, not to automate it. My reading: a written, repeatable run record (D11) is enough. A browser test script that repeats the run is welcome but not required.
5. **Testing a replay from the browser.** The checkout sends a fresh key on each submit, so a replay does not come up in normal use. My reading: D9 is shown by sending the same Pay order request twice with the same key from the checkout's origin in the browser, and checking that the header can be read and the result is the same. If the checkout does not use the header at all, it is enough that the header can be read and that a replay is shown the same as the first answer.
