# Resend Email Service Setup

## Overview

TaskFlow uses **Resend** for transactional email (order confirmations, shipping notifications, email verification, password reset). The application automatically uses:

- `MockEmailService` in **Development** — logs emails to console, sends nothing
- `ResendEmailService` in **Production/Staging** — sends via the Resend API

Email failures are logged but **never fail** the order/checkout — orders complete even if email delivery fails.

---

## Configuration

### 1. Create a Resend Account

1. Sign up at https://resend.com
2. **API Keys** → create a key (starts with `re_`) → copy it

### 2. Choose a Sender Address

| Mode | Sender | Delivers to |
|------|--------|-------------|
| **Testing** (no domain verified) | `onboarding@resend.dev` | Only your Resend account email |
| **Production** | Any address on a verified domain, e.g. `noreply@yourdomain.com` | Anyone |

### 3. Verify Your Domain (Required for real delivery)

1. Resend dashboard → **Domains** → **Add Domain**
2. Add the SPF (TXT) and DKIM (CNAME) records Resend provides to your DNS
3. Wait for propagation (5–30 min) → click **Verify**
4. Send from any address on that domain

Without verification, use `onboarding@resend.dev` — but note it **only delivers to the email address you signed up to Resend with**, and email providers will reject/mark-as-spam mail from unverified domains.

### 4. Environment Variables

Config binds to the `Resend` section — **double underscores** in env vars map to `:`:

**Development (`.env` / `.env.dev`)** — optional; MockEmailService is used anyway:
```bash
Resend__ApiKey=re_your_test_key
Resend__FromEmail=onboarding@resend.dev
```

**Production (`.env` / GitHub secrets)** — required; API **fails fast at startup** if missing:
```bash
Resend__ApiKey=re_your_production_key
Resend__FromEmail=noreply@yourdomain.com   # or onboarding@resend.dev for testing
```

| GitHub secret | Written to `.env` as |
|---|---|
| `RESEND_API_KEY` | `Resend__ApiKey` |
| `RESEND__FROMEMAIL` | `Resend__FromEmail` |

> ⚠️ **Secret value pitfall:** store the *actual* value in the GitHub secret — never the literal text `${{ secrets.RESEND_FROM_EMAIL }}`. The deploy workflow single-quotes values; a literal `${{ }}` in the secret lands verbatim in `.env` and Resend rejects it with `422 Invalid 'from' field`. This exact bug broke production email once — verify secret contents if sends fail.

### 5. Application Configuration

Registered in `Program.cs` — `IEmailService` resolves through the typed `HttpClient` (which carries `BaseAddress = https://api.resend.com/`):

```csharp
// Typed HttpClient registration gives ResendEmailService a BaseAddress.
builder.Services.AddHttpClient<ResendEmailService>(client =>
{
    client.BaseAddress = new Uri("https://api.resend.com/");
});

// Outside Development, IEmailService resolves to the typed client — NOT a
// plain AddScoped<IEmailService, ResendEmailService> (which would get an
// HttpClient with no BaseAddress and fail on every send).
builder.Services.AddScoped<IEmailService>(sp => sp.GetRequiredService<ResendEmailService>());

// Production fail-fast validation
builder.Services.AddOptions<ResendOptions>()
    .Bind(builder.Configuration.GetSection("Resend"))
    .Validate(o => !string.IsNullOrWhiteSpace(o.ApiKey) && !string.IsNullOrWhiteSpace(o.FromEmail),
        "Resend:ApiKey and Resend:FromEmail must be configured in production.")
    .ValidateOnStart();
```

---

## Testing

### Development
Emails are logged, not sent:
```
[MockEmailService] ... to: user@example.com
```

### Production
1. Resend dashboard → **Logs**: https://resend.com/logs
2. API container logs: `docker logs taskflow-api | grep -i resend`
3. Confirm the `from` address is `onboarding@resend.dev` or on a verified domain
4. With `onboarding@resend.dev`, the recipient must be your Resend account email — other addresses are dropped by Resend

---

## Idempotency

`ResendEmailService` sends an `Idempotency-Key` header so retries don't duplicate mail:
- Order confirmation: `order-confirmation-{orderNumber}`
- Shipped notification: `order-shipped-{orderNumber}-{trackingNumber}`

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| API crash-loops at startup | `Resend:ApiKey`/`Resend:FromEmail` empty in production | Set `RESEND_API_KEY` + `RESEND__FROMEMAIL` GitHub secrets, redeploy |
| Resend `422 Invalid 'from' field` | `FromEmail` is a placeholder/invalid sender | Set `RESEND__FROMEMAIL` to `onboarding@resend.dev` or a verified-domain address |
| `Invalid request URI ... BaseAddress` | `IEmailService` resolved without typed client | Fixed in `Program.cs` — keep the `AddHttpClient` registration |
| Emails accepted but never arrive | `onboarding@resend.dev` only delivers to your Resend account email | Verify a domain and use a real sender |
| Emails in spam | Domain not SPF/DKIM verified | Complete domain verification + DMARC |

Password-reset note: `RequestPasswordResetAsync` always returns success (doesn't leak account existence), so the UI shows "check your inbox" even if the send fails — check API logs/Resend logs for the real status.

---

## Cost

- Resend free tier: **3,000 emails/month**, then ~$1 per 1,000
- Sufficient for a portfolio/demo store

## Security

- Never commit `RESEND_API_KEY` — it lives only in `.env` / GitHub secrets
- Rotate the key if exposed
- `.env*` files are gitignored and excluded from Docker build context (`.dockerignore`)

## Related Docs

- [Environment Setup](ENVIRONMENT_SETUP.md)
- [CI/CD Setup](CI_CD_SETUP.md) — secrets list
- [Production Deployment](PRODUCTION_DEPLOYMENT.md)
