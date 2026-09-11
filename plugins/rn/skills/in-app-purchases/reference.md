# Webhook handler skeleton

The shape of a subscription webhook endpoint that stays correct under
retries, replays and out-of-order delivery. Standard Request/Response API
plus Web Crypto, so it runs on Node 18+, Deno and edge runtimes unchanged.
The field names and the signature header follow RevenueCat; map them if
your provider differs. The db functions are yours to implement; their
contracts are the point.

```ts
export async function handleSubscriptionWebhook(req: Request): Promise<Response> {
  // 1. Authenticate the sender. Read the body as text BEFORE parsing: the
  // signature covers the raw bytes, and a re-serialized object will not
  // match. Keep the Authorization check as well; it costs nothing.
  const rawBody = await req.text();
  if (req.headers.get('authorization') !== WEBHOOK_AUTH_VALUE) {
    return new Response('unauthorized', { status: 401 });
  }
  const signature = req.headers.get('x-revenuecat-webhook-signature');
  if (!(await verifySignature(signature, rawBody, WEBHOOK_SIGNING_SECRET))) {
    return new Response('bad signature', { status: 401 });
  }

  const { event } = JSON.parse(rawBody);

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

## Signature verification

RevenueCat's optional HMAC signing sends
`X-RevenueCat-Webhook-Signature: t=<unix seconds>,v1=<hex>`, where v1 is
HMAC-SHA256 over the string `<t>.<raw body>` using the signing secret from
the integration settings (shown once; rotating it invalidates the old one
immediately). Retries are re-signed with a fresh timestamp, so a small
tolerance never rejects a legitimate retry.

```ts
async function verifySignature(
  header: string | null,
  rawBody: string,
  secret: string,
  toleranceSeconds = 300,
): Promise<boolean> {
  if (!header) return false;
  const parts = new Map(header.split(',').map((kv) => kv.split('=', 2) as [string, string]));
  const t = parts.get('t');
  const v1 = parts.get('v1');
  if (!t || !v1) return false;
  if (Math.abs(Date.now() / 1000 - Number(t)) > toleranceSeconds) return false;

  const encoder = new TextEncoder();
  const key = await crypto.subtle.importKey(
    'raw',
    encoder.encode(secret),
    { name: 'HMAC', hash: 'SHA-256' },
    false,
    ['sign'],
  );
  const mac = await crypto.subtle.sign('HMAC', key, encoder.encode(`${t}.${rawBody}`));
  const expected = Array.from(new Uint8Array(mac), (b) => b.toString(16).padStart(2, '0')).join('');
  return constantTimeEqual(expected, v1.toLowerCase());
}

function constantTimeEqual(a: string, b: string): boolean {
  if (a.length !== b.length) return false;
  let diff = 0;
  for (let i = 0; i < a.length; i++) diff |= a.charCodeAt(i) ^ b.charCodeAt(i);
  return diff === 0;
}
```

## Contracts that make the skeleton hold

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
- The 60-second clock: RevenueCat disconnects after 60 seconds and retries
  the delivery up to 5 times (5, 10, 20, 40 and 80 minutes apart), then
  stops. Return 200 as soon as the dedupe record and the upsert are
  committed and queue anything slow, or the retries become duplicates and
  the fifth failure becomes a lost event.
