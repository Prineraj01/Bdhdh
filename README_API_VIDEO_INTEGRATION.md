# API Integration & Video Playback — Developer README

> **Purpose:** This document describes the API client, payload encoding/decoding, stream URL construction, and browser video-player behavior found in the supplied website source. Use it as the implementation specification when integrating the same API into our project.
>
> **Important scope note:** The inspected source is a frontend bundle. It shows how the browser calls the API and plays returned streams, but it does **not** include the server implementation behind `/api/d`, `/s/m/`, `/s/p/`, `/s/o/`, or `/s/f/`. Do not invent backend behavior, upstream providers, authorization rules, or DRM/license details.

---

## 1. Integration requirements

1. Reuse the API contract and stream URL patterns documented below rather than replacing them with unrelated endpoints.
2. Keep the API base same-origin unless our deployment explicitly provides a different compatible origin. The source uses relative paths.
3. Centralize the request wrapper and encoding/decoding functions in one API module.
4. Keep API payload encoding separate from video media playback. `Ze`/`Qe` are the source's transport obfuscation functions; they are not a general media decryptor.
5. Use the stream gateway URLs returned/built by the documented flow. Do not attempt to extract protected source URLs, DRM keys, or bypass access controls.
6. Preserve loading, retry, buffering, error, resume, and fallback behavior.
7. Verify each operation against the live API and actual response shape. The source only establishes the request fields listed here; response object schemas are not fully defined.

## 2. API transport

### Endpoint

`POST /api/d`

### Request headers

```http
Content-Type: application/json
```

### Request body

```json
{
  "p": "<Ze(JSON.stringify(payload))>",
  "t": "<Date.now().toString(36)>"
}
```

- `p`: JSON payload serialized to a string, then encoded with `Ze`.
- `t`: current Unix-millisecond timestamp converted to base 36. The inspected source sends it as a string.
- The source does not show an `Authorization` header in this wrapper.
- The source uses `fetch('/api/d', ...)`, so the request is relative to the current site origin.

### Response handling

The source expects JSON and reads:

- `response.d`, or
- `response.data`

The selected value is passed through `Qe`, then parsed using `JSON.parse`.

If HTTP status is not OK, the client tries to decode `d`/`data` and read its `.error`; otherwise it throws `Error <status>`. Missing response data throws `Invalid response structure`.

### Reference implementation

```js
export async function apiRequest(payload) {
  const encodedPayload = Ze(JSON.stringify(payload));

  const response = await fetch("/api/d", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      p: encodedPayload,
      t: Date.now().toString(36),
    }),
  });

  if (!response.ok) {
    let message = `Error ${response.status}`;

    try {
      const body = await response.json();
      const encoded = body?.d || body?.data;
      if (encoded) {
        const decoded = JSON.parse(Qe(encoded));
        message = decoded?.error || message;
      }
    } catch {
      // Preserve the HTTP status error if the error body is not decodable.
    }

    throw new Error(message);
  }

  const body = await response.json();
  const encoded = body?.d || body?.data;

  if (!body || !encoded) {
    throw new Error("Invalid response structure");
  }

  return JSON.parse(Qe(encoded));
}
```

`Ze` and `Qe` are specified in section 3. Keep them in the same shared module or import them from a single tested utility file.

## 3. Payload encoding and response decoding

### Constants found in source

```js
const qe = [95, 58, 140, 30, 123, 77, 146, 110];
const Je = 167;
```

### URL-safe Base64 helpers

The source uses standard Base64 conversion, then converts it to Base64URL:

- `+` becomes `-`
- `/` becomes `_`
- trailing `=` padding is removed

The decoder reverses those substitutions and restores the required padding.

### `Ze(input)` — encode

The source algorithm:

1. UTF-8 encodes the input with `TextEncoder`.
2. Generates four random bytes using `crypto.getRandomValues`; it has a `Math.random` fallback.
3. Allocates an output byte array of `5 + inputByteLength`.
4. Writes marker byte `167` at index `0`.
5. Writes the four random bytes at indices `1` through `4`.
6. Obfuscates each UTF-8 input byte using the repeating `qe` table, the random bytes, and an index-derived XOR value.
7. Encodes the result as Base64URL without padding.

Exact per-byte operation:

```js
output[5 + i] =
  inputBytes[i] ^
  qe[(i + random[i & 3]) & 7] ^
  random[(i + 1) & 3] ^
  ((i * 13) & 255);
```

### `Qe(input)` — decode

The source decoder supports three input paths:

1. Empty/whitespace input returns an empty string.
2. If the input is even-length hexadecimal, each byte is parsed and XORed with `qe[i % 8]`.
3. Otherwise, it Base64URL-decodes the input:
   - If the decoded data has at least five bytes and byte zero is `167`, bytes `1..4` are treated as the random seed and the main `Ze` transform is reversed.
   - Otherwise, every decoded byte is XORed with the repeating `qe` table as a legacy/fallback path.
4. The final bytes are decoded with `TextDecoder`.

### Reference implementation

```js
const qe = [95, 58, 140, 30, 123, 77, 146, 110];
const Je = 167;

function toBase64Url(bytes) {
  let binary = "";
  const chunkSize = 32768;

  for (let offset = 0; offset < bytes.length; offset += chunkSize) {
    binary += String.fromCharCode(
      ...bytes.subarray(offset, offset + chunkSize)
    );
  }

  return btoa(binary)
    .replace(/\+/g, "-")
    .replace(/\//g, "_")
    .replace(/=+$/, "");
}

function fromBase64Url(value) {
  const base64 = value.replace(/-/g, "+").replace(/_/g, "/");
  const padding = base64.length % 4 === 0
    ? ""
    : "=".repeat(4 - (base64.length % 4));

  const binary = atob(base64 + padding);
  const bytes = new Uint8Array(binary.length);

  for (let i = 0; i < binary.length; i++) {
    bytes[i] = binary.charCodeAt(i);
  }

  return bytes;
}

export function Ze(value) {
  const input = new TextEncoder().encode(value);
  const random = new Uint8Array(4);

  if (typeof crypto !== "undefined" && crypto.getRandomValues) {
    crypto.getRandomValues(random);
  } else {
    for (let i = 0; i < 4; i++) {
      random[i] = (Math.random() * 256) | 0;
    }
  }

  const output = new Uint8Array(5 + input.length);
  output[0] = Je;
  output[1] = random[0];
  output[2] = random[1];
  output[3] = random[2];
  output[4] = random[3];

  for (let i = 0; i < input.length; i++) {
    output[5 + i] =
      input[i] ^
      qe[(i + random[i & 3]) & 7] ^
      random[(i + 1) & 3] ^
      ((i * 13) & 255);
  }

  return toBase64Url(output);
}

export function Qe(value) {
  const input = (value || "").trim();
  if (!input) return "";

  // Legacy hexadecimal format.
  if (input.length % 2 === 0 && /^[0-9a-f]+$/i.test(input)) {
    const output = new Uint8Array(input.length / 2);

    for (let i = 0; i < output.length; i++) {
      const byte = parseInt(input.substring(i * 2, i * 2 + 2), 16);
      output[i] = byte ^ qe[i % qe.length];
    }

    return new TextDecoder().decode(output);
  }

  const bytes = fromBase64Url(input);

  // Current marker-based format produced by Ze().
  if (bytes.length >= 5 && bytes[0] === Je) {
    const random = [bytes[1], bytes[2], bytes[3], bytes[4]];
    const output = new Uint8Array(bytes.length - 5);

    for (let i = 0; i < output.length; i++) {
      output[i] =
        bytes[5 + i] ^
        qe[(i + random[i & 3]) & 7] ^
        random[(i + 1) & 3] ^
        ((i * 13) & 255);
    }

    return new TextDecoder().decode(output);
  }

  // Legacy Base64URL fallback.
  const output = new Uint8Array(bytes.length);

  for (let i = 0; i < bytes.length; i++) {
    output[i] = bytes[i] ^ qe[i % qe.length];
  }

  return new TextDecoder().decode(output);
}
```

### Security interpretation — do not mislabel this

This is a reversible XOR-based obfuscation/encoding scheme with a random per-message byte seed. It is **not AES**, is not authenticated encryption, and should not be described as cryptographically secure encryption. The random bytes are embedded in the encoded output and are therefore available to the decoder.

`Qe` decodes the API transport data and, in one player loader path, a response body that is not already an HLS playlist. It is **not** evidence of HLS segment decryption, DRM decryption, Widevine/FairPlay key handling, or a license-server implementation.

## 4. API operations

All operations below are JSON payloads passed to `apiRequest(payload)`. The source's wrapper adds the `p` and `t` envelope automatically.

| Operation (`op`) | Fields sent by source | Purpose |
|---|---|---|
| `health` | none beyond `op` | Health/status check |
| `batches_query` | `page`, `limit`, `exam`, `cls`, `q` | Search/filter batches |
| `github_batches` | none beyond `op` | Fetch GitHub batch list |
| `batches` | `page`, `limit` | Paginated PiMaxer batch list |
| `batch_subjects` | `id` | Get subjects for a batch |
| `subject_topics` | `batchId`, `subjectId`, `page` | Get topic/chapter list |
| `subject_contents` | `batchId`, `subjectId`, `topicSlug`, `page`, `contentType` | Get topic content; default content type in helper is `videos` |
| `batch_details` | `id` | Batch details |
| `batch_announcements` | `id` | Batch announcements |
| `batch_tests` | `id` | Batch tests |
| `batch_schedule` | `batchId`, `subjectId`, `page` | Batch schedule |
| `radha_comments` | `scheduleId`, `page` | Comments for a schedule |
| `radha_pinned` | `scheduleId` | Pinned comment for a schedule |
| `radha_upvote` | `commentId` | Upvote a comment |
| `pimaxer_contents` | `batchId`, `subjectId`, `tag`, `contentType`, `page` | Notes/DPP/DPP videos |
| `pimaxer_test` | `testId` | Test details |

### Content types seen

For `pimaxer_contents`, the source sends:

- `notes`
- `dpp`
- `dppVideos`

For `subject_contents`, the helper defaults `contentType` to `videos`, but callers may pass another value. Do not assume undocumented values without checking actual app behavior/API responses.

### Payload examples

```js
await apiRequest({ op: "health" });

await apiRequest({
  op: "batches_query",
  page: 1,
  limit: 24,
  exam: "all",
  cls: "all",
  q: "",
});

await apiRequest({
  op: "batch_subjects",
  id: batchId,
});

await apiRequest({
  op: "subject_topics",
  batchId,
  subjectId,
  page: 1,
});

await apiRequest({
  op: "subject_contents",
  batchId,
  subjectId,
  topicSlug,
  page: 1,
  contentType: "videos",
});

await apiRequest({
  op: "pimaxer_contents",
  batchId,
  subjectId,
  tag: "",
  contentType: "notes", // or "dpp" or "dppVideos"
  page: "1",
});
```

The source sometimes sends pagination as a number and sometimes as a string. Preserve the calling helper's observed type unless live API testing shows otherwise.

### Source helper behavior and fallbacks

- Batch search caches successful non-empty results in an in-memory `Map`.
- If `batches_query` fails or returns an unexpected shape, the source falls back to a hard-coded local batch list.
- `github_batches` caches a successful list and falls back to that same local list on error.
- `getHealth()` catches errors and returns a local `{ status: "healthy", timestamp }` object. This is a UI fallback, **not** proof that the backend is healthy.
- The source sanitizes placeholder IDs such as `batch_id` and `lecture_id` through a small helper and substitutes fallback IDs. Do not copy those fallback IDs blindly into our project; use actual selected IDs.

## 5. Video stream URL construction

The source has three main stream routes.

### A. Gateway master HLS

Pattern:

```text
/s/m/{Ze(JSON.stringify(streamPayload))}.m3u8
```

Payload:

```js
{
  b: batchId,    // batch ID
  s: subjectId,  // subject ID
  l: lectureId,  // lecture ID
  v: videoId     // optional video ID
}
```

The source builds it as:

```js
export function buildMasterStreamUrl(batchId, subjectId, lectureId, videoId) {
  const payload = {
    b: batchId,
    s: subjectId,
    l: lectureId,
  };

  if (videoId && videoId.trim()) {
    payload.v = videoId.trim();
  }

  return `/s/m/${Ze(JSON.stringify(payload))}.m3u8`;
}
```

The source marks this as HLS (`isHls: true`, `isMp4: false`) and caches it by batch, subject, lecture, and optional video ID.

### B. HLS proxy for an existing URL

Pattern:

```text
/s/p/{Ze(originalUrl)}.m3u8
```

The source's resolver uses this for absolute HTTP(S) URLs that are not identified as MP4 and are not `.mpd`.

### C. MP4/original URL proxy

Pattern:

```text
/s/o/{Ze(originalUrl)}
```

The source uses this for absolute URLs identified as `.mp4`. Its `generateSignedUrl()` helper also returns this route after encoding the provided value. Despite the helper name, the frontend code alone does not establish that this is a cryptographically signed URL.

### Resolver decision logic observed

1. If the content item has both `batch_id` and `lecture_id`, build the gateway HLS URL using batch, subject, lecture, and optional `videoDetails._id`/`videoDetails.id`.
2. Otherwise read `videoDetails.videoUrl` or `url`.
3. If that URL contains `.mpd`, the inspected resolver returns an empty stream URL.
4. If it contains `.mp4`, route it through `/s/o/`.
5. If it is another absolute HTTP(S) URL, route it through `/s/p/` and append `.m3u8`.
6. If it is already a relative path, the resolver leaves it unchanged.

Do not silently treat `.mpd` as HLS. DASH support is not established by this resolver.

### D. Next-study helper

`getNextStudyStreamUrl(batchId, subjectId, lectureId)` returns:

```js
{
  streamUrl: buildMasterStreamUrl(batchId, subjectId, lectureId),
  isHls: true,
  isMp4: false,
  source: "gateway-hls"
}
```

### E. Prefetch

The source prefetches the master HLS manifest with:

```http
X-SK-M: 1
```

It throttles repeated prefetches for the same stream key for approximately 120 seconds. Prefetch is an optimization only; it should not block playback if it fails.

## 6. Video playback lifecycle

The observed intended user journey is:

1. Load/search batches.
2. Select a batch.
3. Load its subjects.
4. Load subject topics/chapters.
5. Load topic contents with `contentType: "videos"`.
6. Select a lecture/content item.
7. Resolve the lecture identifiers and construct the stream URL.
8. Create or update the player with the selected lecture and stream URL.
9. Load the HLS master manifest.
10. HLS.js reads the master manifest, discovers available levels, loads playlists and media segments, buffers them, and attaches decoded media to the HTML `<video>` element.
11. Update playback progress and controls while playing.

The source's lecture selection handler builds the gateway stream URL from the selected batch, subject, and lecture IDs, stores the selected lecture in player state, navigates to the watch view, restores a saved time if provided, and attempts the stream resolution helper if needed.

### HLS.js configuration observed

The player creates HLS.js with these settings:

| Setting | Value |
|---|---:|
| `enableWorker` | `true` |
| `progressive` | `true` |
| `lowLatencyMode` | `false` |
| `startFragPrefetch` | `true` |
| `startLevel` | `0` |
| `abrEwmaDefaultEstimate` | `1,200,000` |
| `abrBandWidthFactor` | `0.85` |
| `abrBandWidthUpFactor` | `0.6` |
| `capLevelToPlayerSize` | `false` |
| `backBufferLength` | `60` |
| `maxBufferLength` | `60` |
| `maxMaxBufferLength` | `300` |
| `maxBufferSize` | `80,000,000` |
| `maxBufferHole` | `1.5` |
| `highBufferWatchdogPeriod` | `1` |
| `nudgeOffset` | `0.15` |
| `nudgeMaxRetry` | `10` |
| `xhrSetup` | `xhr.withCredentials = false` |

Fragment policy:

- max time to first byte: 35,000 ms
- max load time: 180,000 ms
- timeout retry: 4 attempts, 400 ms initial delay, max 4,000 ms
- error retry: 6 attempts, 500 ms initial delay, max 6,000 ms

Manifest and playlist policy:

- max time to first byte: 30,000 ms
- max load time: 45,000 ms
- timeout retry: 3 attempts, 300 ms initial delay, max 3,000 ms
- error retry: 4 attempts, 500 ms initial delay, max 4,000 ms

### Custom playlist loader behavior

The source wraps the HLS.js playlist loader:

- Adds request header `X-SK-M: 1`.
- On successful response, if the response data is a string and does not begin with `#EXTM3U`, it attempts `Qe()` decoding.
- If decoding fails, it leaves the response handling to the original loader callback.
- Normal HLS playlists beginning with `#EXTM3U` are passed through without decoding.

This is playlist/response-body handling. It does not prove that media segments are XOR-decoded by `Qe`.

### Native HLS fallback

If HLS.js is not supported but the browser reports support for:

```js
video.canPlayType("application/vnd.apple.mpegurl")
```

the source assigns the HLS URL directly to `video.src`, calls `video.load()`, restores the seek position after metadata is loaded, and attempts playback.

### Error handling and MP4 fallback

Observed behavior:

- HLS fatal network errors: call `startLoad()` up to 3 retry cycles.
- HLS fatal media errors: call `recoverMediaError()` up to 3 retry cycles.
- If a fatal error remains and an MP4 fallback URL exists, switch to that fallback if it has not already been attempted.
- A fragment parsing error or response status `422` also triggers MP4 fallback when configured.
- Otherwise show a retry message to the user.
- Native HLS errors also switch to MP4 fallback when available; otherwise show an error.
- Destroy the HLS instance on cleanup/unmount.

Do not create an MP4 fallback URL by guessing. Only use a fallback URL actually supplied by a supported API/content response or an existing project contract.

## 7. Playback UX and local persistence

The inspected player includes:

- Play/pause
- Seek backward/forward by 10 seconds
- Volume slider and mute
- Playback speed
- Fullscreen
- Quality selection and Auto
- Buffering/loading indicators
- Retry action
- Resume from saved position
- Bookmarks/revision notes

Keyboard shortcuts:

| Key | Action |
|---|---|
| `Space` | Play/pause |
| `ArrowLeft` | Seek back 10 seconds |
| `ArrowRight` | Seek forward 10 seconds |
| `ArrowUp` | Increase volume |
| `ArrowDown` | Decrease volume |
| `F` | Toggle fullscreen |
| `M` | Mute/unmute |
| `Escape` | Close player/overlay according to current UI state |

Local storage keys observed:

- `pulsestream_progress_{lectureId}` — last playback position in seconds.
- `pulsestream_bookmarks` — bookmark storage key.

The source writes progress after the current time exceeds 5 seconds. Restore only valid numeric positions and seek only when the media is seekable. Keep storage access guarded because browser storage can fail or be unavailable.

## 8. Additional document/PDF URL helper

The source contains `getShyamPdfUrl(value, isId = false)`:

- If `isId` is true, it encodes `{ i: value }`.
- Otherwise it encodes `{ u: value }`.
- It returns `/s/f/{encodedPayload}`.
- Empty input returns an empty string.

The backend behavior and response type for `/s/f/` are not present in the inspected frontend source. Treat this as a discovered route pattern, not a fully documented PDF API.

## 9. What is not established by the source

The following are **unknown** from the provided frontend bundle and must not be fabricated:

- Backend implementation for `/api/d`.
- Backend implementation for `/s/m/`, `/s/p/`, `/s/o/`, and `/s/f/`.
- Actual upstream video provider, origin storage, or CDN routing.
- Whether the backend performs authorization, token checks, expiry checks, or access validation.
- Complete response schemas for every API operation.
- Whether `.mpd`/DASH streams are supported elsewhere.
- Any DRM system, DRM license endpoint, license challenge/response, content key, Widevine/FairPlay configuration, or protected-segment decryption logic.
- Whether any stream route is cryptographically signed.
- Whether `X-SK-M: 1` is mandatory for every route or only used by the observed player loader/prefetch path.

### Important distinction: transport decoding vs media DRM

- `Qe()` decodes the obfuscated API response envelope and may decode a non-`#EXTM3U` playlist response body.
- HLS.js handles HLS manifests, playlists, segments, buffering, and playback.
- The inspected source does not provide enough evidence to implement DRM decryption or license acquisition.
- Do not attempt to bypass DRM, extract keys, or defeat authorization. If the legitimate stream is DRM-protected, integrate the provider's authorized player/license flow using credentials and documentation supplied by the provider.

## 10. Recommended project architecture

Suggested separation (adapt paths to the existing project rather than duplicating files):

```text
src/
  lib/
    api/
      transport.ts       # apiRequest, HTTP errors
      codec.ts           # Ze, Qe, Base64URL helpers
      operations.ts      # typed operation helpers
    video/
      stream-url.ts      # buildMasterStreamUrl, resolveBestStreamUrl
      hls-config.ts      # shared HLS.js config and loader
      player.tsx         # player lifecycle and controls
      progress.ts        # local progress/bookmark persistence
```

Implementation principles:

- One source of truth for `Ze`/`Qe`.
- One source of truth for `/api/d` request construction.
- One typed function per API operation.
- Validate IDs and required parameters before making calls.
- Keep response parsing and UI state separate.
- Avoid logging full encoded payloads, sensitive URLs, or tokens in production.
- Never hardcode sample batch IDs as real user selections.
- Do not expose secrets in frontend environment variables.
- Do not change route patterns or request headers without testing against the compatible backend.
- Use abort/cleanup logic to prevent stale requests and destroy HLS instances when changing lectures.
- Avoid duplicate HLS instances and revoke object URLs if the implementation creates any.

## 11. Integration test checklist

### API transport
- [ ] `Ze`/`Qe` round-trip test with ASCII and Unicode strings.
- [ ] Confirm Base64URL output contains no `+`, `/`, or trailing `=`.
- [ ] Confirm request body contains `p` and base-36 `t`.
- [ ] Confirm successful response can be read from `d` and `data` formats as supported.
- [ ] Confirm malformed JSON, missing data, HTTP errors, and decode errors show useful messages.

### API operations
- [ ] Health operation.
- [ ] Batch search, pagination, filters.
- [ ] Batch details and subjects.
- [ ] Topics and video contents.
- [ ] Notes, DPP, DPP videos.
- [ ] Announcements, tests, schedule, comments, pinned comment, upvote.
- [ ] Confirm actual response shapes before defining TypeScript interfaces.

### Video
- [ ] Select a valid lecture and verify generated `/s/m/` URL.
- [ ] Confirm master manifest returns valid HLS (`#EXTM3U`) content.
- [ ] Confirm variant playlists and segments load.
- [ ] Confirm `X-SK-M: 1` is sent on the observed HLS loader/prefetch requests.
- [ ] Test HLS.js playback and native HLS fallback.
- [ ] Test quality selection and Auto.
- [ ] Test buffering, network retry, media recovery, and retry UI.
- [ ] Test MP4 fallback only when a valid fallback is actually supplied.
- [ ] Test changing lectures and cleanup.
- [ ] Test progress restore and bookmarks.
- [ ] Test `.mpd` handling explicitly; do not assume it works.

### Deployment
- [ ] Confirm frontend and API routes are deployed on compatible origins.
- [ ] Confirm relative `/api/d` and `/s/...` paths resolve to the intended server.
- [ ] Confirm production CORS/cookie behavior; source sets `withCredentials = false` for HLS XHR.
- [ ] Confirm authorized access works for the intended account/content.
- [ ] Confirm no secrets or privileged upstream credentials are bundled into the browser.

---

# Antigravity — implementation instructions

**Copy this section and give it to Antigravity along with this README and our project repository.**

You are working inside our existing project. Integrate the API and video playback flow documented in this README into the project.

## Mandatory instructions

1. **Inspect the entire existing project first.** Identify its framework, routing, current API layer, state management, content models, player implementation, styling system, and deployment configuration. Do not replace working architecture unnecessarily.
2. **Use this exact API contract** from this README:
   - `POST /api/d`
   - JSON envelope `{ p: Ze(JSON.stringify(payload)), t: Date.now().toString(36) }`
   - response field `d` or `data`, decoded using `Qe`, then JSON-parsed.
   - implement the listed `op` operations and their exact request fields.
3. **Implement `Ze` and `Qe` faithfully.** Use the exact constants and byte operations in this README. Add round-trip tests, including Unicode. Do not substitute Base64-only encoding, AES, a different XOR algorithm, or a guessed cipher.
4. **Use the documented stream routes exactly:**
   - Gateway HLS: `/s/m/{Ze(JSON.stringify({b, s, l, v?}))}.m3u8`
   - HLS proxy: `/s/p/{Ze(originalUrl)}.m3u8`
   - MP4/original proxy: `/s/o/{Ze(originalUrl)}`
   - PDF helper route, if the existing product needs it: `/s/f/{Ze(JSON.stringify({i}) or {u})}`
5. **Implement the full lecture flow:** batch → subjects → topics → contents → select lecture → resolve stream URL → player.
6. **Implement HLS.js playback** with the configuration, custom playlist loader, `X-SK-M: 1` request header, response decode fallback, native HLS fallback, retry/recovery, optional valid MP4 fallback, cleanup, and progress restore described in this README.
7. **Implement player controls** that fit the existing UI: play/pause, ±10s, volume/mute, speed, fullscreen, quality/Auto, buffering/error states, keyboard shortcuts, and local progress/bookmarks where appropriate.
8. **Use real IDs from API responses.** Do not use placeholder IDs or hardcoded example batches as production data.
9. **Preserve existing project features and visual design.** Reuse existing components/styles where sensible. Do not redesign unrelated pages.
10. **Do not guess backend behavior.** The README documents frontend-observed contracts only. If an endpoint fails or a response differs, inspect the actual network response and report the mismatch instead of inventing fields or endpoints.
11. **Do not claim DRM decryption is implemented.** `Qe` is the source's reversible transport/playlist obfuscation decoder, not a DRM key decryptor. Do not add DRM bypass, key extraction, or license circumvention. If a legitimate provider DRM flow is required, stop and request its authorized license integration documentation.
12. **Do not expose secrets.** Do not place upstream credentials, private tokens, or server secrets in frontend code. Do not log sensitive full URLs/payloads in production.
13. **Do not stop after writing code.** Run the project's available lint, typecheck, tests, and production build. Fix integration regressions caused by your changes.
14. **Verify actual playback.** Test at least one authorized lecture end-to-end: API response → stream URL → manifest → segments → playable video. If credentials/content access or backend routes are unavailable, clearly state that playback could not be verified and identify the exact blocker.
15. **At completion, provide a concise implementation report:** files changed, API operations integrated, player behavior implemented, tests/build results, live playback verification status, and any backend details still required.

## Acceptance criteria

- The project uses the documented `/api/d` wrapper and codec.
- Batch/subject/topic/content navigation works using live API data.
- A valid lecture resolves to the documented gateway/proxy route.
- HLS.js or native HLS can play a valid authorized stream.
- Playlist decoding is attempted only under the observed non-`#EXTM3U` condition.
- Errors and retries are visible and do not leave stale player instances.
- Progress and player controls work.
- Existing unrelated project functionality remains intact.
- Unknown backend/DRM behavior is documented as unknown, never fabricated.
