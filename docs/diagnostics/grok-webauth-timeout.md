# Grok Web Auth — `waitForFunction` Timeout Diagnosis

## Error

```
✗ Grok Web 授权失败: page.waitForFunction: Timeout 300000ms exceeded.
    at Object.loginGrokWeb (.../dist/register.onboard-ikaSNPcl.js:904:14)
```

Running `bash onboard.sh webauth` and selecting Grok always fails after 5 minutes.

## Root Cause

The original code at `src/zero-token/providers/grok-web-auth.ts:87-92` used:

```ts
await page.waitForFunction(
  () => {
    return document.cookie.includes("sso") || document.cookie.includes("_ga");
  },
  { timeout: 300000 },
);
```

The `sso` cookie is **HttpOnly** (confirmed via DevTools → Application → Cookies on `grok.com`).
`document.cookie` in JavaScript **cannot see HttpOnly cookies**, so the predicate always
returns `false` and the call times out.

`_ga` is a Google Analytics tracking cookie set on page load regardless of login state —
it is not a Grok session cookie and is an unreliable auth signal.

## Fix

Replaced the `waitForFunction` with a Playwright `context.cookies()` poll, which sees
HttpOnly cookies:

```ts
async function waitForGrokSession(
  context: { cookies: (urls: string | string[]) => Promise<Array<{ name: string }>> },
  timeout = 300000,
): Promise<void> {
  const start = Date.now();
  while (Date.now() - start < timeout) {
    const cookies = await context.cookies("https://grok.com");
    if (cookies.some((c) => c.name === "sso")) {
      return;
    }
    await new Promise((r) => setTimeout(r, 2000));
  }
  throw new Error("Timed out waiting for Grok SSO cookie");
}
```

Then call it as `await waitForGrokSession(context);`.

## How to Inspect Auth Cookies in Chrome DevTools

1. Open `https://grok.com` in Chrome debug mode and log in.
2. Press `F12` → **Application** tab → **Cookies** → expand `https://grok.com`.
3. Look for cookies that appear only after login. Note the **Name**, **Domain**, and
   whether **HttpOnly** is checked.
4. HttpOnly cookies are invisible to `document.cookie` — use Playwright's
   `context.cookies()` API to read them in automation code.

## Files Changed

- `src/zero-token/providers/grok-web-auth.ts` — added `waitForGrokSession` helper,
  replaced broken `waitForFunction` call.
