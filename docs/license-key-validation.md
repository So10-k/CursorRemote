# License Key Validation

This document explains where license keys are validated, how the validation algorithm works, and how the system verifies keys at runtime.

---

## Overview

CursorRemote uses **local client-side validation only** — no network requests or API calls are made to verify a license key. Validation is performed in two places:

| Context | File | When |
|---------|------|------|
| VSCode Extension | `extension/src/license-manager.ts` | When the user enters a key via the Command Palette |
| Server process | `src/server/license.ts` | At server startup, before any operations begin |

Both contexts use the same algorithm.

---

## Validation Algorithm

### Key Format

A valid license key must match the regular expression:

```
/^[A-Z0-9]{4}-[A-Z0-9]{4}-[A-Z0-9]{4}-[A-Z0-9]{4}-[A-Z0-9]{4}$/
```

This means five groups of four uppercase alphanumeric characters joined by dashes, for example:

```
ABCD-1234-EF56-GH78-IJ90
```

### Checksum (Modulo 42)

After the format check passes, the key is verified with a checksum:

1. Remove all dashes — the key becomes 20 characters.
2. Sum the ASCII code of every character.
3. The key is valid only when `sum % 42 === 0`.

```typescript
function validateKey(key: string): boolean {
  const trimmed = key.trim().toUpperCase();
  if (!KEY_FORMAT.test(trimmed)) return false;            // step 1 – format
  const chars = trimmed.replace(/-/g, '');               // step 2 – strip dashes
  const sum = [...chars].reduce((acc, c) => acc + c.charCodeAt(0), 0);
  return sum % 42 === 0;                                  // step 3 – checksum
}
```

This identical function lives in both `license-manager.ts` and `license.ts`.

---

## Where Keys Are Stored

### Extension (VSCode)

The extension stores the key in VSCode's built-in secrets API, which delegates to the OS credential manager (Windows Credential Manager, macOS Keychain, Linux Secret Service):

```typescript
const SECRET_KEY = 'cursorRemote.licenseKey';

// Save
await context.secrets.store(SECRET_KEY, normalizedKey);

// Read
const key = await context.secrets.get(SECRET_KEY);
```

The key is **never written to workspace settings or a plain-text file** by the extension.

### Server

The server reads the key from two locations, in this priority order:

1. **Environment variable** `LICENSE_KEY` — set by the extension when spawning the server process.
2. **File on disk** — `data/license.key` (or the path given by the `DATA_DIR` environment variable).

```typescript
function readStoredKey(): string | null {
  const envKey = process.env.LICENSE_KEY?.trim();
  if (envKey) return envKey;                    // priority 1

  try {
    if (existsSync(LICENSE_PATH)) {
      const raw = readFileSync(LICENSE_PATH, 'utf-8');
      return raw.trim() || null;               // priority 2
    }
  } catch { /* ignore */ }
  return null;
}
```

---

## Verification Flow

### User Enters a Key (Extension)

```
User opens Command Palette
  → "CursorRemote: Enter License Key"
      → showInputBox() with real-time validateInput callback
          → validateKey() called on every keystroke
              ✓ valid  → store in VSCode secrets → show success message → trigger server start
              ✗ invalid → show inline error, do not save
```

### Server Startup

```
main() in src/server/index.ts
  → checkLicense()
      → readStoredKey()   (env var or data/license.key)
      → validateKey()
          ✓ valid  → log "Thank you for supporting the project." → continue
          ✗ invalid / missing → log error with purchase URL → process.exit(1)
```

The server **will not start** without a valid license key.

### How the Extension Passes the Key to the Server

`extension/src/config-bridge.ts` builds the environment object used when spawning the server process:

```typescript
export function buildEnvFromConfig(
  context: vscode.ExtensionContext,
  licenseKey: string | undefined
): Record<string, string> {
  return {
    LICENSE_KEY: licenseKey ?? '',   // ← license key injected here
    CDP_URL:     config.get<string>('cdpUrl', 'http://127.0.0.1:9222'),
    SERVER_PORT: String(config.get<number>('serverPort', 3000)),
    // ...other config values
  };
}
```

---

## UI Behaviour

### No Valid Key

The sidebar tree view shows:

```
  License Key Required   (click to enter)
  Buy License            (click to open store)
  ─────────────────────
  Open Setup Panel
```

### Valid Key Present

```
  CursorRemote v1.x.x
  Server: Running
  CDP: Connected
  Agent: Ready
  Clients: 3
  ─────────────────────
  Open Setup Panel
  Open Web Client
  Show Logs
```

If `autoStart` is enabled in settings (default: `true`), the extension checks the key on activation and starts the server automatically when the key is valid.

---

## Purchase URL

When a key is missing or invalid, both the extension and the server direct the user to the store:

| Context | URL |
|---------|-----|
| Extension | `https://cursor-remote.com/buy?utm_source=extension&utm_medium=command&utm_campaign=license` |
| Server | `https://cursor-remote.com/buy?utm_source=server&utm_medium=cli&utm_campaign=license` |

---

## Quick Reference

| Property | Value |
|----------|-------|
| Validation type | Local only — no API calls |
| Key format | `XXXX-XXXX-XXXX-XXXX-XXXX` (20 alphanumeric chars + 4 dashes) |
| Checksum algorithm | Sum of ASCII codes of the 20 chars must be divisible by 42 |
| Extension storage | VSCode secrets API (OS credential manager) |
| Server storage | `LICENSE_KEY` env var, then `data/license.key` file |
| Failure behaviour | Server exits with code 1; extension prompts the user |
