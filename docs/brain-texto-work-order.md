# Work order — wire B.R.A.I.N. to Texto

Provider decision: **Texto** (texto.com.au). Signed inbound webhook (HMAC-SHA256),
stable inbound `message_id`, AU-only routing, 3¢/SMS, $15/mo dedicated number.

Belongs in `brain-worker`. Parked here because this session has no access to it.

## Step 0 — the one thing that can still kill it

In the Texto dashboard, go to buy the $15/month dedicated number. If that screen
demands an ABN, stop and fall back to Mobile Message. If it doesn't, buy it and
continue. Everything below assumes it went through.

Then: Developer → Inbound Webhook. Leave the URL blank for now, but **copy the
signing secret**.

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
