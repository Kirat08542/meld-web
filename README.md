# meld-web

A web port of [**Meld**](https://github.com/FrancescoGrazioso/Meld) — Spotify's personalization, YouTube Music's catalog — dressed in [**Monochrome**](https://github.com/monochrome-music/monochrome)'s black-and-white aesthetic.

> Spotify drives recommendations and your library. Audio streams from YouTube via [Piped](https://docs.piped.video). Free Spotify account is enough. No Premium needed.

## Quick start

```bash
# 1. Install deps (npm, pnpm, yarn, bun — all fine)
npm install

# 2. Make a Spotify app, add a redirect URI: http://127.0.0.1:5173/callback
#    https://developer.spotify.com/dashboard

# 3. Set your Client ID
cp .env.example .env
# edit .env, paste your VITE_SPOTIFY_CLIENT_ID

# 4. Run
npm run dev
```

Open **http://127.0.0.1:5173** (Spotify rejects `localhost` — use the IP).

## Architecture

```
src/
├── lib/
│   ├── spotify/
│   │   ├── auth.ts            — OAuth PKCE flow (no client secret)
│   │   ├── api.ts             — Spotify Web API client (search, top, library)
│   │   └── types.ts
│   ├── ytmusic/
│   │   ├── piped.ts           — Piped instance client (search + stream URLs)
│   │   └── matcher.ts         — Spotify→YT fuzzy matcher (PORT of Meld's SpotifyMapper.kt)
│   ├── recommendations/
│   │   └── engine.ts          — PORT of Meld's SpotifyRecommendationEngine.kt
│   └── player/
│       └── store.ts           — Zustand store + <audio> controller
├── components/
│   ├── Sidebar.tsx
│   ├── PlayerBar.tsx
│   └── atoms.tsx              — TrackRow, AlbumCard, ArtistCard, Section
└── pages/
    ├── Login.tsx
    ├── Callback.tsx
    ├── Home.tsx
    ├── Search.tsx
    └── Library.tsx
```

## What's ported from Meld (faithfully)

- **`SpotifyMapper.kt` → `matcher.ts`**
  Title normalization (feat/ft/brackets/remaster/remix strip → alphanum), Dice-coefficient bigram similarity, weighted score (`title*0.45 + artist*0.35 + duration*0.20`), early-exit at `0.95`, LRU caches for normalized strings + bigrams.
- **`SpotifyRecommendationEngine.kt` → `engine.ts`**
  User taste profile across 3 time ranges, position-weighted artist affinity, genre Jaccard similarity, 4 candidate buckets (seed artist / same album / genre neighbor / user top), composite scoring (same weights: src 25%, affinity 30%, genre 20%, popularity 10%, recency 15%), per-artist cap of 3 with bucket interleaving.
- **Manual match override** (paste a YT URL for a wrong-matched track) — stored in `localStorage`, takes priority over auto-matching.
- **Match cache** — `spotifyTrackId → videoId` persisted across sessions.

## What's *different* from Meld (and why)

| Meld (Android) | meld-web | Why |
|---|---|---|
| `sp_dc` cookie + TOTP via WebView | OAuth PKCE + Client ID | `sp_dc` is HttpOnly cross-origin — unreachable from browser. PKCE is the only legitimate path. |
| innertube (YT Music internal API) | Piped public instances | YT's API blocks CORS. Piped is the browser-feasible substitute. |
| Native Android player | `<audio>` + Media Session API | Standard web stack. Lockscreen/headset controls still work. |
| Room DB cache | `localStorage` | Plenty for this scope. Could swap to IndexedDB if it gets heavy. |
| Discord Rich Presence, downloads | Not included | Would need a backend. |

## What's *not* built yet

- Playlist detail page (currently lists playlists but doesn't open them)
- Album detail page
- Artist detail page
- Liked Songs view
- Lyrics
- Settings page (Piped instance picker, theme picker)
- Manual match override UI (the underlying API exists in `matcher.ts`)
- Sync (add/remove from playlist)

All the data layer is there for these — they're just UI work.

## Deploying

It's a static site. Build and drop the `dist/` folder anywhere:

```bash
npm run build
# → dist/ goes on Vercel / Netlify / Cloudflare Pages / a webserver
```

When you deploy, **add the production URL as a Redirect URI in your Spotify dashboard** (e.g. `https://yourapp.vercel.app/callback`) and set `VITE_SPOTIFY_REDIRECT_URI` to match.

## Piped instances

If the default instance is slow or down, swap to another. Defaults are baked into `src/lib/ytmusic/piped.ts`. List of public instances: https://piped-instances.kavin.rocks/

## License

MIT for this scaffold. Meld is GPL-3.0, Monochrome is GPL-3.0 — they only inspire here; nothing is copy-pasted from them.
