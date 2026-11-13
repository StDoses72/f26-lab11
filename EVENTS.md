# EVENTS.md

One section per milestone. Fill each one in order as you go.

The prediction in milestone 2 should be written before you run the replay. That is the
part a TA will ask about.

Keep it short and specific. Point at operations, call sites, and what a caller
gets back.

---

## Milestone 1: what runs in the request path, and what runs after it

### The operations table

One row per operation. Read the code, not the file
names. "As wired today" is what the code does now. "Your call" is what you
would do if this were your service. List up to eight operations, and remove
the rows you don't use.

| # | Operation | As wired today | Your call | Why (in one line) |
|---|---|---|---|---|
| 1 | Update the search index | Direct call in `BookingService.book`, before `store.save` (`src/service/bookingService.ts:94`). | Event-driven: subscribe to `booking.confirmed`, after the booking is saved. | Index a confirmed booking after saving it; a search outage should not prevent a valid reservation. |
| 2 | Check for a conflicting booking | Direct call to `conflictingBooking(this.store.all(), ...)` (`src/service/bookingService.ts:71`). | Keep synchronous in `book`, before accepting the booking. | The caller needs to know the room is free before receiving confirmation. |
| 3 | Save the confirmed booking | Direct call to `this.store.save(booking)` (`src/service/bookingService.ts:95`). | Keep synchronous, before publishing confirmation. | A successful response must correspond to a booking already recorded in the store. |
| 4 | Send confirmation and cancellation emails | `EmailConfirmationHandler` subscribes to `booking.confirmed` and `booking.cancelled` (`src/app.ts:76`, `src/app.ts:81`). | Keep event-driven; arrange retries for failed notifications. | A saved reservation is successful even if notification fails; the email should be retried. |
| 5 | Generate a downloadable receipt | `ReceiptGenerator` subscribes to `booking.confirmed` (`src/app.ts:77`). | Make receipt readiness a synchronous prerequisite for reporting full completion. | From the user's perspective, a completed confirmation should include a receipt ready to download. |
| 6 | Schedule or remove a reminder | `ReminderScheduler` subscribes to confirmation and cancellation events (`src/app.ts:78`, `src/app.ts:82`). | Keep event-driven after the booking state changes. | A reminder-scheduling failure should be recovered without preventing the reservation change. |
| 7 | Refresh dashboard statistics | `StatsUpdater` subscribes to confirmation and cancellation events (`src/app.ts:79`, `src/app.ts:83`). | Keep event-driven. | Dashboard figures can catch up from the store without blocking a booking or cancellation. |
| 8 | Record and pay a refund | `cancel` publishes `refund.requested`, handled by `RefundLedger` (`src/service/bookingService.ts:169`, `src/app.ts:85`). | Keep event-driven; accept a successful cancellation with the refund pending if payment fails. | Releasing the room should not wait for the payment provider; unfinished refunds need separate recovery. |

The current bus records failed subscriber deliveries in its dead-letter list
but does not retry automatically. Retrying a failed notification would require
an explicit recovery step such as redelivering the original message.

The receipt choice assumes that full completion includes a downloadable
receipt. If rendering fails after saving, the booking still exists; the
caller must distinguish that partial result from a reservation that was
never saved.

### Two calls in detail

Pick two rows and give each a paragraph rather than a line. One of them has to
be the operation `TICKET.md` implicates.

**Search index update: event-driven after saving.** The caller needs a
confirmed reservation recorded in the booking store before `book` returns.
The search index should be updated by an observer of `booking.confirmed`,
after that booking has been saved. Today, an unavailable search cluster
throws before the save, so the caller receives an error and has no booking.
With an event subscriber, the bus would record an index-update failure as a
dead letter while `book` still returns the saved booking. The search view
could lag until that failed work is recovered. Moving the direct call after
the save alone would still let its exception reach the caller; the bus's
handler-failure isolation matters as well.

**Conflict check: synchronous before accepting.** The caller must know whether
the requested room and time are available before receiving confirmation.
`conflictingBooking` checks the actual booking store, and `book` throws a
`BookingError` when an existing booking overlaps the request. I would keep
this check in the request path: confirming first and checking through an
event afterwards could accept a request that should have been rejected.
The search index is not needed for this check.

### The incident

**The call behind `TICKET.md`.** Name the file, the method,
and the line of the call the incident report is about, and describe what the
caller of that method gets back when that call throws.

`BookingService.book` calls `this.searchIndex.indexBooking(booking, room, now)`
at `src/service/bookingService.ts:94`, before saving the booking on line 95.
If the search cluster is unavailable, this call throws
`SearchIndexUnavailableError`. `book` propagates that error to its caller
instead of returning a `Booking`; the booking is not saved and no
`booking.confirmed` event is published. `cancel` does not call the search
index, which explains why cancellations continued to work during the outage.

---

## Milestone 2: what a second delivery does

### Your prediction (before you run anything)

**What you expect `redeliver` on a refund message to do.** Name what you expect
to change in the ledger, at the payment provider, and in the bus log. Write it
down, then run the demo.

My prediction: redelivering the same refund message will refund the
organizer again and append another ledger entry for the same amount.

I was initially unsure about the bus log. Before running the demo, a code
walkthrough with the agent showed that `publish` appends a new message, while
`redeliver` looks up the existing message and calls `deliver` again. I therefore
expect the refund message to keep the same ID and occupy one log entry, with
its delivery count increasing on each redelivery.

### Before your fix

**The run.** Paste what `npm run demo:replay` printed without editing it. A TA
may ask you about it in milestone 3.

```text

> campus-reservations@1.0.0 demo:replay
> tsx src/demos/replayDemo.ts

booked bk-1001 for 8500 cents
cancelled bk-1001, refund 8500 cents, refund message msg-3

=== after the cancellation (one delivery of the refund message) ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 1
gateway settlements: 1
  stl-1 bk-1001 $85.00
money returned in total: $85.00
dead letters: 0

redelivering msg-3

=== after one redelivery ===
ledger entries: 2
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
  led-2 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 2
gateway settlements: 2
  stl-1 bk-1001 $85.00
  stl-2 bk-1001 $85.00
money returned in total: $170.00
dead letters: 0

redelivering msg-3 again

=== after two redeliveries ===
ledger entries: 3
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
  led-2 bk-1001 $85.00 (cancelled more than two hours ahead)
  led-3 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 3
gateway settlements: 3
  stl-1 bk-1001 $85.00
  stl-2 bk-1001 $85.00
  stl-3 bk-1001 $85.00
money returned in total: $255.00
dead letters: 0

=== bus log ===
  msg-1 booking.confirmed deliveries=1
  msg-2 booking.cancelled deliveries=1
  msg-3 refund.requested deliveries=3
```

**Your prediction against the run.** Where they agree and where they do not.

The run matches the predicted repeated refunds and ledger entries: the three
states have 1, 2, and 3 ledger entries and settlements, with totals of $85.00,
$170.00, and $255.00. The bus-log behavior matches the expectation from our
pre-run walkthrough: there are three published messages, with one refund
message, `msg-3`, delivered three times. No delivery throws, so the dead-letter
list stays empty even though the business effect is wrong.

### Your fix

**The diff.** Paste it.

```diff
diff --git a/src/handlers/refundLedger.ts b/src/handlers/refundLedger.ts
index 600ffca..3a13f19 100644
--- a/src/handlers/refundLedger.ts
+++ b/src/handlers/refundLedger.ts
@@ -18,4 +18,4 @@ export interface LedgerEntry {
- * For a refund message the handler writes a ledger entry, then calls the
- * payment provider for the same amount. The ledger is append only: entries are
- * numbered in the order they were written, and nothing edits or removes one
- * after the fact.
+ * For a refund message the handler calls the payment provider, then records
+ * the settled refund. Completed message IDs prevent duplicate processing in
+ * this instance. The ledger is append only: entries are numbered in the order
+ * they were written, and nothing edits or removes one after the fact.
@@ -26,0 +27 @@ export class RefundLedger implements Subscriber<RefundRequested> {
+  private readonly completedMessageIds = new Set<string>();
@@ -33,0 +35,4 @@ export class RefundLedger implements Subscriber<RefundRequested> {
+    if (this.completedMessageIds.has(message.id)) {
+      return;
+    }
+
@@ -34,0 +40 @@ export class RefundLedger implements Subscriber<RefundRequested> {
+    this.gateway.refund(event.bookingId, event.amountCents);
@@ -43 +49 @@ export class RefundLedger implements Subscriber<RefundRequested> {
-    this.gateway.refund(event.bookingId, event.amountCents);
+    this.completedMessageIds.add(message.id);
```

**What your fix uses to decide what to do on a second delivery, and what it
does not protect.** One or two lines.

The key is `message.id`, kept in an in-memory set only after payment and
ledger recording succeed; a completed ID is skipped on later deliveries.
It does not protect a new message ID for the same refund, process restarts,
or a real provider settling before local completion is recorded.

The provider call now comes before the ledger append. In this lab, the
provider's injected failure happens before settlement, so a failed attempt
leaves neither a ledger entry nor a completed ID and can be retried.

### After your fix

**The run.** Paste what `npm run demo:replay` printed on the fixed code.

```text

> campus-reservations@1.0.0 demo:replay
> tsx src/demos/replayDemo.ts

booked bk-1001 for 8500 cents
cancelled bk-1001, refund 8500 cents, refund message msg-3

=== after the cancellation (one delivery of the refund message) ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 1
gateway settlements: 1
  stl-1 bk-1001 $85.00
money returned in total: $85.00
dead letters: 0

redelivering msg-3

=== after one redelivery ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 1
gateway settlements: 1
  stl-1 bk-1001 $85.00
money returned in total: $85.00
dead letters: 0

redelivering msg-3 again

=== after two redeliveries ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 1
gateway settlements: 1
  stl-1 bk-1001 $85.00
money returned in total: $85.00
dead letters: 0

=== bus log ===
  msg-1 booking.confirmed deliveries=1
  msg-2 booking.cancelled deliveries=1
  msg-3 refund.requested deliveries=3
```

**What stays the same across its three states.** Name the numbers you
compared.

After cancellation, one redelivery, and two redeliveries, there is always
one ledger entry for $85.00, one gateway call, one settlement, and $85.00
returned in total, with zero dead letters. The bus still holds the same
three published messages, and the refund message's delivery count reaches 3.

Verification: two new regression tests failed on the original handler. With
the fix, all 35 tests pass, including repeated deliveries of distinct refund
messages and retry after a provider failure; `npm run typecheck` also passes.

---

## Milestone 3: a delivery that stops halfway

### The failure

**The run.** Run `npm run demo:failure` and paste what the delivery left
behind (the ledger, the provider, and the dead-letter list).

```text

> campus-reservations@1.0.0 demo:failure
> tsx src/demos/failureDemo.ts

armed: call 1 to the payment provider will throw
booked bk-1001 for 8500 cents
cancelled bk-1001, refund 8500 cents, refund message msg-3
the cancel call itself returned: booking is cancelled

=== after the failed delivery ===
ledger entries: 0
gateway calls: 1
gateway settlements: 0
money returned in total: $0.00
dead letters: 1
  msg-3 refund.requested attempt 1 in RefundLedger: payment provider did not answer on call 1 for bk-1001
    failedAtMinute: 480
    payload: {"bookingId":"bk-1001","organizer":"H. Ahmad","organizerEmail":"hahmad@example.edu","amountCents":8500,"reason":"cancelled more than two hours ahead","requestedAtMinute":480}

=== bus log ===
  msg-1 booking.confirmed deliveries=1
  msg-2 booking.cancelled deliveries=1
  msg-3 refund.requested deliveries=1

the refund message is in the log and in the dead-letter list. Milestone 3 asks what you do with it.
```

### Your recovery

**What, if anything, the failed delivery left undone.**

Cancellation succeeded, but the organizer is still owed 8500 cents. The
provider threw before settlement, so there are zero settlements, zero ledger
entries, and no completed ID for this refund. The bus recorded the failed
first delivery of `msg-3` in its dead-letter list; it will not retry on its own.

**The option you took and its evidence.** If you redelivered the
message, paste the state after the redelivery and the bus log line for that
message. If you left the message for a person to handle, write the runbook line
they would follow, and name which fields of the dead-lettered message they read to
follow it and what each one tells them.

Recovery chosen with the agent: redeliver the original refund message once
at the end of `src/demos/failureDemo.ts`. The injected failure occurs before
settlement and affects only the first provider call. The failed attempt was
not marked completed, so the same message can finish its work on retry. We
reuse its ID rather than publishing a second refund request.

```text
redelivering msg-3 after the provider failure

=== after recovery by redelivery ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 2
gateway settlements: 1
  stl-1 bk-1001 $85.00
money returned in total: $85.00
dead letters: 1
  msg-3 refund.requested attempt 1 in RefundLedger: payment provider did not answer on call 1 for bk-1001
    failedAtMinute: 480
    payload: {"bookingId":"bk-1001","organizer":"H. Ahmad","organizerEmail":"hahmad@example.edu","amountCents":8500,"reason":"cancelled more than two hours ahead","requestedAtMinute":480}

=== bus log ===
  msg-1 booking.confirmed deliveries=1
  msg-2 booking.cancelled deliveries=1
  msg-3 refund.requested deliveries=2
```

The refund message now has `deliveries=2`: its first delivery failed and its
second succeeded. There is one ledger entry and one $85.00 settlement across
two provider calls. The dead-letter count remains 1 because it is a history
of failed deliveries, not a pending-work queue; redelivery does not remove
the earlier failure record.

### The other option

**What the option you did not take would have cost.** One line, in terms of who
does the work, when the organizer gets their money, and what has to be true for
it to be safe.

Manual triage would require an operator to reconcile the failed message with
the provider settlements and ledger, delaying the $85 refund until they act;
safe recovery would require confirming whether money already moved and
avoiding a second settlement.
