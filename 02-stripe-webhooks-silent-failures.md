# Stripe webhooks that return 200 and do nothing

The worst webhook bugs look healthy. Stripe gets a 200, the logs stay clean, the customer pays and nothing activates. I hit these four while testing a subscription system.

## A field moved between API versions

`invoice.subscription` no longer exists in API version `2025-03-31.basil`. The value now lives at `invoice.parent.subscription_details.subscription`.

Events are serialized with the API version of the account or the endpoint, not your SDK's. Code reading the old field found nothing and exited normally.

**Fix:** a helper that reads both locations, and a pinned API version on the webhook endpoint.

## A restricted key looked like test mode

The webhook compared its mode against the key with `startsWith('sk_live_')`. A restricted live key starts with `rk_live_`, so the webhook decided it was in test mode and discarded every live event.

**Fix:** `/^(sk|rk)_live_/.test(key)`.

## The idempotency claim could be orphaned

The handler claimed each event with a `create()` and then processed it. If the instance died between those two steps, Stripe's retry saw the claim and treated the event as a duplicate.

**Fix:** claim with `status: "processing"`, seal with `status: "done"`, and treat only `done` as a duplicate.

## One payment fired two updates

`checkout.session.completed` and `invoice.paid` arrive almost together. With a separate read and write, both saw the old state and both logged an extension.

**Fix:** apply the change inside a transaction, so the second event sees the new state and exits.

## How to test this

- [Test clocks](https://docs.stripe.com/billing/testing/test-clocks) simulate renewals, failed retries and cancellation without waiting a month.
- The test card `4000 0000 0000 0341` attaches successfully and fails on renewal.
- `stripe listen` uses a different signing secret from the Dashboard endpoint.
