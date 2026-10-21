# Feedback: sandbox-web, run 8 (Refund a paid order in full)

This describes the web team's part, as found on the run branch `sandbox-web/8/base` of `/Users/zhen/Work/Fab0/sandbox-web` (T1, T2 and T3 merged; the T3 branch has the same tree). The service's side is not reported here. I ran `npm test` on the T3 worktree: 120 tests, 0 failures. I did not re-run `npm run e2e`; what it saw is taken from `docs/e2e-run.md`.

## Delivered

Only D13 to D17 are in the web team's area.

- **D13, a refunded order opened shows as refunded, with the date, and no form.**
  - `src/payments-api.js`: `getOrderPaymentStatus` now returns `status` (`unpaid`, `paid` or `refunded`), `paid` (true only for `paid`) and `refundedAt` (the contract's `refunded_at` for a refunded order, else null). It falls back on the `paid` boolean only when the service sends no `status`. A `status` outside the three values is `unexpected_response`.
  - `src/checkout.js`: `start()` sends a refunded order straight to a new `refunded` phase. `formatRefundDate` shows the date with `Intl.DateTimeFormat('en-US', {dateStyle: 'long'})` in the browser's own time zone. If the status carries no time, the date is taken from the refund on the succeeded payment. With no time at all, the panel shows without a date.
  - `checkout.html` has a new `#refunded-panel` ("This order was refunded", "Refund date"). `styles.css` has `.alert-refunded`. The panel has no card form and no Pay button, and the form is hidden in this phase. The shop's reason is not shown.
- **D14, an order refunded while its checkout is open.**
  - `src/payments-api.js`: `payOrder` reads the contract's `order_refunded` (409) as `kind: 'refused'`, with the service's message as it is, `orderReference` and `refundedAt` (null when the time is missing or odd).
  - `src/checkout.js`: `showRefundedAfterRefusal` shows the service's message and then the refunded panel with the date. The date comes from the refusal, else from a fresh status read. If neither gives one, the panel has no date. A payment call that gives no usable answer, followed by a status read that says `refunded`, also shows the refunded panel.
  - Nothing is recorded: the checkout sends one payment call and no retry.
- **D15, unpaid, paid, declines, retry, already paid and double submit as before.** The paid, ready and already-paid paths are unchanged in `src/checkout.js`. The existing tests still pass, and tests were added for the old answers being unchanged. Record: `docs/e2e-run.md`, section "D15".
- **Client reads refunds (support for D8/D10/D11 of the contract).** `readPayment` copies the refund field by field (`refunded`, `refund` with amount, currency, reason, time). A payment with no refund member, or with `null`, reads as not refunded. A malformed refund makes the answer `unexpected_response`. Card data is still only brand and last four. This is in `src/payments-api.js`.
- **D16, shown in a browser against the running service, with a run record.**
  - `e2e/checkout-run.mjs`, started with `npm run e2e` (`package.json`, Playwright 1.55.0 as a devDependency). It starts the service from its repository (`python3 -m payments --host 127.0.0.1 --port 8000 --db :memory:`) and the checkout (`python3 -m http.server 8080`), with no fake API, and drives Chromium.
  - `docs/e2e-run.md` records the commits (service `3c5293a16540`, checkout `7a9619548610` plus the T3 files), the commands and ports, and how to repeat it. It records each refund as a `curl` call to Refund payment with its payment reference and reason (`pay_1400cdbd579b`, "Out of stock"; `pay_41aae07f0f83`, "Customer cancelled"). It records the orders and cards used (`4242 4242 4242 4242`, `5555 5555 5555 4444`, `4000 0000 0000 0002`, 2599 USD) and what was seen.
  - D13 seen: "This order was refunded", "Refund date October 22, 2026" (time zone fixed to Pacific/Auckland), no form, no button, the reason not shown, one status call and no payment call.
  - D14 seen: from a stale page with the form, Pay gave the service's message "This order was refunded, so it cannot be paid again.", then the refunded panel with the date. The call was answered 409 `order_refunded`, and the order's lookup afterwards showed one payment only, so no attempt was recorded. A reload showed the same panel.
  - Console and cross-origin: no error, apart from Chromium's own console line for the 409 in D14, which the script accepts for that case only. Every service answer carried `Access-Control-Allow-Origin: *`. The record says the script passed in 3 runs in a row.
  - The README has a "Browser run against the service" section that links `docs/e2e-run.md`. It also describes the refunded state and the fixed time zone for the date test.
- **D17, existing suites pass.** `npm test` runs 120 tests with 0 failures: 95 after T1 (22 new, in `tests/payments-api.test.js`), then 25 new in `tests/checkout.test.js`. The new tests fix the time zone (`TZ=Pacific/Auckland`) and cover each state, a payment with and without a refund, the new refusal, and the old answers unchanged.

## Deviations

None from the spec. Against the service's contract:

- The checkout keys on `status` and `refunded_at` and handles `order_refunded` with `refunded_at` as the contract says. It reads `paid` only when `status` is absent, as T1's brief says for older readers.
- `package.json` gained a devDependency (Playwright) and an `e2e` script. The spec asks for a browser run, and the page and its runtime code need no dependency.
- The run record starts the service with `--db :memory:`. The contract's notes name only `--host` and `--port`, so the flag is not in the contract. The record calls this the service's documented command; I could not check it against the service's README from here.

## Open questions

- The T3 browser run found nothing wrong on the checkout's side and no service fault, so no fix was made in `src/` in T3. The record lists no open service fault. Settled by: nothing needed; a different finding would come from a re-run of `npm run e2e` against the final service commit.
- The record is for service commit `3c5293a16540` (the `sandbox-api/8/base` it names) and Chromium only. If the service's run branch moves after that commit, the record would need a re-run to stay true. Settled by: `PAYMENTS_SERVICE_DIR=../sandbox-api npm run e2e` against the final service commit, then updating the commit in `docs/e2e-run.md`.
- The refund date follows the browser's time zone, as the spec says. The run fixes it to `Pacific/Auckland` to predict the date, so the record does not show a real-zone date.
- Only a `status` of `refunded` shows the refunded panel. A `paid` status whose succeeded payment carries a refund is shown as paid. The contract says this cannot happen, so no case handles it.
