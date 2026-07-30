---
name: Authenticate a Kittl app user and connect a third-party provider
description: >-
  Verify a Kittl user on your own backend with a short-lived JWT, and run a third-party
  OAuth flow from inside the sandboxed app without exposing client secrets.
api: https://sdk-docs.kittl.dev/Guides/Advanced/authentication
surface: sdk
operations:
  - kittl.auth.getUserToken
  - kittl.auth.startAuth
  - kittl.auth.exchangeCode
  - kittl.auth.getAuthToken
  - KittlSDK.verifyUserToken
generated: '2026-07-19'
method: generated
source: https://sdk-docs.kittl.dev/Guides/Advanced/authentication
---

# Authenticate a Kittl app user and connect a third-party provider

Two separate concerns, often needed together.

## A. Identify the Kittl user to your own backend

1. **In the app frontend**, get a short-lived Kittl user JWT with
   `kittl.auth.getUserToken()` and send it to your backend.

2. **On your backend**, verify it with `@kittl/sdk-backend`:

   ```ts
   import { KittlSDK, TokenInvalidError } from '@kittl/sdk-backend';

   // Initialize once, at module load time.
   const kittlBackend = new KittlSDK({ appId: process.env.KITTL_APP_ID! });

   try {
     const payload = await kittlBackend.verifyUserToken(token);
     console.log(payload.sub);
   } catch (error) {
     if (error instanceof TokenInvalidError) {
       // Return 401 Unauthorized.
     }
     throw error;
   }
   ```

**Rules**

- `appId` is the JWT **audience** — tokens issued for a different app are rejected. Never
  skip it.
- `payload.sub` is a **pseudonymous per-app user ID**. It is not a stable cross-app Kittl
  identifier; do not treat it as one or try to join on it across apps.
- Signing keys are cached in memory per SDK instance (default: 1 hour, 5 keys). On edge or
  serverless backends where process memory is not reused, supply a persistent
  `SigningKeyCache` so keys survive cold starts.

## B. Connect a third-party OAuth provider

1. **Register the callback with your provider:**
   `https://app.kittl.com/auth/callback/:appId` (`:appId` is your app's ID).

2. **Declare the provider** in `manifest.json` under `config.oauthProviders`. Keys are
   camelCase; the key is the `provider` string you pass to `kittl.auth`.

   ```json
   {
     "config": {
       "oauthProviders": {
         "myProvider": {
           "clientId": "client_id_from_provider",
           "scope": "example:read,example:write",
           "authorizationUrl": "https://myAuthProvider.com/oauth2/authorize",
           "accessType": "offline",
           "tokenUrl": "https://myAuthProvider.com/oauth2/token"
         }
       }
     }
   }
   ```

3. **Run the flow.** Use PKCE whenever the provider supports OAuth 2.1.

   ```ts
   const provider = 'myProvider';

   const startResp = await kittl.auth.startAuth({ provider, generatePKCE: true });
   if (!startResp.isOk) return; // handle startResp.error

   const { code, code_verifier } = startResp.result;

   const exchangeResp = await kittl.auth.exchangeCode({ code, provider, code_verifier });
   if (!exchangeResp.isOk) return; // handle exchangeResp.error
   // exchangeResp.result holds the provider's token response payload
   ```

4. **Restore on reload.** On app load, check for an already-stored token before starting a
   new flow:

   ```ts
   const tokenResp = await kittl.auth.getAuthToken({ provider });
   if (tokenResp.isOk && tokenResp.result) { /* already connected */ }
   ```

**Rules**

- The point of `kittl.auth` is that provider secrets stay out of the sandboxed client —
  do not embed a client secret in app code.
- Set `config.appDevelopment.local.requireHTTPS: true` during local development when the
  provider requires a secure origin.
- `kittl.auth` is the app-sandbox OAuth helper. It is unrelated to `kittl auth login`,
  which is the CLI's developer sign-in.
