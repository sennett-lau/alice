---
name: setup-browser-cookies
preamble-tier: 1
version: 1.0.1
description: |
  Import cookies from your real Chromium browser into the headless browse session.
  Opens an interactive picker UI where you select which cookie domains to import.
  Use before QA testing authenticated pages. Use when asked to "import cookies",
  "login to the site", or "authenticate the browser".
allowed-tools:
  - Bash
  - Read
  - AskUserQuestion
---

## Preamble (run first)

```bash
# Project-local state dir — all session data under .alice/mem/ (gitignored).
eval "$(.alice/bin/alice-slug 2>/dev/null || true)"
mkdir -p "${ROOT:-.}/.alice/mem"
echo "BRANCH: ${BRANCH:-unknown}"
```

# Setup Browser Cookies

## Overview

Imports authenticated browser cookies from a real Chromium profile into the headless browse session through an interactive domain picker.

## When to Use

- Use before browser QA on authenticated pages when the headless session needs the user's real login state.
- Use when asked to import cookies, log in to a site, authenticate the browser, or prepare authed QA.
- Use when `browse` is not already connected to the user's real browser through CDP.

**When NOT to use:**

- Do not use when CDP mode is active; real-browser cookies are already available.
- Do not use for non-Chromium authentication flows that the browse cookie importer cannot access.
- Do not proceed before the browse binary setup check passes.

## Process

Import logged-in sessions from your real Chromium browser into the headless browse session.

### CDP mode check

First, check if browse is already connected to the user's real browser:
```bash
$B status 2>/dev/null | grep -q "Mode: cdp" && echo "CDP_MODE=true" || echo "CDP_MODE=false"
```
If `CDP_MODE=true`: tell the user "Not needed — you're connected to your real browser via CDP. Your cookies and sessions are already available." and stop. No cookie import needed.

### How it works

1. Find the browse binary
2. Run `cookie-import-browser` to detect installed browsers and open the picker UI
3. User selects which cookie domains to import in their browser
4. Cookies are decrypted and loaded into the Playwright session

### Steps

### 1. Find the browse binary

### SETUP (run this check BEFORE any browse command)

```bash
_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
B=""
[ -n "$_ROOT" ] && [ -x "$_ROOT/.alice/skills/browse/dist/browse" ] && B="$_ROOT/.alice/skills/browse/dist/browse"
[ -z "$B" ] && B=.alice/skills/browse/dist/browse
if [ -x "$B" ]; then
  echo "READY: $B"
else
  echo "NEEDS_SETUP"
fi
```

If `NEEDS_SETUP`:
1. Tell the user: "alice browse needs a one-time build (~10–30 seconds; will install bun first if missing). OK to proceed?" Then STOP and wait.
2. On approval, run the bundled setup script:
   ```bash
   .alice/skills/browse/setup
   ```
3. Re-run the SETUP check to confirm `READY:`.

### 2. Open the cookie picker

```bash
$B cookie-import-browser
```

This auto-detects installed Chromium browsers and opens
an interactive picker UI in your default browser where you can:
- Switch between installed browsers
- Search domains
- Click "+" to import a domain's cookies
- Click trash to remove imported cookies

Tell the user: **"Cookie picker opened — select the domains you want to import in your browser, then tell me when you're done."**

### 3. Direct import (alternative)

If the user specifies a domain directly (e.g., `/setup-browser-cookies github.com`), skip the UI:

```bash
$B cookie-import-browser comet --domain github.com
```

Replace `comet` with the appropriate browser if specified.

### 4. Verify

After the user confirms they're done:

```bash
$B cookies
```

Show the user a summary of imported cookies (domain counts).

### Notes

- On macOS, the first import per browser may trigger a Keychain dialog — click "Allow" / "Always Allow"
- On Linux, `v11` cookies may require `secret-tool`/libsecret access; `v10` cookies use Chromium's standard fallback key
- Cookie picker is served on the same port as the browse server (no extra process)
- Only domain names and cookie counts are shown in the UI — no cookie values are exposed
- The browse session persists cookies between commands, so imported cookies work immediately

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "I'll just hardcode the test password into the test." | Real-browser cookies cover SSO, MFA-bypass-via-trust, OAuth callbacks, and session-server quirks that test-credential paths skip. Import once, get the realistic state. |
| "The user is already logged in CDP mode — let me run the picker anyway." | The CDP check exists for a reason. Re-running the picker over CDP can clobber the live session. Honour the early return. |
| "It's fine to log cookie values for debugging." | Cookie values are credentials. Never log them, never paste them into chat, never write them to the report. Domain + count is the only safe surface. |

## Red Flags

- Skipping the CDP mode check and running the picker against a live session.
- Pasting cookie values into chat or any log.
- Importing domains the user didn't ask for ("just in case").
- Running the picker on a binary that's not the project-local `.alice/skills/browse/dist/browse` (could be a stale or different build).

## Verification

After the user confirms cookie selection:

- [ ] `$B cookies` shows non-zero counts for the domains the user asked for.
- [ ] No cookie values appeared in any output — only domains and counts.
- [ ] CDP-mode check ran first and returned `false` (or, if `true`, the skill stopped early).
- [ ] If on macOS, the Keychain dialog was acknowledged by the user (not auto-clicked).
