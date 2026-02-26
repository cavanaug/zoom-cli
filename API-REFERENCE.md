# Zoom API Reference for zoom-cli

**Version**: 1.0  
**Last Updated**: 2026-02-23

## Overview

This document describes the **Zoom API integration** for `zoom-cli`, including OAuth authentication, API endpoints, token management, and rate limiting.

For related documentation, see:
- **[IMPLEMENTATION.md](./IMPLEMENTATION.md)** - Technology stack, project structure, and development guide
- **[CLI-DESIGN.md](./CLI-DESIGN.md)** - User-facing command documentation and workflows
- **[MARKETPLACE.md](./MARKETPLACE.md)** - Zoom Marketplace field-by-field setup with engineering appendix

---

## Table of Contents

1. [OAuth 2.0 Authentication](#oauth-20-authentication)
2. [Required OAuth Scopes](#required-oauth-scopes)
3. [API Endpoint Mapping](#api-endpoint-mapping)
4. [Token Management](#token-management)
5. [API Response Caching](#api-response-caching)
6. [Error Handling](#error-handling)
7. [Rate Limiting](#rate-limiting)
8. [Implementation Phases](#implementation-phases)

---

## OAuth 2.0 Authentication

### Authorization Code Grant Flow (PKCE for CLI)

`zoom-cli` uses **Authorization Code Grant with PKCE** as the primary OAuth 2.0 flow for local CLI authentication.

Why this flow:
- Works for user-run local clients that open a browser for consent.
- Avoids embedding long-lived app secrets in distributed binaries.
- Keeps user tokens on the user's machine (no backend token broker required).

Compatibility note:
- Zoom OAuth docs include PKCE parameters (`code_challenge`, `code_challenge_method`, `code_verifier`) as supported parameters.
- Depending on exact Marketplace app/client configuration, token exchange may still require confidential-client authentication. Validate your specific app configuration during setup.

**Flow Diagram**:
```
1. User runs: zoom-cli auth login
2. CLI generates PKCE values (`code_verifier`, `code_challenge`) and `state`
3. CLI starts local HTTP callback server on first available configured callback port
4. CLI opens browser to: https://zoom.us/oauth/authorize?...
5. User authorizes application in browser
6. Zoom redirects to selected callback URL, for example: http://localhost:53682/callback?code=AUTH_CODE&state=STATE
7. CLI validates `state` and exchanges AUTH_CODE + `code_verifier` for tokens
8. CLI stores access_token + refresh_token securely
```

### OAuth Endpoints

| Endpoint | Purpose | Method |
|----------|---------|--------|
| `https://zoom.us/oauth/authorize` | User authorization | GET |
| `https://zoom.us/oauth/token` | Token exchange/refresh | POST |

### Authorization URL Parameters

```
https://zoom.us/oauth/authorize?
  response_type=code&
  client_id={CLIENT_ID}&
  redirect_uri={SELECTED_ALLOWLISTED_REDIRECT_URI}&
  state={RANDOM_CSRF_STATE}&
  code_challenge={BASE64URL_SHA256_CODE_VERIFIER}&
  code_challenge_method=S256
```

### Redirect URI and Port Strategy (CLI)

PKCE improves client security, but it does not bypass redirect URI validation. The redirect URI still needs to match a URI configured in Zoom Marketplace.

Recommended strategy:
- Register a small fixed set of loopback callback URLs in Marketplace (for example: `http://localhost:53682/callback`, `http://localhost:53683/callback`, `http://localhost:53684/callback`).
- Configure the same ordered list in CLI config.
- At login, bind the first available configured port and use that exact redirect URI in authorize and token requests.
- If no configured callback ports are available, fail with an actionable error.

Avoid relying on fully random ephemeral ports unless your provider explicitly supports wildcard/any-port loopback redirect validation.

### Token Exchange Request (PKCE)

```http
POST https://zoom.us/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
code={AUTH_CODE}&
redirect_uri={SELECTED_ALLOWLISTED_REDIRECT_URI}&
client_id={CLIENT_ID}&
code_verifier={ORIGINAL_CODE_VERIFIER}
```

### Token Exchange Request (Confidential Fallback)

If your app configuration enforces confidential-client auth, Zoom may require client authentication (for example, HTTP Basic auth with `client_id:client_secret`) during token exchange.

```http
POST https://zoom.us/oauth/token
Authorization: Basic {BASE64_CLIENT_ID_COLON_CLIENT_SECRET}
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
code={AUTH_CODE}&
redirect_uri={SELECTED_ALLOWLISTED_REDIRECT_URI}
```

### Token Exchange Response

```json
{
  "access_token": "eyJ...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "eyJ...",
  "scope": "meeting:read meeting_summary:read recording:read user:read"
}
```

### Token Refresh Request

```http
POST https://zoom.us/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token&
refresh_token={REFRESH_TOKEN}&
client_id={CLIENT_ID}
```

If confidential-client auth is required by your app configuration, include client authentication the same way as token exchange.

---

## Required OAuth Scopes

### Core Scopes (Required)

| Scope | Purpose | Commands |
|-------|---------|----------|
| `meeting:read` | List meetings, get details | `ls`, `cat`, `cp`, `download` |
| `meeting_summary:read` | Access AI-generated summaries | `cat summary.md`, `cp`, `download` |
| `recording:read` | List and download recordings | `cat transcript.vtt`, `cp recording.mp4`, `download` |
| `user:read` | Get user profile info | `auth whoami`, `auth status` |

### Optional Scopes (Future Features)

| Scope | Purpose | Status |
|-------|---------|--------|
| `recording:write` | Delete recordings | Phase 4+ |
| `meeting:write` | Create/modify meetings | Future |

### Scope Configuration in Zoom OAuth App

When creating your OAuth app in Zoom Marketplace:

1. Navigate to **Scopes** section
2. Add the following scopes:
   - ✓ `meeting:read` - View and manage user meetings
   - ✓ `meeting_summary:read` - View meeting summaries
   - ✓ `recording:read` - View and manage cloud recordings
   - ✓ `user:read` - View user information

---

## API Endpoint Mapping

### Base URL

```
https://api.zoom.us/v2
```

### Endpoint Reference

| CLI Operation | Zoom API Endpoint | Method | Phase | Notes |
|--------------|-------------------|--------|-------|-------|
| `auth whoami` | `/users/me` | GET | 0 | Get current user info |
| `ls /` | `/users/me/meetings` | GET | 1 | List all meetings |
| `ls "/Meeting/"` | `/users/me/meetings` | GET | 1 | Filter by topic/name |
| `ls "/Meeting/@latest/"` | `/meetings/{meetingId}/recordings` | GET | 2 | List resources for a meeting |
| `cat summary.md` | `/meetings/{meetingId}/meeting_summary` | GET | 1 | Get AI summary |
| `cat metadata.json` | `/meetings/{meetingId}` | GET | 1 | Get meeting details |
| `cat transcript.vtt` | `/meetings/{meetingId}/recordings` | GET | 2 | Get transcript file URL |
| `cp/download transcript` | `{download_url}` | GET | 2 | Download with Bearer token |
| `cp/download recording` | `{download_url}` | GET | 4 | Download with Bearer token |
| `cat chat.txt` | `/meetings/{meetingId}/chat` | GET | 4 | Get chat messages (if available) |

---

## Token Management

### Token Storage

**Location**: `~/.config/zoom-cli/tokens.json`

**Permissions**: `0600` (read/write for owner only)

**Format**:
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_at": "2026-02-24T10:30:00Z",
  "scopes": ["meeting:read", "meeting_summary:read", "recording:read", "user:read"]
}
```

### Token Lifecycle

1. **Initial Authentication** (`auth login`):
   - User authorizes in browser
   - Exchange code for access_token + refresh_token
   - Store tokens with expiration time

2. **Token Usage**:
   - Access tokens expire after 1 hour
   - Before each API call, check if token is expired
   - If expired, auto-refresh using refresh_token

3. **Token Refresh**:
   - Automatically triggered when `expires_at < now + 5 minutes`
   - Exchange refresh_token for new access_token
   - Update stored tokens with new values

4. **Token Invalidation**:
   - If refresh fails (401, invalid_grant), clear tokens
   - Prompt user to run `zoom-cli auth login` again

### CLI Auth Implementation Requirements

- Generate `code_verifier` (high entropy) per login attempt.
- Derive `code_challenge` using SHA-256 and Base64URL encoding.
- Generate and validate `state` to prevent CSRF attacks.
- Bind callback listener to `127.0.0.1` only (not `0.0.0.0`).
- Enforce callback timeout and single-use auth session.
- Never log access/refresh tokens, auth code, or verifier values.
- Store tokens in user-local secure storage (current implementation target: config dir file with strict permissions; future enhancement: OS keychain).

### Token Refresh Implementation

```go
// Pseudocode
func (c *Client) ensureValidToken() error {
    if time.Now().Add(5*time.Minute).After(c.token.ExpiresAt) {
        // Token expired or expiring soon, refresh it
        newToken, err := c.refreshToken(c.token.RefreshToken)
        if err != nil {
            return fmt.Errorf("token refresh failed: %w", err)
        }
        c.token = newToken
        c.saveToken()
    }
    return nil
}
```

---

## API Response Caching

To reduce API calls and improve performance:

### Cache Strategy

| Resource | TTL | Rationale |
|----------|-----|-----------|
| Meeting list | 5 minutes | Meetings don't change frequently |
| Meeting details | 15 minutes | Details rarely change |
| Summaries | 1 hour | Summaries are immutable once generated |
| Recordings list | 15 minutes | Recording processing takes time |

### Cache Implementation

**Location**: `~/.cache/zoom-cli/` or in-memory

**Cache Key Format**:
```
{endpoint}-{params_hash}
```

**Example**:
```
GET /users/me/meetings?from=2026-02-01&to=2026-02-28
Cache Key: "users-me-meetings-abc123def456"
```

### Cache Invalidation

- **Manual**: `zoom-cli cache clear` (future feature)
- **Automatic**: TTL expiration
- **On Error**: If API returns 404 or 410, clear related cache entries

---

## Error Handling

### HTTP Status Codes

| Code | Meaning | CLI Action |
|------|---------|-----------|
| 200 | Success | Process response |
| 401 | Unauthorized | Refresh token or prompt login |
| 403 | Forbidden | Check scopes, show error |
| 404 | Not Found | Resource doesn't exist |
| 429 | Rate Limit | Wait and retry (see Rate Limiting) |
| 500 | Server Error | Retry with exponential backoff |
| 503 | Service Unavailable | Retry with exponential backoff |

### Error Response Format

```json
{
  "code": 1001,
  "message": "User not found."
}
```

### Retry Strategy

```go
// Pseudocode
retryableStatuses := []int{429, 500, 502, 503, 504}
maxRetries := 3
backoff := ExponentialBackoff{
    Initial: 1 * time.Second,
    Max: 30 * time.Second,
    Multiplier: 2,
}
```

---

## Rate Limiting

### Zoom API Rate Limits

- **Light**: 100 requests per day (legacy apps)
- **Medium**: 1,000 requests per day (most OAuth apps)
- **Heavy**: 10,000 requests per day (enterprise)

**Per-second limits**:
- 10 requests per second per endpoint

### Rate Limit Headers

```http
X-RateLimit-Limit: 10
X-RateLimit-Remaining: 5
X-RateLimit-Type: QPS
```

### Rate Limit Handling

1. **Check Headers**: Parse `X-RateLimit-Remaining`
2. **Throttle**: If remaining < 2, sleep 1 second
3. **Handle 429**: Extract `Retry-After` header, wait specified time
4. **Backoff**: Exponential backoff for subsequent retries

**Implementation**:
```go
// Pseudocode
if resp.StatusCode == 429 {
    retryAfter := resp.Header.Get("Retry-After")
    waitDuration := parseRetryAfter(retryAfter) // in seconds
    time.Sleep(waitDuration)
    return c.retry(req)
}
```

---

## Implementation Phases

### Phase 0: Foundation & Authentication

**API Endpoints Required**:
- `POST /oauth/token` - Token exchange and refresh
- `GET /users/me` - User profile (for `auth whoami`)

**Implementation Tasks**:
- [ ] OAuth 2.0 flow (authorization code grant)
- [ ] Token storage (`~/.config/zoom-cli/tokens.json`)
- [ ] Auto-refresh mechanism
- [ ] HTTP client with Bearer token auth

---

### Phase 1: Core Filesystem Metaphor (Read-Only)

**API Endpoints Required**:
- `GET /users/me/meetings` - List all meetings
  - Query params: `type=scheduled|live|upcoming`, `from`, `to`, `page_size`, `next_page_token`
  - Response: List of meetings with ID, topic, start_time, duration
- `GET /meetings/{meetingId}` - Get meeting details
  - Response: Detailed meeting info, participants, settings
- `GET /meetings/{meetingId}/meeting_summary` - Get AI summary
  - Response: Meeting summary text (markdown format)

**Implementation Tasks**:
- [ ] HTTP client with retry logic
- [ ] Rate limit handling
- [ ] Response caching (5-15 min TTL)
- [ ] Error handling for 401, 403, 404, 429, 500

**Example Request** (`ls /`):
```http
GET /v2/users/me/meetings?type=scheduled&page_size=300
Authorization: Bearer {ACCESS_TOKEN}
```

**Example Response**:
```json
{
  "meetings": [
    {
      "uuid": "abc123==",
      "id": 123456789,
      "topic": "Team Standup",
      "type": 8,
      "start_time": "2026-02-23T10:30:00Z",
      "duration": 30,
      "timezone": "America/New_York"
    }
  ],
  "page_size": 300,
  "next_page_token": "xyz789"
}
```

---

### Phase 2: Downloads (Summaries & Transcripts)

**API Endpoints Required**:
- `GET /meetings/{meetingId}/recordings` - List recording files
  - Response: List of recording files (video, audio, transcript, chat)
  - Each file has `id`, `file_type`, `file_size`, `download_url`, `recording_type`

**File Types**:
- `MP4` - Video recording
- `M4A` - Audio-only recording
- `TIMELINE` - Active speaker timeline
- `TRANSCRIPT` - VTT transcript file
- `CHAT` - Chat file
- `CC` - Closed caption file

**Implementation Tasks**:
- [ ] Download files from `download_url` with Bearer token
- [ ] Handle large file downloads (streaming)
- [ ] Progress indication
- [ ] File organization logic

**Example Request** (`ls "/Team-Standup/@latest/"`):
```http
GET /v2/meetings/{meetingId}/recordings
Authorization: Bearer {ACCESS_TOKEN}
```

**Example Response**:
```json
{
  "uuid": "abc123==",
  "id": 123456789,
  "topic": "Team Standup",
  "start_time": "2026-02-23T10:30:00Z",
  "duration": 30,
  "recording_files": [
    {
      "id": "file-abc",
      "meeting_id": "abc123==",
      "recording_start": "2026-02-23T10:30:00Z",
      "recording_end": "2026-02-23T11:00:00Z",
      "file_type": "MP4",
      "file_size": 125829120,
      "download_url": "https://zoom.us/rec/download/xyz...",
      "status": "completed"
    },
    {
      "id": "file-def",
      "file_type": "TRANSCRIPT",
      "file_size": 12582,
      "download_url": "https://zoom.us/rec/download/abc...",
      "status": "completed"
    }
  ]
}
```

**Downloading Files**:
```http
GET {download_url}
Authorization: Bearer {ACCESS_TOKEN}
```

**Important**: The `download_url` requires the Bearer token even though it's a direct URL.

---

### Phase 3: Sync & State Tracking (Deferred)

**Status**: Design pending

---

### Phase 4: Recordings & Advanced Features

**API Endpoints Required**:
- All Phase 2 endpoints (already support recordings)
- `GET /meetings/{meetingId}/chat` (if available)
  - Chat messages from meeting
  - May not be available for all account types

**Implementation Tasks**:
- [ ] Large file handling (resume on failure)
- [ ] Parallel downloads
- [ ] Chat log formatting

---

## Appendix

### OAuth App Setup (Zoom Marketplace)

**Steps**:
1. Go to [Zoom Marketplace](https://marketplace.zoom.us/)
2. Click **Develop** → **Build App**
3. Select **User-managed app** (OAuth)
4. Fill in basic information:
   - App name: `zoom-cli`
   - Short description: CLI tool for managing Zoom meetings
   - Company name: Your name
5. Set **Redirect URL for OAuth**: `http://localhost:53682/callback`
6. Configure OAuth redirect allow list entries that exactly match your callback URL(s)
   - Recommended: add a small fallback set, such as:
     - `http://localhost:53682/callback`
     - `http://localhost:53683/callback`
     - `http://localhost:53684/callback`
7. Use PKCE-enabled authorization requests from the CLI (`code_challenge_method=S256`)
8. Add **Scopes**:
   - `meeting:read`
   - `meeting_summary:read`
   - `recording:read`
   - `user:read`
9. Copy **Client ID** (and `Client Secret` only if your app config requires confidential auth)
10. Add credentials to config file (`~/.config/zoom-cli/config.yaml`)

**Recommended for distributed CLI apps**:
- Prefer PKCE-based local-client flow so user tokens remain on user devices.
- Do not embed a global `client_secret` in distributed binaries.
- If your app config requires confidential token exchange and PKCE-only is not possible, use a backend broker or separate per-user app credentials.

### Complete API Flow Example

**Scenario**: View latest Team Standup summary

1. **Authentication Check**:
   ```
   - Check if ~/.config/zoom-cli/tokens.json exists
   - If not, prompt: "Run 'zoom-cli auth login'"
   - If exists, check if access_token expired
   - If expired, refresh using refresh_token
   ```

2. **List Meetings** (`zoom-cli ls "/Team-Standup/"`):
   ```http
   GET /v2/users/me/meetings?type=scheduled&page_size=300
   Authorization: Bearer {ACCESS_TOKEN}
   ```
   
   - Filter results by topic matching "Team Standup"
   - Sort by start_time descending
   - Take most recent N results

3. **Get Summary** (`zoom-cli cat "/Team-Standup/@latest/summary.md"`):
   ```http
   GET /v2/meetings/{meetingId}/meeting_summary
   Authorization: Bearer {ACCESS_TOKEN}
   ```
   
   - Extract summary text from response
   - Format as markdown
   - Display in terminal (or pipe through pager)

### API Documentation Links

- **Zoom API Reference**: https://developers.zoom.us/docs/api/
- **OAuth 2.0 Guide**: https://developers.zoom.us/docs/integrations/oauth/
- **Meeting APIs**: https://developers.zoom.us/docs/api/rest/reference/zoom-api/methods/#tag/Meetings
- **Recording APIs**: https://developers.zoom.us/docs/api/rest/reference/zoom-api/methods/#tag/Cloud-Recording

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-02-23 | Initial API reference |

---

**End of API Reference**
