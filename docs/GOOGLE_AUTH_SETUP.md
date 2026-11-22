# Google Sign-In / Sign-Up Guide (Google Identity Services)

**Status:** ✅ Implemented and verified
**Flow:** GIS button → Google ID token → `POST /api/auth/google` → JWT session

---

## Overview

TaskFlow uses **Google Identity Services (GIS)** — a "Sign in with Google" button rendered by Google on the frontend. When a user picks an account, Google returns a **signed ID token** (JWT) directly in the browser; the frontend POSTs it to the API, which validates the token signature, issuer, audience, and expiry against Google's OIDC endpoints, then issues the app's own JWT access/refresh tokens.

- Works for **both sign-in and sign-up** — an unknown Google account auto-creates a `Customer` user on first use.
- Sessions use the same JWT + refresh-token mechanism as email/password auth (no cookies).
- No redirect URI is needed for the active flow; the unused legacy `AddGoogle` OAuth handler in `Program.cs` can be removed.

---

## Prerequisites

- Google account with [Google Cloud Console](https://console.cloud.google.com/) access
- TaskFlow running locally or in production

---

## Step 1: Create a Google Cloud Project

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Click the project dropdown → **New Project**
3. Name it `TaskFlow` (or any name) → **Create** → select it

---

## Step 2: Configure the OAuth Consent Screen

1. **APIs & Services** → **OAuth consent screen**
2. User type: **External** → **Create**
3. Fill in app name (`Mercato`/`TaskFlow`), support email, developer contact
4. Add scopes: `email`, `profile`, `openid`
5. Add test users (required while the app is in "Testing" mode — anyone else gets `access_denied`)
6. For production: publish the app and complete verification (privacy policy + terms URLs)

---

## Step 3: Create OAuth 2.0 Credentials

1. **APIs & Services** → **Credentials** → **+ Create Credentials** → **OAuth client ID**
2. Application type: **Web application**
3. Name: `TaskFlow Web Client`
4. Configure **Authorized JavaScript origins** (no path, no trailing slash):
   - Local dev: `http://localhost:5173`
   - Docker dev: `http://localhost`
   - Production: `https://taskflow-das.duckdns.org`
5. **Authorized redirect URIs are NOT required** for the GIS button flow. (The legacy `/signin-google` callback exists in the API but is unused by the frontend.)
6. **Create** → copy the **Client ID** and **Client Secret**

---

## Step 4: Configure Credentials

### Frontend (required)

`VITE_GOOGLE_CLIENT_ID` must be set so the GIS button initializes:

- Local: `frontend/.env` → `VITE_GOOGLE_CLIENT_ID=<client-id>` (or `frontend/.env.local`)
- Production: GitHub secret `VITE_GOOGLE_CLIENT_ID` → baked into the frontend image at build time via `ARG VITE_GOOGLE_CLIENT_ID`

Without it, `GoogleAuthButton` renders nothing.

### Backend (required)

The API validates the ID token's audience against `Authentication:Google:ClientId` and can optionally exchange tokens via `ClientSecret`:

- Local: `Authentication__Google__ClientId` / `Authentication__Google__ClientSecret` in `appsettings` or `.env`
- Production `.env` keys: `Authentication__Google__ClientId`, `Authentication__Google__ClientSecret`
- GitHub secrets: `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`

The **ClientId must match the frontend's `VITE_GOOGLE_CLIENT_ID`** — the ID token's `aud` claim is checked against it.

---

## Step 5: Restart

```bash
# Local
docker compose -f docker-compose.dev.yml up -d --build

# Or frontend-only dev
cd frontend && npm run dev   # reads .env / .env.local
```

For production, push to `main` — the deploy workflow rebuilds the frontend image with `VITE_GOOGLE_CLIENT_ID`.

---

## Step 6: Test

1. Open `http://localhost:5173/register` or `/login`
2. Click **Continue with Google** → Google account chooser appears
   - To use a different account than the one shown: click **"Use another account"**, or test in an incognito window (the chooser shows previously used accounts — normal Google behavior)
3. Pick an account → brief "Signing in…" state → redirect to home, logged in
4. Verify in the DB: a `User` row with `GoogleId` set and `Role = Customer`

### Check the backend logs if it fails

```bash
docker compose -f docker-compose.dev.yml logs api | grep -i google
```

The validator logs the exact JWT validation failure; `AuthService` logs missing claims.

---

## How It Works

1. `GoogleAuthButton` loads `https://accounts.google.com/gsi/client`, calls `google.accounts.id.initialize({ client_id, callback })` and `renderButton`
2. User picks an account → Google calls the callback with `response.credential` (a signed JWT)
3. Frontend POSTs `{ credential }` to `POST /api/auth/google`
4. `GoogleTokenValidator.ValidateToken` validates signature (Google OIDC JWKS), issuer (`accounts.google.com` / `https://accounts.google.com`), audience (`ClientId`), and lifetime → returns `ClaimsPrincipal`
5. `AuthService.LoginWithGoogleAsync` reads `sub`, `email`, `given_name`, `family_name`, `name` — looking up **both raw JWT claim names and .NET-mapped claim types** (`NameIdentifier`, `EmailAddress`, `GivenName`, `Surname`, `Name`) — finds or creates the `User`, issues JWT access + refresh tokens
6. Frontend stores tokens and redirects home

> **Note:** `JwtSecurityTokenHandler` maps inbound claims by default (`sub` → `ClaimTypes.NameIdentifier`, `email` → `emailaddress`). `AuthService` handles both forms — if you add claim lookups elsewhere, use the same dual-lookup helper.

---

## API Endpoints

| Endpoint | Method | Auth | Description |
|----------|--------|------|-------------|
| `/api/auth/google` | POST | No (rate-limited: `auth` policy) | Accepts `{ credential }` (Google ID token), returns app tokens |
| `/api/auth/login` | POST | No (rate-limited) | Email/password login (lockout after 5 failures) |
| `/api/auth/me` | GET | Yes | Current user profile |
| `/api/auth/refresh` | POST | Refresh token | Rotate tokens |

---

## Troubleshooting

### "Sign in with Google" button doesn't appear
- `VITE_GOOGLE_CLIENT_ID` not set at **build time** (prod) or in `frontend/.env.local` (dev)
- Check browser console for GIS script errors; ad-blockers can block `accounts.google.com`

### `400 invalid_request` / `origin_mismatch` from Google
- The page origin isn't in **Authorized JavaScript origins** — add the exact origin (scheme + host + port, no path)

### `401` from `/api/auth/google`
- Check API logs — validator logs the exact reason:
  - `idTokenLength: 0` → frontend sent empty credential (GIS callback issue)
  - `IDX...` / `SecurityTokenMalformed` → token rejected by Google validation (wrong audience, expired, malformed)
  - `Missing required Google claims` → token valid but `sub`/`email` lookup failed
- Most common cause historically: claim-name mapping — now handled in `AuthService`

### "Access blocked" / `access_denied`
- App still in **Testing** mode and the Google account isn't a listed test user → add it under OAuth consent screen → Test users
- Or click "Use another account" in the chooser if an old account is pre-selected

### Console warnings (harmless)
- `Provided button width is invalid` — fixed; GIS needs pixel width, measured from the container
- `Cross-Origin-Opener-Policy ... postMessage` — emitted by Google's own sign-in iframe; appears on all GIS sites
- `Tracking Prevention blocked ... <image-host>` — browser blocking storage for a third-party product image; image still loads

---

## Security Notes

- ID tokens are validated server-side (signature, `iss`, `aud`, `exp`) — never trust client-supplied email/claims
- `/api/auth/google` is covered by the `auth` rate-limit policy (20 req/min/IP)
- Google accounts have no password; "forgot password" doesn't apply — users sign in via the Google button
- Registering with email/password for an email that already exists as a Google account returns 400 "Email already exists" — direct users to the Google button

---

## Related Docs

- [Environment Setup](ENVIRONMENT_SETUP.md)
- [CI/CD Setup](CI_CD_SETUP.md)
- [Production Deployment](PRODUCTION_DEPLOYMENT.md)
