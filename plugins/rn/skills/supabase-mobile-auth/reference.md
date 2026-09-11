# Client wiring patterns

The four pieces that keep mobile sessions alive. Everything here leans on
contracts that have been stable for years: the three-method storage
adapter, the auth lock signature, AppState, AbortController. The storage
engine is swappable; MMKV is the example, not a requirement.

## Storage adapter

```ts
import { MMKV } from 'react-native-mmkv';

const storage = new MMKV({ id: 'auth', encryptionKey: deviceKeychainKey });

export const authStorage = {
  getItem: (key: string) => storage.getString(key) ?? null,
  setItem: (key: string, value: string) => {
    storage.set(key, value);
  },
  removeItem: (key: string) => {
    storage.delete(key);
  },
};
```

Any storage exposing these three methods works. SecureStore fits the
contract, but some iOS releases have historically refused values around
2 KB and a session does not fit that comfortably; Supabase's own docs
suggest AsyncStorage or an encrypted wrapper around it. Encrypted MMKV
gets you the same result with faster reads.

## Refresh lifecycle tied to app state

```ts
import { AppState } from 'react-native';

AppState.addEventListener('change', (state) => {
  if (state === 'active') {
    supabase.auth.startAutoRefresh();
  } else {
    supabase.auth.stopAutoRefresh();
  }
});
```

A refresh firing while the app is suspended can rotate the token without
persisting the response; the next launch then presents the stale token and
trips refresh-reuse detection. Pausing in the background is the fix.

## Serializing refreshes

Two concurrent refresh attempts are the other reuse-detection trigger.
supabase-js ships an official lock for exactly this: `import { processLock }
from '@supabase/supabase-js'` and pass it as `auth.lock`. Reach for the
hand-rolled version below only if you want to see what a lock does or need
custom behavior:

```ts
let queue: Promise<unknown> = Promise.resolve();

export const serialLock = async <R>(
  _name: string,
  _acquireTimeout: number,
  fn: () => Promise<R>,
): Promise<R> => {
  const run = queue.then(fn);
  queue = run.catch(() => undefined);
  return run;
};
```

It ignores the acquire timeout and runs callers one after another, which
is the behavior a mobile app wants. A hung callback cannot block the
chain forever because every network call underneath carries the fetch
timeout below.

## Fetch timeout

The subtle part: replacing `init.signal` outright would silently disable
every per-query `abortSignal()` the app sets, so the caller's signal has to
be forwarded into the timeout controller, reason included, so a cancelled
query still surfaces the caller's own reason rather than looking like a
timeout. Runtimes whose AbortController ignores the reason argument simply
degrade to a plain abort.

```ts
const fetchWithTimeout =
  (ms: number) =>
  (input: RequestInfo | URL, init?: RequestInit): Promise<Response> => {
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), ms);

    const upstream = init?.signal;
    if (upstream) {
      if (upstream.aborted) {
        controller.abort(upstream.reason);
      } else {
        upstream.addEventListener('abort', () => controller.abort(upstream.reason), { once: true });
      }
    }

    return fetch(input, { ...init, signal: controller.signal }).finally(() =>
      clearTimeout(timer),
    );
  };
```

## Putting it together

```ts
export const supabase = createClient(url, anonKey, {
  auth: {
    storage: authStorage,
    lock: serialLock,
    autoRefreshToken: true,
    persistSession: true,
    detectSessionInUrl: false, // token-based native flows only
  },
  global: {
    fetch: fetchWithTimeout(20_000),
  },
});
```

Resulting behavior: sessions survive restarts in encrypted storage,
refreshes stop while backgrounded and never race each other, and no
request can hang past 20 seconds on a stalled connection.
