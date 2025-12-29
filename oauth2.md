---
layout: default
---

For a guide with screenshots, see: https://forums.online-go.com/t/oauth2-how-to/58804

---

# OAuth2 Authentication

OAuth2 allows third-party applications to authenticate users and access the OGS API on their behalf.

## Endpoints

| Endpoint      | URL                                          |
| ------------- | -------------------------------------------- |
| Authorization | `https://online-go.com/oauth2/authorize/`    |
| Token         | `https://online-go.com/oauth2/token/`        |
| Revoke Token  | `https://online-go.com/oauth2/revoke_token/` |
| Introspect    | `https://online-go.com/oauth2/introspect/`   |
| User Info     | `https://online-go.com/oauth2/userinfo/`     |

## Available Scopes

| Scope    | Description               |
| -------- | ------------------------- |
| `read`   | Read access to user data  |
| `write`  | Write access to user data |
| `groups` | Access to user's groups   |

## Application Registration

Register your application to obtain a `client_id` and `client_secret`.

**Limits:** Each user can register up to 5 OAuth2 applications.

### Client Types

-   **Confidential**: Server-side applications that can securely store the client secret. Use this for web backends.
-   **Public**: Mobile apps, SPAs, or desktop apps that cannot securely store secrets. Use PKCE instead of client secrets.

### Via Web Interface

Register at [https://online-go.com/oauth2/applications/](https://online-go.com/oauth2/applications/).

### Via API

```bash
curl -X POST https://online-go.com/api/v1/me/api_clients/ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <your-user-token>" \
  -d '{
    "name": "My Application",
    "client_type": "confidential",
    "authorization_grant_type": "authorization-code",
    "redirect_uris": "https://yoursite.com/callback"
  }'
```

Response:

```json
{
    "id": 123,
    "client_id": "abc123def456",
    "client_secret": "supersecretkey789",
    "name": "My Application",
    "client_type": "confidential",
    "authorization_grant_type": "authorization-code",
    "redirect_uris": "https://yoursite.com/callback"
}
```

**Important**: Save the `client_secret` immediately - it cannot be retrieved later.

### Redirect URI Schemes

Supported URI schemes for redirect URIs:

-   `https://` - Web applications (recommended)
-   `http://` - Local development only
-   `expo://` - Expo mobile apps
-   `gobo://` - Custom mobile scheme

---

## Authorization Code Flow

This is the recommended flow for third-party applications.

### Step 1: Redirect User to Authorization

```
https://online-go.com/oauth2/authorize/?client_id=YOUR_CLIENT_ID&response_type=code&redirect_uri=https://yoursite.com/callback&scope=read+write&state=RANDOM_STATE
```

**Parameters:**

| Parameter       | Required | Description                                                                 |
| --------------- | -------- | --------------------------------------------------------------------------- |
| `client_id`     | Yes      | Your application's client ID                                                |
| `response_type` | Yes      | Must be `code`                                                              |
| `redirect_uri`  | Yes      | Must exactly match a registered redirect URI                                |
| `scope`         | No       | Space-separated scopes: `read`, `write`, `groups`                           |
| `state`         | Yes\*    | Random value for CSRF protection (\*technically optional but always use it) |

### Step 2: Handle the Callback

After authorization, OGS redirects to your callback URL:

**Success:**

```
https://yoursite.com/callback?code=AUTH_CODE&state=YOUR_STATE
```

**Error:**

```
https://yoursite.com/callback?error=access_denied&error_description=User+denied+access&state=YOUR_STATE
```

Always validate the `state` parameter matches what you sent.

### Step 3: Exchange Code for Tokens

Use HTTP Basic Authentication (recommended):

```bash
curl -X POST https://online-go.com/oauth2/token/ \
  -u "YOUR_CLIENT_ID:YOUR_CLIENT_SECRET" \
  -d "grant_type=authorization_code" \
  -d "code=AUTH_CODE" \
  -d "redirect_uri=https://yoursite.com/callback"
```

Or include credentials in the body:

```bash
curl -X POST https://online-go.com/oauth2/token/ \
  -d "grant_type=authorization_code" \
  -d "code=AUTH_CODE" \
  -d "redirect_uri=https://yoursite.com/callback" \
  -d "client_id=YOUR_CLIENT_ID" \
  -d "client_secret=YOUR_CLIENT_SECRET"
```

**Response:**

```json
{
    "access_token": "your-access-token",
    "token_type": "Bearer",
    "expires_in": 2592000,
    "refresh_token": "your-refresh-token",
    "scope": "read write"
}
```

### Token Expiration

| Token         | Lifetime                  |
| ------------- | ------------------------- |
| Access Token  | 30 days (2592000 seconds) |
| Refresh Token | 30 days (2592000 seconds) |

**Note:** Both tokens expire after 30 days. After the refresh token expires, users must re-authenticate.

### Step 4: Use Access Token

```bash
curl -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  https://online-go.com/api/v1/me/
```

### Step 5: Refresh Tokens

Before the access token expires, obtain a new one:

```bash
curl -X POST https://online-go.com/oauth2/token/ \
  -u "YOUR_CLIENT_ID:YOUR_CLIENT_SECRET" \
  -d "grant_type=refresh_token" \
  -d "refresh_token=YOUR_REFRESH_TOKEN"
```

### Revoke Tokens

To invalidate a token (e.g., on logout):

```bash
curl -X POST https://online-go.com/oauth2/revoke_token/ \
  -u "YOUR_CLIENT_ID:YOUR_CLIENT_SECRET" \
  -d "token=TOKEN_TO_REVOKE"
```

---

## JavaScript Example

```javascript
// Step 1: Initiate login
function loginWithOGS() {
    const clientId = "your-client-id";
    const redirectUri = "https://yoursite.com/callback";
    const state = crypto.randomUUID();

    sessionStorage.setItem("oauth_state", state);

    const params = new URLSearchParams({
        client_id: clientId,
        redirect_uri: redirectUri,
        response_type: "code",
        scope: "read write",
        state: state,
    });

    window.location.href = `https://online-go.com/oauth2/authorize/?${params}`;
}

// Step 2: Handle callback
async function handleCallback() {
    const params = new URLSearchParams(window.location.search);

    // Check for errors
    const error = params.get("error");
    if (error) {
        const description = params.get("error_description") || "Unknown error";
        throw new Error(`OAuth error: ${error} - ${description}`);
    }

    // Validate state
    const state = params.get("state");
    if (state !== sessionStorage.getItem("oauth_state")) {
        throw new Error("State mismatch - possible CSRF attack");
    }

    // Exchange code for tokens (send to your backend)
    const code = params.get("code");
    const response = await fetch("/api/auth/ogs/callback", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ code }),
    });

    if (!response.ok) {
        throw new Error("Token exchange failed");
    }

    const data = await response.json();
    // Handle authenticated user...
}
```

---

## PKCE for Public Clients

For mobile apps and SPAs that cannot securely store a client secret, use PKCE (Proof Key for Code Exchange).

### Complete PKCE Flow

```javascript
// Generate PKCE parameters
function generateCodeVerifier() {
    const array = new Uint8Array(32);
    crypto.getRandomValues(array);
    return btoa(String.fromCharCode(...array))
        .replace(/\+/g, "-")
        .replace(/\//g, "_")
        .replace(/=/g, "");
}

async function generateCodeChallenge(verifier) {
    const encoder = new TextEncoder();
    const data = encoder.encode(verifier);
    const hash = await crypto.subtle.digest("SHA-256", data);
    return btoa(String.fromCharCode(...new Uint8Array(hash)))
        .replace(/\+/g, "-")
        .replace(/\//g, "_")
        .replace(/=/g, "");
}

// Step 1: Authorization request with PKCE
async function loginWithPKCE() {
    const clientId = "your-client-id";
    const redirectUri = "https://yoursite.com/callback";
    const state = crypto.randomUUID();
    const codeVerifier = generateCodeVerifier();
    const codeChallenge = await generateCodeChallenge(codeVerifier);

    // Store for later
    sessionStorage.setItem("oauth_state", state);
    sessionStorage.setItem("code_verifier", codeVerifier);

    const params = new URLSearchParams({
        client_id: clientId,
        redirect_uri: redirectUri,
        response_type: "code",
        scope: "read write",
        state: state,
        code_challenge: codeChallenge,
        code_challenge_method: "S256",
    });

    window.location.href = `https://online-go.com/oauth2/authorize/?${params}`;
}

// Step 2: Token exchange with PKCE (no client_secret needed)
async function exchangeCodeWithPKCE(code) {
    const codeVerifier = sessionStorage.getItem("code_verifier");

    const response = await fetch("https://online-go.com/oauth2/token/", {
        method: "POST",
        headers: { "Content-Type": "application/x-www-form-urlencoded" },
        body: new URLSearchParams({
            grant_type: "authorization_code",
            code: code,
            redirect_uri: "https://yoursite.com/callback",
            client_id: "your-client-id",
            code_verifier: codeVerifier,
        }),
    });

    return response.json();
}
```

---

## Error Responses

Common OAuth2 errors returned in callbacks or token requests:

| Error                 | Description                               |
| --------------------- | ----------------------------------------- |
| `invalid_request`     | Missing or invalid parameter              |
| `invalid_client`      | Client authentication failed              |
| `invalid_grant`       | Authorization code is invalid or expired  |
| `unauthorized_client` | Client not authorized for this grant type |
| `access_denied`       | User denied the authorization request     |
| `invalid_scope`       | Requested scope is invalid or unknown     |

---

## Security Best Practices

-   **Always use and validate the `state` parameter** to prevent CSRF attacks
-   **Use HTTPS** for all OAuth2 communications
-   **Keep `client_secret` confidential** - never expose in client-side code
-   **Use PKCE** for public clients (mobile apps, SPAs)
-   **Store tokens securely** - use HttpOnly cookies or secure storage
-   **Implement token refresh** before expiry for seamless sessions
-   **Revoke tokens on logout** to prevent unauthorized access
