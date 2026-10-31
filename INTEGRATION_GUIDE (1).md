# Non-Translated, Nigerian & Sports API: Integration Guide

**Base URL:** `https://axis.yocinema.dpdns.org`

This API serves three catalogs from one backend:

| Catalog | What it is |
|---|---|
| **Non-translated movies & TV** | Original-language titles, with subtitles |
| **Nigerian movies & TV** | The same API, filtered to `country = Nigeria` |
| **Sports** | Leagues, live matches, fixtures, replays and highlights |

It is built for apps and websites. This guide documents every endpoint and also shows **how the official YOCINEMA Android app uses it**, so you can copy a flow that is known to work.

> The translated catalog is a separate API with its own documentation. Nothing here applies to it.

---

## Contents

1. [Getting access](#getting-access)
2. [Authentication & errors](#authentication--errors)
3. [Apps vs websites](#apps-vs-websites)
4. [Hosts you will see](#hosts-you-will-see)
5. [How the official app uses the API](#how-the-official-app-uses-the-api)
6. [Endpoint reference: movies & TV](#endpoint-reference-movies--tv)
7. [Captions (subtitles)](#captions-subtitles)
8. [Nigerian vs non-translated feeds](#nigerian-vs-non-translated-feeds)
9. [Sports](#sports)
10. [Media links](#media-links)
11. [Code examples](#code-examples)
12. [Caching & rate limits](#caching--rate-limits)
13. [Support](#support)

---

## Getting access

1. **Create an API key** in the dashboard: <https://dash.yocinema.dpdns.org>
2. **Ask for non-translated/Nigerian access to be enabled on that key.** It is off by default. Message us on [Telegram](https://t.me/jeliostarrdev) or [WhatsApp](https://wa.me/256707934960) and we'll switch it on for free.

Until step 2 is done, every request returns `403` with the message *"not enabled for movies-demo access"* (see [errors](#authentication--errors)).

---

## Authentication & errors

Send your key on **every** request:

```
X-API-Key: <your key>
```

All requests are `GET`. Every request, including failed ones, counts against your key's daily limit.

### Error responses

Errors are JSON with the message in `error` (and sometimes a `code`):

```json
{ "error": "API key required", "code": "KEY_REQUIRED" }
```

Be defensive when parsing: read `error`, then fall back to `message`, then to `data.error` / `data.message`. The official app does exactly this.

| Status | Message contains | Meaning | What to do |
|---|---|---|---|
| 401 | `required` | No key was sent | Send `X-API-Key` |
| 401 | *(anything else)* | Key not recognized | Check the key |
| 403 | `not enabled for movies-demo access` | Key works but this catalog isn't enabled | [Request access](#getting-access) |
| 403 | `revoked` / `deleted` | Key was revoked or removed | Create a new key |
| 403 | `paused` / `suspended` | Key is disabled | Contact support |
| 403 | `expired` | Key or plan expired | Renew in the dashboard |
| 403 | `inactive` | Account inactive | Contact support |
| 403 | `Origin not allowed` | Browser origin not allow-listed | See [websites](#apps-vs-websites) |
| 429 | | Daily request limit reached | Wait for reset or upgrade |
| 400 | | Bad or missing parameter | Fix the request |
| 404 | | Title / event / video not found | Show "not available" |
| 502 / 504 | | Upstream failure or timeout | Retry with backoff |
| 500 | | Unexpected error | Retry once, then report |

Match on **status first, then message text**. The message text is the only way to tell the 403 reasons apart.

---

## Apps vs websites

### Native apps (Android, iOS)

Call the API directly with `X-API-Key`. Native requests carry no browser `Origin` header, so there is nothing to allow-list. Don't hardcode a key in a publicly distributed app. Have the user sign in with their own key (as the official app does) or route through your own server.

### Websites

Two options:

- **Recommended: call from your backend.** Your server holds the key and passes titles and media links to your front end. No CORS issues, and the key is never exposed.
- **Call from the browser.** Your site's exact origin (e.g. `https://yoursite.com`) must be allow-listed by us, otherwise you get `403 Origin not allowed`. The key is visible in your page's network tab, so only do this with a key you can rotate.

Media links (`url`, `downloadUrl`, subtitles, images) work **directly in the browser without any key**, so only API calls need to go through your backend. See [Media links](#media-links).

---

## Hosts you will see

| Purpose | Host |
|---|---|
| **API (this guide)** | `https://axis.yocinema.dpdns.org` |
| Video & downloads | `https://stream.yocinema.dpdns.org` |
| Subtitles | `https://edge.jeliostarrdev.qzz.io` |
| Images (covers, avatars, banners) | `https://cdn.yocinema.dpdns.org` / `https://cdn.jeliostarrdev.qzz.io` |

Media links look like `https://<host>/m/<long-token>`. If your site uses a Content-Security-Policy, allow **both parent domains** (`*.yocinema.dpdns.org` and `*.jeliostarrdev.qzz.io`) for `media-src`, `img-src` and `connect-src`. Don't hardcode individual media hosts: they can change.

---

## How the official app uses the API

The Android app is the reference client. These are its real flows, in the order a user hits them.

### Home (non-translated)

`GET /api/home`. The app then:

1. Shows `banners` (up to 8) as the hero carousel. If there are none, it builds a hero from the first items of the `subjects` sections.
2. Renders each `section` by `type`: `subjects` → poster rail, `categories` → chip rail, `appointment` → "coming soon" rail.
3. **Hides** `appointment` sections and any section whose title contains "nollywood" on the non-translated home.
4. Applies the [feed filter](#nigerian-vs-non-translated-feeds) to every rail.
5. "View all" on a rail → `GET /api/browse?rankingId=<section.rankingId>` when the section has one, otherwise `browse?genre=<section title>`.
6. A category chip → `GET /api/browse?query=<chip.query>`.

### Nigerian movies

`GET /api/browse?country=Nigeria&sort=Latest&page=1&limit=…`, with `type=movie` or `type=tv` for the Movies / TV tabs (no `type` = All). Search inside this section calls `/api/search` and then keeps only `country == Nigeria` results.

### Filter screen (Movie / TV / Anime tabs)

1. Load options once: `GET /api/browse?filters=true` → `{ genres, countries, years, sortOptions }`.
2. Query: `GET /api/browse?type=movie|tv|animation&genre=&country=&year=&sort=Hottest&page=&limit=`. "All" means leave that parameter out.

### Search

`GET /api/search?q=…&limit=30`. The search landing page shows `GET /api/trending?limit=30` before the user types.

### Title page

1. `GET /api/details/:detailPath`
2. For TV: `GET /api/seasons/:detailPath` (season/episode pickers)
3. "More like this": `GET /api/browse?country=<title's country>&genre=<first genre>&sort=Hottest&limit=20`, excluding the current title, keep 15.
4. Dubs: show `details.dubs` where the dub's `type == 0` (audio dubs) as a language picker. A chosen dub's own `detailPath` replaces the original path for all playback calls.

### Play

1. `GET /api/movie/:detailPath` or `GET /api/tv/:detailPath?season=S&episode=E` (use the dub's `detailPath` if one is selected).
2. Keep only **free** qualities: `vipLocked != true` and `url` not empty. Sort by `resolution` descending. If there are none, use `best_free`. If that's also empty, show "No playable source".
3. In parallel (after the stream call returns): `GET /api/captions/:detailPath?season=S&episode=E` (omit season/episode for movies). Captions are optional: if this fails, play without subtitles.
4. Give the quality `url` to the player.

### Download

The same quality list (step 2 above). Download the quality's `url` (or `downloadUrl`) and name the file yourself, e.g. `Title_S1E3_1080P.mp4`. No headers are needed on the media link.

### Sports

See [Sports](#sports).

---

## Endpoint reference: movies & TV

All paths are relative to `https://axis.yocinema.dpdns.org`. `detailPath` is the slug of a title, e.g. `oppenheimer-Akh5Nrwl7o`. Path form (`/api/movie/<detailPath>`) and query form (`/api/movie?detailPath=<detailPath>`) are equivalent for `details`, `seasons`, `movie`, `tv` and `captions`.

| Endpoint | Purpose |
|---|---|
| `GET /api/home` | Banners, sections, category chips |
| `GET /api/trending` | Trending titles |
| `GET /api/search` | Search / autocomplete |
| `GET /api/browse` | Filtered list, filter options, category chips, rankings |
| `GET /api/catalog` | Simple movie / tv / animation list |
| `GET /api/details/:detailPath` | Synopsis, cast, dubs, trailer |
| `GET /api/seasons/:detailPath` | Seasons and episode counts |
| `GET /api/movie/:detailPath` | Movie stream and download links |
| `GET /api/tv/:detailPath` | Episode stream and download links |
| `GET /api/captions/:detailPath` | Subtitle tracks |

### Subject (title in a list)

```json
{
  "title": "Oppenheimer",
  "subjectId": "1234567890",
  "subjectType": 1,
  "type": "movie",
  "detailPath": "oppenheimer-Akh5Nrwl7o",
  "genre": "Drama, History",
  "imdbRating": "8.3",
  "country": "United States",
  "description": "…",
  "releaseDate": "2023-07-21",
  "duration": 10800,
  "cover": "https://…/m/<token>",
  "hasResource": true
}
```

`subjectType`: `1` movie, `2` TV. `duration` is in seconds. `hasResource: false` means nothing playable exists. Field names vary slightly by endpoint: home/details use `countryName` and `imdbRatingValue` instead of `country` and `imdbRating`, so read both. Lists come back as `results` (search, browse, catalog) or `items` (trending); the official app accepts either.

### `GET /api/home`

```json
{
  "bannerCount": 8,
  "banners": [ { "...Subject": "", "bannerImage": "https://…" } ],
  "sections": [
    { "type": "subjects", "title": "Trending Now", "position": 3, "count": 12, "items": [ ], "rankingId": "123" },
    { "type": "categories", "title": "Categories", "position": 5, "items": [],
      "categories": [ { "title": "Hollywood", "query": "type=/home/movieFilter&…", "image": "https://…" } ] },
    { "type": "appointment", "title": "Coming Soon", "items": [ ] }
  ]
}
```

- Pass a category `query` **unchanged** to `/api/browse?query=`.
- A section with `type: "sport_live"` carries a `route` (`/api/sports/live`); call that route to show live matches.

### `GET /api/trending`

`?limit=20` (max 100) → `{ "count": 20, "items": [Subject] }`

### `GET /api/search`

| Param | Default | Notes |
|---|---|---|
| `q` | | **Required** |
| `limit` | 20 | 1–50 |
| `mode=suggest` | | Autocomplete: returns `{ suggestions: [{ word, type }] }` (limit default 10, max 25) |

```json
{ "query": "bridgerton", "count": 20, "total": 37, "results": [ Subject ] }
```

### `GET /api/browse`

| Use | Request | Response |
|---|---|---|
| Filtered list | `?type=&genre=&country=&year=&sort=&page=1&limit=20` | `{ type, genre, country, year, sort, page, count, results }` |
| Filter options | `?filters=true` | `{ genres, countries, years, sortOptions }` |
| Category chip | `?query=<chip.query>&page=1&limit=20` | `{ query, tabId, filter, page, count, results }` |
| Ranking ("View all") | `?rankingId=<id>&page=1&limit=24` | `{ rankingId, title, page, count, hasMore, nextPage, results }` |

- `type`: `movie`, `tv`, `animation`, `all` (default).
- `sort`: `Latest`, `Hottest`, `ForYou`, `Rating`.
- Keep `limit` at 50 or below.
- Page through a ranking with `page = nextPage` while `hasMore` is `true`. For other lists, there's more when a page returns a full `limit`.

### `GET /api/catalog`

`?type=movie|tv|animation&page=1&limit=20` → `{ type, page, count, results }`. Other `type` values return 400.

### `GET /api/details/:detailPath`

```json
{
  "detailPath": "…", "type": "movie", "title": "Oppenheimer", "description": "…",
  "genre": "Drama, History", "releaseDate": "2023-07-21", "duration": 10800, "durationText": "3h 0m",
  "imdbRatingValue": "8.3", "countryName": "United States", "subtitles": "English, French, …",
  "cover": "https://…", "hasResource": true,
  "trailer": { "url": "https://…", "cover": "https://…" },
  "cast": [ { "name": "…", "role": "Cast", "character": "…", "avatarUrl": "https://…" } ],
  "dubs": [ { "lanName": "French", "lanCode": "fr", "type": 0, "kind": "dub", "detailPath": "oppenheimer-fr-…" } ]
}
```

- `cast[].role`: `Cast`, `Director` or `Staff`.
- `dubs[].type`: `0` = audio dub, `1` = subtitle-only variant. Offer only `type 0` as a language switcher.
- `trailer.url` may be empty.

### `GET /api/seasons/:detailPath`

```json
{
  "seasonCount": 6, "totalEpisodes": 93, "globalResolutions": [360, 480, 720, 1080],
  "seasons": [ { "season": 1, "maxEp": 13, "availableResolutions": [360, 480, 720, 1080] } ]
}
```

Build the episode list from `seasons[].maxEp`.

### `GET /api/movie/:detailPath` and `GET /api/tv/:detailPath`

TV adds `?season=1&episode=1` (both default to `1`).

```json
{
  "title": "Oppenheimer", "detailPath": "…",
  "qualities": [
    { "resolution": 1080, "size_mb": 2210.45, "duration_sec": 10800, "codec": "h264",
      "vipLocked": false, "url": "https://…/m/<token>", "downloadUrl": "https://…/m/<token>" }
  ],
  "best_free": { }
}
```

| Field | Use |
|---|---|
| `url` | Playback (supports seeking) |
| `downloadUrl` | Same file served as an attachment |
| `vipLocked: true` | Not available. Skip it. |
| `best_free` | Best playable quality, or `null` |

TV responses also include `season`, `episode` and `available_seasons`.

---

## Captions (subtitles)

### `GET /api/captions/:detailPath`

| Param | Default | Notes |
|---|---|---|
| `season` | `0` | **Movies: omit.** TV: the real season. |
| `episode` | `0` | **Movies: omit.** TV: the real episode. |

```json
{
  "detailPath": "lucifer-UQASHYbVPB2", "title": "Lucifer", "season": 1, "episode": 1,
  "captionCount": 13,
  "captions": [
    { "id": "…", "lan": "en", "lanName": "English",
      "url": "https://edge.jeliostarrdev.qzz.io/m/<token>",
      "downloadUrl": "https://edge.jeliostarrdev.qzz.io/m/<token>",
      "size": 52310, "delay": 0 }
  ]
}
```

### Facts to build around

- **Format:** `.srt` (SubRip), served from the subtitle host ([hosts](#hosts-you-will-see)). The link is self-authorizing: **no headers or key** needed, and it expires after about 6 hours.
- **Language fields:** `lan` is a code (`en`), `lanName` is the display name (`English`). The official app uses `lan`, falling back to `lanName`, as the language and `lanName`, falling back to `lan`, as the label.
- **Default track:** the app auto-selects English (`lan == "en"` or `lanName == "English"`) and otherwise starts with subtitles **off**. A subtitle menu lets the user switch tracks or turn them off.
- **Many titles have no captions.** Nigerian titles typically return an empty list. Treat an empty `captions` array as normal and hide the subtitle button.
- **Never block playback on captions.** The app starts the video as soon as the stream call returns and loads captions alongside it. If the captions call fails, it plays without them, and if a single track fails to load, the video is unaffected.
- **Call order:** request the stream (`/api/movie` or `/api/tv`) first, then captions. If captions return `404 No playable video id found`, wait 2–3 minutes and retry.
- **Use `GET`.** Don't probe caption URLs with `HEAD` before using them; it adds a round trip and isn't needed.
- **`delay` field:** the official app does not apply it. Instead it gives users a manual subtitle-delay control (0 to 15 seconds). Do the same, or apply `delay` yourself if you find it useful.
- **TV:** pass the real `season` and `episode`. Captions are per episode.
- **Caching:** the app caches the captions response for 30 minutes (keyed by title + season + episode). The links inside expire in about 6 hours, so don't store them longer.
- **Dubs:** captions for a dubbed version are fetched with the dub's own `detailPath`.

### Using captions

**Android (Media3 / ExoPlayer):** attach `.srt` directly:

```kotlin
val subs = captions.mapNotNull { c ->
    val url = c.url ?: return@mapNotNull null
    val isEnglish = c.lan.equals("en", true) || c.lanName.equals("English", true)
    MediaItem.SubtitleConfiguration.Builder(Uri.parse(url))
        .setMimeType(MimeTypes.APPLICATION_SUBRIP)
        .setLanguage(c.lan ?: "und")
        .setSelectionFlags(if (isEnglish) C.SELECTION_FLAG_DEFAULT or C.SELECTION_FLAG_AUTOSELECT else 0)
        .build()
}
val item = MediaItem.Builder().setUri(videoUrl).setSubtitleConfigurations(subs).build()
```

**Web (`<video>`):** browsers only accept WebVTT in `<track>`, so convert on the fly:

```js
function srtToVtt(srt) {
  return "WEBVTT\n\n" + srt.replace(/\r+/g, "").replace(/(\d{2}:\d{2}:\d{2}),(\d{3})/g, "$1.$2");
}

async function addSubtitle(video, cap) {
  const res = await fetch(cap.url);                     // plain GET, no headers
  const vtt = srtToVtt(await res.text());
  const track = document.createElement("track");
  Object.assign(track, {
    kind: "subtitles", label: cap.lanName, srclang: cap.lan,
    src: URL.createObjectURL(new Blob([vtt], { type: "text/vtt" })),
    default: cap.lan === "en",
  });
  video.appendChild(track);
}
```

If the browser blocks the cross-origin `fetch` of a subtitle, fetch it on your backend and serve it from your own domain.

---

## Nigerian vs non-translated feeds

There is **no separate Nigerian endpoint**. Nigerian titles live in the same catalog and are identified by `country == "Nigeria"` (case-insensitive; read `country` or `countryName`). The official app builds two feeds from the same API by filtering results client-side:

```text
playable(title)  = (type == "movie" or type == "tv") and hasResource != false

Non-translated feed (no country filter):
    keep playable titles that are NOT Nigerian

Nigerian feed (countryFilter = "Nigeria"):
    keep playable titles whose country equals "Nigeria"
```

Apply this to **every** list you render (home rails, browse, search, trending), because the API returns mixed results.

```js
const isNigerian = s => (s.country ?? s.countryName ?? "").toLowerCase() === "nigeria";
const isPlayable = s => (s.type === "movie" || s.type === "tv") && s.hasResource !== false;

function cleanForFeed(items, countryFilter = null) {
  return items.filter(s =>
    isPlayable(s) &&
    (countryFilter == null ? !isNigerian(s)
                           : (s.country ?? s.countryName ?? "").toLowerCase() === countryFilter.toLowerCase())
  );
}
```

For the Nigerian feed, ask the server to do most of the work: `browse?country=Nigeria&sort=Latest`. Then still run `cleanForFeed(items, "Nigeria")`.

---

## Sports

All paths: `GET https://axis.yocinema.dpdns.org/api/sports/...` with `X-API-Key`.

| Endpoint | Params | Returns |
|---|---|---|
| `/api/sports/leagues` | none | `{ items: [{ id, name, localName }] }` |
| `/api/sports/live` | `limit`, `cursor`, `leagueId`, `sport` | Matches currently live |
| `/api/sports/schedule` | `limit`, `cursor`, `leagueId`, `sport`, `date`, `from`, `to` | Scheduled, finished, postponed and cancelled matches |
| `/api/sports/events/:id` | none | `{ item }`: one match (404 if unknown) |
| `/api/sports/media` (alias `/replays`) | `days` (1–14, default 3), `limit` (1–100, default 40) | `{ items, count }`: recent matches that have replays or highlights, newest first |

- **Pagination:** list responses are `{ items, count, hasMore, nextCursor }`. Pass `nextCursor` back as `cursor`. `count` is the total number of matches, not the page size. `limit` defaults to 20 (max 100).
- **Dates:** `date` / `from` / `to` are `YYYY-MM-DD` and compare against the **UTC** start time. `to` excludes events later on that same date, so use the next day's date as your upper bound.
- **Strict params:** unknown query parameters return `400`.
- **Times:** `startTime` / `endTime` are ISO-8601 UTC strings. Convert to the viewer's time zone for display.

### Match object

```json
{
  "id": "123456",
  "sport": "football",
  "league": "Premier League", "leagueId": "10", "round": "Matchday 10", "season": "2026/27",
  "teams": {
    "home": { "id": "1", "name": "Team A", "abbreviation": "TMA", "score": 2, "logo": "https://…" },
    "away": { "id": "2", "name": "Team B", "abbreviation": "TMB", "score": 1 }
  },
  "status": "live",
  "startTime": "2026-10-31T15:00:00.000Z", "endTime": null,
  "liveStream": { "available": true, "url": "https://…/index.m3u8", "format": "hls", "authorized": true },
  "streams": [],
  "replay": [],
  "highlights": [ { "id": "…", "title": "Goal 12'", "url": "https://…", "cover": "https://…", "durationSeconds": 42, "createdAt": "…" } ]
}
```

- `status`: `scheduled`, `live`, `finished`, `postponed`, `cancelled`. Treat anything else as unknown.
- `score` is a number or `null`. `logo` is **omitted** when there is none, so use a placeholder.
- **Can the viewer watch live?** Only when `status == "live"` **and** `liveStream.available == true` **and** `liveStream.url` is non-empty. Otherwise don't show a Watch button.
- `streams` is always empty. Ignore it.
- `replay` and `highlights` are often empty. Don't render tabs or buttons for empty lists.
- Optional fields may also appear (`statusLive`, `timeDesc`, `leagueLogo`, `teamMatchInfo`, `result`). Ignore anything you don't need.

### How the official app builds its sports feed

On opening Sports (and on pull-to-refresh) it makes three requests in parallel:

| Request | Used for |
|---|---|
| `live` (all pages: `limit=50`, follow `nextCursor` until `hasMore` is false) | **Live** tab. Sorted: playable matches first, then by kickoff. |
| `schedule?limit=60` | **Fixtures**: only `status == scheduled`, sorted by kickoff |
| `media?days=3&limit=40` | **Replays** (finished with a `replay`) and **Highlights** (has `highlights`, excluded if already shown under Replays) |

- A league filter chip re-runs the feed with `leagueId=<id>`. League list: `leagues`, cached 1 hour.
- **Polling:** only while at least one match is live, re-fetch every **30 seconds** and flash any match whose score changed. Stop polling when nothing is live or the screen is hidden. Poll failures are silent. Keep showing the last good data.
- **Playing a match:** `live` match → `liveStream.url`. Finished match → `replay[0].url`, otherwise `highlights[0].url`. The match-detail page lists every replay and highlight clip.
- **Match details:** `GET /api/sports/events/:id` (cached about 45 seconds).

### Playing sports media

| Media | Playback |
|---|---|
| Replays and highlights | Play the `url` directly. No special headers. |
| **Live** (`liveStream.url`, HLS `.m3u8`) | The live provider rejects requests that don't carry a browser-like identity. See below. |

**Native apps (live):** send these headers on the playlist **and** segment requests:

```
Referer: https://sportsnow.top/
Origin: https://sportsnow.top
User-Agent: Mozilla/5.0 (Linux; Android 14) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Mobile Safari/537.36
```

The official app does this only for live hosts (`*.aisports.mobi`, and `lacdn.aoneroom.com` manifests), never for replays or highlights. In ExoPlayer: `DefaultHttpDataSource.Factory().setDefaultRequestProperties(headers)` or `OkHttpDataSource` with those headers.

```kotlin
val headers = mapOf(
    "Referer" to "https://sportsnow.top/",
    "Origin" to "https://sportsnow.top",
    "User-Agent" to "Mozilla/5.0 (Linux; Android 14) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Mobile Safari/537.36",
)
val factory = DefaultHttpDataSource.Factory().setDefaultRequestProperties(headers)
val source = HlsMediaSource.Factory(factory).createMediaSource(MediaItem.fromUri(liveUrl))
```

**Websites (live):** browsers do not allow pages to set `Referer`, `Origin` or `User-Agent`, so a plain `<video>` or hls.js player cannot play live matches directly. Replays and highlights work normally. For live in a browser, contact us ([Support](#support)) to discuss options. Note that anything routed through the API's own proxy counts **every** playlist and segment request against your daily limit.

---

## Media links

Every playable or displayable link in a response (`url`, `downloadUrl`, covers, banners, avatars, trailers, subtitles) is an **opaque link that works on its own**. Treat it as a string you pass to a player or `<img>`.

- **No headers or key** are needed on these links.
- **They expire** (about 6 hours for video, downloads and subtitles; about 24 hours for images). Don't store them. Store `detailPath` (+ season/episode) and fetch fresh links on play. The official app caches stream responses for only **60 seconds**.
- **If playback fails** with 403/404 midway, request fresh links and resume at the same position.
- **Image caching:** the same cover returns a **different link** on each API response, so URL-based caches never hit. Use a stable cache key such as `subject:<subjectId>:cover` (the official app uses this pattern with Coil: `diskCacheKey` / `memoryCacheKey`). On the web, cache by `subjectId` in your own store.
- **Don't parse or modify** links. Anyone holding a link can use it until it expires.
- **Sports media** (live streams, replays, highlights, logos) are normal provider URLs, not the opaque links above, so they are used as returned.

---

## Code examples

### Kotlin: Retrofit + OkHttp

```kotlin
class ApiKeyInterceptor(private val key: () -> String?) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val req = chain.request().newBuilder().apply {
            key()?.takeIf { it.isNotBlank() }?.let { header("X-API-Key", it) }
        }.build()
        return chain.proceed(req)
    }
}

interface DemoApi {
    @GET("api/home") suspend fun home(): Response<HomeResponse>
    @GET("api/browse") suspend fun browse(
        @Query("type") type: String? = null, @Query("genre") genre: String? = null,
        @Query("country") country: String? = null, @Query("year") year: String? = null,
        @Query("sort") sort: String? = null, @Query("page") page: Int = 1, @Query("limit") limit: Int = 20,
    ): Response<BrowseResponse>
    @GET("api/search") suspend fun search(@Query("q") q: String, @Query("limit") limit: Int = 20): Response<BrowseResponse>
    @GET("api/details/{path}") suspend fun details(@Path("path") p: String): Response<Details>
    @GET("api/seasons/{path}") suspend fun seasons(@Path("path") p: String): Response<Seasons>
    @GET("api/movie/{path}") suspend fun movie(@Path("path") p: String): Response<StreamResponse>
    @GET("api/tv/{path}") suspend fun tv(@Path("path") p: String, @Query("season") s: Int, @Query("episode") e: Int): Response<StreamResponse>
    @GET("api/captions/{path}") suspend fun captions(@Path("path") p: String, @Query("season") s: Int? = null, @Query("episode") e: Int? = null): Response<CaptionsResponse>
    @GET("api/sports/live") suspend fun live(@Query("limit") limit: Int = 50, @Query("cursor") cursor: String? = null, @Query("leagueId") leagueId: String? = null): Response<SportsList>
    @GET("api/sports/schedule") suspend fun schedule(@Query("limit") limit: Int = 60, @Query("leagueId") leagueId: String? = null, @Query("date") date: String? = null): Response<SportsList>
    @GET("api/sports/media") suspend fun media(@Query("days") days: Int = 3, @Query("limit") limit: Int = 40): Response<SportsList>
    @GET("api/sports/events/{id}") suspend fun event(@Path("id") id: String): Response<SportsItem>
}

val client = OkHttpClient.Builder()
    .addInterceptor(ApiKeyInterceptor { BuildConfig.API_KEY })
    .connectTimeout(20, TimeUnit.SECONDS).readTimeout(30, TimeUnit.SECONDS)
    .build()

val api = Retrofit.Builder()
    .baseUrl("https://axis.yocinema.dpdns.org/")
    .client(client)
    .addConverterFactory(MoshiConverterFactory.create())
    .build().create(DemoApi::class.java)
```

Make every response field **nullable or defaulted**. The official app does this so a missing key never crashes parsing.

Pick the free quality:

```kotlin
val free = (stream.qualities ?: emptyList())
    .filter { it.vipLocked != true && !it.url.isNullOrBlank() }
    .sortedByDescending { it.resolution ?: 0 }
val pick = free.firstOrNull() ?: stream.bestFree?.takeIf { it.vipLocked != true && !it.url.isNullOrBlank() }
```

### JavaScript (Node backend)

```js
const BASE = "https://axis.yocinema.dpdns.org";
const KEY  = process.env.YOCINEMA_API_KEY;

async function api(path) {
  const res = await fetch(BASE + path, { headers: { "X-API-Key": KEY } });
  const body = await res.json().catch(() => ({}));
  if (!res.ok) {
    const msg = body.error ?? body.message ?? body.data?.error ?? `HTTP ${res.status}`;
    throw Object.assign(new Error(msg), { status: res.status, code: body.code });
  }
  return body;
}

// Nigerian movies
const ng = await api("/api/browse?country=Nigeria&sort=Latest&type=movie&limit=20");
const items = cleanForFeed(ng.results ?? ng.items ?? [], "Nigeria");

// Non-translated movie: stream + captions
const path = "oppenheimer-Akh5Nrwl7o";
const stream = await api(`/api/movie/${path}`);
const q = (stream.qualities ?? [])
  .filter(x => x.vipLocked !== true && x.url)
  .sort((a, b) => (b.resolution ?? 0) - (a.resolution ?? 0))[0] ?? stream.best_free;
const caps = await api(`/api/captions/${path}`).catch(() => ({ captions: [] }));

// Sports: live now
const live = await api("/api/sports/live?limit=50");
const watchable = live.items.filter(m => m.status === "live" && m.liveStream?.available && m.liveStream.url);
```

### Web player (non-sports)

```html
<video id="v" controls playsinline></video>
<script>
  // `q` and `caps` come from your backend (above)
  const v = document.getElementById("v");
  v.src = q.url;                                  // plays directly, no headers
  (caps.captions || []).forEach(c => addSubtitle(v, c));   // see "Captions"
</script>
```

---

## Caching & rate limits

Every request counts toward your key's daily limit, so cache. These TTLs are what the official app uses:

| Data | Cache for |
|---|---|
| Home, browse, ranking, trending, catalog | 5 minutes |
| Filter options (`browse?filters=true`) | 30 minutes |
| Title details | 30 minutes |
| Seasons | 1 hour |
| Captions response | 30 minutes |
| **Stream / quality links** (`movie`, `tv`) | **60 seconds** (links expire) |
| Sports leagues | 1 hour |
| Sports live | 20 seconds |
| Sports schedule | 3 minutes |
| Sports replays/highlights (`media`) | 10 minutes |
| Sports single match | 45 seconds |

Other tips:

- **Coalesce duplicate requests.** If several screens ask for the same thing at once, send one request. The app does this.
- **Debounce search** (about 250 ms) and use `mode=suggest` for autocomplete.
- **Paint from a local cache first** (the app stores the last home and browse pages on disk and shows them instantly while refreshing) so a cold start never shows an empty screen.
- **Retry** `502` / `504` / `500` up to 2–3 times with exponential backoff (1 s, 2 s, 4 s). Don't retry other `4xx`, except `404` on captions.
- On **429**, stop calling and tell the user their daily limit is reached.
- Use **client timeouts** of about 20 s connect / 30 s read.

---

## Support

For key access, enabling non-translated/Nigerian content, higher limits or website allow-listing:

- Telegram: <https://t.me/jeliostarrdev>
- WhatsApp: <https://wa.me/256707934960>
- Dashboard: <https://dash.yocinema.dpdns.org>
