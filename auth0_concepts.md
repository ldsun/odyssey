# Auth0 Concepts

A practical explanation of Auth0's core building blocks.

---

## Tenant

**What it is:** The top-level Auth0 account. Everything lives inside a tenant — all your users, applications, connections, and settings. It is identified by a domain name (e.g. `your-name.us.auth0.com`).

> **Key insight:** Auth0 stores the login session at the tenant level, not the application level.

**Mental model:** Think of the tenant as your Auth0 "workspace". It is the boundary of your identity universe. All your applications share the same tenant, so a user who signs up via one app exists in the same user pool as one who signs up via another. This is what makes **SSO work** — once a user is logged in to any app in the tenant, other apps in the same tenant can silently pick up that session without asking for credentials again.

**Environments = separate tenants.** Best practice is one tenant per environment (dev / staging / prod). Tenants are completely isolated — users, configs, and tokens from one tenant are not valid in another.

---

## Application

**What it is:** A registered client that is allowed to authenticate users through the tenant. Each application gets a unique **Client ID** and, depending on its type, a **Client Secret**. It also has its own allowed callback URLs, logout URLs, and token settings.

**Mental model:** An application uses Auth0's service to verify its users. It must be registered with Auth0 in order to use that service — the Client ID is the credential that proves it is allowed to do so.

**How an application proves its own identity to Auth0** depends on where it runs:

- **Regular Web App** — proves identity with a **static shared secret** (the Client Secret) stored server-side. It's like a password: the same secret is sent on every token exchange. Safe because it never leaves the server.

- **SPA** — cannot use a static secret because everything runs in the browser, which is readable by anyone (DevTools, extensions, XSS). Instead, it uses **PKCE**: on each login the app generates a temporary key pair — a random `code_verifier` (the private side, kept in memory) and a `code_challenge` (its hash, the public side, sent to Auth0 upfront). When exchanging the authorization code for tokens, the app reveals the `code_verifier`. Auth0 hashes it and checks it matches the challenge — proving the app that initiated the login is the same one completing it. The pair is thrown away after one use, so intercepting one exchange gives an attacker nothing reusable.

  The mental model: **a temporary public/private key pair, generated fresh per login and discarded immediately after.**

**Application types:**

| Type | Proves identity via | When to use |
|---|---|---|
| **Single Page Application (SPA)** | PKCE (per-login key pair) | Browser-only React / Vue apps |
| **Regular Web Application** | Client Secret (static, server-side) | Server-rendered apps with a backend |
| **Native** | PKCE | iOS, Android, or desktop apps |
| **Machine to Machine (M2M)** | Client Secret | Service-to-service calls (e.g. backend → Auth0 Management API) |

**Why use separate applications for separate frontends?** Each app has its own allowed callback URLs, token lifetimes, and allowed connections. Keeping them separate means a staff/admin app and a consumer app can be configured independently. SSO still works between them because it operates at the tenant level.

---

## Connection

**What it is:** The login method offered to users of an application. It determines what happens when a user clicks "Sign in".

**Mental model:** The application decides *that* Auth0 handles authentication. The connection decides *how* users actually log in. When a user hits "Sign in", the connection is what they see next:

- `Username-Password-Authentication` → an email and password form
- `google-oauth2` → redirected to Google's login page
- `saml-enterprise` → redirected to their company's SSO

An application can allow multiple connections — e.g. "sign in with email/password or Google". Auth0 then shows both options on the login screen.

| Connection type | How users log in |
|---|---|
| **Database** | Email + password stored in Auth0 or your own DB |
| **Social** | Google, Apple, GitHub, Facebook OAuth |
| **Enterprise** | SAML, OIDC, Active Directory / LDAP |
| **Passwordless** | Email magic link, SMS OTP |

A connection can be **shared across multiple applications**. If two apps both allow the same connection, they see the same underlying user record — the user doesn't have separate accounts.

---

## Database (Connection)

**What it is:** A connection type where credentials (email + hashed password) are stored either in Auth0's own managed database or in your own database via a custom script. Most setups use the Auth0-managed option.

**Mental model:** A Database connection is Auth0's own user store. When a user signs up with email/password, Auth0 handles hashing, breach detection, and MFA — you don't touch the credential store directly.

**Why you might have multiple Database connections:**

A common pattern is to have one connection for real users and a separate isolated connection for automated testing. Test accounts in the test connection are invisible to real applications (since the applications are not configured to allow that connection), so test runs can freely create and delete users without touching the real user list.

---

## Resource Server (API)

**What it is:** Your backend API, registered with Auth0 so that tokens can be issued for it. The key field is the **identifier** — a string (usually a URL) that becomes the `aud` (audience) claim in the JWT.

**Mental model:** There are three parties in every OAuth flow:

| Party | Role | Example |
|---|---|---|
| **Authorization Server** | Issues tokens | Auth0 |
| **Client** | Requests tokens | Your frontend app |
| **Resource Server** | Receives and validates tokens | Your backend API |

The Resource Server registration is how you tell Auth0 that your backend exists. Without it, Auth0 doesn't know what to put in the `aud` claim and won't issue a proper JWT.

**How the audience ties them together:**

1. The frontend requests a token for a specific audience (e.g. `https://api.example.com`)
2. Auth0 checks: is that audience registered? Yes → issues a JWT with `"aud": "https://api.example.com"`
3. The backend receives the token and validates: does `aud` match what I expect? Yes → accept the request

**What the registration does NOT do** — it doesn't configure your backend. It is purely an Auth0-side declaration that a resource server exists at this identifier and applications are allowed to request tokens for it.

**RS256 and the JWKS endpoint** — tokens are signed with Auth0's private key (RS256). The backend verifies the signature using Auth0's public key, fetched from the JWKS endpoint (`/.well-known/jwks.json`). The backend never shares a secret with Auth0 — it just fetches the public key and checks the signature. This means even if someone steals a token, they cannot forge a new one without Auth0's private key.

---

## How the pieces fit together

```
User clicks "Sign in"
    │
    ▼
Browser redirects to Auth0 (tenant domain)
    with client_id=...       ← identifies which Application
    and redirect_uri=...     ← must be in the app's Allowed Callback URLs
    │
    ▼
Auth0 checks: which Connections does this Application allow?
    → Username-Password-Authentication ✓
    │
    ▼
User enters email + password
    Auth0 verifies against the Connection's user store
    │
    ▼
Auth0 issues tokens (access token + ID token)
    and redirects back to redirect_uri with an auth code
    │
    ▼
App exchanges auth code for tokens
    Stores them in memory (SPA — not in localStorage)
    │
    ▼
App calls backend with Authorization: Bearer <access_token>
    Backend validates token against Auth0's JWKS endpoint
```

---

## SSO

There are two distinct SSO concepts in Auth0 — it is worth keeping them separate.

### 1. Auth0's own SSO (within a tenant)

Auth0 provides this natively, no third party needed. When a user logs in through one application, Auth0 sets a **session cookie on the tenant domain**. This cookie is independent of the application and Client ID.

When the same user opens a second application in the same tenant and that app calls `loginWithRedirect`, Auth0 detects the existing session and silently issues a new token — no password entry required.

This works across any two applications in the same tenant, even with different Client IDs, as long as both allow a common connection.

### 2. Enterprise SSO / Federation

This is what the **"SSO Integrations"** section in the Auth0 dashboard is for. Here, Auth0 acts as a bridge to an external corporate identity provider — Okta, Azure AD, Google Workspace, Ping, etc. The user logs in with their corporate credentials at the upstream provider; that provider confirms their identity back to Auth0; Auth0 then issues a token to your application.

```
User → your app → Auth0 → Okta (corporate IdP) → Auth0 → your app
```

This is used when your users already have identities managed elsewhere — e.g. a company's employees log in with their work Okta account rather than creating a new password in Auth0.

### Auth0 and Okta

Okta acquired Auth0 in 2021. They are the same company but remain separate products with different focuses:

| | Auth0 | Okta |
|---|---|---|
| **Target** | Developers integrating auth into their own apps | IT teams managing workforce identity |
| **How you use it** | Integrate via SDK into your codebase | IT deploys and configures it for employees |
| **Typical use case** | Your app's sign-up / login flow | Employee SSO for internal tools (Slack, Salesforce, etc.) |

They can work together: Okta manages corporate identity; Auth0 federates from Okta so your app's users log in with their corporate credentials.
