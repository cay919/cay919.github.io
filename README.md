Readme
Tuneless — Project Handoff (v3)
Last updated: June 2026. This supersedes v2. It reflects the current working build of `index.html` (formerly `songless.html`), deployed to GitHub Pages.
---
1. What It Is
A Songless-style music guessing game. The player connects their Spotify Premium account, picks one of their own playlists (or their Liked Songs), and guesses songs from progressively longer clips. Audio is streamed in-browser via the Spotify Web Playback SDK.
Single file: `index.html`. No build system, no dependencies to install.
External resources loaded at runtime: Google Fonts (Space Grotesk, Space Mono) and the Spotify Web Playback SDK (`https://sdk.scdn.co/spotify-player.js`).
State persists in the browser via `localStorage` (auth tokens + leaderboard).
Deployed at: `https://cay919.github.io/`
Current status: fully working. Auth, playlist loading, Liked Songs, clip playback, guessing, scoring, sounds, seeking, the leaderboard, and home/back navigation all function. The remaining constraints are imposed by Spotify (dev-mode limits), not bugs — see §12.
---
2. How to Run
Production (GitHub Pages)
Visit `https://cay919.github.io/` — no setup needed. The file is served as `index.html` so GitHub Pages serves it at the root URL directly.
Local development
Open PowerShell (or any terminal).
`cd` into the project folder.
`python -m http.server 8888 --bind 127.0.0.1`
Open: `http://127.0.0.1:8888/index.html`
To stop: `Ctrl + C`.
Critical for local dev: Use `127.0.0.1`, not `localhost` or `[::1]`. The Spotify SDK's WebSocket fails on IPv6 loopback. You must also add `http://127.0.0.1:8888/index.html` as an additional Redirect URI in your Spotify Developer dashboard if you want local testing to work (the production GitHub Pages URI is already registered).
---
3. Spotify Developer App Setup
Dashboard: https://developer.spotify.com/dashboard
Client ID (hardcoded): `2b41a394f163478586d9241cbbf8158f`
Redirect URI (production): `https://cay919.github.io/`
Redirect URI (local dev, optional): `http://127.0.0.1:8888/index.html`
Web Playback SDK must be checked under "APIs used."
User Management: every person who will play must be added with their exact name + Spotify email. In Development Mode, a non-allowlisted user can sometimes log in but then gets 403 on API calls. This is the most common cause of a 403 for a new player when the app owner works fine.
App owner must have Spotify Premium (a 2026 dev-mode requirement). Each listener also needs Premium for the Web Playback SDK to produce audio.
Scopes requested (exact string in `startAuth`)
```
user-read-private user-read-email user-library-read
playlist-read-private playlist-read-collaborative
streaming user-modify-playback-state user-read-playback-state
```
`user-library-read` is required for Liked Songs (`/me/tracks`). `streaming` + `user-read-email` are required by the Web Playback SDK. If you change scopes, the user must disconnect & reconnect — cached tokens don't gain new scopes.
---
4. ⚠️ The February 2026 Spotify API Migration (essential context)
Spotify migrated existing development-mode apps to a much stricter API (announced for Feb 2026, enforced on existing apps March 9, 2026). The original Tuneless code predated this and broke. The current build is written for the post-migration API. Future maintainers must keep this in mind — older Spotify tutorials/Stack Overflow answers are now wrong.
What changed and how the code adapts:
Change	Impact	How the code handles it
`GET /playlists/{id}/tracks` renamed to `GET /playlists/{id}/items`	Old path 403s/fails for dev-mode apps	`loadPlaylistTracks` calls `/items`
Playlist item's `track` field renamed to `item`	Track data was missing	Normalizer reads `i.item || i.track`
`GET /me` no longer returns `product`, `email`, `country`, `followers`	Premium check misfired	`loadUser` only warns if `product` is present
Playlist contents readable only for playlists you own or collaborate on	Followed/editorial/algorithmic playlists 403	Playlist grid is filtered to owned + collaborative only
`GET /me/tracks` needs `user-library-read`	Liked Songs 403'd	Scope added
Dev Mode = "surface metadata only," Extended Quota Mode now requires a registered business (250k+ MAU)	No realistic path out of dev-mode limits	App runs within dev-mode constraints (see §12)
Playback control endpoints (`/me/player/*`) still work in dev mode, which is why the game is viable at all.
---
5. Authentication Flow (PKCE + refresh)
Spotify killed the implicit grant, so Tuneless uses Authorization Code with PKCE.
The Client ID is hardcoded — users do not need to paste anything. Clicking "Connect with Spotify" immediately kicks off the OAuth redirect.
User clicks "Connect with Spotify" → `startAuth(HARDCODED_CLIENT_ID)` is called directly.
`startAuth()` generates a PKCE verifier + challenge (`randStr`, `sha`).
The verifier is saved to both `sessionStorage` and `localStorage` (`tl_verifier`) as a redundancy.
Redirect to Spotify → user approves → redirected back to `https://cay919.github.io/?code=...`.
`init()` detects the code and calls `exchangeCode(code)` → stores access token and refresh token.
The `?code=` is removed from the URL so a refresh doesn't re-exchange a used code.
Token refresh is implemented. PKCE returns a `refresh_token`; `refreshAccessToken()` uses it. `getValidToken()` returns a token guaranteed valid for ~60s, refreshing when needed. The SDK's `getOAuthToken` callback and every `spFetch` call route through this, and `spFetch` retries once on 401. Result: sessions survive past the 1-hour token expiry without a manual reconnect.
Hardcoded config constant
At the top of the `<script>` block:
```javascript
const HARDCODED_CLIENT_ID = '2b41a394f163478586d9241cbbf8158f';
```
This constant is used in three places: `btnConnect` handler, `exchangeCode`, and `refreshAccessToken` — all fall back to it if `tl_client_id` is missing from localStorage (e.g. after a manual storage wipe).
Full reset (browser console)
```javascript
['tl_token','tl_token_exp','tl_client_id','tl_redirect_uri','tl_verifier','tl_refresh']
  .forEach(k => localStorage.removeItem(k)); sessionStorage.clear();
```
(Leaderboard lives under `tl_leaderboard` and is intentionally not cleared by this.)
---
6. localStorage Keys
Key	Purpose
`tl_token`	Spotify access token
`tl_token_exp`	Access token expiry timestamp (ms)
`tl_refresh`	Refresh token (PKCE) — enables silent re-auth
`tl_client_id`	Spotify app Client ID (also hardcoded as fallback)
`tl_redirect_uri`	Redirect URI used at auth time
`tl_verifier`	PKCE verifier (also mirrored in sessionStorage)
`tl_leaderboard`	JSON array of completed-game records (see §9.6)
---
7. Game Logic & Scoring
Duration tiers (seconds): `[0.5, 1, 5, 10, 20]`
Points by tier: `{0.5:5, 1:4, 5:3, 10:2, 20:1}` — guess earlier, score more.
Rounds per game: 10. Max score: 50 (5 × 10).
Wrong-guess penalty: −1 per wrong submission, floored at 0.
Track pool: every playable track in the chosen source (deduped by id). Local files, podcast episodes, and removed/null entries are filtered out; only `spotify:track:` URIs are kept.
Autocomplete searches the entire playlist pool (`STATE.allTracks`), not just the 10 songs in play — this is what makes the decoy list large and the game hard. Up to 250 matches render; the dropdown scrolls.
Round flow
Round starts at tier 0 (0.5s); the clip auto-plays.
Skip → advance to the next tier with no penalty; playback continues from where the previous clip ended (pickup), not from 0.
Wrong guess → −1, plus a sound, then advance to the next tier (same pickup continuation). Keep guessing.
Correct guess → award the current tier's points, play the "correct" sound, reveal.
Skip or wrong guess on the final tier (20s) → round lost, reveal.
Reveal (win or lose) → album art shown, answer card displayed, and a full 30s of the track becomes playable/scrubbable before "Next song."
---
8. Feature Reference (everything built on top of the base game)
Sound effects — synthesized via the Web Audio API (`sfxCtx`, `tone`, `playSound`); no audio files. Rising chime on correct, low falling buzz on wrong.
Wrong-guess continuation — see §7. Lets the player keep going through tiers with a small penalty instead of ending the round.
30s reveal playback — after any round ends, the full clip is replayable and seekable.
Click-to-seek — clicking the waveform jumps within the current tier's window. The waveform spans exactly `[0, limit]`, so a click is inherently constrained to the time the player has unlocked.
Pickup continuation — skipping/advancing resumes from the previous tier's end position via `position_ms`, and pressing play replays the whole unlocked window from 0.
Leaderboard — per-browser, stored in `localStorage`. Records name, score, accuracy %, playlist, and completion timestamp. Sorted by score; the latest run is highlighted; the name is editable to tell attempts apart; "View leaderboard" button on the playlist screen; Clear button.
Home / Back navigation — the "tuneless" logo is Home (→ landing); a back arrow goes one screen up (game/results → playlists → landing). Both stop playback. A "Continue to your playlists" button prevents the landing page being a dead end when already connected.
Liked Songs — a synthetic "❤️ Liked Songs" card (`__liked__`) always shown first; always playable since it's the user's own library.
No-setup connect flow — the Client ID is hardcoded. Friends visiting the site just click "Connect with Spotify" and authorize — no developer dashboard required on their end.
---
9. Architecture & Key Functions
Single file, three logical sections: `<style>`, the screen markup (`#screenLogin`, `#screenPlaylists`, `#screenGame`, `#screenResults`, plus the auth modal), and one `<script>`.
9.1 Auth & networking
Function	Purpose
`init()`	Entry point. Checks URL for `?code=`, else uses/refreshes cached token.
`startAuth(clientId)`	Begins PKCE OAuth (uses `randStr`, `sha`). Called with `HARDCODED_CLIENT_ID`.
`exchangeCode(code)`	Exchanges auth code for access + refresh tokens. Falls back to `HARDCODED_CLIENT_ID`.
`refreshAccessToken()`	Uses `tl_refresh` to get a fresh access token. Falls back to `HARDCODED_CLIENT_ID`.
`getValidToken()`	Returns a token valid ~60s, refreshing if needed.
`storeToken`, `getStoredToken`, `getRedirectUri`, `authHeaders`	Token storage + helpers.
`spFetch(path, opts)`	Wraps Spotify Web API calls; injects a valid token; retries once on 401.
`fullDisconnect()`	Clears all auth state (incl. `tl_refresh`) and disconnects the SDK.
9.2 Spotify SDK & playback engine
Function	Purpose
`initSpotifyPlayer()`	Creates `Spotify.Player`; only after a confirmed token.
`playUri(uri, positionMs)`	`PUT /me/player/play?device_id=…`; on 403/404 transfers to the device (`ensureDevice`) and retries. Returns success bool.
`ensureDevice()`	`PUT /me/player` to make the SDK device the active Connect device.
`explainPlaybackError(status, msg)`	Maps Spotify's opaque errors to actionable messages.
`playClip(startSec=0)`	Plays the current tier's clip from `startSec`.
`startClipWatch(startSec, limit)`	The timing engine — see §10.
`endClip()`	Stops at the limit, restores UI.
`hardPause()`	Pauses and retries until the SDK confirms it stopped.
`stopPlayback()`	Invalidates the current clip (generation bump) + hardPause.
`seekTo(targetSec)`	Waveform-click seek within the tier window.
9.3 Playlists & tracks
Function	Purpose
`loadPlaylists()`	`GET /me/playlists`; filters to owned + collaborative; counts hidden ones.
`renderPlaylists()`	Renders the grid; prepends the Liked Songs card; shows the hidden-count note.
`loadPlaylistTracks(source)`	`'__liked__'` → `/me/tracks`; otherwise `/playlists/{id}/items`. Normalizes `item`/`track`, filters to real tracks, paginates.
`loadUser()`	`GET /me`; sets user chip; guarded Premium check.
9.4 Round flow & UI
Function	Purpose
`startGame()`	Loads tracks, dedupes, shuffles 10 rounds, stores full pool in `STATE.allTracks`.
`loadRound()`	Resets per-round state (incl. `reveal`, `clipLoaded`); auto-plays tier 0.
`updateDurationPills()`	Highlights tier progress (all "done" during reveal).
`buildWaveform()`, `updateWaveformProgress(pct)`	The visual bar (60 bars).
`revealRound(correct, pts)`	Reveal banner, answer card, history, enters 30s reveal mode.
`showResults()`	End screen + records leaderboard entry.
`showScreen(name)`, `goHome()`, `goBack()`	Navigation + back/home button visibility.
`showToast(msg, dur)`	Transient status message.
9.5 Sounds
`sfxCtx()` (lazy AudioContext), `tone(freq, offset, dur, type, peak)`, `playSound('correct'|'wrong')`.
9.6 Leaderboard
`loadLeaderboard()`, `saveLeaderboard(arr)`, `renderLeaderboard()`, `showLeaderboardOnly()`, `escapeHtml(s)`.
Entry shape: `{ id, name, score, max, pct, playlist, ts }`. Capped at 200 entries.
---
10. Playback Timing Engine (deep-dive — read before touching playback)
This is the most subtle part of the codebase. It was rebuilt specifically to kill a "the whole track keeps playing" bug. Three ideas:
Time the clip from when playback ACTUALLY begins. After issuing play, `startClipWatch` polls `getCurrentState()` until the SDK reports it's truly playing at/after the requested position, then anchors timing to that real start. This matters because the REST play call returns (HTTP 204) before audio actually starts; timing from the request moment made very short clips (the old 0.1s tier) cut early or miss their stop entirely.
Retry the pause until confirmed. `hardPause()` issues `pause()`, checks `getCurrentState()`, and repeats (up to 7×) until the SDK confirms `paused`. A pause issued while a play command is still settling gets silently ignored — that race is exactly what let a clip run away into the full song.
Generation guard. Every new clip increments `STATE.clipGen`. An older `startClipWatch` that's mid-`await` checks the generation and bails, so overlapping clips (e.g. rapid skip) can't fight each other.
Pickup continuation (§7) is achieved by passing `position_ms` to `playUri` (= previous tier's limit). Browser autoplay policy requires `player.activateElement()` inside a user gesture before audio will play — it's called on Start, on the Play button, and in `playClip`.
Tradeoff to know: the shortest tier is 0.5s (was 0.1s, which was too short to be reliable given start latency). Even at 0.5s there's a small amount of unavoidable imprecision from the gap between requesting playback and audio beginning, but the retry-until-paused logic guarantees it never overruns into the rest of the song.
---
11. Common Bugs & Troubleshooting
Symptom	Likely cause	Fix
Page shows 404 on GitHub Pages	File not named `index.html`	The file must be `index.html` at the repo root for GitHub Pages to serve it at the root URL
Page shows 404 / "File not found" locally	Server started in the wrong folder, or file misnamed	`cd` into the project folder and ensure `index.html` is there
SDK won't connect / WebSocket fails	Served over `localhost` or `[::1]`	Use `http://127.0.0.1:8888/...` for local dev
403 on `/me/player/play`	(a) device not active, (b) user not allowlisted, (c) not Premium	Code transfers-then-retries for (a); for (b) add the user under User Management with exact name+email; for (c) use Premium. Exact reason shows under the play button.
"No playable tracks"	A followed/editorial playlist (403 on item read), or all-local/episode playlist	Pick a playlist you own, or Liked Songs. Followed playlists are hidden from the grid.
403 on `/me/tracks` (Liked Songs)	Missing `user-library-read` scope on the cached token	Disconnect & reconnect to re-authorize with the new scope
Premium warning on a Premium account	`/me` no longer returns `product` (Feb 2026)	Already handled — warning only fires if `product` is present
Track plays past its time limit	Pause raced a still-settling play command	Fixed by `hardPause` retry + start-anchored timing (§10)
Session drops after ~1 hour	(Old issue) no refresh token	Fixed — refresh tokens implemented
Suggestions list cut off at screen bottom	Dropdown opened downward off-screen	Fixed — opens upward, scrolls, shows match count
`redirect_uri_mismatch` error from Spotify	The URI in the dashboard doesn't exactly match what the app sends	Ensure `https://cay919.github.io/` is registered exactly (trailing slash matters). For local dev also add `http://127.0.0.1:8888/index.html`.
---
12. Known Constraints & Limitations (not bugs)
Development Mode caps: up to 25 users (legacy apps may be grandfathered; standard new-app limit is lower). App owner must have Premium, and Extended Quota Mode is effectively unavailable to personal apps (requires a registered business with 250k+ MAU). Tuneless is designed to live within these limits.
Each new player must be added to the Spotify Developer dashboard under User Management with their exact name and Spotify email before they can play. They can log in but will get a 403 on API calls until allowlisted.
Playable sources: only playlists the user owns/collaborates on, plus Liked Songs. Followed/editorial/algorithmic playlists cannot be read.
Premium required for any audio (Web Playback SDK limitation).
0.5s precision: the shortest clip is best-effort due to playback start latency (see §10).
Leaderboard is per-browser (`localStorage`). Not shared across devices/browsers; cleared if site data is wiped.
Token lifetime: access tokens last 1 hour; refresh is automatic while `tl_refresh` exists, but there is no server, so everything is client-side.
---
13. History of Resolved Issues (chronological)
`response_type must be code` — implicit grant killed by Spotify → switched to PKCE.
`code_verifier was invalid` — verifier now stored in sessionStorage and localStorage.
`No playlists found` — `tracks.total` unreliable in the list API → filter on `p.id`.
Preview URLs null — Spotify killed 30s previews → switched to the Web Playback SDK.
`Invalid token scopes` from SDK — added `user-read-email`.
WebSocket failures — switched server bind from `[::1]` to `127.0.0.1`.
403 on `/me/player/play` — fixed with `?device_id=`, transfer-then-play retry (`ensureDevice`/`playUri`), `activateElement`, and token refresh.
Feb/Mar 2026 API migration — `/tracks`→`/items`, `track`→`item`, `/me` field removals, owned-only playlist reads, `user-library-read` for Liked Songs. (See §4.)
"No playable tracks" — was a swallowed 403 on followed playlists → grid now filtered to owned + Liked Songs; real errors surfaced.
Autocomplete too easy — was searching only the 10 in-play tracks → now searches the full deduped playlist pool.
Suggestions cut off — dropdown now opens upward, scrolls, shows a count header.
Token expiry every hour — refresh tokens implemented.
Clip overrun / whole-track playback — rebuilt timing engine (start-anchored watch + retry-until-paused + generation guard). Shortest tier moved 0.1s → 0.5s.
Client ID popup friction — Client ID hardcoded (`HARDCODED_CLIENT_ID`) so friends can play without any developer setup. "Connect with Spotify" now goes directly to OAuth redirect.
Deployed to GitHub Pages — file renamed `songless.html` → `index.html`; redirect URI updated to `https://cay919.github.io/`.
---
14. Possible Future Work
"Resume game" instead of restart when leaving mid-game (currently a fresh 10 songs each time).
Difficulty modes (e.g. title-only vs title+artist matching; fewer/more tiers).
Per-playlist leaderboards or filtering the leaderboard view by playlist.
Optional cross-device leaderboard would require a backend (out of scope for the single-file design).
Graceful handling if Spotify tightens dev mode further (e.g. a Spotify Connect fallback that remote-controls the user's existing active device instead of the Web Playback SDK).
---
15. Quick Facts Cheat-Sheet
Live URL: `https://cay919.github.io/`
File: `index.html` (formerly `songless.html`), single file, ~1,270 lines, no build.
Client ID: `2b41a394f163478586d9241cbbf8158f` (hardcoded as `HARDCODED_CLIENT_ID` constant)
Redirect URI: `https://cay919.github.io/` (register this exactly in Spotify dashboard)
Durations `[0.5, 1, 5, 10, 20]`s · Points `{0.5:5,1:4,5:3,10:2,20:1}` · 10 rounds · max 50 · wrong = −1.
Playlist tracks: `GET /playlists/{id}/items` (new). Liked Songs: `GET /me/tracks`. Play: `PUT /me/player/play?device_id=`. Transfer: `PUT /me/player`. Token: `https://accounts.spotify.com/api/token`.
Adding a new player: Spotify Dashboard → your app → Settings → User Management → add their exact name + Spotify email.
After any scope change: disconnect & reconnect.
The big lesson for future sessions: this app targets the post-February-2026 Spotify API. Treat pre-2026 docs/answers as outdated.
