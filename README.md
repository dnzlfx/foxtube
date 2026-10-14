# FoxTube

A YouTube-style frontend built as an overlay on top of
[Invidious](https://github.com/iv-org/invidious), with a self-contained
recommendation engine, a Nginx proxy layer and a Docker-based deployment.

FoxTube does not replace Invidious: it injects its own JavaScript and CSS into
the Invidious pages through Nginx `sub_filter`, so you keep upstream playback
and API compatibility while getting a custom feed, a "For you" ranking and a
personalized profile.

## What it does

- **Custom dark UI** injected over the stock Invidious pages (`assets/css/default.css`, `assets/js/yt-actions.js`).
- **Local recommendation engine** — candidate generation plus multi-objective ranking, all client-side, no backend training required. See `docs/recommendation-engine-guide.md`.
- **Personal profile** from a Google Takeout export (subscriptions and watch history). Only the sanitized example (`data/foxtube-profile.example.json`) is committed; the real profile is git-ignored because it contains personal data.
- **Nginx overlay** that serves the profile and rewrites the feed, history and subscriptions routes (`nginx/default.conf`).
- **Optional Postgres schema** for a future backend (`db/foxtube_recommendations_schema.sql`) — not used by the current overlay build.

## Requirements

- Docker and Docker Compose v2.
- A public domain (optional) if you want HTTPS and cookie-scoped sessions.

## Setup

1. Clone the repository.

   ```bash
   git clone https://github.com/dnzlfx/foxtube.git
   cd foxtube
   ```

2. Create your local configuration from the examples (these files hold secrets and are git-ignored):

   ```bash
   cp .env.example .env
   cp config/config.example.yml config/config.yml
   ```

3. Edit `.env` and `config/config.yml` and set, at minimum:

   - `SERVER_SECRET_KEY` in `.env` — shared secret between Invidious and the Companion service. Generate one with `openssl rand -hex 16`.
   - `invidious_companion_key` in `config/config.yml` — must match `SERVER_SECRET_KEY`.
   - `hmac_key` in `config/config.yml` — also from `openssl rand -hex 16`.
   - `domain` (and `https_only`) if you serve it from a real domain.
   - `PROXY_PORT` in `.env` if you do not want the default `3002` on the host.

4. Generate your recommendation profile from a Google Takeout export and place it at `data/foxtube-profile.json`. The committed example shows the expected shape:

   ```json
   {
     "generatedAt": "2026-01-01T00:00:00Z",
     "source": "google-takeout-example.zip",
     "subscriptions": [
       { "channelId": "UC…", "title": "Example Channel", "url": "https://www.youtube.com/channel/UC…" }
     ],
     "watchHistory": [
       { "videoId": "dQw4w9WgXcQ", "title": "Example Watched Video" }
     ]
   }
   ```

   `config/config.yml` must mount this file; the Nginx overlay serves it at `/foxtube-profile.json`.

5. Start the stack.

   ```bash
   docker compose up -d
   ```

6. Open the instance at `http://localhost:${PROXY_PORT}` (default `3002`), or your domain.

## Layout

```
assets/
  css/default.css          # FoxTube theme injected into Invidious
  js/yt-actions.js         # recommendation engine + UI behavior
  branding/                # logo and favicon
  ytube-clone.html         # feed shell
config/
  config.example.yml       # Invidious config template (copy to config.yml)
data/
  foxtube-profile.example.json  # profile shape; real file is git-ignored
db/
  foxtube_recommendations_schema.sql
docs/
  ALGORITHM.md
  css-design-guide.md
  recommendation-engine-guide.md
nginx/
  default.conf             # proxy + sub_filter overlay injection
docker-compose.yml
```

## How the overlay works

`docker-compose.yml` runs four services: `invidious`, `invidious-companion`,
`invidious-db` (Postgres) and `invidious-proxy` (Nginx). The proxy is what makes
FoxTube different — for the feed, history and subscriptions routes it proxies to
Invidious but injects `<script src="/js/yt-actions.js">` before `</body>` and
rewrites the CSS query string, then the JavaScript renders the FoxTube feed from
the profile and the recommendation engine.

## Security notes

- `config/config.yml` and `.env` are git-ignored; never commit real values. Only the `*.example` files are tracked.
- The real `data/foxtube-profile.json` is personal data (watch history) and is git-ignored.
- `/foxtube-profile.json` is protected by a hotlink guard in `nginx/default.conf`; add your own host to the allowed referer list before exposing it publicly.

## License

MIT — see `LICENSE`.
