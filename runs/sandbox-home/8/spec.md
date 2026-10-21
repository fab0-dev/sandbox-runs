# Spec: Refund a paid order in full

## Intent

The Feature "Card payments" lets customers pay for their order by card at checkout, see whether the payment went through, and lets the shop see each payment's outcome. Today, once an order is paid it stays paid. When an item turns out to be out of stock, or the customer cancels, the shop has no way to give the money back through the payments service, and nothing in the payment record shows that it did.

This Story lets the shop refund a paid order in full, with a reason, through the payments service. The refund is kept with the payment, so the shop's payment list and the per-order lookup show what happened to the money. That keeps the Feature's promise that the shop can see each payment's outcome. A payment is refunded at most once, and a refund the service cannot make is refused with a reason the shop can act on. On the checkout, a refunded order shows as refunded, with the date, and is not offered for payment again.

Two teams take part:
- **service** (sandbox-api): makes the refund, refuses the ones it must refuse, keeps the refund with the payment, and shows it in its answers.
- **web** (sandbox-web): shows a refunded order as refunded at checkout, with the date, instead of the card form.

## Scope

In:
- Refunding one succeeded payment in full: its whole amount, in its currency, in one go.
- A reason given by the shop with every refund, kept with the refund.
- Refusing a refund, without changing anything, when the payment does not exist, did not succeed (it was declined, so the order was never paid by it), or was already refunded. Each refusal has its own reason the shop can act on.
- At most one refund per payment, also when two refund requests for the same payment arrive at the same time.
- Showing the refund (when, how much, why) on the payment wherever the service returns a payment: Refund payment, Order payment status and List payments.
- Telling, in Order payment status, an order that was refunded apart from one that is paid and one that is unpaid.
- Refusing to pay an order that was refunded, so it is not charged again.
- The checkout showing a refunded order as refunded, with the refund date, with no card form, and the same when a payment is refused because the order was refunded meanwhile.
- Documenting in the service how to make a refund and each refusal, so a person can repeat them on purpose.
- Showing the checkout's refunded state in a browser against the running service, and adding it to the run record the web team keeps from the previous Story.
- Keeping card data safe as before: a refund carries at most the card's brand and last four digits.

Out:
- Partial refunds, and more than one refund per payment.
- Refunds the customer asks for or makes themselves. The checkout offers no refund action.
- A shop back-office screen. The shop refunds through the service's call, as it already sees payments through List payments and Order payment status.
- Real money and a real payment provider. Refunds run in the service's test mode, like payments (see Open questions).
- Cancelling an order, restocking, receipts and emails to the customer about the refund.
- Showing the shop's refund reason to the customer (see Open questions).
- Narrowing List payments to refunded payments, and any new filter or sort.
- Authentication and access control for refunds or any other call, and keeping data across a service restart. These stay open from earlier Stories (see Open questions).

## Done when

D1: The shop can refund a succeeded, not yet refunded payment by naming the payment and giving a reason. The answer shows the payment as refunded, with the amount and currency refunded equal to the payment's own, the reason as given, and the time of the refund.
D2: A refund request without a reason, or with a reason that is blank, is refused, and nothing changes.
D3: Refunding a payment that does not exist is refused with a reason that says the payment was not found. Nothing is recorded.
D4: Refunding a declined payment is refused with a reason that says the payment took no money, so there is nothing to refund. Nothing changes.
D5: Refunding a payment that was already refunded is refused with a reason that says it was already refunded, and when. The first refund stays as it was, and no second refund is recorded.
D6: Each refusal in D2 to D5 has its own code, which the shop can tell apart from the others and from a success, and a message in plain words.
D7: When two refund requests for the same payment arrive at the same time, exactly one succeeds and the other is refused as already refunded.
D8: After a refund, Order payment status for that order says the order is refunded, gives the refund's time, and shows the refund on the succeeded payment among the order's attempts. An unpaid order and a paid, unrefunded order are reported as before.
D9: After a refund, List payments shows the refund on that payment (when, how much, why), in its usual place in the list. Narrowing by `succeeded` and by `declined` gives the same payments as before the refund.
D10: Paying an order that was refunded is refused with its own code and a message the customer can understand. No payment is tried and no attempt is recorded.
D11: Nothing the service returns, lists or records about a refund contains a full card number or a security code.
D12: The service's documentation says how to refund a payment, gives each refusal with its code, and lets a person repeat each of D1 to D5 and D10 on purpose with the test cards.
D13: Opening the checkout for a refunded order shows that the order was refunded, with the date of the refund, and offers no card form and no way to pay.
D14: If the order is refunded while its checkout is open, paying from that checkout shows the customer the service's message for a refunded order, and then shows the order as refunded, with the date. No attempt is recorded.
D15: The checkout for an unpaid order and for a paid, unrefunded order behaves as before.
D16: D13 and D14 are shown in a browser against the running service, started from its repository, with no fake API. The web team's run record says how the refund was made, which order and card were used, and what was seen, so a person can repeat it.
D17: Each team's existing test suite still passes, and the cases of the previous Stories (paying, declines, retry, already paid, double submit, the shop's list and lookup) behave as before.

## Interfaces

The service offers these. The service team writes the contract from them, as a change to the current one. Names give the meaning; the contract settles the exact form.

**Refund payment**: new, called by the shop.
- Takes: the payment reference, and the shop's reason for the refund (plain text, required, not blank).
- Returns the payment, shaped as in Pay order, now carrying its refund.
- Refuses the request, changing nothing, when:
  - the reason is missing or blank (D2);
  - the payment is not found (D3);
  - the payment was declined (D4);
  - the payment was already refunded; the answer says when (D5).
- Each refusal has its own code and a plain-words message. A success can always be told apart from a refusal.

**Refund on a payment**: a change to the payment shape used by Pay order, Order payment status and List payments.
- A payment that was not refunded says so.
- A refunded payment carries: the amount and currency refunded (always the payment's own), the shop's reason, and the time of the refund.
- The payment's outcome stays `succeeded`. A refund does not change the outcome, the decline reason or the payment's place in a list.

**Order payment status**: changed.
- Tells which of three states the order is in: unpaid, paid, or refunded.
- For a refunded order, gives the time of the refund, so the checkout can show its date.

**Pay order**: changed.
- Refuses to pay an order that was refunded, with its own code and a message the checkout can show to the customer as is (D10). This sits beside the existing refusal for an order that is already paid.

**List payments**: unchanged, except that each payment carries its refund as above.

**Cross-origin access**: Order payment status and Pay order stay callable from the checkout's origin as now, including the new answers. Refund payment is called by the shop, not by the checkout.

**Refund documentation**: offered by the service as documentation, beside `docs/test-cards.md`: how to refund, and each refusal with its code (D12).

The web checkout offers no interface to the service. It consumes Order payment status and Pay order.

## Open questions

1. **What "unpaid" means for a payment.** The Story says a refund of a payment that is "unpaid, declined or already refunded" is refused. A payment is either succeeded or declined, so a payment that is unpaid has no meaning of its own. My reading: the shop names a payment, and "unpaid" covers a declined payment (it took no money, so the order is unpaid by it) and a payment reference that does not exist. Both are refused with their own reason (D3, D4). An order that has no succeeded payment therefore has nothing the shop can refund.
2. **Naming the payment, not the order.** The Story speaks of refunding a paid order but also says a payment is refunded at most once. My reading: the shop names the payment, which it finds with Order payment status or List payments. An order has at most one succeeded payment, so this is the same as refunding the order, and it leaves no doubt which payment is refunded.
3. **Can a refunded order be paid again?** The Story says the checkout shows a refunded order "instead of offering to pay it again", which reads as final: the order was cancelled or cannot be delivered. My reading: the service refuses to pay a refunded order (D10), so a second checkout tab or a stale page cannot charge it again. If the shop wants to take a new payment for the same order later, that is a separate Story.
4. **The form of the reason.** The Story asks for "a reason" and gives two examples, out of stock and customer cancelled. My reading: plain text from the shop, required and not blank, kept and shown as given. The service may set a sensible length limit and says so in its contract. A fixed list of reason codes for reporting would be a later Story if the product owner wants one.
5. **Test mode.** There is no real payment provider, so no money moves back. My reading: a refund in test mode always succeeds when it is allowed. There are no test cards for a refund that the provider fails. A failed refund from a provider comes with a real provider.
6. **Showing the reason and the time to the customer.** The Story asks the checkout to show the refund date only. My reading: the checkout shows that the order was refunded and the date, not the shop's reason, which may be written for the shop and not for the customer. The date is shown in the customer's own local time, as a date a person reads easily.
7. **No access control on refunds.** Refunds move money back, and the service still has no authentication, so anyone who can reach it could call Refund payment. My reading: this is accepted for now, because the service runs in test mode only and access control is already left to its own Story. That Story should cover Refund payment first.
8. **What "paid" means after a refund.** Today Order payment status says whether the order is paid. My reading: a refunded order is reported as refunded, not as paid and not as unpaid (D8), and the checkout keys on that state. How the service keeps existing readers of "paid" working is left to its contract.
