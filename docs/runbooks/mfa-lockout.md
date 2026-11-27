# MFA lockout

## Symptoms

- A user can't finish signing in. After their password, the code step keeps saying "That code is wrong", or "Your sign-in expired" after 5 tries.
- They've lost the phone with their authenticator app (new phone, app deleted, factory reset) **and** don't have their recovery codes.
- auth logs show `mfa failed` lines for their `userId`, with `reason` `wrong_code` or `too_many_attempts`.

How it works: see [ADR 0009](../decisions/0009-totp-mfa.md) and [Login with MFA](../architecture/auth-flow.md#login-with-mfa).

!!! tip "See it in Grafana"
    The **Failed MFA codes** panel counts `mfa failed`. To see one user's attempts:

    ```logql
    {container="auth"} | json | message="mfa failed" | userId="<userId>"
    ```

!!! danger "Check identity before you reset"
    Resetting MFA removes the protection it gives. "I lost my phone" is exactly what someone with a stolen password would say. Only reset when you're sure it's the account owner.

## Check

1. **Is it a clock problem, not a lost device?** Codes are accepted for 30 seconds either side. A phone whose clock is several minutes off produces codes that are always wrong. Ask the user to turn on automatic date and time, then try again. If that works, stop here.

2. **Do they have recovery codes?** On the code step, **Use a recovery code** accepts one of the 10 codes from setup (the `blob-storage-recovery-codes.txt` download, or wherever they copied them). Case, spaces and dashes don't matter. If one works, they're in. Ask them to turn MFA off and on again from the home page, to get a new app setup and fresh codes. Stop here.

3. **Find the user:**

    ```bash
    docker compose -p auth-service logs --no-log-prefix auth | grep '"mfa failed"' | tail -20
    ```

    Note the `userId` and `clientIp`. Many failures from IPs the user doesn't recognise means someone else is trying their password: they should change it after the reset.

4. **Confirm it's them.** Use something only the real owner can do: reply from the account's email address and answer details only they know (recent file names, when they signed up). Write down what you checked.

## Fix

Run the reset with the account's email. It turns MFA off, deletes the TOTP secret and recovery codes, and signs the user out everywhere:

```bash
# Docker
docker compose -p auth-service exec auth bun run reset-mfa user@example.com

# Local (from Blob_auth/, uses DB_PATH or ./auth.sqlite)
bun run reset-mfa user@example.com
```

It prints nothing on success and logs `mfa reset by admin` with the `userId`. "No user with that email." means the address is wrong.

Then tell the user:

1. Sign in with just the password.
2. Set up two-factor authentication again on the home page, with the new phone.
3. Save the new recovery codes somewhere other than the phone.
4. Change the password if step 3 of the check showed attempts from IPs they don't recognise.

## If MFA_ENCRYPTION_KEY was lost

If auth_service's `MFA_ENCRYPTION_KEY` changed or was lost, **every** MFA user fails: their stored secrets can't be decrypted, and their recovery codes no longer match. The auth logs show `mfa failed` for everyone, or an `unhandled error` on `/login/mfa`.

- Put the old key back if you can. Nothing else needs to change.
- If it's gone, reset each MFA user with `reset-mfa` and ask them to set it up again. A new key can be made with `bun run gen-mfa-key`.

## Prevent

- Keep `MFA_ENCRYPTION_KEY` backed up alongside `AUTH_PRIVATE_KEY`, in a secrets manager ([Security checklist](../operations/security.md)).
- The setup screen asks users to save their recovery codes before it switches to "on". Remind users to keep them off the phone.
