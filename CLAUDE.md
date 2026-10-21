# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A deployment kit only: `docker-compose.yml`, `.env.example` and `README.md`. There is no application source, build, lint or test suite. The app ships as prebuilt upstream images (`ghcr.io/duartesantos8/opengym-api`, `ghcr.io/duartesantos8/opengym-web`, from <https://github.com/DuarteSantos8/openGym>). App behaviour and the full env-var reference live upstream, not here.

## Commands

```bash
cp .env.example .env          # required: api's env_file is `required: true`, compose fails without .env
docker compose config -q      # validate compose file + interpolation (the closest thing to a lint/test)
docker compose up -d          # start / apply .env changes (`restart` does NOT re-read .env)
docker compose ps             # health of all three services
docker compose logs api | grep 'gym-api on'   # shows the rpID/origin the API actually loaded
docker compose exec api wget -qO- http://127.0.0.1:3000/api/health
```

## Architecture

Traffic path: `Internet → Cloudflare → cloudflared → web:80 → api:3000`. No service publishes a port; the Cloudflare Tunnel is the only way in. Never add `ports:`.

- **api**: Node backend. All persistent state (users, passkeys, session secret, uploads, encrypted coach key in `coach.json`) is in the `api_storage` volume at `/data`. Project name is pinned (`name: opengym-kit`), so the volume is `opengym-kit_api_storage`; README backup/restore commands depend on that name.
- **web**: nginx (uid 101) serving the React PWA and proxying `/api` to `BACKEND`:`PORT`. Same-origin is required for passkeys.
- **cloudflared**: outbound-only tunnel, configured entirely by `TUNNEL_TOKEN`; route in the Cloudflare dashboard must point to `http://web:80`.

Startup is ordered by healthchecks: cloudflared waits for web healthy, web waits for api healthy. Healthchecks use `127.0.0.1`, not `localhost` (nginx listens IPv4-only).

## Non-obvious invariants

- **`.env` does double duty**: compose reads it for `${...}` interpolation, and api also loads the whole file via `env_file`. web deliberately has no `env_file`: its entrypoint runs envsubst over the nginx template, so only explicitly listed vars belong there. `$$http_cf_connecting_ip` is a compose escape so nginx receives the literal `$http_cf_connecting_ip`.
- **`HOST` override**: `.env` `HOST` is the public hostname, feeding `RP_ID=${HOST}` and `ORIGIN=https://${HOST}`. The api `environment:` then sets `HOST=0.0.0.0` because the API reads `HOST` as its bind address. Keep both. Changing the public `HOST` invalidates every registered passkey.
- **Upload size chain**: `MEDIA_VIDEO_MAX_MB` (.env, 40) < `MEDIA_UPLOAD_MAX` (web, `48m`) < Cloudflare's 100 MB free-plan cap. Change them together.
- **Service wiring**: web's `BACKEND=api` / `PORT=3000` must match the api service name and its `PORT`. `TRUST_PROXY=1` on api is only safe because api is reachable solely through web.
- **Hardening baseline**: every service merges `<<: *security-baseline` (no-new-privileges, AppArmor, `cap_drop: ALL`) and runs `read_only: true`. Any new writable path needs a `tmpfs` entry (web's need `uid=101,gid=101`). The AI coach supports only API-key providers; CLI-based providers would need SETUID/SETGID caps this kit does not grant.
- **AI coach config** is done in the app's admin dashboard, not via env. Only `COACH_JOB_TIMEOUT_MS` and `COACH_DISABLED` exist as env knobs.

## Keeping docs in sync

README mirrors the config: the "Configuration Guide" categories match the `.env.example` sections (STANDARD vs ADVANCED), the "Resource Management" section lists the `deploy.resources.limits` values, and the containers table lists services/ports. Update the README when changing any of these.
