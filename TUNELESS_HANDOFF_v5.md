# Tuneless — Project Handoff (v4)

_Last updated: June 2026. This supersedes v3. Reflects the current working build of `index.html` deployed to GitHub Pages, with a global Supabase leaderboard and silence-detection clip offset._

---

## 1. What It Is

A Songless-style music guessing game. The player connects their **Spotify Premium** account, picks one of their own playlists (or their Liked Songs), and guesses songs from progressively longer clips. Audio is streamed in-browser via the **Spotify Web Playback SDK**.

- Single file: `index.html`. No build system, no dependencies to install.
- External resources loaded at runtime: Google Fonts (Space Grotesk, Space Mono) and the Spotify Web Playback SDK (`https://sdk.scdn.co/spotify-player.js`).
- Spotify auth tokens cached in `localStorage`. Leaderboard stored globally in **Supabase** (with `localStorage` as offline fallback).
- **Deployed at:** `https://cay919.github.io/`

**Current status: fully working.** Auth, playlist loading, Liked Songs, clip playback, guessing, scoring, sounds, seeking, global leaderboard, silence detection, and home/back navigation all function.

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
`tl_leaderboard` (local cache) is intentionally not cleared by this.

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
| `tl_leaderboard` | JSON array — local cache of scores, used as offline fallback |

---

## 7. Game Logic & Scoring

- **Duration tiers (seconds):** `[0.5, 1, 5, 10, 20]`
- **Points by tier:** `{0.5:5, 1:4, 5:3, 10:2, 20:1}` — guess earlier, score more.
- **Rounds per game:** 10. **Max score:** 50.
- **Wrong-guess penalty:** −1 per wrong submission, floored at 0.
- **Track pool:** every playable track in the chosen source (deduped by id). Local files, podcast episodes, and null entries filtered out; only `spotify:track:` URIs kept.
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
| `startClipWatch(startSec, limit)` | Timing engine — polls SDK until confirmed playing, then starts timer. |
| `endClip()` | Stops at the limit, resets UI. |
| `hardPause()` | Pauses and retries until SDK confirms stopped (kills overrun race). |
| `stopPlayback()` | Increments `clipGen` (invalidates in-flight watch) + `hardPause`. |
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
| `loadPlaylists()` | `GET /me/playlists`; filters to owned + collaborative. |
| `loadPlaylistTracks(source)` | Paginates `/me/tracks` or `/playlists/{id}/items`; normalizes item shape. |
| `loadUser()` | `GET /me`; sets user chip; guarded Premium check. |

### 9.5 Round flow & UI
| Function | Purpose |
|---|---|
| `startGame()` | Loads + dedupes + shuffles tracks; picks 10 rounds. |
| `loadRound()` | Resets state, runs silence scan, then calls `playClip()`. |
| `revealRound(correct, pts)` | Shows result banner, answer card, enters 30s reveal mode. |
| `showResults()` | End screen; submits score to Supabase; renders leaderboard. |
| `showLeaderboardOnly()` | Opens leaderboard from playlist screen with no game summary. |

### 9.6 Leaderboard (Supabase)
| Function | Purpose |
|---|---|
| `sbInsert(entry)` | `POST /rest/v1/scores` — inserts a new score row. Returns the Supabase UUID. |
| `sbUpdateName(id, name)` | `PATCH /rest/v1/scores?id=eq.{id}` — updates the display name live. |
| `sbFetchLeaderboard()` | `GET /rest/v1/scores` — fetches top 100 scores sorted by score desc. |
| `renderLeaderboard()` | Fetches remote scores, falls back to local, renders rows with current user highlighted. |
| `loadLeaderboard()` / `saveLeaderboard(arr)` | localStorage helpers used as offline fallback and local write-cache. |

Score row shape (Supabase): `{ id (uuid), name, score, max, pct, playlist, spotify_id, ts }`.

---

## 10. Deep Dives

### 10.1 Playback Timing Engine (read before touching playback)

Three ideas that make clip timing reliable:

1. **Time from when playback ACTUALLY begins.** `startClipWatch` polls `getCurrentState()` until the SDK reports it's truly playing at/after the requested absolute position, then anchors timing to that confirmed start. The REST play call returns before audio actually starts — timing from the request causes short clips to overrun.

2. **Retry the pause until confirmed.** `hardPause()` issues `pause()`, checks state, and repeats up to 7× until the SDK confirms `paused`. Fixes the race where a pause issued mid-settle gets silently ignored.

3. **Generation guard.** Every new clip increments `STATE.clipGen`. Any older `startClipWatch` mid-`await` checks the generation and bails, preventing overlapping clips from fighting.

All positions passed to `playUri` and `player.seek` are **absolute** (`STATE.soundOnsetMs + clip-relative ms`). `startClipWatch` converts the confirmed absolute `beginPos` back to clip-relative seconds for the UI progress display.

### 10.2 Silence Detection (read before touching loadRound)

Some tracks have silent intros that would make the 0.5s and 1s tiers unplayable. Before the first clip of each round:

1. `loadRound` calls `playUri` at position 0 to prime the SDK.
2. `detectSoundOnset` mutes the player (`setVolume(0)`), then seeks forward in **500ms steps**, reading RMS energy from a Web Audio `AnalyserNode` tapped onto the SDK's hidden `<audio>` element.
3. When RMS exceeds the threshold (6/255), it backs up 250ms and returns that as `STATE.soundOnsetMs`.
4. Volume is restored, player is paused, and `playClip()` is called normally — all subsequent clip offsets, seeks, and reveal playback are relative to `soundOnsetMs`.

**Fallback:** any error in the scan leaves `soundOnsetMs = 0` and the game plays from position 0 as before. The scan is capped at 8 seconds (16 × 500ms steps).

**`MediaElementSourceNode` constraint:** a browser throws if you try to wrap the same `<audio>` element in a `MediaElementSourceNode` twice. `_ensureAnalyser` guards against this by reusing the existing node.

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
| Track plays past time limit | (Old issue) pause race | Fixed — `hardPause` retry + start-anchored timing |
| **`redirect_uri_mismatch`** | URI mismatch in Spotify dashboard | Ensure `https://cay919.github.io/` registered exactly (trailing slash matters) |
| **Leaderboard shows local scores only** | Supabase unreachable or key wrong | Check `SUPABASE_URL` / `SUPABASE_ANON` constants; check Supabase dashboard for errors |
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
- **Leaderboard "Clear"** only removes the local cache — it cannot delete global Supabase rows (by design).
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

---

## 15. Possible Future Work

- "Resume game" instead of restart when leaving mid-game.
- Difficulty modes (title-only vs title+artist; fewer/more tiers).
- Per-playlist leaderboard filtering.
- Graceful handling if Spotify tightens dev mode further (Spotify Connect fallback).

---

## 16. Quick Facts Cheat-Sheet

- **Live URL:** `https://cay919.github.io/`
- **File:** `index.html`, single file, ~1,420 lines, no build.
- **Spotify Client ID:** `2b41a394f163478586d9241cbbf8158f` (constant `HARDCODED_CLIENT_ID`)
- **Spotify Redirect URI:** `https://cay919.github.io/` (exact, trailing slash)
- **Supabase URL:** `https://naslliaqhilrcepgcmvo.supabase.co`
- **Supabase key:** publishable/anon key in `SUPABASE_ANON` constant (`sb_publishable_...`)
- Durations `[0.5, 1, 5, 10, 20]`s · Points `{0.5:5,1:4,5:3,10:2,20:1}` · 10 rounds · max 50 · wrong = −1.
- API endpoints: tracks `GET /playlists/{id}/items`, liked `GET /me/tracks`, play `PUT /me/player/play?device_id=`, transfer `PUT /me/player`, token `https://accounts.spotify.com/api/token`.
- Adding a new player: Spotify Dashboard → your app → Settings → User Management → exact name + email.
- After any scope change: **disconnect & reconnect**.
- **This app targets the post-February-2026 Spotify API.** Pre-2026 docs are outdated.
