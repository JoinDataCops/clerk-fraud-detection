# Clerk fraud detection

Search "Clerk fraud detection" and you get two kinds of results: Clerk's own marketing pages, and a pile of stuff about county clerks and court records. **There is no real playbook.** So here is one, written by someone who has shipped Clerk into production and then watched fake signups walk straight through it.

Here is the honest read. Clerk has real fraud controls. They are competent, they are on by default, **and most teams never bother to learn what they actually cover**. But Clerk's built-in controls all live at the same layer: the credentials and the request rate. **Clerk does not natively score who is behind the request.** It does not tell you the IP is a datacenter proxy, the device has fingerprinted itself onto 40 other accounts, or the behavior looks scripted.

This is not a "Clerk is bad" post. Clerk is a good auth platform. This is a post about a specific, real gap, and the webhook pattern that closes it. [DataCops](/signup-cops) is what I plug into that webhook, and I will be specific about why and where. Related: [Fraud traffic validation](/fraud-traffic-validation), [Auth0 signup fraud](/resources/auth0-signup-fraud), [Best signup fraud detection 2026](/resources/best-signup-fraud-detection-2026).

## Quick stuff people keep asking

**Does Clerk have fraud detection?** It has fraud controls, which is not quite the same thing. Clerk blocks disposable email domains, restricts plus-addressed and subaddressed emails, locks out brute-force attempts, checks passwords against HaveIBeenPwned, and can require Cloudflare Turnstile. That is solid abuse prevention. It is not risk scoring. Clerk decides "is this credential allowed and is this request rate sane," not "is this signup likely fraudulent based on IP, device, and behavior."

**How do I block disposable emails in Clerk?** It is a built-in setting. In the Clerk dashboard, under the email configuration for your instance, enable the option to block disposable email addresses. Clerk maintains the domain list. It catches the common throwaway providers without you maintaining anything. Understand the limit: disposable-domain lists always trail new domains, and they do nothing against real-looking inboxes on aged domains created specifically to look legitimate.

**Can Clerk detect bot signups?** To a point. Turnstile, Cloudflare's CAPTCHA alternative, is built into Clerk's signup flow and stops naive bots. But CAPTCHA-class challenges in 2026 are a weak filter. Bots solve them at very high rates, and an AI agent driving a real browser session sails through Turnstile while looking like a normal user. Clerk catches the dumb bots. It does not catch the modern ones.

**Does Clerk integrate with Cloudflare Turnstile?** Yes, natively. Clerk uses Turnstile as its bot-protection challenge on sign-up and sign-in, and it is largely managed for you. Treat it as a speed bump, not a wall. It raises the cost of trivial automation. It does not stop a determined or AI-driven attacker.

**How do I add fraud detection to a Clerk webhook?** Clerk emits webhook events through its event system, including a `user.created` event. Point that webhook at your own endpoint. When a user is created, your endpoint receives the payload, calls a fraud-detection service with the signup's signals, gets a risk decision, and acts on it: flag the user in your database, hold them in a pending state, restrict entitlements, or queue for review. This is the standard production pattern, and it is where real risk scoring gets added.

**What does Clerk do about brute force attacks?** Clerk has built-in brute-force protection. Repeated failed sign-in attempts trigger lockouts and rate limiting on the account and the request source. This protects existing accounts from credential-stuffing and password guessing. Note what it does not address: a brute-force lockout protects a real account from takeover. It does nothing about mass creation of new fake accounts, which is a different attack with a different shape.

**Can Clerk block plus-addressed emails?** Yes. Clerk can restrict subaddressed emails, the `user+anything@gmail.com` pattern, so one inbox cannot mint unlimited distinct-looking signups. Enable it if you run a free trial. Know the limit: it stops the lazy version of trial abuse. It does nothing against an attacker using many genuinely separate inboxes, or catalog email aliasing on their own domain.

## The gap: Clerk validates the credential, not the actor

Line up everything Clerk does and a clear pattern shows up. Disposable-email blocking, subaddress restriction, brute-force lockout, HaveIBeenPwned checks, Turnstile. Every one of those inspects either the credential being presented or the rate of requests. None of them inspect the actor behind the request.

That is the gap, and it matters because modern signup fraud has moved past the credential. The attacker is not using `test@mailinator.com` anymore. They have a real-looking inbox on a domain they registered last month. They are not hammering your endpoint 500 times a minute, so rate limiting never trips. They are driving a real headless browser, so Turnstile passes. Every credential-layer and rate-layer control Clerk has waves them through, because on those axes they look fine.

What gives them away is the stuff Clerk does not look at. The IP: is it residential, or a datacenter range, a VPN exit, a proxy, a Tor node. The device fingerprint: is this the 41st account from one device. The email domain freshness: was that domain registered eleven days ago. The behavior: did the form fill in 0.4 seconds with no mouse movement, did 600 accounts arrive in a tight burst from one ASN. Clerk has no native concept of any of this. It is not a Clerk failure. It is a scope boundary. Clerk is an authentication and user-management platform. Actor-level risk scoring is a different product category.

Here is the proof of what slips through. PillarlabAI ran a honeypot, a deliberately attractive signup target, and pulled 3,000 signups. 77% of them were fraudulent. Not 7%. 77%. And 650 of those accounts traced back to a single device fingerprint. One device. A credential-layer control sees 650 different email addresses, 650 different passwords, 650 requests spread out enough to dodge rate limits, and waves all 650 through as distinct, legitimate users. A device-fingerprint signal sees one machine and collapses the whole thing instantly. That is the exact difference between what Clerk inspects and what catches modern fraud.

And the damage does not stop at a dirty user table. Those 650 fake accounts each generated a signup event. If your signup event fires a conversion to Meta or Google, you just told the ad platforms that 650 fake accounts are 650 ideal customers. The optimizer believes you. It goes and finds more traffic that looks like those fakes. Your cost per real acquisition climbs, your ROAS degrades, and the root cause is sitting in your user table wearing 650 names and one fingerprint. Garbage signups in, garbage ad optimization out.

## The webhook pattern that closes it

You do not rip out Clerk. You extend it. Clerk stays the front door and does what it is good at: credential validation, session management, the brute-force and Turnstile basics. You add an actor-risk layer behind it using the webhook.

The shape is straightforward. Clerk fires `user.created` to your endpoint. Your endpoint calls a fraud-detection service with the signup's signals, the IP, the device data, the email domain. The service returns a risk assessment with context: IP type, device-fingerprint reuse, domain age, behavioral flags. Your endpoint decides. Low risk, activate normally. Elevated risk, hold the user in a pending or limited state, withhold the trial entitlement, or route to manual review. The decision happens before the account gets the keys.

One detail that matters: do the risk check before you grant entitlements, not after. If you provision the trial, the credits, the API key first and check fraud later, the abuser already got the value. The webhook decision has to gate activation, not just annotate a record.

This is the layer I run DataCops in. Why DataCops specifically for the Clerk webhook: it scores exactly the signals Clerk does not. IP intelligence across residential, datacenter, VPN, proxy, and Tor, against a 361.8 billion-plus IP database, so a datacenter or proxy signup gets surfaced. Device and identity intelligence at signup through SignUp Cops, so the 650-accounts-one-device pattern actually shows up instead of presenting as 650 separate users. And because DataCops also runs the first-party analytics and CAPI pipeline, the fraud signal and the conversion signal live in the same place. A signup flagged as fraud does not get sent to Meta and Google as a conversion. That closes the ad-optimization leak, not just the user-table leak.

There is a free tier worth knowing about for this exact pattern: 2,000 signup verifications a month at no cost, which covers a lot of early-stage signup volume before you pay anything.

I will be straight about the limits. DataCops is a newer brand than the long-established fraud incumbents, and SOC 2 Type II is still in progress, so a regulated buyer with a hard compliance gate may need to wait for that to close. DataCops surfaces risk with context, it gives you a decision to act on, it does not silently "block" fraud for you, and you should not want it to. The action stays in your webhook handler, where you control it. And to be exact about one thing: DataCops does not claim to catch 100% of fraud. Nothing does. It surfaces the actor-level signals Clerk structurally cannot see, and that is the whole point of putting it on the webhook.

## Decision guide

**You run a free trial or freemium product.** Subaddress and disposable-email blocking are table stakes. Turn them on, then add the webhook risk layer, because trial abuse has long since moved past plus-addressing.

**Your signup numbers look great but activation and revenue do not.** That gap is fake accounts inflating the top of the funnel. A device-fingerprint signal on the webhook will tell you fast.

**You are sending signup as a conversion event to Meta or Google.** This is urgent. Filter fraud before the conversion fires, or you are paying the ad platforms to find you more fakes.

**You only need to stop low-effort spam.** Clerk's built-ins plus Turnstile may genuinely be enough. Do not over-buy. Add the webhook layer when you see real targeted abuse.

**You are comparing Clerk against Auth0 on fraud.** They are close, and both validate at the credential and rate layer. Neither natively scores actor risk. Whichever you pick, you are adding the webhook layer regardless.

**You have a hard SOC 2 requirement today.** Note DataCops has SOC 2 Type II in progress and weigh that timeline.

## Clerk is the lock. It is not the bouncer.

The mistake I see people make with Clerk is reading the security feature list, seeing disposable-email blocking and brute-force protection and Turnstile, and concluding fraud is handled. It is not handled. It is handled at one layer, the credentials and the request rate. The actor behind the request, the IP, the device, the behavior, is completely unexamined by default.

Clerk is the lock on the door. A lock checks that the key fits. It does not check who is holding the key, or notice that the same person has walked through 650 times wearing 650 different coats. That check is the webhook layer, and it is on you to add it.

So go look. Pull your last 1,000 signups and ask one question Clerk cannot answer for you: how many distinct devices created those accounts? If that number is a lot smaller than 1,000, you already have the answer about whether the credential layer was enough.

---

Research by [DataCops](https://www.joindatacops.com) — first-party tracking, consent infrastructure, fraud prevention, and server-side CAPI for Meta, Google, TikTok, and LinkedIn.
