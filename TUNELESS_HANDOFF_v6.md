# Tuneless — Project Handoff (v6)

_Last updated: October 2026. Supersedes v5. Reflects `index.html` with the v6 changes: hard-stop timing engine, version-aware song deduplication, 50-song playlist minimum, and a resilient leaderboard. **The v6 changes were tested with unit/simulation tests only — they have not yet been verified against live Spotify or Supabase (see §0).**_

---

## 0. What Changed in v6 (and what is still unverified)

| # | Issue (priority order) | What was done | Verification status |
|---|---|---|---|
| 1 | Clips sometimes play past their limit (worst at 0.5s) | Rebuilt the stop engine (§10.1): position-based deadline, 20ms watchdog, instant GainNode mute, pause-until-confirmed with generation guard | Simulated with a lagging fake player (300ms start lag / 250ms pause lag): audible overrun **≤10ms**. **Not yet tested on real Spotify.** |
| 2 | Duplicate songs / multiple versions in the pool | `dedupeTracks()` collapses same-id, same-ISRC, and same-normalised-title+shared-artist tracks to one (§10.3) | 12 Node unit tests pass (remaster, remix, live, feat., covers kept, non-Latin titles, etc.) |
| 3 | Minimum 50 songs to be playable | `MIN_PLAYLIST_SONGS = 50`; small playlists hidden from selection; rule re-checked on the deduped pool at start (§10.4) | Logic reviewed; **not yet tried against real playlists** |
| 4 | Leaderboard empty / lost after inactivity | Timeouts + retry, correct auth header for `sb_publishable_` keys, persistent cache of last good board, retry queue for failed uploads, keep-alive workflow (§11) | **Could not reach Supabase from the build sandbox.** Root cause is inferred, not confirmed — see "Leaderboard diagnosis" below |

### Leaderboard diagnosis (read this first)
The most likely cause of "empty after weeks of inactivity" is that **Supabase free-tier projects auto-pause after ~7 days without API activity**. While paused, every request fails, and v5 silently fell back to the *local* cache — which is empty on any device that hasn't played. Data in a paused project is normally retained, but only for a limited window (check the Supabase dashboard for the current retention period).

**Action required by the project owner:**
1. Open https://supabase.com/dashboard → project **Tuneless**. If it says *Paused*, click **Restore project** and wait for it to come back.
2. Add `.github/workflows/supabase-keepalive.yml` (included alongside this build) to the GitHub Pages repo so it pings the API twice a week and can't pause again.
3. Load the game → open the leaderboard. If it still fails, the note under the board now shows the reason (`HTTP 401`, `timed out`, …) and the browser console has details. A `401/403` points at the key or RLS policies rather than a paused project.

Secondary suspect fixed in code: v5 sent the publishable key (`sb_publishable_…`) as an `Authorization: Bearer` header. That key is not a JWT and belongs in `apikey` only (`sbHeaders()` now does this; legacy `eyJ…` keys still get both).

### Known issues / open items
- **Rename of leaderboard entries probably doesn't persist remotely.** The `owner update` RLS policy compares `spotify_id` to a JWT `sub` claim, but the game only uses the anon key (no user JWT), so the PATCH matches 0 rows and PostgREST returns 200 with `[]`. `sbUpdateName` now logs a warning when this happens. Fixing it properly needs either a different policy/column-grant (which lets anyone rename any row) or a Supabase Edge Function. Not changed, because it is a security-model decision.
- **No-gain fallback is weaker.** If the Web Audio graph can't be built (`_ensureAnalyser()` fails — e.g. no `<audio>` element found, or a browser blocks `createMediaElementSource`), the hard stop falls back to `setVolume(0)` plus pausing slightly early. In simulation this path can still overrun by up to ~200ms with a very slow pause. The first round of a session may use this path until the SDK's `<audio>` element exists.
- **Dedupe is heuristic.** It can occasionally merge two genuinely different songs (e.g. a title containing a word like "Live" after a dash) or miss a version whose title differs completely. Spotify's Feb 2026 API may omit `external_ids`/`popularity`/`linked_from`; the code uses them only if present. Rules live in `VERSION_NOISE` — adjust there.
- **The playlist-card count is the raw Spotify total** (includes duplicates, local files, podcasts). The 50-song rule is therefore re-enforced on the real deduped pool when the game starts; a playlist that fails is hidden for the rest of the session with a toast.
- **Liked Songs load is capped at 200 tracks** (unchanged), so a huge Liked list is sampled from the most recent 200.
- **Unescaped HTML:** track names/artists are inserted with `innerHTML` in suggestions, reveal, and results (playlist names are now escaped). Low risk (data comes from Spotify) but should be wrapped in `escapeHtml()`.
- **Leaderboard "Clear"** still only clears `tl_leaderboard`; it does not clear the new cache (`tl_lb_cache`) or unsent queue (`tl_lb_pending`).
- `STATE.likedCount` and `STATE.ineligible` are not reset on disconnect/reconnect (a page reload does).

---

## 1. What It Is

A Songless-style music guessing game. The player connects their **Spotify Premium** account, picks one of their own playlists (or their Liked Songs), and guesses songs from progressively longer clips. Audio is streamed in-browser via the **Spotify Web Playback SDK**.

- Single file: `index.html`. No build system, no dependencies to install.
- External resources loaded at runtime: Google Fonts (Space Grotesk, Space Mono) and the Spotify Web Playback SDK (`https://sdk.scdn.co/spotify-player.js`).
- Spotify auth tokens cached in `localStorage`. Leaderboard stored globally in **Supabase** (with `localStorage` as offline fallback).
- **Deployed at:** `https://cay919.github.io/`

**Current status:** all features implemented; the v6 fixes (§0) await live verification. Auth, playlist loading, Liked Songs, clip playback, guessing, scoring, sounds, seeking, leaderboard, silence detection, and navigation are in place.

---

## 2. How to Run

### Production (GitHub Pages)
Visit `https://cay919.github.io/` — no setup needed.

### Local development
1. Open PowerShell (or any terminal).
2. `cd` into the project folder.
3. `python -m http.server 8888 --bind 127.0.0.1`
4. Open: `http://127.0.0.1:8888/index.html`
5. To stop: `Ctrl + C`.

**Critical for local dev:** Use `127.0.0.1`, **not** `localhost` or `[::1]`. The Spotify SDK's WebSocket fails on IPv6 loopback. Also add `http://127.0.0.1:8888/index.html` as an additional Redirect URI in your Spotify Developer dashboard if you want local testing to work.

---

## 3. Spotify Developer App Setup

Dashboard: https://developer.spotify.com/dashboard

- **Client ID (hardcoded):** `2b41a394f163478586d9241cbbf8158f`
- **Redirect URI (production):** `https://cay919.github.io/`
- **Redirect URI (local dev, optional):** `http://127.0.0.1:8888/index.html`
- **Web Playback SDK** must be checked under "APIs used."
- **User Management:** every person who will play must be added with their **exact name + Spotify email**. In Development Mode, a non-allowlisted user can log in but gets **403** on API calls.
- **App owner must have Spotify Premium** (a 2026 dev-mode requirement). Each listener also needs Premium for the Web Playback SDK to produce audio.

### Scopes requested (exact string in `startAuth`)
```
user-read-private user-read-email user-library-read
playlist-read-private playlist-read-collaborative
streaming user-modify-playback-state user-read-playback-state
```
**If you change scopes, the user must disconnect & reconnect** — cached tokens don't gain new scopes.

---

## 4. ⚠️ The February 2026 Spotify API Migration (essential context)

Spotify migrated existing development-mode apps to a much stricter API (enforced **March 9, 2026**). The current build is written for the **post-migration** API. Treat pre-2026 tutorials and Stack Overflow answers as outdated.

| Change | Impact | How the code handles it |
|---|---|---|
| `GET /playlists/{id}/tracks` **renamed** to `GET /playlists/{id}/items` | Old path 403s | `loadPlaylistTracks` calls `/items` |
| Playlist item's `track` field **renamed** to `item` | Track data missing | Normalizer reads `i.item \|\| i.track` |
| `GET /me` no longer returns `product`, `email`, `country`, `followers` | Premium check misfired | `loadUser` only warns if `product` is present |
| Playlist **contents readable only for playlists you own or collaborate on** | Followed playlists 403 | Grid filtered to owned + collaborative only |
| `GET /me/tracks` needs `user-library-read` | Liked Songs 403'd | Scope added |
| Extended Quota Mode requires registered business (250k+ MAU) | No path out of dev-mode | App lives within dev-mode constraints (see §12) |

---

## 5. Authentication Flow (PKCE + refresh)

**The Client ID is hardcoded** — "Connect with Spotify" immediately kicks off the OAuth redirect, no setup needed from the player.

1. User clicks "Connect with Spotify" → `startAuth(HARDCODED_CLIENT_ID)` called directly.
2. PKCE verifier + challenge generated; verifier saved to both `sessionStorage` and `localStorage`.
3. Redirect to Spotify → user approves → returned to `https://cay919.github.io/?code=...`.
4. `init()` detects the code → `exchangeCode(code)` → stores access token **and** refresh token.
5. `?code=` removed from URL to prevent re-exchange on refresh.

Token refresh is fully implemented. `getValidToken()` guarantees a token valid for ~60s, refreshing silently when needed. Sessions survive past the 1-hour expiry with no manual reconnect.

### Hardcoded config constants
At the top of the `<script>` block:
```javascript
const HARDCODED_CLIENT_ID = '2b41a394f163478586d9241cbbf8158f';
const SUPABASE_URL  = 'https://naslliaqhilrcepgcmvo.supabase.co';
const SUPABASE_ANON = '<publishable key>';   // sb_publishable_...
```
`HARDCODED_CLIENT_ID` is used in three places: `btnConnect` handler, `exchangeCode`, and `refreshAccessToken`.

### Full auth reset (browser console)
```javascript
['tl_token','tl_token_exp','tl_client_id','tl_redirect_uri','tl_verifier','tl_refresh']
  .forEach(k => localStorage.removeItem(k)); sessionStorage.clear();
```
`tl_leaderboard`, `tl_lb_cache`, and `tl_lb_pending` are intentionally not cleared by this.

---

## 6. localStorage Keys

| Key | Purpose |
|---|---|
| `tl_token` | Spotify access token |
| `tl_token_exp` | Access token expiry timestamp (ms) |
| `tl_refresh` | Refresh token (PKCE) — enables silent re-auth |
| `tl_client_id` | Spotify Client ID (hardcoded fallback also exists) |
| `tl_redirect_uri` | Redirect URI used at auth time |
| `tl_verifier` | PKCE verifier (also mirrored in sessionStorage) |
| `tl_leaderboard` | JSON array — this device's own scores, used as an offline fallback |
| `tl_lb_cache` | `{ts, total, rows}` — last **successful** fetch of the global leaderboard; shown when Supabase is unreachable |
| `tl_lb_pending` | JSON array — scores that failed to upload; retried automatically on next sign-in / leaderboard open |

---

## 7. Game Logic & Scoring

- **Duration tiers (seconds):** `[0.5, 1, 5, 10, 20]`
- **Points by tier:** `{0.5:5, 1:4, 5:3, 10:2, 20:1}` — guess earlier, score more.
- **Rounds per game:** 10. **Max score:** 50.
- **Wrong-guess penalty:** −1 per wrong submission, floored at 0.
- **Track pool:** every playable track in the chosen source, **deduplicated by song** (one entry per song across remasters/remixes/live/etc. — §10.3). Local files, podcast episodes, and null entries filtered out; only `spotify:track:` URIs kept.
- **Minimum size:** a source needs **≥ 50 unique playable songs** (`MIN_PLAYLIST_SONGS`). Smaller playlists are hidden from the selection screen (§10.4).
- **Autocomplete** searches the **entire playlist pool** (`STATE.allTracks`), not just the 10 in play.

### Round flow
1. Round loads → silence scan runs (see §10.2) → first clip auto-plays from the detected audio onset.
2. **Skip** → next tier, no penalty; playback picks up from the previous clip's end position.
3. **Wrong guess** → −1, advance to next tier with pickup continuation.
4. **Correct guess** → award tier points, reveal.
5. **Skip/wrong on final tier (20s)** → round lost, reveal.
6. **Reveal** → album art shown, 30s of the track (from onset) replayable/scrubbable.

---

## 8. Feature Reference

1. **Sound effects** — synthesized via Web Audio API; no audio files.
2. **Wrong-guess continuation** — keeps playing with more time unlocked, small penalty.
3. **30s reveal playback** — full clip replayable and seekable after any round ends.
4. **Click-to-seek** — waveform click jumps within the current tier window.
5. **Pickup continuation** — skipping/advancing resumes from the previous tier's end.
6. **Global leaderboard** — scores stored in Supabase, visible to all players. Falls back to localStorage if offline. Rows belonging to the current Spotify account are highlighted.
7. **Home / Back navigation** — logo = Home; back arrow goes one screen up. Both stop playback.
8. **Liked Songs** — always shown first; always playable.
9. **No-setup connect flow** — Client ID hardcoded; friends just click and authorize.
10. **Silence detection** — before each round's first clip, the game scans the track muted to find where audio actually begins, then offsets all clip windows to that position (see §10.2).

---

## 9. Architecture & Key Functions

Single file: `<style>`, screen markup, one `<script>`.

### 9.1 Auth & networking
| Function | Purpose |
|---|---|
| `init()` | Entry point. Checks URL for `?code=`, else uses/refreshes cached token. |
| `startAuth(clientId)` | Begins PKCE OAuth. Called with `HARDCODED_CLIENT_ID`. |
| `exchangeCode(code)` | Exchanges auth code for access + refresh tokens. Falls back to `HARDCODED_CLIENT_ID`. |
| `refreshAccessToken()` | Uses `tl_refresh` to silently get a new token. Falls back to `HARDCODED_CLIENT_ID`. |
| `getValidToken()` | Returns a token valid ~60s, refreshing if needed. |
| `spFetch(path, opts)` | Wraps all Spotify Web API calls; injects token; retries once on 401. |
| `fullDisconnect()` | Clears all auth state and disconnects the SDK player. |

### 9.2 Spotify SDK & playback engine
| Function | Purpose |
|---|---|
| `initSpotifyPlayer()` | Creates `Spotify.Player`; only called after a confirmed token. |
| `playUri(uri, positionMs)` | `PUT /me/player/play?device_id=…`; retries on 403/404 after `ensureDevice`. |
| `ensureDevice()` | Transfers playback to the SDK device so it becomes active. |
| `playClip(startSec)` | Plays the current tier's clip, offset by `STATE.soundOnsetMs`. |
| `startClipWatch(startSec, limit)` | Timing engine — anchors to the SDK's confirmed position, then runs the 20ms watchdog that enforces the deadline (§10.1). |
| `triggerStop(gen)` | Idempotent per clip generation; called by the watchdog or the SDK state listener. |
| `endClip()` | Mutes, pauses until confirmed, resets UI. |
| `hardPause(gen)` | Pauses and retries (up to 15×) until SDK confirms stopped; aborts if a newer clip has started. |
| `stopPlayback()` | Increments `clipGen`, clears timers, mutes, `hardPause`. |
| `silenceNow()` / `restoreSound()` | Instant GainNode mute (fallback: `setVolume(0)`) and its undo; `playClip`/`seekTo` call `restoreSound()`. |
| `seekTo(targetSec)` | Waveform-click seek, offset by `STATE.soundOnsetMs`. |

### 9.3 Silence detection
| Function | Purpose |
|---|---|
| `detectSoundOnset(uri)` | Scans track muted via Web Audio AnalyserNode; returns first ms with real audio. |
| `_ensureAnalyser()` | Connects the SDK's `<audio>` element to an AnalyserNode (once per element). |
| `_getRms()` | Reads frequency energy from the analyser (0–255 scale). |

### 9.4 Playlists & tracks
| Function | Purpose |
|---|---|
| `loadPlaylists()` | `GET /me/playlists`; filters to owned + collaborative; then `resolveTrackCounts()`. |
| `resolveTrackCounts()` / `fetchTrackTotal(id)` | Fetches the item total (`limit=1`) for sources whose count is missing, 4 at a time. |
| `_plCount(pl)` / `_eligible(id, count)` | Read a source's count / apply the 50-song rule (unknown counts stay visible and are enforced at start). |
| `dedupeTracks(tracks)` / `normalizeTitle()` / `versionPenalty()` | Song-level deduplication (§10.3). Pure functions between `// <dedupe>` markers — easy to unit test. |
| `loadPlaylistTracks(source)` | Paginates `/me/tracks` or `/playlists/{id}/items`; normalizes item shape. |
| `loadUser()` | `GET /me`; sets user chip; guarded Premium check. |

### 9.5 Round flow & UI
| Function | Purpose |
|---|---|
| `startGame()` | Loads tracks, runs `dedupeTracks`, enforces the 50-song minimum, shuffles, picks 10 rounds. |
| `loadRound()` | Resets state, runs silence scan, then calls `playClip()`. |
| `revealRound(correct, pts)` | Shows result banner, answer card, enters 30s reveal mode. |
| `showResults()` | End screen; submits score to Supabase; renders leaderboard. |
| `showLeaderboardOnly()` | Opens leaderboard from playlist screen with no game summary. |

### 9.6 Leaderboard (Supabase)
| Function | Purpose |
|---|---|
| `sbHeaders()` / `sbRequest()` | Correct auth headers per key type; `fetch` with an 8s timeout. |
| `sbInsert(entry)` | `POST /rest/v1/scores`. Returns the Supabase UUID, or `null` on failure (caller queues it). |
| `flushPendingScores()` | Re-uploads everything in `tl_lb_pending`; keeps what still fails. |
| `sbUpdateName(id, name)` | `PATCH` the display name; logs a warning if RLS matched 0 rows (see Known issues). |
| `sbFetchLeaderboard()` | Top 100 by score + total row count (`Prefer: count=exact`). Returns `{rows,total}` or `{error}`; one retry on network/5xx. |
| `mergeEntries(...lists)` | Combines lists, dedupes on `ts|score`, sorts, caps at 100. |
| `renderLeaderboard()` | Paints the cached board instantly, fetches fresh, caches it on success; on failure shows cache + device scores + reason + Retry button. |
| `loadLeaderboard()` / `saveLeaderboard(arr)` | This device's own scores (`tl_leaderboard`). |

Score row shape (Supabase): `{ id (uuid), name, score, max, pct, playlist, spotify_id, ts }`.

---

## 10. Deep Dives

### 10.1 Playback Timing Engine / Hard Stop (read before touching playback)

**Why clips overran (v5 root causes):** the stop was a single `setTimeout` measured from a polled "playing" sample; `pause()` then took 100–300ms to take effect (audio kept playing, which dominates a 0.5s clip); a late timer, a buffering stall, or a pause issued while a play was settling could each push the stop later; and a leftover pause retry could even hit the *next* clip.

v6 enforces the limit in four layers:

1. **Position-based deadline.** The stop point is `soundOnsetMs + limit·1000` as an *absolute track position*, so start latency can't move it. `startClipWatch` anchors to the SDK's confirmed position (sampled mid-request) once it reports playing.
2. **20ms watchdog** (`STATE.clipWatchdog`) instead of one timer. Every ~200ms it re-reads the SDK position and only ever moves the deadline **earlier** if playback is ahead of the estimate — it never extends a clip. `player_state_changed` also checks the deadline as a safety net.
3. **Instant mute, then pause.** Audio is routed `<audio> → GainNode → speakers` (the analyser taps *before* the gain). At the deadline `silenceNow()` sets gain to 0 synchronously, so nothing audible leaks while the SDK processes `pause()`. If the graph is unavailable, it falls back to `setVolume(0)` and issues the stop `pauseRttMs/2` early (smoothed pause round-trip, ≤150ms).
4. **Pause until confirmed, generation-guarded.** `hardPause(gen)` retries up to 15× (40ms apart) until `getCurrentState().paused`, and aborts if `clipGen` has moved on. `triggerStop(gen)` is idempotent so the watchdog, state listener, and resync can't double-fire.

Sound is restored by `restoreSound()` at the start of `playClip` and after `seekTo`'s resume. **If you add a new way to start playback, call `restoreSound()` or it will be silent.** The priming play at the start of `loadRound` is deliberately muted so the first ~100ms of the track at position 0 is never heard.

All positions passed to `playUri` and `player.seek` are **absolute** (`STATE.soundOnsetMs + clip-relative ms`).

**Testing aid:** the engine can be exercised headlessly with a fake player (start lag + pause lag). Worst-case simulated overrun with the gain mute was ≤10ms; without it ~200ms.

### 10.2 Silence Detection (read before touching loadRound)

Some tracks have silent intros that would make the 0.5s and 1s tiers unplayable. Before the first clip of each round:

1. `loadRound` calls `playUri` at position 0 to prime the SDK.
2. `detectSoundOnset` mutes the player (`setVolume(0)`), then seeks forward in **500ms steps**, reading RMS energy from a Web Audio `AnalyserNode` tapped onto the SDK's hidden `<audio>` element.
3. When RMS exceeds the threshold (6/255), it backs up 250ms and returns that as `STATE.soundOnsetMs`.
4. Volume is restored, player is paused, and `playClip()` is called normally — all subsequent clip offsets, seeks, and reveal playback are relative to `soundOnsetMs`.

**Fallback:** any error in the scan leaves `soundOnsetMs = 0` and the game plays from position 0 as before. The scan is capped at 8 seconds (16 × 500ms steps).

**`MediaElementSourceNode` constraint:** a browser throws if you try to wrap the same `<audio>` element in a `MediaElementSourceNode` twice. `_ensureAnalyser` guards against this by reusing the existing node.

### 10.3 Song Deduplication (read before touching the pool)

`dedupeTracks(tracks)` runs in `startGame()` on everything loaded from the source. Two tracks are the **same song** if any holds:
- same Spotify `id` (or `linked_from.id`);
- same ISRC, when Spotify supplies `external_ids.isrc`;
- same **normalised title** and at least one **shared artist id**.

`normalizeTitle()` lower-cases, strips diacritics, removes bracketed segments and trailing ` - …` segments that contain version words (`VERSION_NOISE`: remaster, remix, live, acoustic, demo, mono/stereo, single/album/radio/extended/club/dub/instrumental/edit/bonus/deluxe/anniversary/session/version/mix/take, feat/ft/with/from, a bare year…), then keeps only letters/digits (Unicode-aware, so Japanese/Korean/etc. titles don't collapse). Meaningful parts such as `(Part 2)` are kept, and covers by *different* artists are kept as separate songs.
Which copy survives: the one with the lowest `versionPenalty()` (cleanest title), then higher `popularity` if present, then earliest in the playlist.

### 10.4 Minimum Playlist Size

`MIN_PLAYLIST_SONGS = 50`. Two checkpoints:
1. **Selection screen:** `renderPlaylists()` hides any source whose known count is <50 (and Liked Songs if <50). Counts come from the playlist object, or from `resolveTrackCounts()` (a `limit=1` request) when missing. Sources with an unknown count stay visible.
2. **Game start:** after load + dedupe, if unique playable songs <50 the game doesn't start, the source is added to `STATE.ineligible`, hidden, and a toast explains. This catches playlists padded with duplicates/local files/podcasts.

---

## 11. Supabase Setup

### Project details
- **Project name:** Tuneless
- **Project URL:** `https://naslliaqhilrcepgcmvo.supabase.co`
- **Dashboard:** https://supabase.com/dashboard
- **Anon/publishable key:** stored in `SUPABASE_ANON` constant in `index.html` (starts with `sb_publishable_...`). Safe to embed — RLS prevents abuse.
- **Secret key:** never embed in the HTML. Server-side use only.

### Database table
```sql
create table scores (
  id          uuid primary key default gen_random_uuid(),
  name        text not null default 'Player',
  score       int  not null,
  max         int  not null,
  pct         int  not null,
  playlist    text,
  spotify_id  text,
  ts          bigint not null
);
alter table scores enable row level security;
create policy "public read"   on scores for select using (true);
create policy "public insert" on scores for insert with check (true);
create policy "owner update"  on scores for update
  using (spotify_id = current_setting('request.jwt.claims', true)::json->>'sub');
```

### RLS policy summary
- **Anyone** can read all scores (global leaderboard is public).
- **Anyone** can insert a new score (no auth required to submit).
- **Only the score's owner** (matched by `spotify_id`) can update their row — enforced by Supabase JWT claims. In practice, name edits work via the anon key because Supabase allows the PATCH when the `spotify_id` matches via the policy.
- **Nobody** can delete rows from the client (no delete policy).

### Free-tier pausing & keep-alive
Free projects are paused after ~7 days of no API activity; while paused all requests fail. Mitigations: (a) `.github/workflows/supabase-keepalive.yml` pings the API Mondays/Thursdays; (b) the client keeps `tl_lb_cache` (last good board) and `tl_lb_pending` (unsent scores) so the board is still shown and scores aren't lost while the server is down. GitHub disables scheduled workflows after 60 days of repo inactivity — re-enable in the Actions tab if needed. Upgrading to a paid plan removes auto-pause entirely.

### API key header
`sb_publishable_…` keys go in the `apikey` header **only**. Do not send them as `Authorization: Bearer` (they aren't JWTs). Legacy `eyJ…` anon keys use both. Handled in `sbHeaders()`.

### Supabase setup checklist (if rebuilding from scratch)
1. Create a new project at supabase.com.
2. In Settings → Security: enable **Data API**, leave automatic RLS off (policies are written manually).
3. Run the SQL above in the SQL Editor.
4. Copy the **Project URL** (no trailing path) and the **Publishable key** into `index.html`.

---

## 12. Common Bugs & Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Page shows **404** on GitHub Pages | File not named `index.html` | Must be `index.html` at repo root |
| Page shows **404** locally | Wrong folder or misnamed file | `cd` into project folder, check filename |
| SDK WebSocket fails | Served over `localhost` or `[::1]` | Use `http://127.0.0.1:8888/...` |
| **403 on `/me/player/play`** | Device not active / user not allowlisted / not Premium | Transfer-then-retry handles (a); add user in Spotify User Management for (b); Premium required for (c) |
| **"No playable tracks"** | Followed/editorial playlist or all-local files | Pick a playlist you own, or Liked Songs |
| **403 on `/me/tracks`** | Missing `user-library-read` scope | Disconnect & reconnect |
| Session drops after ~1 hour | (Old issue) | Fixed — refresh tokens implemented |
| Track plays past time limit | (Recurring since v1) pause latency + late timer | **v6 rebuild** — position deadline, watchdog, instant gain mute (§10.1). Needs live confirmation |
| **`redirect_uri_mismatch`** | URI mismatch in Spotify dashboard | Ensure `https://cay919.github.io/` registered exactly (trailing slash matters) |
| **Leaderboard shows local scores only / "Couldn't reach the global leaderboard (…)"** | Project **paused** (most common after inactivity), offline, or key/RLS problem | Read the reason in the note + console. Dashboard → Restore project if paused; `HTTP 401/403` → check key and RLS policies. Add the keep-alive workflow |
| **Playlist missing from the picker** | Under 50 songs (or <50 after de-duplication), or not owned/collaborative | By design (§10.4). Add songs, or check the dev console log `Track pool: …` |
| **Clip still audible past its limit** | Gain graph not built (fallback path) or SDK not confirming pause | Console: look for `Silence detection: analyser setup failed` / `hardPause: SDK never confirmed paused`. See Known issues in §0 |
| **No sound at all after a stop** | A new playback path didn't call `restoreSound()` | Call `restoreSound()` before starting playback (§10.1) |
| **Score not appearing for others** | RLS policy missing or table not created | Re-run the SQL setup; check Supabase → Authentication → Policies |
| **Silence scan delays every round** | Expected — scan runs once per round before first clip | Normal; status bar shows "Scanning for audio start…" |
| **Scan finds wrong onset** | Analyser not ready on very first round | Rare; `soundOnsetMs` falls back to 0 gracefully |

---

## 13. Known Constraints & Limitations

- **Development Mode caps:** up to 25 users (legacy apps may be grandfathered). App owner must have Premium. Extended Quota Mode unavailable without a registered business.
- **Each new player must be added** to Spotify User Management with exact name + email before they can play.
- **Playable sources:** only playlists the user owns/collaborates on, plus Liked Songs.
- **Premium required** for any audio.
- **0.5s precision:** shortest clip is best-effort due to playback start latency.
- **Leaderboard "Clear"** only removes this device's `tl_leaderboard` — it cannot delete global Supabase rows (by design) and does not clear `tl_lb_cache` / `tl_lb_pending`.
- **Supabase free tier auto-pauses** after ~7 days idle (§11).
- **Silence scan adds ~1–3s** to round load time for songs with silent intros; songs without silent intros are detected quickly (usually 1 step).
- **Token lifetime:** 1 hour; refresh is automatic while `tl_refresh` exists.

---

## 14. History of Resolved Issues (chronological)

- `response_type must be code` — implicit grant killed by Spotify → switched to PKCE.
- `code_verifier was invalid` — verifier now stored in sessionStorage **and** localStorage.
- `No playlists found` — `tracks.total` unreliable → filter on `p.id`.
- Preview URLs null — Spotify killed 30s previews → switched to Web Playback SDK.
- `Invalid token scopes` from SDK — added `user-read-email`.
- WebSocket failures — switched server bind from `[::1]` to `127.0.0.1`.
- **403 on `/me/player/play`** — fixed with `?device_id=`, transfer-then-play retry, `activateElement`, token refresh.
- **Feb/Mar 2026 API migration** — `/tracks`→`/items`, `track`→`item`, `/me` field removals, owned-only playlist reads, `user-library-read`. (See §4.)
- **"No playable tracks"** — swallowed 403 on followed playlists → grid filtered; errors surfaced.
- **Autocomplete too easy** — now searches full pool, not just 10 in-play tracks.
- **Suggestions cut off** — dropdown opens upward, scrolls, shows match count.
- **Token expiry every hour** — refresh tokens implemented.
- **Clip overrun** — rebuilt timing engine; shortest tier moved 0.1s → 0.5s.
- **Client ID popup friction** — Client ID hardcoded; connect goes directly to OAuth.
- **Deployed to GitHub Pages** — renamed `songless.html` → `index.html`; redirect URI updated.
- **Silent intro problem** — songs with silent intros made short tiers unplayable → silence detection scans track muted before each round and offsets all clips to the audio onset.
- **Per-device leaderboard** — scores were local only → replaced with Supabase global leaderboard; all players share one ranked table; local cache used as fallback.
- **v6 — Clip overrun (long-standing)** — replaced single-timer stop with position-based deadline + 20ms watchdog + GainNode instant mute + generation-guarded `hardPause`. Simulation-verified only.
- **v6 — Duplicate/alternate versions** — `dedupeTracks` collapses remasters/remixes/live/etc. (§10.3).
- **v6 — 50-song minimum** — small playlists hidden; rule re-checked after dedupe (§10.4).
- **v6 — Empty leaderboard after inactivity** — probable Supabase auto-pause; added timeout/retry, correct key header, persistent board cache, unsent-score queue, and a keep-alive workflow. Root cause unconfirmed (§0).

---

## 15. Possible Future Work

- "Resume game" instead of restart when leaving mid-game.
- Difficulty modes (title-only vs title+artist; fewer/more tiers).
- Per-playlist leaderboard filtering.
- Fix leaderboard renames properly (Edge Function or revised RLS) and escape all `innerHTML` inserts.
- Cache loaded tracks so "Play again" doesn't refetch the whole playlist.
- Graceful handling if Spotify tightens dev mode further (Spotify Connect fallback).

---

## 16. Quick Facts Cheat-Sheet

- **Live URL:** `https://cay919.github.io/`
- **File:** `index.html`, single file, ~1,900 lines, no build. Also: `.github/workflows/supabase-keepalive.yml`.
- **Spotify Client ID:** `2b41a394f163478586d9241cbbf8158f` (constant `HARDCODED_CLIENT_ID`)
- **Spotify Redirect URI:** `https://cay919.github.io/` (exact, trailing slash)
- **Supabase URL:** `https://naslliaqhilrcepgcmvo.supabase.co`
- **Supabase key:** publishable/anon key in `SUPABASE_ANON` constant (`sb_publishable_...`)
- Durations `[0.5, 1, 5, 10, 20]`s · Points `{0.5:5,1:4,5:3,10:2,20:1}` · 10 rounds · max 50 · wrong = −1 · **min 50 unique songs per playlist**.
- API endpoints: tracks `GET /playlists/{id}/items`, liked `GET /me/tracks`, play `PUT /me/player/play?device_id=`, transfer `PUT /me/player`, token `https://accounts.spotify.com/api/token`.
- Adding a new player: Spotify Dashboard → your app → Settings → User Management → exact name + email.
- After any scope change: **disconnect & reconnect**.
- **This app targets the post-February-2026 Spotify API.** Pre-2026 docs are outdated.
