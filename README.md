# Clerk fraud detection in 2026: an honest inventory and a working webhook pattern

Clerk is excellent identity infrastructure. It is not a fraud engine. The 2026 SERP for Clerk fraud detection is a wasteland of Clerk's own marketing pages plus unrelated county clerk results. Founders shipping Next.js apps on top of Clerk keep asking the same question and not finding the answer: what does Clerk actually do for signup fraud, and what do I need to bolt on?

This page is the inventory. Every Clerk built-in named honestly, mapped against the specific fraud vectors each one fails to cover, plus a copy-pasteable webhook recipe (user.created hits a fraud-decision endpoint, and if the score is high you call Clerk Backend API to ban or lock the user before activation).

The context for 2026. Imperva 2025: bad bots are 37% of all internet traffic, automated traffic is 51% of web traffic, the first time it has surpassed human activity. MyEmailVerifier roll-up: 20-30% of new SaaS account registrations are fraudulent or bot-generated, spiking to 40-60% during promotional peaks. ipasis: ~33% of freemium SaaS accounts use disposable email domains. Onsefy: a mid-sized SaaS at 25% fake-account rate burns $5K-$15K/mo ($60K-$180K/yr) on infrastructure, email, and support for fraudulent users.

MRC's 2026 report: 64% of merchants saw a meaningful increase in first-party misuse, with 25% reporting increases of 25%+. BleepingComputer's March 2026 piece on modern fraud chains framed it neatly: single-signal defenses always lag behind, attacks are a relay race stitching bots, residential proxies, aged emails, and manual ATO. Clerk's bot protection is single-signal (Cloudflare Turnstile only).

February 2026 Clerk raised the free tier from 10K to 50K MAU, bundled MFA into Pro, and moved Enterprise Connections to metered. May 2026 Clerk shipped Application Logs as an event stream for auth, billing, and orgs events. April 2026 CVE-2026-0000 was disclosed, an authorization bypass when combining reverification with role/permission/feature/plan checks (patched April 22).

The net of all that. Clerk is shipping fast on identity. The fraud surface remains exactly what it was in 2024: a static disposable-email list, +-subaddress block, Cloudflare Turnstile, account lockout, HIBP password check. The free tier expansion 5x'd the bot-signup blast radius before pricing applies pressure to clean it up.

---

## Quick stuff people keep asking

**Does Clerk have fraud detection?** Partially. Clerk has bot sign-up protection (Cloudflare Turnstile), disposable email blocking (static list), +-subaddress restriction, brute-force lockout, HaveIBeenPwned password check, and geo-blocking. These are identity controls, not a fraud engine. Clerk does not natively score IP reputation, device fingerprint, behavioral velocity, or multi-account linkage.

**How do I block disposable emails in Clerk?** Clerk Dashboard, User & Authentication, Email and SMS, toggle the disposable-email block. Static list shipped August 2023. Sophisticated abusers use rotating private domains that the static list never sees.

**Can Clerk detect bot signups?** Single-signal only. Cloudflare Turnstile rendered via the `<div id="clerk-captcha" />` element. Invisible CAPTCHA was deprecated. Turnstile is good against unsophisticated bots and farmable for Turnstile-solving services that cost ~$1 per 1,000 solves on the open market.

**Does Clerk integrate with Cloudflare Turnstile?** Yes, it is the default bot-protection signal. No additional configuration if you use Clerk's hosted forms.

**How do I add fraud detection to a Clerk webhook?** Subscribe to user.created via svix (Clerk's webhook infrastructure) or Clerk Application Logs (May 2026), POST to your fraud-decision endpoint, and if the score is high call the Clerk Backend API to ban (`users.ban`) or lock (`users.lock`) the user. Pattern below.

**What does Clerk do about brute force attacks?** Account lockout shipped December 2023, kicks in on repeated failed attempts. Effective against credential-stuffing on a single account. Does not address signup-side abuse where each attempt creates a new account.

**Can Clerk block plus-addressed emails?** Yes, the +-subaddress restriction toggle blocks `user+anything@example.com` patterns. Independent toggle from disposable email block. Does nothing against rotating private domains.

---

## What Clerk actually does for fraud (the honest inventory)

**1. Disposable email blocking**

The Good: shipped August 2023. Toggle in the Clerk Dashboard. Catches the most obvious mailinator, tempmail, 10minutemail traffic.

Frustrations: static list. Sophisticated abusers use rotating private domains that never hit the list. Per ipasis 2026, ~33% of freemium SaaS accounts use disposable domains, but the share moving to private rotating domains is rising as the static lists catch up to the public providers.

Wish List: dynamic list refresh. Hooks into a third-party email-reputation API.

Value for Money: **6.5/10.** Worth turning on. Insufficient by itself.

---

**2. +-subaddress restriction**

The Good: blocks `user+a@example.com`, `user+b@example.com` patterns. Independent toggle from disposable block. Catches the lazy free-trial abuser pattern.

Frustrations: does nothing against attackers who control their own domain (`user@privatedomain.com`, `user2@privatedomain.com`). Modern free-trial abuse rarely uses + addressing because the technique is well-known.

Wish List: detection of catch-all domains, not just + patterns.

Value for Money: **6/10.** Free toggle, turn it on, do not call it fraud detection.

---

**3. Cloudflare Turnstile (bot signup protection)**

The Good: replaced the older Visual CAPTCHA in 2024. Rendered via `<div id="clerk-captcha" />`. Frictionless for real users. Cloudflare's signal is genuinely strong against unsophisticated bots.

Frustrations: single-signal. Turnstile-solving services exist and price at ~$1 per 1,000 solves. Modern fraud chains (per BleepingComputer March 2026) are a relay race that stitches bots, residential proxies, and aged emails. The single-signal defense always lags behind. Clerk does not augment Turnstile with IP reputation, device fingerprint, behavioral velocity, or multi-account linkage.

Wish List: native risk-scoring on top of Turnstile. Pluggable signal pipeline.

Value for Money: **6.5/10.** Necessary. Not sufficient.

---

**4. Account lockout (brute-force protection)**

The Good: shipped December 2023. Effective against credential-stuffing on existing accounts. Configurable.

Frustrations: addresses ATO (account takeover), not signup-side abuse. Each new account is a fresh slate against the lockout.

Wish List: signup-side velocity limits per IP, ASN, device fingerprint.

Value for Money: **7/10.** Real protection on the right surface (existing accounts). Wrong surface for signup fraud.

---

**5. HaveIBeenPwned password check**

The Good: blocks signups with passwords that have appeared in known breaches. Encourages users toward unique passwords. Cheap signal, high value.

Frustrations: addresses password reuse, not bot signups, disposable emails, or trial abuse. Orthogonal to the fraud-detection problem most operators face.

Wish List: integration with credential-stuffing signal (failed logins on the same IP across accounts).

Value for Money: **8/10.** Excellent feature, wrong category for fraud.

---

**6. Geo-blocking**

The Good: block signups from specific countries. Useful for SaaS with regulatory exposure.

Frustrations: VPNs and residential proxies route around it trivially. Modern abuse routes through the same regions you serve real users.

Wish List: ASN and proxy detection, not country-only.

Value for Money: **5.5/10.** Helps with compliance posture, does not stop sophisticated fraud.

---

**7. MFA (Require MFA toggle, Feb 2026)**

The Good: single-toggle Require MFA across the entire app. Strong protection against ATO once accounts exist.

Frustrations: addresses ATO, not signup fraud. Disposable-email and bot-signup vectors are unchanged.

Wish List: signup-time risk scoring that triggers step-up MFA only when the risk score warrants.

Value for Money: **8/10.** Excellent ATO protection. Orthogonal to signup fraud.

---

## What Clerk does NOT do (the gap)

**8. IP reputation and risk scoring**

The Good: the right architecture for 2026 fraud. Most cloud IPs are not running people, they are running bots. Datacenter detection is the easiest layer to win.

Frustrations: Clerk does not score IPs natively. Imperva 2025 says automated traffic is 51% of web traffic. The IP layer is the cheapest, fastest fraud signal and Clerk leaves it on the table.

Wish List: native IP reputation, residential vs datacenter vs VPN vs proxy vs Tor categorization.

Value for Money: **N/A** (not shipped).

---

**9. Device fingerprinting**

The Good: canvas, WebGL, audio, screen, fonts, plugins fingerprint identifies the same physical device across new accounts. Catches multi-account abuse where each attempt uses a fresh email.

Frustrations: Clerk does not fingerprint devices natively. This is the single biggest gap for trial-abuse use cases. Stytch publishes device-intelligence benchmarks; Auth0 has paid Attack Protection. Clerk has neither.

Wish List: native browser fingerprint at the signup form.

Value for Money: **N/A** (not shipped).

---

**10. Behavioral velocity**

The Good: signup rate per IP, per ASN, per device fingerprint per minute is a strong fraud signal. 50 signups from the same ASN in 5 minutes is not normal traffic.

Frustrations: Clerk does not surface velocity controls. Each signup is evaluated in isolation.

Wish List: configurable velocity limits in the dashboard.

Value for Money: **N/A** (not shipped).

---

**11. Multi-account linkage**

The Good: linking new accounts to known-bad accounts via shared device, IP, payment method, or behavioral signature is how mature fraud teams catch professional abusers.

Frustrations: Clerk does not link accounts on shared signals. Once a user is banned, the same actor can sign up again with a fresh email.

Wish List: native account-linkage graph.

Value for Money: **N/A** (not shipped).

---

## The webhook pattern (copy-pasteable, Next.js + svix)

Clerk's user.created event is the natural integration point. As of May 2026, Clerk Application Logs is also a clean stream. Here is the pattern.

### Step 1: configure the webhook

In Clerk Dashboard, Webhooks, create endpoint pointing to your fraud-decision API route (e.g. `https://yourdomain.com/api/clerk-fraud`). Subscribe to `user.created`. Copy the signing secret.

### Step 2: verify and route the event

```ts
// app/api/clerk-fraud/route.ts (Next.js 15 App Router)
import { Webhook } from 'svix';
import { headers } from 'next/headers';
import { clerkClient } from '@clerk/nextjs/server';

const SIGNING_SECRET = process.env.CLERK_WEBHOOK_SIGNING_SECRET!;

export async function POST(req: Request) {
  const headerPayload = headers();
  const svix_id = headerPayload.get('svix-id');
  const svix_timestamp = headerPayload.get('svix-timestamp');
  const svix_signature = headerPayload.get('svix-signature');

  if (!svix_id || !svix_timestamp || !svix_signature) {
    return new Response('missing svix headers', { status: 400 });
  }

  const body = await req.text();
  const wh = new Webhook(SIGNING_SECRET);

  let evt;
  try {
    evt = wh.verify(body, {
      'svix-id': svix_id,
      'svix-timestamp': svix_timestamp,
      'svix-signature': svix_signature,
    }) as { type: string; data: any };
  } catch (err) {
    return new Response('invalid signature', { status: 401 });
  }

  if (evt.type !== 'user.created') {
    return new Response('ok', { status: 200 });
  }

  const user = evt.data;
  const ip = user.last_sign_in_ip || user.first_sign_in_ip;
  const email = user.email_addresses?.[0]?.email_address;
  const userAgent = user.last_sign_in_user_agent;

  // Step 3: call your fraud decision
  const decision = await fetch('https://datacops.yourdomain.com/api/decide', {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ ip, email, userAgent, userId: user.id }),
  }).then((r) => r.json());

  // Step 4: act on the decision
  if (decision.score >= 75) {
    await clerkClient.users.banUser(user.id);
    return new Response('banned', { status: 200 });
  }

  if (decision.score >= 50) {
    await clerkClient.users.lockUser(user.id);
    return new Response('locked for review', { status: 200 });
  }

  return new Response('ok', { status: 200 });
}
```

### Step 3: the fraud-decision endpoint

This is where DataCops or any equivalent fraud layer slots in. POST receives `{ ip, email, userAgent, userId }`. Returns `{ score, reasons }`. The decision engine evaluates IP reputation (residential vs datacenter vs VPN vs proxy vs Tor), email validation (disposable, fresh domain, alias techniques), browser fingerprint if collected client-side, behavioral velocity per IP and ASN, and multi-account linkage to existing banned users.

With DataCops the IP reputation database is 361B+ entries (202B residential, 146.4B datacenter, 11.9B VPN, 620M proxy) and the fraud-email-domain list is 160K+ entries. SignUp Cops is the product surface that powers this decision endpoint.

### Step 4: act before activation

Ban via `clerkClient.users.banUser(userId)` if the score is high. Lock via `clerkClient.users.lockUser(userId)` for manual review at medium scores. Both happen before the user can authenticate further sessions.

Advanced: collect a browser fingerprint client-side on the signup form (canvas, WebGL, audio, screen, fonts) and POST it as `unsafeMetadata` to Clerk's `signUp.create()` call, then read it from the user.created event for the decision. Clerk does not collect this natively.

---

## When to stop bolting on and add a fraud layer

**Pre-launch.** Don't bother. Ship the product. Turn on Clerk's defaults (disposable email block, +-subaddress block, Turnstile, account lockout, HIBP, MFA). The bot signup blast radius is small enough that manual review handles it.

**Past 50K MAU on the new free tier.** Now bot-signup blast radius is real. The 5x increase in free-tier ceiling (Feb 2026) means the inflection point arrives sooner than under the old 10K limit. Add a webhook decision layer.

**Free trial product with paid conversion.** Bolt on day one. Trial abuse drains infrastructure and pollutes conversion rate metrics. Onsefy's $5K-$15K/mo waste range applies once you cross 10K monthly signups at typical 25% fake rates.

**B2B with org abuse.** Clerk Organizations introduce a different fraud surface: invite spam, fake org creation, seat-abuse for free-tier features. Add the layer when the first paid org reports phantom seats.

**Compliance-bound.** Anyone subject to KYC, AML, or financial regulation needs a fraud layer at signup, full stop. Clerk's defaults are not the compliance surface.

---

## Clerk vs Auth0 vs Stytch on fraud (be honest)

**Auth0 (Okta).** Has Attack Protection / Bot Detection as a paid add-on. Stronger native fraud surface than Clerk. Practitioner reports of ~3x pricing increases post-Okta acquisition.

**Stytch.** Publishes device-intelligence benchmarks. Closest to native auth + fraud pitch in the category. B2B-focused.

**Clerk.** Single-signal Turnstile only. Strong identity surface, weak fraud surface. The webhook pattern above is how production teams compensate.

**The architectural take.** None of the auth providers ship a complete fraud engine. Auth0 is closest, Stytch second, Clerk third. All three are better paired with an out-of-band fraud layer than relied on alone. CVE-2026-0000 (Clerk authorization bypass, April 2026) is a reminder that auth-platform-native authorization is not a substitute for an out-of-band trust check.

---

## So what should you actually use?

Want Clerk's identity surface plus native bot detection? Try Auth0. Budget for the post-Okta pricing.

Want device intelligence baked in? Try Stytch.

Want Clerk's DX (which is genuinely the best in the category) and need to add a fraud layer? Keep Clerk and bolt on a webhook decision endpoint. The pattern above is the recipe.

Want the webhook decision endpoint as a managed service with the IP reputation database, browser fingerprint, email validation, and Clerk Backend API integration already built? Try DataCops SignUp Cops. Free tier is real (500 signup verifications on Basic, 2,000 sessions per month, no card).

---

## The mistake I see people make

Treating Clerk's defaults as a complete fraud surface. Disposable-email block plus Turnstile plus account lockout sounds like a stack. It is a starting point. The 2026 attack pattern (BleepingComputer's relay race framing) chains residential proxies plus aged email plus Turnstile-solving services plus manual ATO. Single-signal defenses always lag behind. The webhook decision layer is the answer Clerk's docs do not write.

---

## Now your turn

If you run Clerk in production today, what is the actual fraud signal you wish was native, IP reputation, device fingerprint, or behavioral velocity?

---

Research by [DataCops](https://www.joindatacops.com) · First-party tracking, consent infrastructure & fraud prevention.
