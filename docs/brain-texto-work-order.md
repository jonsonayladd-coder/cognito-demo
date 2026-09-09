# Work order — wire B.R.A.I.N. to Texto

Provider decision: **Texto** (texto.com.au). Signed inbound webhook (HMAC-SHA256),
stable inbound `message_id`, AU-only routing, 3¢/SMS, $15/mo dedicated number.

Belongs in `brain-worker`. Parked here because this session has no access to it.

## Step 0 — RESOLVED, 9 Sep 2026

Dedicated number purchased on Texto. **No ABN was required.** The blocker that
stalled this project for three days on Twilio does not exist here, and the
Mobile Message fallback is no longer needed.

Remaining from this step: Developer → Inbound Webhook. Leave the URL blank until
the Worker is deployed, but **copy the signing secret** and confirm HMAC signing
is enabled — the signature header is only sent when it is switched on.

## Step 1 — deploy first, for once

`deploy.bat`. The build is written, tested and committed but has never been
deployed, and Texto cannot POST to a Worker that isn't live. This is the one time
deployment comes before the code change rather than after.

Confirm `https://brain.allgoodnow.workers.dev/verify` answers with a real token
before going further.

Then set the secrets:

```
wrangler secret put TEXTO_INBOUND_WEBHOOK_SECRET
wrangler secret put TEXTO_API_KEY
```

## Step 2 — the inbound adapter

New route `POST /hooks/texto`. Rules from `architecture.md` still hold: write raw,
return 200, classify in the consumer, never in the request.

```js
// Verify over the RAW BYTES. Parsing and re-serialising breaks the signature.
async function verifyTexto(raw, header, secret) {
  const provided = (header || "").replace(/^sha256=/, "");
  if (!/^[0-9a-f]{64}$/i.test(provided)) return false;

  const key = await crypto.subtle.importKey(
    "raw", new TextEncoder().encode(secret),
    { name: "HMAC", hash: "SHA-256" }, false, ["verify"]
  );
  const sig = Uint8Array.from(provided.match(/../g).map(h => parseInt(h, 16)));
  return crypto.subtle.verify("HMAC", key, sig, raw);   // constant-time
}

export async function handleTextoInbound(request, env) {
  const raw = await request.arrayBuffer();              // NOT request.json()

  const ok = await verifyTexto(
    raw,
    request.headers.get("x-texto-signature"),
    env.TEXTO_INBOUND_WEBHOOK_SECRET
  );
  if (!ok) return new Response("invalid signature", { status: 401 });

  const e = JSON.parse(new TextDecoder().decode(raw));
  if (e.event !== "message.inbound") return new Response(null, { status: 204 });

  // Raw insert. The chain trigger fires here. Dedupe on the vendor's own id.
  await env.DB.prepare(`
    INSERT INTO messages_raw
      (gateway_msg_id, from_phone, body, num_media, received_at, payload_json)
    VALUES (?1, ?2, ?3, 0, ?4, ?5)
    ON CONFLICT(gateway_msg_id) DO NOTHING
  `).bind(
    e.message_id,
    e.from,                       // already E.164 from Texto
    e.body,
    e.received_at,
    JSON.stringify(e)
  ).run();

  await env.CLASSIFY_QUEUE.send({ gateway_msg_id: e.message_id });

  return new Response(null, { status: 200 });
}
```

Notes on that code, each one deliberate:

- **`arrayBuffer()`, not `json()`.** Re-serialising changes the bytes and every
  signature fails. This is the single most common way this integration breaks.
- **Signature check before anything touches the database.** A record that failed
  verification must never enter the chain — the chain cannot un-attest it later.
- **`ON CONFLICT DO NOTHING` on `gateway_msg_id`.** Texto retries up to 3 times
  and reuses `message_id`, so retries are idempotent by construction. Needs a
  unique index on that column if there isn't one already.
- **Await the queue send, don't `waitUntil` it.** If the queue push fails, you
  want to return non-2xx and let Texto retry the whole thing. The dedupe above
  makes that safe. `waitUntil` would swallow the failure and lose the
  classification silently.
- **15-second budget.** Classification stays in the consumer. This handler does
  two writes and returns.

## Step 3 — opt-out is not a no-op

`is_optout: true` means Texto has *already* recorded the opt-out at their end.
That worker's messages stop arriving and nothing in your system explains why.

Handle it explicitly:

- write an `events` row of type `optout`
- set the worker's `status` so the board shows them as unreachable, not absent
- surface it on the supervisor board — an unreachable worker is not the same as
  a worker who didn't show up, and an attendance log that conflates them is wrong
  in exactly the way that matters

A worker texting STOP as a joke is a realistic Monday. It should not silently
delete them from the record.

## Step 4 — outbound

```js
async function sendSms(env, to, body) {
  const r = await fetch("https://api.texto.com.au/send", {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${env.TEXTO_API_KEY}`,
      "Content-Type": "application/json"
    },
    body: JSON.stringify({ to, message: body, sender: env.TEXTO_NUMBER })
  });
  if (!r.ok) throw new Error(`texto ${r.status}: ${await r.text()}`);
  return r.json();
}
```

Rate limits: 5 req/s and 5 concurrent per key. Fine for confirmations. For a
`SITREP` fan-out to forty workers use `/send-batch` (max 1,000) rather than forty
calls, or you will hit the limiter.

## Step 5 — tests, added to the existing 34

Four new behaviour checks, all of which must actually run:

1. A POST with a wrong signature is rejected 401 **and writes nothing**.
2. The same `message_id` delivered twice produces exactly one `messages_raw` row
   and one classified event.
3. `is_optout: true` produces an optout event and flips worker status.
4. A body whose bytes are valid but whose JSON is re-serialised differently still
   verifies — i.e. prove you're hashing raw bytes, not a parsed round-trip.

Label them VERIFIED only after they run.

## Step 6 — live test, one phone

Point the Texto webhook at the deployed URL. Text `IN` from your own handset.
Then check, in order:

- `messages_raw` has one row with Texto's `message_id`
- `/verify` still recomputes the chain clean
- the event classified correctly through the queue
- text a 200-character message and confirm it arrives as one webhook, not several

Only after all four does a second phone get added.

## What this does not cover

`DOCKET` / `RECEIPT` / `HOLDPOINT` / `PHOTO` are MMS. Texto's API has no MMS
endpoint. Phase 1 is attendance and is fully covered; photo capture needs a second
provider and is a Phase 2 decision. Don't let it into this work order.

Texto deletes message data after 90 days. D1 is the record of truth, which was
already the design — but it means Texto can never corroborate the chain later.
Your copy is the only copy. That is worth knowing before the word "evidence" is
used with anyone.

---

# Phase 2 — photos without MMS

Texto has no MMS endpoint, and MMS is the wrong tool anyway. Workers get a
one-time upload link by SMS and the file goes straight to R2.

Cost: one outbound SMS (3¢) instead of 29–34¢ inbound MMS. No second provider,
no second account, no second adapter.

## Why not MMS

- **Carriers recompress MMS.** The image you receive is the carrier's re-encode,
  usually with EXIF stripped. The SHA-256 in `media` would be a hash of the
  carrier's version, not of what the handset produced. A direct upload gives you
  the original bytes.
- **Inbound MMS to virtual numbers is patchy across Australian carriers.** A
  browser upload works on every handset.
- **MMS needs a data connection too**, so nothing is given up on reception.
- The premise survives: no install, no login, no account. One page, reached from
  a text they already have.

## Flow

1. Worker texts `DOCKET` (or `PHOTO`, `HOLDPOINT`).
2. Inbound webhook lands as normal — signature verified, raw row written, chained.
3. Consumer classifies it, matches the sender against `workers`, and mints a token.
4. Outbound SMS: `Upload here: brain.allgoodnow.workers.dev/u/x7k2m9 — expires in 15 min`
5. Worker taps it. One `<input type="file" accept="image/*" capture="environment">`
   and a button. Nothing else on the page.
6. Worker streams the body to R2, hashes the bytes, writes `media` against the
   `messages_raw` id of the text that asked — so the photo is chained to the
   request, not floating loose.

## Schema addition

```
upload_tokens   token TEXT PRIMARY KEY,        -- 128-bit random, base32
                message_id INTEGER NOT NULL,   -- the DOCKET text that triggered it
                worker_id INTEGER NOT NULL,
                site_id INTEGER NOT NULL,
                purpose TEXT NOT NULL,          -- docket | receipt | holdpoint | photo
                created_at TEXT NOT NULL,
                expires_at TEXT NOT NULL,
                used_at TEXT                    -- non-null once spent
```

Rules, all enforced server-side rather than by the page:

- Single use. Check `used_at IS NULL` and stamp it in the same statement that
  accepts the upload, or two taps race and you get two files on one token.
- Short expiry. Fifteen minutes. An unexpired token in a forwarded text is an
  open door onto the site record.
- The token carries the identity. Never accept a worker id, site id or purpose
  from the page — they come from the token row only.
- Cap the body size and check the content type on the server. `accept=` on the
  input is a hint to the file picker, not a constraint.
- No listing, no browsing, no index. `/u/:token` serves exactly one form and
  accepts exactly one file.

`media` already has what's needed: `r2_key`, `sha256`, `mime`, `exif_json`,
`received_at`. Add the `upload_token` for provenance.

## What this does not prove

`capture="environment"` hints at the camera. It does not force it, and any
handset can pick a photo from last week's gallery. MMS has exactly the same
hole — nothing about a photo arriving over the carrier network proves when it
was taken.

So this is a docket **log**, not verification. Record EXIF when it is present,
never rely on it, and never let anyone describe it as proof the photo was taken
at that time on that site. Same discipline as the GPS rule: the system records
what it received and when it received it, and claims nothing more.

---

# CORRECTIONS — verified against the live Cloudflare account, 9 Sep 2026

Everything above was written against v1 (`brain-worker`, D1 `brain`). **That is
the wrong target.** The live project is **brain-v2**:

- Repo: `C:\Users\me\Downloads\sitewire-mvp`
- Worker: `sitewire-mvp` — **already deployed**, last modified 7 Sep 2026.
  The deploy-first step above does not apply.
- D1: `brain-v2-customer` (`b5471ff4-1b4d-482a-a91e-3ee7924bfaf9`) and
  `brain-v2-demo` (`1a29e6ec-5986-4441-81e1-1ac90bc0f380`).
- The old D1 `brain` (`fd8c6969…`) is v1 and is not the target.

Confirm the workers.dev subdomain before pasting a webhook URL. The v1 notes say
`allgoodnow`, which would make it `sitewire-mvp.allgoodnow.workers.dev`, but that
is inference, not verified.

## Real `messages_raw` (read from the live database, not from architecture.md)

```sql
CREATE TABLE messages_raw (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  site_id      TEXT NOT NULL,
  gateway_id   TEXT,
  from_phone   TEXT,
  to_phone     TEXT,
  body         TEXT,
  num_media    INTEGER NOT NULL DEFAULT 0,
  received_at  TEXT NOT NULL,
  prev_hash    TEXT,
  row_hash     TEXT,
  seal_status  TEXT,
  payload_json TEXT
);
CREATE UNIQUE INDEX idx_raw_dedupe ON messages_raw(site_id, gateway_id);
```

Three corrections to the adapter above:

1. The column is **`gateway_id`**, not `gateway_msg_id`.
2. Dedupe is **composite**: `ON CONFLICT(site_id, gateway_id) DO NOTHING`.
   A bare conflict target on `gateway_id` will not compile against this index.
3. **`site_id` is NOT NULL and must be resolved before the insert.** Texto's
   `to` field — the number that received the message — is the routing key.
   One number means one site. A second site needs either a second Texto number
   or a phone→site lookup, and that decision is now load-bearing rather than
   cosmetic. `to_phone` stores the raw value regardless.

The chain lives in the row (`prev_hash`, `row_hash`, `seal_status`) and is
enforced by `messages_raw_seal_once`. The adapter must follow exactly whatever
the existing `/simulate` path does to compute and seal — do not hand-roll a
second insert path. Read that handler before writing this one.

## `media` — the Phase 2 section above is wrong for v2

```sql
CREATE TABLE media (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  site_id TEXT NOT NULL, raw_id INTEGER NOT NULL, idx INTEGER NOT NULL DEFAULT 0,
  r2_key TEXT NOT NULL, content_type TEXT, bytes INTEGER,
  sha256 TEXT NOT NULL, created_at TEXT NOT NULL
);
```

It is `raw_id`, `content_type` and `bytes` — and **there is no `exif_json`
column**. v2 does not store EXIF at all. Either add the column deliberately or
drop the EXIF language from the Phase 2 plan; do not leave the doc claiming
something the schema cannot hold.

## Trigger drift between the two databases — worth a look

`brain-v2-customer` has four triggers on `messages_raw`:
`seal_once`, `no_delete`, `identity_immutable`, **`body_immutable`**.

`brain-v2-demo` has only three. **It is missing `body_immutable`** — message
bodies can be edited in the demo database but not in the customer one.

If that is deliberate (so demos can be reset), fine — write it down. If it is
drift, the demo is quietly weaker than the thing it demonstrates, which is the
worst way for that difference to exist.

## Current contents of `brain-v2-customer`

9 raw messages, 3 events, 2 workers, 1 site, 0 media. Test seed, not real data —
safe to work against.
