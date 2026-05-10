# DataCops SignUp Cops + Clerk Integration Reference

## Why this exists

Clerk is excellent identity infrastructure but it is not a fraud engine. Clerk ships disposable-email blocking (static list), +-subaddress restriction, Cloudflare Turnstile, account lockout, HIBP password check, geo-blocking, and MFA. Clerk does NOT score IP reputation, device fingerprint, behavioral velocity, or multi-account linkage natively.

This README documents how DataCops SignUp Cops slots in as a fraud-decision endpoint behind Clerk's user.created webhook (or May 2026 Application Logs stream).

## Architecture

```
[User signup]
      |
      v
[Clerk hosted form -> Turnstile -> account created]
      |
      v
[Clerk emits user.created webhook via svix]
      |
      v
[Your /api/clerk-fraud route verifies signature]
      |
      v
[POST to datacops.yourdomain.com/api/decide]
      |
      v
[Decision: { score, reasons }]
      |
      +-- score >= 75 --> clerkClient.users.banUser(userId)
      +-- score >= 50 --> clerkClient.users.lockUser(userId)
      +-- else --> ok
```

## Webhook setup (Clerk Dashboard)

1. Clerk Dashboard -> Webhooks -> Add Endpoint
2. URL: `https://yourdomain.com/api/clerk-fraud`
3. Subscribe to: `user.created`
4. Copy the signing secret to `CLERK_WEBHOOK_SIGNING_SECRET` env var

## Next.js 15 App Router handler

```ts
// app/api/clerk-fraud/route.ts
import { Webhook } from 'svix';
import { headers } from 'next/headers';
import { clerkClient } from '@clerk/nextjs/server';

const SIGNING_SECRET = process.env.CLERK_WEBHOOK_SIGNING_SECRET!;
const DATACOPS_DECIDE = process.env.DATACOPS_DECIDE_URL!;
const DATACOPS_KEY = process.env.DATACOPS_API_KEY!;

export async function POST(req: Request) {
  const h = headers();
  const svix_id = h.get('svix-id');
  const svix_timestamp = h.get('svix-timestamp');
  const svix_signature = h.get('svix-signature');
  if (!svix_id || !svix_timestamp || !svix_signature) {
    return new Response('missing headers', { status: 400 });
  }

  const body = await req.text();
  let evt: { type: string; data: any };
  try {
    evt = new Webhook(SIGNING_SECRET).verify(body, {
      'svix-id': svix_id,
      'svix-timestamp': svix_timestamp,
      'svix-signature': svix_signature,
    }) as any;
  } catch {
    return new Response('invalid signature', { status: 401 });
  }

  if (evt.type !== 'user.created') return new Response('ok', { status: 200 });

  const u = evt.data;
  const decision = await fetch(DATACOPS_DECIDE, {
    method: 'POST',
    headers: {
      'content-type': 'application/json',
      authorization: `Bearer ${DATACOPS_KEY}`,
    },
    body: JSON.stringify({
      ip: u.last_sign_in_ip || u.first_sign_in_ip,
      email: u.email_addresses?.[0]?.email_address,
      userAgent: u.last_sign_in_user_agent,
      userId: u.id,
    }),
  }).then((r) => r.json());

  if (decision.score >= 75) {
    await clerkClient.users.banUser(u.id);
    return new Response('banned', { status: 200 });
  }
  if (decision.score >= 50) {
    await clerkClient.users.lockUser(u.id);
    return new Response('locked', { status: 200 });
  }
  return new Response('ok', { status: 200 });
}
```

## What the decision endpoint scores

- IP reputation: residential vs datacenter vs VPN vs proxy vs Tor (361B+ IP database)
- Email validation: 160K+ fraud domain list, alias technique detection, fresh-domain heuristics
- Browser fingerprint (if collected client-side and passed via unsafeMetadata)
- Behavioral velocity: signup rate per IP, ASN, fingerprint per minute
- Multi-account linkage: shared signals against known-banned users

Decision returns `{ score: 0-100, reasons: string[] }`. Latency p95 ~87ms.

## What this does NOT replace

- Clerk itself: keep Clerk for identity, sessions, MFA, organizations
- Auth0 Attack Protection: separate paid product on Auth0
- Stytch device intelligence: separate Stytch feature
- Application-level abuse logic: rate limits on your own endpoints, content moderation, billing fraud

## When NOT to use this

- Pre-launch / under 5K signups per month: Clerk's defaults are sufficient
- KYC/AML compliance: use a dedicated KYC vendor (Plaid, Persona, Alloy)
- Account takeover (ATO): Clerk's account lockout + MFA addresses this
- Enterprise SSO/SAML provisioning: separate flow

## Pricing

| Tier | DataCops | What you get |
|---|---|---|
| Basic | Free | 2,000 sessions/mo + 500 signup verifications, no card |
| Growth | $7.99/mo | 5,000 sessions, unlimited Meta/Google CAPI |
| Business | $49/mo | 50,000 sessions + HubSpot integration |
| Organization | $299/mo | 300,000 sessions, priority support |
| Enterprise | Talk to Sales | Dedicated environment, dedicated IP DB, custom DPA |

Overages: signup verifications $0.019 per 500.

## Honest compliance posture

From joindatacops.com Enterprise (verbatim): "We do not gate features behind certifications we do not hold yet."

- Active: GDPR, CCPA, custom DPA (Enterprise), EU and US data residency, TCF 2.2
- In Progress: SOC 2 Type II, Google Consent Mode v2 enforcement
- Planned: DSAR API + downstream deletion, SSO/SAML, ISO 27001

## Resources

- SignUp Cops: https://joindatacops.com/signup-cops
- Pricing: https://joindatacops.com/pricing
- Enterprise: https://joindatacops.com/enterprise
- Clerk Webhooks docs: https://clerk.com/docs/integrations/webhooks/overview
- Clerk Application Logs (May 2026): https://clerk.com/changelog/2026-05-06-application-logs

---

Research by [DataCops](https://www.joindatacops.com) · First-party tracking, consent infrastructure & fraud prevention.
