# Auth

**Status:** Draft · **Prefix:** AUTH

## Purpose

Accounts, sign-in, second factor, recovery, sessions and API keys for a multi-user web app.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| AUTH-1 | User: `id`, `email` (unique, lower-cased), `email_verified_at?`, `username` (unique, `[a-z0-9-]`, 3–40, no `+`), `name`, `locale`, `locked` (default false), `role`: `user`\|`admin` (instance admin), `avatar_url?`, `bio?`, `time_zone` (IANA, required; sign-up proposes the browser's), `week_start` (`monday`\|`sunday`, default `monday`). | signup `Ann@X.com` → stored `ann@x.com` |
| AUTH-2 | Sign up with `email`, `password`, `username`, `name` (1–100 chars) and any field another spec requires at sign-up; `locale` defaults to `en`. Sign-up can be turned off instance-wide, except for invite links a requiring spec defines (e.g. `@spechub/teams:TEAM-5`). Taken email or username → `taken`. | signup disabled, no invite → refused |
| AUTH-3 | Passwords: ≥ 10 chars, ≤ 128, not in the instance's breached/common list, not containing the username or email local part; no composition rules. Admins: ≥ 15 chars and 2FA required before admin actions. | `password1234` → refused (common); `Password1` → refused (short) |
| AUTH-4 | Passwords stored with a slow adaptive hash (argon2id, or bcrypt cost ≥ 12). | manual: stored hash format |
| AUTH-5 | Email verification: single-use random token (≥ 128 bits), valid 24 h; verifying sets `email_verified_at`. Unverified users can sign in but can't use actions a requiring spec marks as needing a verified email. | expired token → refused |
| AUTH-6 | Sign in by email + password; unknown email, wrong password and password-less accounts all give the same `invalid_credentials`. Locked users → `locked`, only after a correct password (else `invalid_credentials`). | unknown email vs wrong password → same error |
| AUTH-7 | OIDC sign-in (at least Google, Microsoft). Provider email must be verified. An existing account with the same email is linked only if that account's email is verified. | unverified local account + Google same email → refused |
| AUTH-8 | Magic-link sign-in: single-use, valid 10 minutes. | link at 11 min → refused |
| AUTH-9 | TOTP 2FA (RFC 6238, 30 s, accepts current and previous step): setup needs current password, is enabled only after one valid code; 10 single-use backup codes; secret encrypted at rest; disabling needs password + code. | backup code reused → refused |
| AUTH-10 | 2FA user signing in by OIDC or magic link still must give a code. | Google sign-in, 2FA on → code asked |
| AUTH-11 | Password reset: response identical whether the email exists; token single-use, valid 1 h; ≤ 3 requests per email per 15 min; a new request voids older ones; reset signs out all sessions. | reset → old session rejected |
| AUTH-12 | Change password needs the current one; new ≠ old. | same password → refused |
| AUTH-13 | Sessions: opaque token of ≥ 128 random bits, only its SHA-256 stored; HTTP-only, Secure, SameSite=Lax cookie; idle timeout 30 days, sliding; list and revoke own sessions. | revoke session → next request 401 |
| AUTH-14 | API keys: `prefix_` + ≥ 128 random bits; only a SHA-256 hash stored; shown once; `{note, expires_at? (default 90 days), last_used_at, scope: user\|team}`; sent as `Authorization: Bearer`; expired by timestamp. Rotate = new key, old one revoked. | expired key → 401 |
| AUTH-15 | Rate limits (fixed window, fail closed): sign-up, sign-in, reset, verify: 10/min per IP and per email; 2FA endpoints 5/min per user. Over → 429 with `Retry-After`. | 11th sign-in in a minute → 429 |
| AUTH-16 | Delete account: password (or OIDC re-auth) + 2FA code if on; removes personal data; owned resources follow the requiring spec's rule (e.g. `@spechub/teams:TEAM-9`), and without one they are deleted with the account. | delete → can't sign in, email reusable |
| AUTH-17 | State changes from cookie sessions need a non-GET request from the same origin: SameSite=Lax cookie plus an `Origin` check; cross-origin → 403. | POST with foreign `Origin` → 403 |
| AUTH-18 | Instance `mode`: `multi` (default) or `single_owner`. In `single_owner` the first sign-up becomes the owner and an instance admin, and sign-up then closes; more users exist only if an admin creates them (AUTH-22). | single_owner: second sign-up → refused |
| AUTH-19 | In `single_owner` mode `email` is optional and sign-in is by `username`; without an email, password reset is only through an operator recovery action on the server that issues a one-time reset link (valid 1 h). | no email: forgot-password → explains operator recovery |
| AUTH-20 | An API key acts as its creator, limited to the creator's current access to the key's scope; it stops working when the creator loses that access or is deleted. | creator removed from team → team key 401 |
| AUTH-21 | API requests (session or key) are limited to 120 per minute per key or session by default (operator-configurable); responses carry `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`; over → 429 with `Retry-After`. | 121st call in a minute → 429 |
| AUTH-22 | Instance admins can create users (email or username, name, time zone) even with sign-up off; the new user gets a one-time set-password link valid 72 h. Admins can lock and unlock users. | admin creates user → link sets password once |
| AUTH-23 | Sensitive actions (changing email or password, 2FA changes, API keys, account deletion, and any action a requiring spec marks sensitive) need a password, OIDC or 2FA confirmation within the last 10 minutes. | change email 11 min after sign-in → re-auth asked |

## Out of scope

- SAML/SCIM, passkeys (planned next), CAPTCHA.

## Decisions

- NIST 800-63B password rules instead of the source's composition rules (7 chars, upper/lower/digit).
- Rate limiting fails closed; the source failed open.
- Magic link 10 minutes, as the source intended (its code said 10 hours).
- 0.2: no hard dependency on `@spechub/teams`; personal apps use `single_owner` mode. Time zone lives on the user because most requiring specs need one.
