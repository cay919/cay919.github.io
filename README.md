# Tuneless 🎧

A Songless-style music guessing game that plays songs from **your own Spotify playlists**. Hear a tiny clip, guess the song, and unlock longer clips if you need them. The earlier you guess, the more points you earn.

**Play it:** https://cay919.github.io/

## What you need

- A **Spotify Premium** account (required for in-browser playback).
- Your account must be added to the app's allowlist by the project owner (Spotify Developer Dashboard → User Management). Without this, you can log in but the game can't load your music.
- A modern desktop browser. Keep the tab visible while a round loads.

## How to play

1. Click **Connect with Spotify** and approve access.
2. Pick one of your playlists (or **Liked Songs**). A playlist needs at least **50 unique songs**.
3. Each game is **10 rounds**. In each round a clip plays, and you search for the song by title or artist. Search covers your whole playlist, not just the 10 songs in play.
4. Wrong guess or **Skip** unlocks a longer clip.
5. After each round you can replay and scrub the song for up to **60 seconds**.

### Scoring

| Clip length | 0.5s | 1s | 5s | 10s | 20s |
|---|---|---|---|---|---|
| Points | 5 | 4 | 3 | 2 | 1 |

- A wrong guess costs **1 point** (your score never drops below 0).
- Max score from the rounds is **50**.
- **Big-playlist bonus:** +1 point for a playlist of 100 songs, +2 for 150, +3 for 200, and so on. The bonus is added when the game ends and shown on the results screen.
- Songs with a silent intro are detected, and clips should start at the first audible sound.

### Leaderboard

Scores are shared globally. Your most recent game is highlighted and each entry shows the playlist and its song count.
