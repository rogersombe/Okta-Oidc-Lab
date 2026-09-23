# P5a — OIDC Authorization Code Flow (Okta)

## Overview

Registered a web application in Okta, ran the OAuth 2.0 / OIDC Authorization Code flow end to end, and decoded the resulting ID token to document the identity claims returned. Demonstrates practical understanding of the token-based authentication flow underlying modern SSO — the same mechanism used by Entra ID, Google Workspace, and every other major identity provider.

## What this proves

- Can register and configure an OIDC client application in an identity provider (client type, grant type, redirect URIs, scopes)
- Understands the two-stage token exchange: front-channel authorization code retrieval, back-channel code-for-token exchange
- Can read and interpret a decoded JWT — not just glance at it, but explain what each claim means and why it's there
- Diagnosed and resolved two distinct real-world misconfigurations without external troubleshooting help

## Setup

- **Identity provider:** Okta (Integrator Free tenant)
- **App type:** Web Application, OIDC, Authorization Code grant only
- **Client authentication:** Client secret
- **Redirect URI:** `http://localhost:8080/authorization-code/callback`
- **Scopes requested:** `openid profile email`

## The debugging story (the actual evidence of understanding)

Two failures occurred in sequence, both instructive:

**1. `access_denied` — Policy evaluation failed**
The app registration succeeded and was assigned to my user, but the authorization request still failed with a 400 and `access_denied`. Root cause: Okta's `default` Authorization Server enforces its own independent **Access Policy** layer — separate from app-level user assignment. A brand-new authorization server ships with zero policies, meaning *no client can obtain a token from it* until a policy and rule are explicitly created.

This is a distinction worth stating plainly in interview: app assignment answers "is this user allowed to use this app," while the authorization server's access policy answers "is this client allowed to request tokens from this server at all." Two independent gates, easy to configure one and forget the other.

Fix: Security → API → `default` → Access Policies → added a policy scoped to the client, with a rule permitting the Authorization Code grant type.

**2. Tooling friction — PowerShell's `curl` alias**
Once the Okta side was correctly configured, the token exchange request failed with a PowerShell parser error. `curl` in PowerShell is aliased to `Invoke-WebRequest`, which does not share the same flag syntax as the real curl binary — the command was being silently reinterpreted rather than failing cleanly. Resolved by explicitly invoking `curl.exe` to bypass the alias.

## Evidence

- [ ] Screenshot: authorization request redirect showing `code=` parameter in the browser address bar
- [ ] Screenshot: token exchange response (redact `access_token` and `id_token` values — show response shape/keys only)
- [ ] Screenshot: decoded ID token claims (jwt.io or equivalent)

## Decoded ID token — claims documented

| Claim | Value (example) | What it means |
|---|---|---|
| `iss` | `https://integrator-5120541.okta.com/oauth2/default` | Confirms which authorization server issued the token — a relying party must validate this matches the expected issuer |
| `aud` | Client ID | Confirms the token was issued *for this specific app* — prevents a token meant for App A being replayed against App B |
| `sub` | Okta internal user ID | Stable, unique identifier for the authenticated user |
| `exp` / `iat` | Unix timestamps | Token lifetime — this token expires 1 hour after issuance |
| `amr` | `["mfa", "otp", "pwd", "okta_verify"]` | Authentication Methods Reference — documents *how* the user proved their identity, not just that they did. Evidence that MFA was actually enforced at sign-in, not merely configured as policy |
| `name`, `email`, `preferred_username` | Profile claims | Returned because `profile` and `email` scopes were explicitly requested — token contents are scope-driven, not automatic |

The `amr` claim is the most interesting line for interview purposes: it's direct evidence that step-up authentication (password + OTP + Okta Verify) occurred, which is the kind of detail that separates "I called an OIDC endpoint" from "I understand what a relying party should actually be checking."

## Known limits / not covered in this lab

- No refresh token flow exercised (out of scope for this lab; candidate for a follow-up exercise)
- No PKCE (Proof Key for Code Exchange) — this app uses confidential client authentication (client secret) rather than a public client pattern; PKCE would be the correct addition for a SPA or mobile client
- Access Policy scoped broadly ("any scopes") rather than tightly to `openid profile email` — acceptable for a lab, would be tightened for production
