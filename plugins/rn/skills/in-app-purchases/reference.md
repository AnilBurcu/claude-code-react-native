# Webhook handler skeleton

The shape of a subscription webhook endpoint that stays correct under
retries, replays and out-of-order delivery. Standard Request/Response API,
so it runs on Node 18+, Deno and edge runtimes unchanged. The field names
follow RevenueCat's webhook payload; map them if your provider differs.
The db functions are yours to implement; their contracts are the point.

```ts
export async function handleSubscriptionWebhook(req: Request): Promise<Response> {
  // 1. Authenticate the sender. The header value is exactly the string you
  // configured in the provider's dashboard; compare the whole value.
  if (req.headers.get('authorization') !== WEBHOOK_AUTH_VALUE) {
    return new Response('unauthorized', { status: 401 });
  }

  const { event } = await req.json();

  // 2. Dedupe by event id. Providers retry on any non-2xx response and on
  // timeouts you may have already processed. Answer duplicates with 200 so
  // the sender stops resending.
  const firstTime = await recordEventOnce(event.id, event.environment);
  if (!firstTime) {
    return new Response('duplicate', { status: 200 });
  }

  // 3. Branch by event type BEFORE reading optional fields. Only a handful
  // of fields exist on every event; TRANSFER, for example, carries
  // transferred_from/transferred_to instead of a single app_user_id, and
  // no product_id.
  if (event.type === 'TRANSFER') {
    await applyTransfer({
      fromUserIds: event.transferred_from ?? [],
      toUserIds: event.transferred_to ?? [],
      environment: event.environment,
    });
    return new Response('ok', { status: 200 });
  }

  // 4. Upsert to the state the event describes. Never increment, never
  // toggle: replays and reordering must converge to the same row.
  await upsertSubscription({
    userId: event.app_user_id,
    productId: event.product_id,
    status: statusFromEventType(event.type),
    expiresAt: event.expiration_at_ms ? new Date(event.expiration_at_ms) : null,
    // SANDBOX or PRODUCTION. Store it on the row and filter every admin
    // query and revenue metric by it, or sandbox testing pollutes reality.
    environment: event.environment,
  });

  return new Response('ok', { status: 200 });
}
```

Contracts that make the skeleton hold:

- `recordEventOnce`: an atomic insert against a unique constraint on the
  event id, returning false on conflict. A read-then-write check is a race,
  not a dedupe.
- `upsertSubscription`: one row per user and product; applying the same
  event twice produces the same row.
- Field presence is per event type. Treat everything beyond id, type and
  timestamp as optional and validate before use; a handler that assumes
  every field on every event works until the first TRANSFER.
- The failure window: if processing dies AFTER the dedupe insert but BEFORE
  the upsert, a retry would be swallowed as a duplicate and the event is
  lost. Close it by writing the dedupe record and the subscription change
  in one transaction, or by deleting the dedupe record when processing
  throws.
