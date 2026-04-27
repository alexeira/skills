---
name: better-auth-device-authorization
description: Implement OAuth 2.0 Device Authorization Grant (RFC 8628) in Better Auth for limited-input devices — CLIs, smart TVs, IoT, gaming consoles. Use this whenever the user mentions "device flow", "device authorization", "device code", `deviceAuthorization` plugin, RFC 8628, CLI login/auth, smart TV auth, IoT auth, or needs to authenticate a device that can't easily render a browser. Covers server + client plugin setup, the verification page, the approve/deny page, polling with proper error handling, and CLI integration.
version: 1.0.0
---

# Better Auth — Device Authorization

Implements [RFC 8628](https://datatracker.ietf.org/doc/html/rfc8628) via Better Auth's `deviceAuthorization` plugin. Use this when one device (CLI, TV, console, IoT) needs to authenticate a user who completes the approval on a *separate* device with a real browser.

## The flow at a glance

1. **Device requests codes** → server returns `device_code`, `user_code`, `verification_uri`, `verification_uri_complete`.
2. **Show user** the `user_code` and `verification_uri` (or open `verification_uri_complete` directly).
3. **Device polls** the token endpoint at the server-suggested interval.
4. **User authenticates** in their browser, enters the code, approves or denies.
5. **Polling resolves** with an `access_token` (or `access_denied` / `expired_token`).

To use the access token against your API, the **Bearer plugin** must also be enabled. Without it, the token is issued but downstream API calls have nothing to validate it.

## 1. Server setup

```ts title="auth.ts"
import { betterAuth } from "better-auth";
import { deviceAuthorization, bearer } from "better-auth/plugins";

export const auth = betterAuth({
  // ...rest of config
  plugins: [
    deviceAuthorization({
      verificationUri: "/device", // page where users will enter the code
      // optional advanced hooks — see §6
    }),
    bearer(), // required if the device will call your API with the access_token
  ],
});
```

Then run the migration to create the device-flow tables:

```bash
npx auth migrate
# or, to inspect first:
npx auth generate
```

## 2. Client setup

```ts title="auth-client.ts"
import { createAuthClient } from "better-auth/client";
import { deviceAuthorizationClient } from "better-auth/client/plugins";

export const authClient = createAuthClient({
  baseURL: process.env.AUTH_BASE_URL, // e.g. https://api.example.com
  plugins: [deviceAuthorizationClient()],
});
```

## 3. Device side — request code + poll

```ts
import { authClient } from "./auth-client";
import open from "open"; // optional, for CLIs

const CLIENT_ID = "your-client-id";

async function login() {
  const { data, error } = await authClient.device.code({
    client_id: CLIENT_ID,
    scope: "openid profile email", // optional
  });
  if (error || !data) throw new Error(error?.error_description ?? "device.code failed");

  console.log(`\nGo to: ${data.verification_uri}`);
  console.log(`And enter code: ${data.user_code}\n`);
  // Or jump straight there:
  await open(data.verification_uri_complete).catch(() => {});

  return pollForToken({
    deviceCode: data.device_code,
    interval: data.interval ?? 5, // server-suggested polling cadence (seconds)
  });
}
```

### Polling — handle every documented error

Polling is the part that goes wrong most often. The spec defines five errors and each one means something specific:

| Error                   | What it means                          | What to do                                |
| ----------------------- | -------------------------------------- | ----------------------------------------- |
| `authorization_pending` | User hasn't approved yet               | Keep polling at current interval          |
| `slow_down`             | You're polling too fast                | Add ≥5s to your interval and continue     |
| `expired_token`         | `device_code` expired before approval  | Stop. Restart the whole flow              |
| `access_denied`         | User clicked Deny                      | Stop. Surface a friendly message          |
| `invalid_grant`         | Bad `device_code` / `client_id` combo  | Stop. This is a bug in the device         |

Treating any of these as a generic "retry" is wrong — `slow_down` will keep snowballing, and `expired_token` will spin forever.

```ts
type PollArgs = { deviceCode: string; interval: number };

async function pollForToken({ deviceCode, interval }: PollArgs) {
  while (true) {
    const { data, error } = await authClient.device.token({
      grant_type: "urn:ietf:params:oauth:grant-type:device_code",
      device_code: deviceCode,
      client_id: CLIENT_ID,
      fetchOptions: {
        headers: { "user-agent": "my-cli/1.0.0" },
      },
    });

    if (data?.access_token) return data;

    switch (error?.error) {
      case "authorization_pending":
        break;
      case "slow_down":
        interval += 5;
        break;
      case "access_denied":
        throw new Error("User denied the authorization request.");
      case "expired_token":
        throw new Error("The code expired before you finished. Run login again.");
      case "invalid_grant":
        throw new Error("Invalid device or client. Check your client_id.");
      default:
        throw new Error(error?.error_description ?? "Unknown polling error.");
    }

    await new Promise((r) => setTimeout(r, interval * 1000));
  }
}
```

## 4. Verification page (`/device`)

Where users land to type in the code. The path here must match the `verificationUri` you configured on the server. Format the input forgivingly — strip dashes, uppercase — because users will type what they see (`ABCD-1234`) but the server stores it normalized.

```tsx title="app/device/page.tsx"
"use client";
import { useState } from "react";
import { authClient } from "@/lib/auth-client";

export default function DevicePage() {
  const [userCode, setUserCode] = useState("");
  const [error, setError] = useState<string | null>(null);

  async function onSubmit(e: React.FormEvent) {
    e.preventDefault();
    setError(null);

    const formatted = userCode.trim().replace(/-/g, "").toUpperCase();
    const res = await authClient.device({ query: { user_code: formatted } });

    if (res.data) {
      window.location.href = `/device/approve?user_code=${formatted}`;
    } else {
      setError("Invalid or expired code");
    }
  }

  return (
    <form onSubmit={onSubmit}>
      <label htmlFor="code">Device code</label>
      <input
        id="code"
        value={userCode}
        onChange={(e) => setUserCode(e.target.value)}
        placeholder="ABCD-1234"
        autoComplete="off"
        autoCapitalize="characters"
        inputMode="text"
        maxLength={12}
      />
      <button type="submit">Continue</button>
      {error && <p role="alert">{error}</p>}
    </form>
  );
}
```

## 5. Approve / Deny page

The user **must be authenticated** before approving — otherwise anyone could approve a device for any account. If there's no session, redirect to login with a return URL so they come back here after signing in.

```tsx title="app/device/approve/page.tsx"
"use client";
import { useState } from "react";
import { useSearchParams } from "next/navigation";
import { authClient } from "@/lib/auth-client";

export default function DeviceApprovePage() {
  const params = useSearchParams();
  const userCode = params.get("user_code") ?? "";
  const { data: session } = authClient.useSession();
  const [busy, setBusy] = useState(false);

  if (!session?.user) {
    const returnTo = encodeURIComponent(`/device/approve?user_code=${userCode}`);
    if (typeof window !== "undefined") {
      window.location.href = `/login?redirect=${returnTo}`;
    }
    return null;
  }

  async function decide(action: "approve" | "deny") {
    setBusy(true);
    try {
      if (action === "approve") {
        await authClient.device.approve({ userCode });
      } else {
        await authClient.device.deny({ userCode });
      }
      window.location.href = "/device/done";
    } finally {
      setBusy(false);
    }
  }

  return (
    <div>
      <h1>Authorize device?</h1>
      <p>A device is requesting access to your account.</p>
      <p>Code: <code>{userCode}</code></p>
      <button onClick={() => decide("approve")} disabled={busy}>Approve</button>
      <button onClick={() => decide("deny")} disabled={busy}>Deny</button>
    </div>
  );
}
```

Server-side equivalents (require session cookies):

```ts
import { headers } from "next/headers";

await auth.api.deviceApprove({ body: { userCode }, headers: await headers() });
await auth.api.deviceDeny({ body: { userCode }, headers: await headers() });
```

## 6. Advanced configuration

### Restrict which clients can use the device flow

Not every OAuth client should be allowed to do device flow — it's specifically for input-constrained devices, and you usually want to gate it.

```ts
deviceAuthorization({
  verificationUri: "/device",
  validateClient: async (clientId) => {
    const client = await db.oauthClients.findFirst({ where: { id: clientId } });
    return !!client?.allowDeviceFlow;
  },
  onDeviceAuthRequest: async (clientId, scope) => {
    // Audit log — useful for spotting abuse / unexpected client IDs
    await auditLog.write({ event: "device_auth_request", clientId, scope });
  },
});
```

### Custom code generation

The default `user_code` charset is `ABCDEFGHJKLMNPQRSTUVWXYZ23456789` — note the deliberate exclusion of `0/O/1/I` so a user reading from a TV at 3 meters can type it without ambiguity. If you customize it, **keep that property**.

```ts
deviceAuthorization({
  verificationUri: "/device",

  generateDeviceCode: async () => {
    const { randomBytes } = await import("node:crypto");
    return randomBytes(32).toString("hex");
  },

  generateUserCode: async () => {
    const charset = "ABCDEFGHJKLMNPQRSTUVWXYZ23456789"; // no confusables
    const { randomInt } = await import("node:crypto"); // unbiased, unlike Math.random()
    let code = "";
    for (let i = 0; i < 8; i++) code += charset[randomInt(charset.length)];
    return code;
  },
});
```

`Math.random()` is fine for the demo in the official docs but inappropriate for security tokens — `crypto.randomInt` is unbiased and cryptographically strong.

## 7. Full CLI example

```ts title="cli/login.ts"
#!/usr/bin/env node
import { createAuthClient } from "better-auth/client";
import { deviceAuthorizationClient } from "better-auth/client/plugins";
import open from "open";

const authClient = createAuthClient({
  baseURL: process.env.AUTH_BASE_URL ?? "http://localhost:3000",
  plugins: [deviceAuthorizationClient()],
});

const CLIENT_ID = process.env.CLI_CLIENT_ID!;

async function main() {
  const { data, error } = await authClient.device.code({ client_id: CLIENT_ID });
  if (error || !data) {
    console.error("Could not start device login:", error?.error_description);
    process.exit(1);
  }

  console.log("\nVisit: " + data.verification_uri);
  console.log("Code:  " + data.user_code + "\n");
  await open(data.verification_uri_complete).catch(() => {});

  let interval = data.interval ?? 5;
  while (true) {
    const res = await authClient.device.token({
      grant_type: "urn:ietf:params:oauth:grant-type:device_code",
      device_code: data.device_code,
      client_id: CLIENT_ID,
      fetchOptions: { headers: { "user-agent": "my-cli/1.0.0" } },
    });

    if (res.data?.access_token) {
      // Persist the token securely — keychain on macOS, libsecret on Linux,
      // Credential Manager on Windows. Plain ~/.config files are convenient
      // but readable by other processes — only acceptable for low-value tokens.
      console.log("Logged in!");
      return;
    }

    switch (res.error?.error) {
      case "authorization_pending": break;
      case "slow_down": interval += 5; break;
      case "access_denied": console.error("Denied."); process.exit(1);
      case "expired_token": console.error("Code expired, try again."); process.exit(1);
      default: console.error(res.error?.error_description); process.exit(1);
    }
    await new Promise((r) => setTimeout(r, interval * 1000));
  }
}

main();
```

### Using the access token

With the Bearer plugin enabled, attach the token as a normal `Authorization` header:

```ts
await fetch(`${BASE_URL}/api/me`, {
  headers: { Authorization: `Bearer ${accessToken}` },
});
```

## Common pitfalls

- **Forgetting the Bearer plugin** — login "works" but every subsequent API call returns 401. The device flow only issues the token; Bearer validates it.
- **Polling too fast and ignoring `slow_down`** — your CLI gets rate-limited and login appears to hang. Always honor `interval` and increase it on `slow_down`.
- **Treating all errors as retryable** — `expired_token`, `access_denied`, and `invalid_grant` are terminal. Spinning on them wastes the user's time and hides real failures.
- **Letting unauthenticated users hit `/device/approve`** — anyone who guesses or scrapes a `user_code` could approve a device for an active session. Always gate the approval page on a valid session.
- **Customizing the user-code charset and reintroducing confusables** (`0/O/1/I/l`) — looks fine on a desktop, fails when read off a TV across the room.
- **Dropping the `user-agent` on token polling** — some deployments fingerprint clients here for abuse detection; an empty UA can get throttled.
