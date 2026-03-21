# OAuth and OIDC: From Zero to Understanding

## The Problem: Passwords Everywhere

Imagine you use 50 different apps. Each one asks you to create an account with a username and password. Now you have 50 passwords. You reuse them (bad), forget them (annoying), and every app stores your password in their database (risky — if one gets hacked, attackers try your password everywhere).

Even worse: some apps want to access your data on other apps. For example, a photo printing service wants to access your Google Photos. Do you give it your Google password? Absolutely not — it could read your emails, change your settings, or lock you out.

We need two things:

1. **A way for apps to access your stuff without knowing your password** (OAuth)
2. **A way to prove who you are without creating a new account everywhere** (OIDC)

---

## Part 1: OAuth — "Let this app access my stuff"

### The Analogy

You're staying at a hotel. You want the valet to park your car. You don't give them your house key — you give them a **valet key** that only starts the engine and opens the driver's door. It can't open the trunk or the glove box.

OAuth works the same way. Instead of giving an app your password (the master key), you give it a **token** (the valet key) that only allows specific actions.

### The Parties

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  Resource     │     │  Authorization    │     │  Client          │
│  Owner (You)  │     │  Server           │     │  (The App)       │
│               │     │  (e.g., Google)   │     │  (e.g., Photo    │
│  The person   │     │  Guards your data │     │   Printer)       │
│  who owns     │     │  and issues       │     │                  │
│  the data     │     │  access tokens    │     │  Wants to access │
│               │     │                   │     │  your data       │
└──────────────┘     └──────────────────┘     └──────────────────┘
                              │
                              │ protects
                              ▼
                     ┌──────────────────┐
                     │  Resource Server  │
                     │  (e.g., Google    │
                     │   Photos API)    │
                     │                  │
                     │  Where your data │
                     │  actually lives  │
                     └──────────────────┘
```

- **Resource Owner** — You. The human who owns the data.
- **Client** — The app that wants to access your data (e.g., a photo printing service).
- **Authorization Server** — The gatekeeper that authenticates you and issues tokens (e.g., Google's login page).
- **Resource Server** — The API that holds your data (e.g., Google Photos API). Often run by the same company as the Authorization Server.

### The OAuth Flow (Authorization Code Grant)

This is the most common and most secure OAuth flow.

```
You (Browser)          Photo Printer App         Google (Auth Server)       Google Photos (API)
    │                        │                          │                        │
    │  1. "Connect Google    │                          │                        │
    │      Photos"           │                          │                        │
    │───────────────────────>│                          │                        │
    │                        │                          │                        │
    │  2. Redirect to Google │                          │                        │
    │<───────────────────────│                          │                        │
    │     URL includes:                                 │                        │
    │     - client_id (who's asking)                    │                        │
    │     - redirect_uri (where to come back)           │                        │
    │     - scope (what access: "read photos")          │                        │
    │     - state (CSRF protection)                     │                        │
    │                                                   │                        │
    │  3. You see Google login page                     │                        │
    │──────────────────────────────────────────────────>│                        │
    │                                                   │                        │
    │  4. You log in + consent                          │                        │
    │     "Allow Photo Printer to                       │                        │
    │      read your photos?"                           │                        │
    │──────────────────────────────────────────────────>│                        │
    │                                                   │                        │
    │  5. Redirect back to Photo Printer                │                        │
    │<──────────────────────────────────────────────────│                        │
    │     URL includes:                                 │                        │
    │     - code (one-time authorization code)           │                        │
    │     - state (must match step 2)                   │                        │
    │                                                   │                        │
    │───────────────────────>│                          │                        │
    │                        │                          │                        │
    │                        │  6. Exchange code for     │                        │
    │                        │     access token          │                        │
    │                        │     (server-to-server)    │                        │
    │                        │  Sends: code +            │                        │
    │                        │    client_id +             │                        │
    │                        │    client_secret           │                        │
    │                        │─────────────────────────>│                        │
    │                        │                          │                        │
    │                        │  7. Access token returned │                        │
    │                        │     (e.g., "ya29.abc123") │                        │
    │                        │<─────────────────────────│                        │
    │                        │                          │                        │
    │                        │  8. Use token to get photos                       │
    │                        │─────────────────────────────────────────────────>│
    │                        │     Header: Authorization: Bearer ya29.abc123    │
    │                        │                          │                        │
    │                        │  9. Here are the photos  │                        │
    │                        │<─────────────────────────────────────────────────│
    │                        │                          │                        │
    │  10. Your photos!      │                          │                        │
    │<───────────────────────│                          │                        │
```

### What's Exchanged and Why

| Step | What | Why |
|------|------|-----|
| 2 | `client_id` | Tells Google which app is asking (Photo Printer registered with Google beforehand) |
| 2 | `redirect_uri` | Where to send the user back — must match what was pre-registered to prevent hijacking |
| 2 | `scope` | What permissions are requested (e.g., "read photos" but not "delete photos") |
| 2 | `state` | A random string to prevent CSRF attacks — the app checks it matches when the user returns |
| 5 | `code` | A one-time authorization code — proof that you said "yes" — but NOT the access token itself |
| 6 | `client_secret` | Proves the app is who it claims to be (only the real Photo Printer knows this secret) |
| 7 | `access_token` | The "valet key" — a bearer token that grants limited access to your photos |

### Why the Code → Token Exchange?

Why not just return the access token directly in step 5?

Because step 5 goes through **your browser** (via URL redirect). URLs can be logged, cached, or intercepted. The authorization code is short-lived (minutes) and can only be used once. The actual access token is exchanged server-to-server (step 6), never touching the browser.

### What OAuth Does NOT Do

OAuth tells the Photo Printer app: *"This token lets you read photos."*

It does NOT tell the app: *"The person who authorized this is terry@gmail.com."*

OAuth is about **authorization** (what you can do), not **authentication** (who you are). This is where OIDC comes in.

---

## Part 2: OIDC — "This is who I am"

### The Problem OIDC Solves

You want to log into a new website. Instead of creating yet another account, you click "Sign in with Google." The website needs to know **who you are** (your name, email, etc.) — not access your photos or files.

OAuth alone can't do this reliably. Some developers hacked it by requesting a "profile" scope and calling a user-info API, but this was inconsistent across providers and had security gaps.

**OIDC (OpenID Connect)** is a standardized identity layer built **on top of** OAuth. It adds one key thing: an **ID Token** that proves who you are.

### OIDC = OAuth + Identity

```
┌─────────────────────────────────────────────────┐
│                    OIDC                          │
│                                                  │
│   Everything OAuth does (access tokens, scopes) │
│                                                  │
│   + ID Token (who you are)                       │
│   + UserInfo endpoint (more details about you)   │
│   + Standardized scopes (openid, profile, email) │
│   + Discovery document (.well-known/openid-      │
│     configuration)                               │
│                                                  │
└─────────────────────────────────────────────────┘
```

### The OIDC Flow

The flow is almost identical to OAuth, with a few additions:

```
You (Browser)          New Website              Identity Provider (e.g., Google)
    │                      │                              │
    │  1. "Sign in with    │                              │
    │      Google"         │                              │
    │─────────────────────>│                              │
    │                      │                              │
    │  2. Redirect to Google                              │
    │<─────────────────────│                              │
    │     scope: "openid profile email"  ← NEW (openid   │
    │                                      scope required)│
    │                                                     │
    │  3-5. Same as OAuth (login, consent, code return)   │
    │                                                     │
    │                      │  6. Exchange code             │
    │                      │─────────────────────────────>│
    │                      │                              │
    │                      │  7. Response includes:        │
    │                      │     - access_token            │
    │                      │     - id_token  ← NEW         │
    │                      │<─────────────────────────────│
    │                      │                              │
    │                      │  The website now knows:       │
    │                      │  WHO you are (from id_token)  │
    │                      │                              │
    │  8. Welcome, Terry!  │                              │
    │<─────────────────────│                              │
```

### The ID Token

The ID token is a **JWT (JSON Web Token)** — a signed JSON blob with three parts separated by dots:

```
eyJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJodHRwczovL2dvb2dsZS5jb20iLCJzdWIiOiIxMjM0NTY3ODkiLCJlbWFpbCI6InRlcnJ5QGdtYWlsLmNvbSIsIm5hbWUiOiJUZXJyeSIsImdyb3VwcyI6WyJBcmdvQ0QgQWRtaW5zIl0sImV4cCI6MTcxMTEyMzQ1Nn0.SIGNATURE
```

Decoded, the middle part (the "payload") looks like:

```json
{
  "iss": "https://accounts.google.com",
  "sub": "123456789",
  "email": "terry@gmail.com",
  "name": "Terry",
  "groups": ["ArgoCD Admins"],
  "iat": 1711123456,
  "exp": 1711127056,
  "aud": "photo-printer-client-id"
}
```

| Field | Meaning |
|-------|---------|
| `iss` (issuer) | Who created this token (the identity provider's URL) |
| `sub` (subject) | A unique, stable identifier for you at this provider |
| `email` | Your email address |
| `name` | Your display name |
| `groups` | Groups you belong to (used for RBAC) |
| `iat` (issued at) | When the token was created |
| `exp` (expires) | When the token expires |
| `aud` (audience) | Who this token is intended for (the client_id of the app) |

The token is **signed** by the identity provider's private key. The website verifies the signature using the provider's public key (published at the JWKS endpoint). This means the token can't be tampered with — if anyone changes the email or groups, the signature won't match.

### The Discovery Document

OIDC standardizes how apps find the identity provider's endpoints. Every OIDC provider publishes a JSON document at:

```
https://<provider>/.well-known/openid-configuration
```

This returns:

```json
{
  "issuer": "https://accounts.google.com",
  "authorization_endpoint": "https://accounts.google.com/o/oauth2/v2/auth",
  "token_endpoint": "https://oauth2.googleapis.com/token",
  "userinfo_endpoint": "https://openidconnect.googleapis.com/v1/userinfo",
  "jwks_uri": "https://www.googleapis.com/oauth2/v3/certs",
  "scopes_supported": ["openid", "profile", "email"],
  "claims_supported": ["sub", "name", "email", "picture"]
}
```

This is why OIDC is much easier to integrate than raw OAuth — the app just needs the issuer URL and can auto-discover everything else.

---

## Part 3: OAuth vs OIDC — The Difference

| | OAuth 2.0 | OIDC |
|---|---|---|
| **Purpose** | Authorization — "what can this app do?" | Authentication — "who is this person?" |
| **Token** | Access token (opaque, for API calls) | ID token (JWT, contains identity) + access token |
| **Tells the app** | "This token can read photos" | "This is terry@gmail.com, member of ArgoCD Admins" |
| **Use case** | Third-party API access | Single Sign-On (SSO), federated login |
| **Standardized?** | Scopes/endpoints vary by provider | Standardized scopes, claims, discovery |
| **Built on** | — | OAuth 2.0 (extends it) |

**In short:** OAuth = authorization ("can I access this?"). OIDC = authentication ("who am I?") built on top of OAuth.

---

## Part 4: Real-World Examples

### Example 1: OAuth — GitHub App Accessing Your Repos

**Scenario:** A CI/CD tool (like CircleCI) wants to read your GitHub repos to run tests.

- **You** = Resource Owner
- **CircleCI** = Client
- **GitHub** = Authorization Server + Resource Server
- **Scope** = `repo:read`

Flow:
1. You click "Connect GitHub" in CircleCI
2. Redirected to GitHub: "Allow CircleCI to read your repos?"
3. You approve
4. CircleCI gets an access token scoped to `repo:read`
5. CircleCI calls GitHub API with that token to clone your code

CircleCI never knows your GitHub password. You can revoke access anytime. CircleCI can only read repos — it can't delete them or access your billing.

### Example 2: OIDC — SSO Login to ArgoCD via Authentik

**Scenario:** You want to log into ArgoCD without creating an ArgoCD-specific account.

- **You** = Resource Owner
- **ArgoCD (via Dex)** = Client
- **Authentik** = Identity Provider (Authorization Server)
- **Scopes** = `openid profile email groups`

Flow:
1. You click "Login via Authentik" on ArgoCD
2. Redirected to Authentik login page
3. You enter your Authentik username/password (or skip if already logged in — SSO!)
4. Authentik returns an authorization code to ArgoCD's Dex
5. Dex exchanges the code for an ID token
6. The ID token says: `email: terry@gmail.com, groups: ["ArgoCD Admins"]`
7. ArgoCD maps `ArgoCD Admins` → `role:admin`
8. You're logged in as admin

No ArgoCD password exists. Your identity lives in one place (Authentik). Add a new app? Just create another OIDC provider in Authentik — no new credentials needed.

### Example 3: OAuth — Mobile App Accessing Your Calendar

**Scenario:** A meeting scheduler app on your phone wants to read your Google Calendar.

- **Scope** = `calendar.readonly`
- **Special consideration** = Mobile apps can't keep a `client_secret` safe (users can decompile the app)

This uses the **PKCE (Proof Key for Code Exchange)** extension — a variation of the authorization code flow designed for public clients (mobile apps, SPAs) that can't store a secret. Instead of a client_secret, the app generates a one-time cryptographic challenge that proves it's the same app that started the flow.

### Example 4: OAuth — Machine-to-Machine (Client Credentials)

**Scenario:** Your backend server needs to call another API — no human involved.

This uses the **Client Credentials Grant** — the simplest OAuth flow:

```
Your Server                          API Server (Auth)
    │                                      │
    │  POST /oauth/token                   │
    │  client_id=xxx                       │
    │  client_secret=yyy                   │
    │  grant_type=client_credentials       │
    │─────────────────────────────────────>│
    │                                      │
    │  { "access_token": "abc123" }        │
    │<─────────────────────────────────────│
```

No browser, no redirect, no human. This is what the Tailscale operator uses — it sends its OAuth client_id and client_secret directly to the Tailscale API to get a token.

---

## Summary

```
Problem: "I have 50 passwords and apps want my data"
                    │
                    ▼
        ┌───────────────────┐
        │     OAuth 2.0      │
        │                    │
        │  "Here's a limited │
        │   access token,    │
        │   not my password" │
        └────────┬──────────┘
                 │
                 │  "But who IS the user?"
                 ▼
        ┌───────────────────┐
        │      OIDC          │
        │  (built on OAuth)  │
        │                    │
        │  "Here's an ID     │
        │   token that says  │
        │   who they are"    │
        └───────────────────┘
```

OAuth solved the "don't give apps your password" problem.
OIDC solved the "prove who you are across apps" problem.
Together, they power virtually every "Sign in with..." button and API integration on the internet.
