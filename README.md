# Tracking Meta ad conversions through a LINE LIFF webview (when the Pixel silently undercounts)

> **TL;DR** — You run Meta (Facebook/Instagram) ads that send people into a **LINE** Official Account or a **LIFF** app, you drop the Meta Pixel on the LIFF page like every guide says, and your conversions still report as near-zero. The Pixel isn't broken — it's running inside LINE's **in-app webview**, where the ad-click attribution (`fbclid` / `_fbp`) never arrives and third-party storage is restricted. The fix is two-part: **(1)** capture attribution at the last *real-browser* touchpoint before the LINE hand-off, and **(2)** fire the actual conversion **server-side via the Conversions API (CAPI)**, deduplicated by `event_id`. This writeup is the specific recipe — general "Pixel vs CAPI" articles don't cover the LIFF hand-off break.

---

## Why the client Pixel undercounts inside LIFF

Follow the actual click path of a Meta ad that targets a LINE audience:

```
Meta ad (Facebook/IG feed)
  → Meta in-app browser opens your landing/redirect URL     [fbclid in URL, _fbp cookie set HERE]
    → user taps "Add LINE" / "Open in LINE"
      → LINE app opens the OA chat or LIFF app              [DIFFERENT app, DIFFERENT cookie jar]
        → conversion happens here (onboarding / subscribe / form submit)
```

Three things break client-side attribution at the LINE boundary:

1. **Cookie jar reset.** The `_fbp` / `_fbc` cookies and the `fbclid` query param live in Meta's in-app browser. When the OS hands off to the **LINE app**, the LIFF page loads in LINE's *own* webview with a separate storage context. The pixel on the LIFF page has nothing to attribute the conversion to.
2. **In-app webview storage restrictions.** LIFF webviews (and in-app browsers generally) restrict third-party cookies / cross-site storage, so even a correctly-installed Pixel often can't persist or read `_fbp` reliably.
3. **The conversion is late and authenticated.** The real conversion (completed onboarding, started trial, submitted a lead form) typically happens *after* a LINE login step, server-side — exactly where a browser pixel has the least visibility.

Net effect: `PageView` might fire occasionally, but the high-value `Lead` / `CompleteRegistration` / `Purchase` event inside LIFF is unmatched, so Meta can't optimize delivery and your ROAS looks fake-bad.

---

## The fix, in two parts

### Part 1 — Capture attribution at the last real-browser touchpoint

Don't try to attribute *inside* LIFF. Put a thin public **redirect page** between the ad and LINE — e.g. `https://example.app/go/line` — that runs in the normal in-app browser (where `fbclid` *does* arrive), and there:

- Let the **Pixel** fire a lightweight intent event (e.g. a custom `LineClickIntent` or a standard `Lead` with a `content_name`), so you still get a click-side signal.
- Read `fbclid` from the URL and the `_fbp` cookie, and **carry them forward** into LINE — via LIFF state / a query param on the LIFF endpoint, or by stashing them against a short-lived session you can rejoin after LINE login.

```ts
// /go/line  (public page, runs in the in-app browser BEFORE the LINE hand-off)
const url = new URL(location.href);
const fbclid = url.searchParams.get("fbclid");          // present here, gone after LINE
const fbp = document.cookie.match(/_fbp=([^;]+)/)?.[1]; // Pixel-set first-party cookie

// fbc is derived from fbclid per Meta's spec: fb.1.<unix_ms>.<fbclid>
const fbc = fbclid ? `fb.1.${Date.now()}.${fbclid}` : undefined;

// carry attribution across the hand-off (LIFF state / signed param / server session)
const liff = new URL("https://liff.line.me/<liff-id>");
if (fbc) liff.searchParams.set("fbc", fbc);
if (fbp) liff.searchParams.set("fbp", fbp);
location.href = liff.toString();
```

> Treat the carried `fbc`/`fbp` as best-effort. If they survive the hand-off, match quality goes up; if they don't, CAPI still attributes via hashed identifiers (email/phone) you collect during onboarding.

### Part 2 — Fire the real conversion server-side via CAPI

When the conversion actually completes (your backend inserts the subscription / lead row), POST the event to Meta's Conversions API from the server. This is the event that matters and the one the client pixel kept missing.

```ts
// server-side, after the conversion row is committed
import crypto from "node:crypto";
const sha256 = (s: string) =>
  crypto.createHash("sha256").update(s.trim().toLowerCase()).digest("hex");

async function fireCapiLead(opts: {
  email?: string; phone?: string;
  fbc?: string; fbp?: string;
  eventId: string;            // <-- dedup key, see below
  clientIp?: string; userAgent?: string;
}) {
  const body = {
    data: [{
      event_name: "Lead",                       // or CompleteRegistration / Purchase
      event_time: Math.floor(Date.now() / 1000),
      action_source: "website",
      event_id: opts.eventId,                   // dedup against any client event
      user_data: {
        ...(opts.email && { em: [sha256(opts.email)] }),
        ...(opts.phone && { ph: [sha256(opts.phone)] }),
        ...(opts.fbc && { fbc: opts.fbc }),      // NOT hashed
        ...(opts.fbp && { fbp: opts.fbp }),      // NOT hashed
        ...(opts.clientIp && { client_ip_address: opts.clientIp }),
        ...(opts.userAgent && { client_user_agent: opts.userAgent }),
      },
    }],
  };

  const res = await fetch(
    `https://graph.facebook.com/v21.0/${process.env.META_DATASET_ID}/events` +
      `?access_token=${process.env.META_CAPI_ACCESS_TOKEN}`,
    { method: "POST", headers: { "content-type": "application/json" },
      body: JSON.stringify(body) }
  );
  // log res.ok / events_received / fbtrace_id to your audit trail
  return res.json();
}
```

Key points:

- **Hash PII** (`em`, `ph`) with SHA-256, lowercased + trimmed. **Do not hash** `fbc`, `fbp`, IP, or user-agent.
- The **dataset/pixel ID** and the **CAPI access token** are server-only secrets — never ship the token to the client.
- Log every send (`events_received`, `fbtrace_id`, failures) to your own audit table. CAPI failures are silent to the user; you want them visible to you.

### The `event_id` dedup rule

If you fire *both* a client Pixel event (from `/go/line`) and a server CAPI event, Meta will **double-count** unless they share an `event_id`. Generate the ID once, use it in both places, and Meta collapses them into one conversion. For a LIFF flow where the client event is just "click intent" and the server event is the real "Lead," they're usually *different* events and don't need to share an ID — but the moment you mirror the *same* event on both sides, the shared `event_id` is mandatory.

---

## Verify it end-to-end

- **Events Manager → Test Events** — grab the test code, pass it as `test_event_code` in the CAPI body, run a real ad-click-to-conversion, and watch the server event land with `action_source: website` and a populated `user_data`.
- **Event Match Quality** — check the score on your server event. Carrying `fbc`/`fbp` from `/go/line` plus hashed email/phone should push it into the "Good"/"Great" range.
- **Deduplication column** — if you mirror a client + server event, confirm Events Manager shows them deduplicated, not doubled.

---

## When you do NOT need this

- Your ad sends users to a **normal web page** that converts in a real browser (not LINE/LIFF, not an in-app webview) → the standard Pixel + optional CAPI is enough; the hand-off break doesn't apply.
- You don't run paid Meta acquisition into LINE at all → nothing to attribute.

This recipe is specifically for the **paid-Meta → LINE/LIFF** funnel, which is extremely common in Thailand / Japan / Taiwan and almost absent from the English "Pixel vs CAPI" literature.

---

## Why this writeup exists

Search "Meta Pixel vs Conversions API" and there are hundreds of solid articles. Search "Meta Pixel not tracking inside LINE LIFF" and there's almost nothing — yet the LINE-driven funnel is a default growth motion across Asia. The non-obvious part isn't "use CAPI" (everyone says that); it's *why* the client pixel fails specifically at the LINE hand-off, and that you need a **pre-LINE attribution bridge** to keep match quality up once you move the conversion server-side.

---

## Found this useful?

A ⭐ helps others running the LINE acquisition funnel find it. I write up the non-obvious gotchas from building small LINE / Next.js / marketing-stack projects — issues and corrections welcome. For help wiring a production LINE × Meta CAPI attribution pipeline (server events, dedup, audit logging), see [GitHub Sponsors](https://github.com/sponsors/MankhongGarden).

## License

MIT — see [LICENSE](LICENSE).
