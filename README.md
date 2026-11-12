<div align="center">

# Roilo API: AI Video Ads From Your Product Photos

**Send product photos. Get back a finished video ad.**

Roilo makes AI UGC video ads. It writes the script and generates the scenes with your product and presenter. It records the voiceover, burns in captions and makes the cuts.<br>
The video comes in 9:16, 1:1 or 16:9. Make videos from your own app with the API, or by chat in Claude through the Roilo MCP server.

[![API v1](https://img.shields.io/badge/API-v1-111111)](https://roilo.co/developers)
[![Works with Claude](https://img.shields.io/badge/works_with-Claude-111111)](https://roilo.co/channels/claude)

[Website](https://roilo.co) · [Developer reference](https://roilo.co/developers) · [Use Roilo in Claude](https://roilo.co/channels/claude) · [How it works](https://roilo.co/how-it-works) · [Pricing](https://roilo.co/pricing)

</div>

> **This repo is the quickstart. It holds no code.**
> The full, current API reference is at **[roilo.co/developers](https://roilo.co/developers)**. If this README and that page ever differ, the page is right.

## Contents

- [What Roilo makes: AI UGC video ads](#what-roilo-makes-ai-ugc-video-ads)
- [How a video gets made](#how-a-video-gets-made)
- [Quickstart: REST API](#quickstart-rest-api)
- [Endpoints](#endpoints)
- [Video statuses and fields](#video-statuses-and-fields)
- [Webhooks](#webhooks)
- [Use Roilo as a video creator in Claude (MCP)](#use-roilo-as-a-video-creator-in-claude-mcp)
- [Errors, retries and limits](#errors-retries-and-limits)
- [Use cases](#use-cases)
- [FAQ](#faq)

## What Roilo makes: AI UGC video ads

Roilo turns a product or a service into a video ad that is ready to post. You upload photos of your product and a presenter. The presenter can be you, or someone from Roilo's library. Roilo does the rest.

- **It works out what to say first.** Before anything renders, Roilo reads your product. It finds the target buyer, the problem they want solved, what makes the product different, the dream outcome and the time to result.
- **Your product. Your presenter. Your place.** Scenes stay true to the photos you sent. For service businesses, that includes your real venue. Every scene is checked against your photos. A scene that doesn't match is rejected and made again.
- **It arrives finished.** The voiceover is recorded, captions are burned in and the cuts are made. Nothing is left to edit.
- **The shape your platform wants.** You get 9:16 vertical (the default), 1:1 square or 16:9 widescreen. Quality is 480p, 720p or 1080p.
- **You pay only for delivered videos.** Credits are held when a video starts. They are charged when the video is delivered and refunded if it fails.

The API makes the same videos as the website. You don't book a creator, rerun scenes one by one or fight with prompts.

## How a video gets made

```text
POST /v1/videos
   │
   ▼
queued ──(review: true)──► awaiting_review ──► approve, or it continues at review_expires_at
   │                                              │
   ▼                                              │
processing ◄──────────────────────────────────────┘
   │
   ├──► completed   output.video_url is ready to download
   └──► failed      error.code says why; held credits are refunded
```

Generation runs in the background. The create call returns `202` straight away. Then you wait for a webhook, or you poll the video.

## Quickstart: REST API

Base URL: `https://roilo.co/api/v1`. Requests and responses are JSON over HTTPS. Uploads are multipart. Timestamps are ISO 8601 in UTC.

### 1. Create an API key

[Sign up](https://roilo.co/register) for free. You need a verified email. Then open the account menu, choose **API keys** ([/settings/api-keys](https://roilo.co/settings/api-keys)) and click **Create key**. Keys start with `rk_live_` and are shown only once. You can give each key a monthly credit limit.

```bash
export ROILO_API_KEY="rk_live_..."
```

### 2. Check the key

```bash
curl https://roilo.co/api/v1/me \
  -H "Authorization: Bearer $ROILO_API_KEY"
```

This returns your available credits and how much this key has spent this month. A missing, malformed or revoked key returns `401 invalid_api_key`.

### 3. Pick a mode and a presenter

```bash
# Video modes: credit cost, photo limits, aspect ratios, qualities
curl -s https://roilo.co/api/v1/modes \
  -H "Authorization: Bearer $ROILO_API_KEY" | jq '.data[] | {id, name, credit_cost}'

# Your own presenters plus stock presenters
curl -s https://roilo.co/api/v1/actors \
  -H "Authorization: Bearer $ROILO_API_KEY" | jq '.data[] | {id, name, is_stock}'
```

`/v1/voices` and `/v1/caption-styles` work the same way.

### 4. Upload photos (optional)

You can skip this step and send public `https` links to your photos instead.

```bash
curl https://roilo.co/api/v1/files \
  -H "Authorization: Bearer $ROILO_API_KEY" \
  -F "file=@tumbler-front.jpg"
# → { "id": "file_...", "expires_at": "..." }
```

Uploads can be JPEG, PNG or WebP. You can reuse a file ID across videos until it expires.

### 5. Create the video

```bash
IDEMPOTENCY_KEY=$(uuidgen)   # reuse this same value if you retry

curl https://roilo.co/api/v1/videos \
  -H "Authorization: Bearer $ROILO_API_KEY" \
  -H "Idempotency-Key: $IDEMPOTENCY_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "mode": "MODE_ID",
    "actor_id": "ACTOR_ID",
    "product_images": [
      "file_...",
      "https://cdn.example.com/tumbler-side.jpg"
    ],
    "aspect_ratio": "9:16",
    "product": { "name": "Insulated tumbler", "category": "home" },
    "webhook_url": "https://example.com/webhooks/roilo",
    "metadata": { "sku": "TMB-001" }
  }'
```

Replace `MODE_ID` and `ACTOR_ID` with IDs from step 3. A few notes:

- **Choose a presenter.** Send `actor_id`, or send your own presenter photos in `actor_images`. Without `actor_id`, a product video needs at least the number of presenter photos its mode sets.
- **Photo counts vary by mode.** Each mode sets its own minimum and maximum number of photos.
- **You can say more about the product.** Add `target_audience`, `pain_point`, `dream_outcome`, `competitive_advantage`, `social_proof` and `result` to the `product` object.
- **Other settings.** `voice_id`, `caption_style_id` and `quality` are also available.
- **`metadata` comes back unchanged** on the video object.
- **`"review": true`** pauses the video so you can check the brief first. See [Review before generation](#review-before-generation).

The call returns `202` with the video object, in status `queued`.

### 6. Get the finished video

The recommended way is to listen for the `video.completed` [webhook](#webhooks). You can also poll:

```js
// Node 18+
const BASE = "https://roilo.co/api/v1";
const headers = { Authorization: `Bearer ${process.env.ROILO_API_KEY}` };

export async function waitForVideo(id, everyMs = 15_000) {
  for (;;) {
    const res = await fetch(`${BASE}/videos/${id}`, { headers });
    if (!res.ok) throw new Error(`Roilo responded ${res.status}`);
    const video = await res.json();
    if (video.status === "completed") return video.output; // video_url, thumbnail_url, ...
    if (video.status === "failed") throw new Error(video.error.code);
    await new Promise((r) => setTimeout(r, everyMs));
  }
}
```

Download links are signed and expire after one hour (see `output.url_expires_at`). To get fresh links, fetch the video again.

## Endpoints

| Method | Path | What it does |
|---|---|---|
| `GET` | `/v1/me` | Check the key; read credits and this key's spend |
| `GET` | `/v1/modes` | Video modes, credit cost, photo limits, aspect ratios, qualities |
| `GET` | `/v1/actors` | Your presenters and stock presenters |
| `GET` | `/v1/voices` | Your voices and stock voices |
| `GET` | `/v1/caption-styles` | Caption styles |
| `POST` | `/v1/files` | Upload one image (`201`) |
| `POST` | `/v1/videos` | Start a video (`202`) |
| `GET` | `/v1/videos` | List videos, newest first (`limit`, `cursor`) |
| `GET` | `/v1/videos/:id` | Read one video |
| `POST` | `/v1/videos/:id/approve` | Release a video held for review |
| `DELETE` | `/v1/videos/:id` | Delete a video (`204`) |

For every field, limit and example, see [roilo.co/developers](https://roilo.co/developers).

## Video statuses and fields

| `status` | Meaning |
|---|---|
| `queued` | Accepted. Inputs are being prepared. |
| `awaiting_review` | Paused for you (`review: true`). Approve it, or it continues on its own when `review_expires_at` passes. |
| `processing` | The video is being generated. |
| `completed` | `output` is set. |
| `failed` | `error` says why. Held credits are refunded. |

Write your code against `status` and `error.code`.

The fields you will use most:

| Field | What it holds |
|---|---|
| `id` | The video ID, starting with `vid_` |
| `output` | `null` until the video is completed. Then it holds `video_url`, `thumbnail_url`, `duration_sec`, `aspect_ratio`, `quality` and `url_expires_at` |
| `error` | `code`, `message` and, sometimes, `param` (the field at fault) |
| `brief` | The drafted brief, while the video is awaiting review |
| `credits_held`, `credits_charged` | Credits held for the video, and credits charged on delivery |
| `metadata` | Your own object, returned unchanged |

### Review before generation

If you send `"review": true`, Roilo first analyzes your photos, then stops at `awaiting_review`. Fetch the video to read the drafted `brief`. Then approve it:

```bash
curl -X POST https://roilo.co/api/v1/videos/vid_.../approve \
  -H "Authorization: Bearer $ROILO_API_KEY"
```

The request body is optional. To change the brief, send a `brief` that holds only the fields you want to change. If the video isn't waiting for review, approving it returns `409 not_awaiting_review`.

## Webhooks

Set `webhook_url` on a video. Roilo then POSTs JSON to that URL for three events:

| Event | When it fires |
|---|---|
| `video.awaiting_review` | The brief is ready for you to check |
| `video.completed` | The video is ready |
| `video.failed` | The video failed, and its credits were refunded |

The body holds the event `id`, `type` and `created_at`, plus the full video object under `data`.

- **Reply fast.** Return any `2xx` within 10 seconds. Redirects are not followed.
- **Failed deliveries are retried.** The gaps grow longer between tries, up to six attempts in total.
- **Expect duplicates.** An event can arrive more than once. De-duplicate on the `Roilo-Event-Id` header, or on the body `id`.
- **Links can go stale.** The links in a webhook may expire before a retry arrives. Fetch the video for fresh ones.

### Verify the signature

Each webhook carries this header: `Roilo-Signature: t=<unix seconds>,v1=<hex HMAC-SHA256>`

- **What is signed:** the HMAC covers `${t}.${rawBody}`.
- **The secret:** your key's webhook secret. It starts with `whsec_`, is shown once, and can be rotated on the API keys page.
- **Freshness:** reject any `t` more than 300 seconds from now.
- **Comparison:** compare signatures in constant time.
- **Raw body:** always verify against the raw request body, before you parse the JSON.

<details>
<summary>Sample code: Node.js and Python</summary>

**Node.js (Express)**

```js
import express from "express";
import { createHmac, timingSafeEqual } from "node:crypto";

export function verifyRoiloSignature(secret, rawBody, header = "", toleranceSec = 300) {
  const { t = "", v1 = "" } = Object.fromEntries(
    header.split(",").map((part) => part.trim().split("="))
  );
  if (!/^\d+$/.test(t) || Math.abs(Date.now() / 1000 - Number(t)) > toleranceSec) return false;
  const expected = createHmac("sha256", secret).update(`${t}.`).update(rawBody).digest();
  const given = Buffer.from(v1, "hex");
  return given.length === expected.length && timingSafeEqual(given, expected);
}

const app = express();
const seen = new Set(); // use a shared store in production

app.post("/webhooks/roilo", express.raw({ type: "*/*" }), (req, res) => {
  const ok = verifyRoiloSignature(
    process.env.ROILO_WEBHOOK_SECRET,
    req.body,
    req.get("Roilo-Signature")
  );
  if (!ok) return res.sendStatus(400);

  const eventId = req.get("Roilo-Event-Id");
  if (seen.has(eventId)) return res.sendStatus(200); // duplicate delivery
  seen.add(eventId);

  const event = JSON.parse(req.body.toString("utf8"));
  // event.type: "video.awaiting_review" | "video.completed" | "video.failed"
  // event.data: the full video object
  res.sendStatus(200); // reply within 10 seconds; do slow work after replying
});

app.listen(3000);
```

**Python**

```python
import hashlib
import hmac
import time


def verify_roilo_signature(secret: str, raw_body: bytes, header: str, tolerance_sec: int = 300) -> bool:
    """Check a Roilo-Signature header against the raw request body."""
    parts = dict(p.strip().split("=", 1) for p in header.split(",") if "=" in p)
    t, v1 = parts.get("t", ""), parts.get("v1", "").lower()
    if not t.isdecimal() or abs(time.time() - int(t)) > tolerance_sec:
        return False
    expected = hmac.new(secret.encode(), t.encode() + b"." + raw_body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected.encode(), v1.encode())


# Flask:
# verify_roilo_signature(os.environ["ROILO_WEBHOOK_SECRET"],
#                        request.get_data(), request.headers.get("Roilo-Signature", ""))
```

</details>

## Use Roilo as a video creator in Claude (MCP)

Ask Claude to make your product video, and Roilo makes it. Roilo's hosted MCP server connects to Claude as a custom connector, using a Roilo API key.

| | |
|---|---|
| **Connector URL** | `https://roilo.co/api/mcp` |
| **Transport** | Streamable HTTP, stateless |
| **Auth** | Your Roilo API key as a bearer token: `Authorization: Bearer rk_live_...` |

Each tool takes the same fields and returns the same objects as its REST route. Credits are charged the same way as over REST. When a call fails, the tool error carries the same error body as REST.

### Add Roilo to Claude (web and Desktop)

This works on Claude's Free, Pro, Max, Team and Enterprise plans. Free plans can add one custom connector, and that is enough for Roilo.

1. In Claude, go to **Customize → Connectors**, then **+ Add → Add custom connector**.
2. Name it `Roilo` and enter the server URL `https://roilo.co/api/mcp`. Click **Continue**.
3. Under **Authentication**, choose **No sign in**.
4. Under **Request headers**, add `Authorization`. Its value is `Bearer`, a space, then your key: `Bearer rk_live_...`.
5. Click **Add**. In any chat, click **+ → Connectors**, turn Roilo on and ask for a video.

On Team or Enterprise, only an Owner can add the connector, under **Organization settings → Connectors**. Everyone then shares that one key. The full guide is at [roilo.co/channels/claude](https://roilo.co/channels/claude).

### Other AI assistants

Any assistant that can add a remote connector with custom headers can use the same entry:

```json
{
  "mcpServers": {
    "roilo": {
      "type": "http",
      "url": "https://roilo.co/api/mcp",
      "headers": { "Authorization": "Bearer rk_live_..." }
    }
  }
}
```

### Tools

| Tool | What it does | REST route |
|---|---|---|
| `create_video` | Start a video | `POST /v1/videos` |
| `get_video` | Read a video's status and download link | `GET /v1/videos/:id` |
| `list_videos` | List recent videos | `GET /v1/videos` |
| `approve_video` | Release a video held for review | `POST /v1/videos/:id/approve` |
| `delete_video` | Delete a video | `DELETE /v1/videos/:id` |
| `upload_image` | Upload one image as base64, up to about 16 MB. Send larger files to `create_video` as `https` links | `POST /v1/files` |
| `list_modes` | List video styles and what each costs in credits | `GET /v1/modes` |
| `list_actors` | List your presenters and stock presenters | `GET /v1/actors` |
| `list_voices` | List your voices and stock voices | `GET /v1/voices` |
| `get_credits` | Show credits left and this key's spend this month | `GET /v1/me` |

### Things to ask

> Make a 9:16 vertical video for this tumbler, using the photos on this product page: https://…

> Which video styles do you offer, and what does each cost in credits?

> Remake that video in 16:9 with captions for a Facebook ad.

> Is my video ready? Send me the download link.

> How many credits do I have left, and how much has this key spent this month?

Give Claude public image links, like the ones on your shop or product page. A video takes a few minutes. Claude can check back and tell you when yours is ready.

## Errors, retries and limits

Every error has the same shape:

```json
{ "error": { "code": "...", "message": "...", "param": "..." } }
```

- **Branch on the HTTP status and `code`.** The wording of `message` can change.
- **Retry `429` and `5xx` responses after a pause.** On a create, reuse the same `Idempotency-Key` to make the retry safe. For 24 hours, the same key returns the original response. For any other `4xx`, fix the request instead.
- **Credit errors.** `402 insufficient_credits` means the account is out of credits. `402 key_credit_limit_reached` means this key hit its monthly limit.
- **Rate limits are per key, per minute.** Calls from Claude have a limit of their own. Every response carries `X-RateLimit-Limit`, `X-RateLimit-Remaining` and `X-RateLimit-Reset`. A `429 rate_limited` response also carries `Retry-After`.
- **Too many videos at once.** `429 too_many_active_videos` means the account already has too many videos in progress.

For the full code list and current limits, see [roilo.co/developers](https://roilo.co/developers).

## Use cases

| Who | What you send | Read more |
|---|---|---|
| Shopify and ecommerce stores | Product photos and a presenter | [roilo.co/for/ecommerce](https://roilo.co/for/ecommerce) |
| Shopee and Lazada sellers | Product photos and a presenter | [roilo.co/for/marketplaces](https://roilo.co/for/marketplaces) |
| Property listing videos | Photos of the real space and a presenter, as a service video | [roilo.co/for/property](https://roilo.co/for/property) |
| Venues and local services | Venue photos and a presenter, plus an optional product, as a service video | [How it works](https://roilo.co/how-it-works) |

To make a service video, send `"business_type": "service"` with `venue_images` and a presenter (`actor_id` or an actor image). You can also add a `business_name`.

<details>
<summary>Categories the API accepts</summary>

**Product:** `general`, `beauty`, `fashion`, `tech`, `food`, `fitness`, `home`, `pet`, `kids`, `health`

**Service:** `beauty_wellness`, `food_beverage`, `property`, `interior_renovation`, `fitness`, `clinic_health`, `education`, `automotive`, `events_media`, `general_service`

</details>

## FAQ

**Is there an API for making product videos with AI?**
Yes. Roilo's REST API takes product photos and a mode. It returns a finished video with a script, a presenter, a voiceover and captions. Start with the [Quickstart](#quickstart-rest-api).

**Is there an MCP server for making video ads?**
Yes. Roilo's hosted MCP server makes finished product video ads. Its address is `https://roilo.co/api/mcp`, and it takes your Roilo API key. See [Use Roilo as a video creator in Claude](#use-roilo-as-a-video-creator-in-claude-mcp).

**Can I use Claude as a video creator?**
Yes, with Roilo. [Add Roilo to Claude](#add-roilo-to-claude-web-and-desktop), then ask in chat. You can set the style, the shape (9:16, 1:1 or 16:9), the presenter, the voice and the captions. Or you can let Claude ask you.

**Do I need a paid Claude plan?**
No. Free plans can add one custom connector on Claude web and Desktop, and that is enough for Roilo.

**Does connecting Roilo to Claude cost extra?**
No. There is no separate charge for the connection. Videos use Roilo credits at website prices.

**What happens if a video fails?**
You pay nothing for it. Its held credits are refunded.

**Do I need my own presenter?**
No. You can use your own photos, or pick a presenter from Roilo's library (`GET /v1/actors`).

**Can I check the brief before the video is made?**
Yes. Send `"review": true` and read the drafted brief. Edit it if you want, then approve it. The script is written from the brief you approve.

**Is there an SDK?**
This repo doesn't include one. The API is plain HTTPS and JSON, so any language with an HTTP client works. In Claude, you need no code at all.

**How do I keep spending under control?**
Give each key a monthly credit limit. You can also revoke a key at any time on the API keys page.

## Links

- Developer reference: [roilo.co/developers](https://roilo.co/developers)
- Use Roilo in Claude: [roilo.co/channels/claude](https://roilo.co/channels/claude)
- How it works: [roilo.co/how-it-works](https://roilo.co/how-it-works)
- Pricing: [roilo.co/pricing](https://roilo.co/pricing)
- Guides: [roilo.co/b](https://roilo.co/b)
- Contact: [roilo.co/contact](https://roilo.co/contact)

Using the API or the MCP server means you accept the [API and MCP access terms](https://roilo.co/terms#api-terms). See also the [Terms](https://roilo.co/terms) and the [Privacy Policy](https://roilo.co/privacy).

<div align="center">

**[Roilo](https://roilo.co)**: your photos in, one finished ad out.

</div>
