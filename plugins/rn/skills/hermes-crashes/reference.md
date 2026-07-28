# Error normalization patterns

Copy-paste candidates for the capture layer. Plain TypeScript with no SDK
types; the reporter calls are shown generically, adjust the options object
to whatever your reporter accepts.

## normalizeError

Turns anything thrown or rejected into a real Error with a stable name, so
the reporter groups by error identity instead of by capture callsite.

```ts
export function normalizeError(input: unknown): Error {
  if (input instanceof Error) return input;

  if (typeof input === 'object' && input !== null) {
    const obj = input as { message?: unknown; code?: unknown };
    const message =
      typeof obj.message === 'string' ? obj.message : safeStringify(input);
    const err = new Error(message);
    // A code inside the name splits dashboard issues correctly:
    // "Error [23505]" groups apart from "Error [PGRST301]".
    err.name =
      typeof obj.code === 'string' && obj.code.length > 0
        ? `Error [${obj.code}]`
        : 'NonErrorThrown';
    (err as Error & { cause?: unknown }).cause = input;
    return err;
  }

  return new Error(`Non-error thrown: ${String(input)}`);
}

function safeStringify(value: unknown): string {
  try {
    return JSON.stringify(value) ?? String(value);
  } catch {
    return String(value);
  }
}
```

## Keeping network noise out of the error feed

```ts
export function isAbortError(error: unknown): boolean {
  if (typeof error !== 'object' || error === null) return false;
  if ((error as { name?: unknown }).name === 'AbortError') return true;
  const message = (error as { message?: unknown }).message;
  return typeof message === 'string' && message.toLowerCase().includes('abort');
}
```

At every capture site:

```ts
if (isAbortError(error)) {
  // One fixed fingerprint per flow: every timeout lands in ONE issue you
  // can watch, instead of hundreds of issues you learn to ignore.
  capture(normalizeError(error), {
    level: 'warning',
    fingerprint: ['network-abort', flowName],
  });
} else {
  capture(normalizeError(error), { tags: { flow: flowName } });
}
```

The two functions are boring on purpose. They earn their keep the first
week a release meets bad hotel wifi.
