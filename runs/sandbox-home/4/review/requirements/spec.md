# Spec: Pay an order by card at checkout

## Intent

The Feature "Card payments" lets customers pay for their order by card at checkout. This Story delivers that payment from start to finish. The customer enters card details at checkout. The service tries to take the payment and says whether it went through, and why not when it didn't. The checkout shows the customer that answer. The service also keeps each payment's outcome, so the shop can see what happened to every payment. This Story covers all three of the Feature's "Done when" points.

Two teams build it:
- **service** (sandbox-api): takes the payment, decides the outcome and keeps it.
- **web** (sandbox-web): the checkout step that asks for card details and shows the result.

## Scope

In:
- Paying one order by card, for that order's amount, in one attempt.
- The outcomes "succeeded" and "declined". A declined payment always has a reason the customer can understand.
- Checking card details before any payment is tried (card number, expiry, security code, cardholder name), with a clear message for each problem.
- Trying again after a declined payment, with the same card or another one.
- Not charging an order twice. Once an order is paid, another payment for it is refused.
- Keeping every payment attempt and its outcome on the service, so it can be looked up by order or listed for the shop.
- Keeping card data safe. The full card number and the security code are never stored, shown back or listed. A payment shows at most the card's brand and its last four digits.

Out:
- Real money and a real payment provider. Payments are processed in a test mode, where known test card details always give known outcomes (see Open questions).
- Refunds, partial payments, split payments, saved cards, wallets (Apple Pay, Google Pay) and any payment method other than a card.
- Extra card checks such as 3-D Secure.
- Building the order, the cart or the rest of checkout. The checkout step starts from an order that already exists and has an amount.
- A shop back-office screen. The shop sees outcomes through the service's payment listing in this Story (see Open questions).
- Receipts and emails.

## Done when

D1: At checkout, a customer with an unpaid order can enter card details (cardholder name, card number, expiry date, security code) and submit a payment for the order's amount.
D2: When the card details are valid and the card is approved, the payment succeeds. The customer sees that the order is paid and the amount that was charged.
D3: When the card is declined, the customer sees that the payment did not go through and why, in plain words. There is a distinct reason for each of: card declined, insufficient funds, expired card, incorrect card details, and payment could not be processed (a temporary failure, try again).
D4: Card details that are clearly invalid (missing fields, a card number that fails the standard check digit, an expiry date in the past, a security code of the wrong length) are rejected before any payment is tried. The customer sees which field is wrong.
D5: After a declined payment, the customer can try again on the same checkout without re-entering the order. A later success marks the order as paid.
D6: An order that is already paid cannot be paid again. A second attempt is refused with a reason the customer can understand, and no second charge is recorded.
D7: While a payment is being processed, the customer cannot submit it a second time. A double click or a resubmit results in at most one payment attempt.
D8: Every payment attempt, whether it succeeded or was declined, is recorded on the service with: the order, the amount and currency, the outcome, the decline reason when there is one, the card brand and last four digits, and when it happened.
D9: The shop can list payments, newest first, and see each one's outcome and reason. The shop can also look up all payment attempts for a given order.
D10: Nothing the service returns, lists or records contains a full card number or a security code.
D11: The service documents the test card details that produce each outcome in D2 and D3, so that a person can repeat every outcome on purpose.

## Interfaces

The service offers these to the web checkout and to the shop. The service team writes the contract from them. Names give the meaning; the contract settles the exact form.

**Pay order**: called by the web checkout.
- Takes: the order reference, the amount and currency to charge, and the card details (cardholder name, card number, expiry month and year, security code).
- Returns a payment with:
  - a payment reference;
  - the order reference;
  - the amount and currency;
  - an outcome of `succeeded` or `declined`;
  - for `declined`, a reason code (`card_declined`, `insufficient_funds`, `expired_card`, `incorrect_card_details`, `processing_error`) and a message the checkout can show to the customer as is;
  - the card brand and last four digits;
  - the time of the attempt.
- Refuses the request, without trying a payment, when:
  - the card details are invalid, naming each invalid field (D4);
  - the order is already paid (D6);
  - the amount is not a positive amount.
- A decline is a normal answer, not an error. The web checkout can always tell three things apart: payment succeeded, payment declined (with a reason), and request refused (with which fields or why).

**Order payment status**: called by the web checkout and by the shop.
- Takes an order reference.
- Returns whether the order is paid, plus the order's payment attempts, newest first, each shaped as in Pay order.

**List payments**: called by the shop.
- Returns recorded payments, newest first, each shaped as in Pay order.
- Can be narrowed to one outcome (`succeeded` or `declined`).

**Test cards**: offered by the service as documentation, not as a call.
- The card details that produce each outcome and each decline reason (D11).

The web checkout offers no interface to the service. It consumes Pay order and Order payment status.

## Open questions

1. **Payment provider.** The Story does not name a real payment provider. My reading: payments run in a test mode on the service. Known test card details give each outcome on purpose (D11), and no real money moves. Connecting a real provider would be a later Story.
2. **Where the amount comes from.** It is not settled whether the service already knows orders and their amounts. My reading: the web checkout passes the order reference and the amount, and the service charges that amount. Checking the amount against the service's own order record is left for when orders exist on the service.
3. **How the shop sees outcomes.** The Feature says the shop can see each payment's outcome, but the Story names no shop screen. My reading: the service's List payments and Order payment status satisfy this for now (D9). A back-office screen in the web app would be a separate Story if the product owner wants one.
4. **Currency.** The Story does not mention currency. My reading: one currency per payment, taken from the order, and every payment carries its currency explicitly. Supporting several currencies needs nothing beyond that in this Story.
