# Zoom Marketplace Setup for zoom-cli

**Version**: 1.0  
**Last Updated**: 2026-02-26

## Overview

This guide is the single source of truth for configuring the Zoom Marketplace app used by `zoom-cli`.

It covers:
- Field-by-field Marketplace setup answers
- Recommended callback/redirect strategy for CLI apps
- Scope selection guidance
- Troubleshooting common setup errors
- Engineering appendix (PKCE and token exchange details)

Related docs:
- **[IMPLEMENTATION.md](./IMPLEMENTATION.md)** - build plan and module ownership
- **[API-REFERENCE.md](./API-REFERENCE.md)** - endpoint mapping and OAuth lifecycle
- **[CLI-DESIGN.md](./CLI-DESIGN.md)** - user-facing auth command behavior

---

## Marketplace Tab Mapping

Use this quick index while stepping through the Zoom Marketplace UI.

| Marketplace UI Area | What to Fill | See Section |
|---|---|---|
| Basic Information | Name, description, support/privacy URLs, contact | [Marketplace Field-by-Field Answers](#marketplace-field-by-field-answers) |
| OAuth Redirect / Allow List | Callback URLs and fallback ports | [OAuth Redirect / Allow List](#2-oauth-redirect--allow-list) |
| Scopes | Minimum required API permissions | [Scopes](#3-scopes) |
| Activation / Distribution | Dev vs publish path and admin constraints | [Activation / Distribution](#4-activation--distribution) |
| Local Client Setup | `config.yaml` redirect list and credentials | [Local CLI Configuration](#local-cli-configuration) |
| Auth Runtime Behavior | Port selection at login and error conditions | [Runtime Port Selection Behavior](#runtime-port-selection-behavior) |
| Implementation Details | PKCE params, callback validation, token exchange | [Engineering Appendix](#engineering-appendix) |

---

## Recommended App Type

For a distributed CLI where each user authenticates their own Zoom account:
- Use **OAuth app**
- Prefer **User-managed app**
- Use **Authorization Code + PKCE**

Use Account-level OAuth only for tenant-admin driven enterprise installs.

---

## Marketplace Field-by-Field Answers

Open: Zoom Marketplace -> Develop -> Build App -> OAuth

### 1) Basic Information

- **App Name**: `zoom-cli`
- **Short Description**: `CLI tool for browsing and exporting Zoom meeting summaries, transcripts, and recordings.`
- **Company Name**: your org or your name
- **Developer Contact Email**: monitored support email
- **Support URL**: support/docs page for your CLI
- **Privacy Policy URL**: page describing token handling and data usage

Suggested privacy text:

```text
This CLI uses OAuth Authorization Code with PKCE. OAuth tokens are stored locally on the user device. Our service does not require access to user Zoom access or refresh tokens for normal CLI operation.
```

### 2) OAuth Redirect / Allow List

Use pre-registered loopback callback URLs.

- **Primary Redirect URL**: `http://localhost:53682/callback`
- **Recommended fallback URLs**:
  - `http://localhost:53683/callback`
  - `http://localhost:53684/callback`

Add all callback URLs above to the OAuth redirect allow list.

Important:
- Redirect URI must match exactly what the CLI sends.
- PKCE does not remove redirect URI validation.
- Do not rely on fully random ephemeral callback ports unless Zoom explicitly allows any-port loopback redirects for your exact app configuration.

### 3) Scopes

Add the minimum scopes needed by current CLI features:

- `meeting:read`
- `meeting_summary:read`
- `recording:read`
- `user:read`

Scope justification template:

```text
meeting:read - required for listing meetings and meeting metadata in ls/cat commands.
meeting_summary:read - required for reading AI summaries via summary.md resources.
recording:read - required for transcript/recording discovery and download commands.
user:read - required for auth status and whoami user profile output.
```

### 4) Activation / Distribution

- Keep app in development mode while validating auth flow.
- Publish only when metadata/compliance requirements are complete.
- If users see admin approval errors, verify account policies for Marketplace app install/authorization.

---

## Local CLI Configuration

PKCE-first configuration:

```yaml
auth:
  client_id: "your-zoom-client-id"
  redirect_url: "http://localhost:53682/callback"
  redirect_urls:
    - "http://localhost:53682/callback"
    - "http://localhost:53683/callback"
    - "http://localhost:53684/callback"
```

If your app configuration requires confidential token exchange, also include `client_secret`:

```yaml
auth:
  client_id: "your-zoom-client-id"
  client_secret: "your-zoom-client-secret"
  redirect_url: "http://localhost:53682/callback"
  redirect_urls:
    - "http://localhost:53682/callback"
    - "http://localhost:53683/callback"
    - "http://localhost:53684/callback"
```

---

## Runtime Port Selection Behavior

At `auth login` time:
- Try configured `redirect_urls` in order.
- Bind the first available local port.
- Use the selected redirect URI in both authorization request and token exchange request.
- If all configured ports are busy, exit with an actionable error.

---

## Troubleshooting

- **Redirect mismatch / invalid redirect_uri**
  - Confirm exact URL match (scheme, host, port, path) between Marketplace and CLI request.
- **Port already in use**
  - Free the local port or add another pre-registered callback port and retry.
- **invalid_scope**
  - Add missing scopes in Marketplace and reinstall/reauthorize app.
- **state mismatch**
  - Restart login; ensure callback hit belongs to the same auth session.
- **Admin approval required**
  - Account admin must allow/install or pre-approve app scopes.

---

## Engineering Appendix

### A) PKCE Parameters

Per login attempt:
- Generate `code_verifier` (high entropy, RFC 7636 compatible)
- Derive `code_challenge = BASE64URL(SHA256(code_verifier))`
- Send `code_challenge_method=S256`
- Generate and validate `state`

### B) Authorization Request

```text
GET https://zoom.us/oauth/authorize
  ?response_type=code
  &client_id={CLIENT_ID}
  &redirect_uri={SELECTED_ALLOWLISTED_REDIRECT_URI}
  &state={STATE}
  &code_challenge={CODE_CHALLENGE}
  &code_challenge_method=S256
```

### C) Callback Handling

Expected callback:

```text
http://localhost:{PORT}/callback?code={AUTH_CODE}&state={STATE}
```

Validation requirements:
- `state` must match in-memory login session
- callback accepted once, then listener shuts down
- enforce timeout (recommended: 120 seconds)

### D) Token Exchange (PKCE)

```http
POST https://zoom.us/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
code={AUTH_CODE}&
redirect_uri={SELECTED_ALLOWLISTED_REDIRECT_URI}&
client_id={CLIENT_ID}&
code_verifier={CODE_VERIFIER}
```

### E) Confidential Fallback

If app config requires confidential-client auth, include client authentication (for example HTTP Basic with `client_id:client_secret`) for token and refresh requests.

### F) Token Storage and Refresh

- Persist tokens with strict local file permissions (`0600`) or OS keychain (future enhancement).
- Refresh when expiry is near.
- On refresh failure (`invalid_grant`), clear tokens and require `auth login`.
- Never log token values, auth code, or `code_verifier`.

---

## Copy/Paste Setup Checklist

1. Create User-managed OAuth app in Zoom Marketplace.
2. Add redirect URLs:
   - `http://localhost:53682/callback`
   - `http://localhost:53683/callback`
   - `http://localhost:53684/callback`
3. Add scopes:
   - `meeting:read`
   - `meeting_summary:read`
   - `recording:read`
   - `user:read`
4. Copy `client_id` (and `client_secret` only if required by app config).
5. Update local `~/.config/zoom-cli/config.yaml`.
6. Run `zoom-cli auth login`.
7. Verify with `zoom-cli auth status` and `zoom-cli auth whoami`.
