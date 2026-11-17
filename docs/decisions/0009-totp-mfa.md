# 0009. Optional TOTP multi-factor authentication with recovery codes

## Status

Accepted, 2026-11-17.

## Context

Until now a password was all it took to sign in. Passwords get reused, phished and leaked, and anyone who has one can read and delete that user's files. We want a second factor that:

- works for every user without hardware or a phone number,
- needs no paid third-party service (SMS or email delivery),
- keeps the other services unaware of how a user signed in, unless they choose to check.

## Decision

auth_service offers **TOTP** (RFC 6238: SHA-1, 6 digits, 30-second steps), the codes that Google Authenticator, 1Password, Authy and similar apps generate. It's **optional per user**.

- **Setup:** `POST /mfa/setup` returns a secret and an `otpauth://` link, which the frontend shows as a QR code. `POST /mfa/enable` turns MFA on only after a correct code, so a mis-scanned QR code can't lock anyone out. It returns **10 one-time recovery codes**, shown once.
- **Login:** with MFA on, `POST /login` checks the password but starts no session. It returns an `mfaToken`, a 5-minute JWT with audience `mfa`, which no service accepts as an access token. `POST /login/mfa` exchanges it plus a code for the normal session. Each `mfaToken` allows 5 wrong codes and one sign-in.
- **Replay:** each code works once. auth_service stores the last accepted 30-second step, and marks recovery codes as used.
- **Storage:** TOTP secrets are encrypted with AES-256-GCM, and recovery codes are stored as HMAC-SHA-256. Both keys are derived from `MFA_ENCRYPTION_KEY`, which is kept apart from the database.
- **Signal to other services:** access tokens carry `amr`, `["pwd"]` or `["pwd","otp"]`, kept across refreshes. A service can require MFA for a sensitive action by checking it, with no call to auth_service.
- **Sensitive actions:** turning MFA off needs the password and a code. Deleting the account needs a code too when MFA is on.
- **Lockout:** a user who lost both their phone and their recovery codes is reset by an admin with `bun run reset-mfa <email>`. See [MFA lockout](../runbooks/mfa-lockout.md).

See [Login with MFA](../architecture/auth-flow.md#login-with-mfa).

## Consequences

Good:

- A stolen password alone no longer opens an account that has MFA on.
- No new infrastructure, no per-message cost, works offline on the phone.
- The other services need no change. `amr` is there when they want it.
- A copy of the SQLite file alone doesn't let anyone generate codes or test recovery codes.

Bad:

- It's optional, so it protects only users who turn it on. Enforcing it for everyone is a later decision.
- TOTP can still be phished in real time: a fake login page can relay a code within its 30 seconds. Passkeys don't have this weakness.
- `MFA_ENCRYPTION_KEY` is one more secret to manage. Losing it locks every MFA user out until an admin resets them, and it's now required to start auth_service.
- Lockouts need a person to check identity before running the reset. There's no self-service email reset, on purpose: email is often the account an attacker already controls.
- The count of wrong codes per `mfaToken` is in memory, like the rate limiter. It resets on restart and isn't shared between instances.

## Alternatives considered

- **SMS codes**: phone numbers can be taken over by SIM swapping, delivery costs money and needs a provider, and we'd have to store phone numbers (more personal data under GDPR).
- **Email codes**: an attacker who has the password often has the email account too, and it adds an email provider.
- **Passkeys / WebAuthn**: phishing-resistant and the best long-term option, but more work on both ends (registration ceremonies, multiple credentials per user, browser support quirks). A good next step on top of TOTP, not instead of it.
- **Recovery codes hashed with `Bun.password` (Argon2id)**: about 160 ms per code. Checking a wrong recovery-shaped code against 10 hashes cost up to 1.6 s of CPU. The codes are random (50 bits), so a keyed HMAC is enough and needs only one indexed lookup.
- **Mandatory MFA for everyone**: safer, but it would lock out existing users who have no authenticator app. Start optional, measure adoption, then decide.
