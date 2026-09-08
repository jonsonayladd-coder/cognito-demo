# B.R.A.I.N. — SMS provider bottleneck: research and decision

Researched 8 Sep 2026. Parked in this repo because it is the one this session had
access to; it belongs with `brain-worker`. Move it there.

## The short version

The wall is not where it looks. Twilio's "business account" requirement applies to
**alphanumeric sender IDs** and to the **Business** end-user type. B.R.A.I.N. cannot
use an alphanumeric sender ID at all — nobody can reply to one — so that whole
regulatory branch is irrelevant to this product.

What B.R.A.I.N. needs is a **two-way Australian virtual mobile number (VMN, +614)**.
Virtual numbers sit entirely outside the ACMA Sender ID Register. Twilio's own
published requirement for an AU **mobile** number as an **Individual** is:

- name + government-issued ID or passport
- Australian address + proof of address (utility bill, tax notice, rent receipt, title deed)

No ABN. No ASIC company extract. No public-facing website. (Those are all in the
*Business* column, and the *website* requirement is in the *Local number* column —
either of which would explain what got hit.)

So there are three live paths, none of them blocked.

## Path A — Mobile Message (recommended for Phase 1)

Australian, direct-to-carrier, self-serve, no sales call.

- Free dedicated Australian VMN, claimed after the first credit purchase.
  Claiming asks for **full name (a person, not a business), date of birth, service
  address**. No ABN.
- 1.6¢/SMS on first purchase, then 2–4¢. No monthly fee, no API fee.
- **Inbound is free.**
- Documented inbound webhook (`POST` JSON) and DLR webhook, set in Settings > API.
- 50 free credits to test before spending anything.
- Number goes stale if unused: send at least one message from it every six months.

Caveat: the API is SMS-only. `DOCKET` / `RECEIPT` / `HOLDPOINT` / `PHOTO` are MMS and
are **not** covered by this provider. That is fine — Phase 1 is attendance, which is
pure SMS. MMS is a Phase 2 provider decision, not a Phase 1 blocker.

## Path B — Cellcast

Australian, supports both SMS and MMS, so it is the one to look at when photos land.

- Dedicated VMN $18/month or $168/year, kept permanently. Free shared number also
  available (replies work on it, but a shared number is wrong for this product).
- SMS ~2.9–4.7¢ depending on volume. Replies free.
- **MMS ~29–34¢ each.** Ten times SMS. Put this in the cost model before promising
  anyone photo dockets — 40 workers sending one docket photo a day is ~$250/month/site
  on its own, more than the entire attendance load.

## Path C — Twilio, Individual end-user type

Still the best docs, still the most expensive, and now the only one of the three with
a **signed webhook** (see Finding 2 below), which for this product is not a nicety.

- AU mobile numbers: two-way SMS supported, MMS-enabled AU long codes available.
- Regulatory Bundle, Individual, reviewed in ~3 business days.
- Outbound AU SMS US$0.0515. Inbound is billed (unlike A and B). Number rental on top.
- Note Twilio's own line: its services are "intended for business use or use in
  connection with an individual's trade, craft, or profession only." Building this for
  payment qualifies; it is not a consumer-use loophole.

## The ABN question, answered separately

Get one anyway. It is free, it takes 10–15 minutes at **abr.gov.au** (not a paid
third-party site), and with a TFN on hand it usually issues within minutes. Being paid
~$2k to build software is carrying on an enterprise, which is the eligibility test.

It is not required for Path A or Path C-as-individual, so it is **not** on the critical
path to a working number this week. But it is required by ClickSend for Australian
direct customers, it will be required the day an alphanumeric sender ID is wanted, and
the ACMA Scam Prevention Framework is tightening KYC across every provider through
2026 — the identity bar goes up from here, never down. Do it in parallel, not first.

A fourth option exists and should be refused for now: registering the number under SIG
or Jimmy's entity. It solves consent cleanly (they would be messaging their own
workers) and costs nothing, but the number then belongs to them. With no contract and
no signed scope, do not hand over the one asset the pilot runs on.

## Cost model (one site, 40 workers, 2 messages/day, 22 days ≈ 1,760 inbound + 1,760 outbound)

| Provider | Number | Inbound | Outbound | ~Monthly |
|---|---|---|---|---|
| Mobile Message | free | free | 2–4¢ | **$35–70** |
| Cellcast | $18/mo | free | 2.9–3.7¢ | $69–83 |
| ClickSend | $19/mo | free | 7.2¢ | $146 (ABN required) |
| Twilio | rental | billed | 5.15¢ | ~$91 + inbound + rental |

SMS remains the entire cost line and it scales linearly with adoption. This is the
number a builder will ask for.

## Finding 1 — Mobile Message gives you no inbound message ID

`architecture.md` says to deduplicate on the gateway's message ID. Mobile Message's
inbound payload does not contain one:

```json
{ "to": "...", "sender": "...", "message": "...", "received_at": "...",
  "type": "inbound", "original_message_id": "...", "original_custom_ref": "..." }
```

`original_message_id` is the *outbound* message being replied to — empty when a worker
texts `IN` cold, which is the normal case. Meanwhile the gateway **retries up to 10
times with backoff from 60s to 1 hour** on any non-200. So duplicates are guaranteed
and there is no vendor key to dedupe on.

The adapter must synthesise an idempotency key — e.g. `sha256(to | sender | received_at
| body)` — and the schema needs `messages_raw.gateway_msg_id` to accept a synthesised
value with a provenance flag saying it was synthesised rather than vendor-issued. Two
identical texts a second apart from one worker are indistinguishable under that key;
decide deliberately whether that collapses (correct for retries) or is a lost message
(wrong for a worker who genuinely sent `IN` twice). Twilio's `MessageSid` has none of
this problem.

Also unresolved: whether long *inbound* messages arrive split one webhook per part. The
docs state that explicitly for status updates and ambiguously for inbound. Test it with
a 200-character text before the parser assumes whole messages.

## Finding 2 — an unsigned webhook is an evidence problem, not a security nit

Mobile Message documents no HMAC signature. Its security guidance is "check the payload
format matches what you expect", which is not authentication. Twilio signs every
request with `X-Twilio-Signature`.

For an ordinary notification app this is a shrug. For B.R.A.I.N. it goes to the heart of
the product: anyone who learns the webhook URL can POST a forged attendance record, and
the hash chain will faithfully, verifiably attest to it. The chain proves *nothing was
altered after arrival*. It says nothing about whether what arrived was real. A builder
relying on that record in a dispute would be relying on the wrong guarantee.

Whichever provider is chosen, before Phase 1:

- unguessable webhook path plus a shared secret compared in constant time
- provider IP allowlist if one is published; ask for it
- record on every stored message which authentication was satisfied — signature,
  shared secret, or none — so `/verify` can report the weakest link rather than implying
  a strength the transport never had
- do not let "hash-chained and tamper-evident" reach Jimmy or SIG without that
  distinction attached. It is the same class of overclaim as GPS-verified check-in.

If the evidentiary claim is the product, Twilio's signed webhook may be worth its
price. That is a decision to make deliberately, not by defaulting to the cheap option.

## What to do

1. Sign up to Mobile Message, buy the smallest credit pack, claim the free dedicated
   number (name, DOB, address). Cost: a few dollars. No ABN, no waiting.
2. Point the existing `/simulate` payload shape at a real adapter for the Mobile Message
   inbound format. The adapter already exists in the design; this is the afternoon it
   was designed for.
3. Test with one phone — yours — before anyone else's number touches it. Verify the
   duplicate behaviour and the multi-part behaviour deliberately.
4. In parallel, apply for the ABN at abr.gov.au. It unblocks everything downstream and
   is needed to invoice regardless.
5. Decide Finding 2 before Phase 1 goes to a real site, not after.

Phase 1 is ten workers on one site, attendance only. Every provider above clears that
bar. The bottleneck was a category error, not a wall.

## Sources

- ACMA, SMS Sender ID Register — https://www.acma.gov.au/sms-sender-id-register
- Twilio, Australia regulatory guidelines — https://www.twilio.com/en-us/guidelines/au/regulatory
- Twilio, Australia SMS guidelines — https://www.twilio.com/en-us/guidelines/au/sms
- Mobile Message, webhooks — https://help.mobilemessage.com.au/api/setting-up-webhooks
- Mobile Message, free dedicated number — https://help.mobilemessage.com.au/sending-receiving-sms/how-to-get-your-free-dedicated-number
- Mobile Message, API docs — https://mobilemessage.com.au/api-documentation
- Cellcast pricing — https://www.cellcast.com/au/lp/send-more-for-less
- ClickSend, ACMA schedule (ABN requirement) — https://www.clicksend.com/us/legal/acma-schedule/
- Telnyx, Australia DID requirements — https://support.telnyx.com/en/articles/3505912-australia-did-requirements
- Australian Business Register — https://www.abr.gov.au
