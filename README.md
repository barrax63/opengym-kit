# openGym Kit

**openGym Kit** is an open-source template that quickly sets up a self-hosted gym and body-weight tracker featuring openGym with passkey login, an optional AI coach, and a Cloudflare Tunnel as the only way in: no ports are exposed and there is no local access.

### What's included

✅ [**openGym API**](https://github.com/DuarteSantos8/openGym) - Node.js backend for passkey (WebAuthn) authentication, per-user JSON storage, push notifications and the AI coach

✅ [**openGym Web**](https://github.com/DuarteSantos8/openGym) - React PWA served by an unprivileged nginx, proxying `/api` so the whole app sits on one origin (required for passkeys)

✅ [**Cloudflared**](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) - Secure reverse proxy and the single entry point to your openGym instance, without exposing any ports

### What you can build

⭐️ **Personal Training Planner** with weekly routines or rotations over 5,600+ exercises with animated demos

⭐️ **Guided Workouts** with rest timers, plate math, supersets, warm-ups, cardio and automatic PR detection

⭐️ **Progress Tracking** with estimated 1RM curves, a muscle map, body-weight charts and progress photos

⭐️ **AI Coach** that drafts and revises your plan from what you log, using your own API key

⭐️ **Family or Small-Gym Instance** with per-profile data, invite-only signup and an admin dashboard

⭐️ **Installable Phone App** (PWA) with passkey sign-in, offline mode and sync across devices

Installation
------------

### Prerequisites

*   Docker and Docker Compose installed on your system (amd64 or arm64)
*   A Cloudflare account with a domain managed in Cloudflare (passkeys require HTTPS on a real hostname)
*   A Cloudflare Tunnel (see [Set up the Cloudflare Tunnel](#set-up-the-cloudflare-tunnel))
*   (Optional) API key from Anthropic, OpenAI, Google Gemini or an OpenAI-compatible endpoint (e.g. Ollama) for the AI coach

### Running openGym using Docker Compose

#### Standard Setup

Create the [Cloudflare Tunnel](#set-up-the-cloudflare-tunnel) first, then:

```bash
git clone https://github.com/barrax63/opengym-kit.git
cd opengym-kit
cp .env.example .env # Update the STANDARD CONFIGURATION section inside
docker compose up -d
```

> [!IMPORTANT]
> Make sure to update the following variables in your `.env` file:
>
> *   `HOST` - Your public hostname (e.g. `gym.example.com`): bare name, no `https://`, no port, no trailing slash
> *   `CLOUDFLARED_TUNNEL_TOKEN` - Your Cloudflare tunnel token
> *   `TZ` - Your timezone (e.g., `Europe/Berlin`, `America/New_York`)

> [!WARNING]
> Passkeys are bound to `HOST`. Changing it later invalidates every registered passkey, so settle on the final hostname before anyone signs up.

⚡️ Quick start and usage
------------------------

The openGym Docker Kit is built around a Docker Compose file that's pre-configured with network isolation, security hardening, and persistent storage.

After completing the installation steps above, simply visit `https://<HOST>` in your browser. All traffic reaches openGym through the Cloudflare Tunnel.

### First-time Setup

1.  **Create Admin Profile**: Right after the first start, tap **Create profile** and register a passkey. With `FIRST_USER_ADMIN=1` the first profile becomes the admin, so do this before sharing the address
2.  **Lock Down Signup**: Set `INVITE_ONLY=1` in `.env` and run `docker compose up -d`. New users then need an invite code from the admin dashboard
3.  **Set Up the AI Coach** (optional): see [Enable the AI coach](#enable-the-ai-coach-optional)
4.  **Install the App**: Add the site to your home screen (iOS: Share → Add to Home Screen · Android: ⋮ → Add to Home screen)

Containers and access
---------------------

The kit consists of multiple containers with restricted access for enhanced security. For more information on how to access the services, please refer to the table below.

| Container | Version | Hostname | Port | Network accessible? |
| --- | --- | --- | --- | --- |
| `api` | `latest` | api | 3000 | From Docker network only |
| `web` | `latest` | web | 80 | From Docker network only (via cloudflared) |
| `cloudflared` | `latest` | cloudflared | -/- | Outbound to Cloudflare only |

No container publishes a port on the host. The only way in is `Internet → Cloudflare → cloudflared → web → api`.

### Set up the Cloudflare Tunnel

The tunnel is required: it is the only path from the outside to openGym.

1.  [Create a Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/get-started/create-remote-tunnel/) and add a public hostname route for your `HOST` that points to `http://web:80`.

2.  Copy the generated **tunnel token** into your `.env` file as `CLOUDFLARED_TUNNEL_TOKEN`.

3.  Set `HOST` in `.env` to exactly the public hostname of that route.

4.  Start the stack (cloudflared starts together with openGym):

    ```bash
    docker compose up -d
    ```

Cloudflared connects outbound to Cloudflare's edge using the `token`, so no additional configuration files are required. Your openGym instance will be accessible via your custom domain without exposing any ports.

> [!NOTE]
> If you put **Cloudflare Access** in front of openGym, add a bypass policy for `/api/*` (the phone app pairs through it) and for `/icon-180.png`, `/icon-512.png` and `/manifest.json` (iOS fetches the home screen icon without your session).

### Enable the AI coach (optional)

The coach runs on your server with your own provider key. This kit uses the API-key providers only, which need nothing extra in the image: a coach job is a single HTTPS request from the `api` container.

1.  Sign in with your admin profile and open **Settings → Account → Admin dashboard → AI Coach**.
2.  Toggle the card on and pick a provider: **Anthropic**, **OpenAI**, **Google Gemini** or **OpenAI-compatible** (Ollama, LM Studio, vLLM, OpenRouter, ...).
3.  **Use an API key** → paste it. It is encrypted into `coach.json` inside the `api_storage` volume and never shown again.
4.  **List models** → pick a model, then **Test the Coach**.
5.  Users find it under **Plan → Coach**.

> [!TIP]
> A local model on a CPU may need more than the default five minutes per job. Set `COACH_JOB_TIMEOUT_MS=1500000` in `.env`. `COACH_DISABLED=1` forces the coach off instance-wide.

Upgrading
---------

### Update all containers to latest versions

```bash
git pull
docker compose pull
docker compose up -d
```

This will:

*   Pull the latest code changes
*   Download the latest container images
*   Recreate containers with new versions
*   Preserve all your data in volumes

### Enable auto-updates

You can enable automatic updates by adding a cron job. Run:

```bash
crontab -u <USER> -e
```

Add this entry:

```bash
# Every Sunday at 00:00 run git pull and compose up with latest images
0 0 * * 0 cd /home/<USER>/opengym-kit && /usr/bin/git pull && /usr/bin/docker compose pull && /usr/bin/docker compose up -d >> /home/<USER>/opengym-kit/cron.log 2>&1
```

👓 Recommended reading
----------------------

openGym offers extensive documentation and guides for getting the most out of your instance.

### Official Documentation

*   [openGym README](https://github.com/DuarteSantos8/openGym#readme)
*   [Self-hosting Guide](https://github.com/DuarteSantos8/openGym/blob/main/docs/SELF_HOSTING.md)
*   [AI Coach](https://github.com/DuarteSantos8/openGym/blob/main/docs/AI_COACH.md)
*   [Importing Data](https://github.com/DuarteSantos8/openGym/blob/main/docs/DATA_IMPORTS.md) (FitNotes, Strong, Hevy, Apple Health)
*   [Phone App](https://github.com/DuarteSantos8/openGym/blob/main/docs/MOBILE.md)
*   [FAQ](https://github.com/DuarteSantos8/openGym/blob/main/docs/FAQ.md)
*   [API Documentation](https://opengym.ch/api.html)

### Key Features

*   **Passkey Sign-in**: Face ID, Touch ID or fingerprint, with optional password sign-in
*   **Multi-Device Sync**: Field-by-field merge when two devices edit at once, full offline mode
*   **Progression Rules**: Linear, Greyskull LP, double/triple progression, automatic deloads
*   **Import & Export**: FitNotes, Strong, Hevy and Apple Health imports, full JSON export
*   **Push Notifications**: Rest timer and workout-day reminders in each user's local time
*   **User Management**: Admin dashboard, invite codes, activity log

Configuration Guide
-------------------

### Environment Variables

The `.env.example` file contains all configurable options divided into two sections:

#### Standard Configuration (Required)

*   **HOST**: Your public hostname, used for the passkey `RP_ID` and `ORIGIN=https://HOST`
*   **CLOUDFLARED\_TUNNEL\_TOKEN**: Cloudflare tunnel token for remote access
*   **TZ**: Your timezone (e.g., `Europe/Berlin`)

#### Advanced Configuration (Optional)

*   **Interface**: Passkey prompt name (`RP_NAME`), default language (`DEFAULT_LANG`)
*   **Access Control**: First-user admin, admin ids, invite-only signup, guest mode, password sign-in
*   **Security & Runtime**: Session length, activity log retention and IP recording
*   **AI Coach**: Job timeout, instance-wide kill switch
*   **Media Uploads**: Per-profile quota, video size cap, minimum free disk space
*   **Notifications**: Contact address sent with push notifications (`VAPID_SUBJECT`)

The full list of options is in the [openGym configuration reference](https://github.com/DuarteSantos8/openGym#quick-start).

### Security Features

This setup includes several security hardening measures:

*   **No new privileges**: Containers cannot gain additional privileges
*   **AppArmor**: Additional security layer for container isolation
*   **Capability dropping**: All capabilities dropped, none added back
*   **Read-only filesystems**: All containers, with tmpfs for scratch paths only
*   **Unprivileged web server**: nginx runs as uid 101
*   **Network isolation**: No published ports; all traffic enters through the Cloudflare Tunnel, and the API is reachable only through the web container
*   **Trusted client addresses**: The web container overwrites proxy headers and only passes Cloudflare's `CF-Connecting-IP`
*   **Resource limits**: CPU and memory constraints to prevent resource exhaustion
*   **Health checks**: Automatic container health monitoring and ordered start-up

Tips & tricks
-------------

### Accessing the API

The API is only accessible from within the Docker network for security. To check it:

```bash
# Health check from inside the container
docker compose exec api wget -qO- http://127.0.0.1:3000/api/health

# Show the passkey settings the server actually loaded
docker compose logs api | grep 'gym-api on'
```

### Backup and Restore

Everything (profiles, passkeys, workouts, uploads, session secret, encrypted coach key) lives in the `api_storage` volume.

#### Backup Volume

```bash
docker run --rm -v opengym-kit_api_storage:/data:ro -v $(pwd):/backup alpine tar czf /backup/opengym-backup-$(date +%F).tar.gz /data
```

#### Restore Volume

```bash
docker compose down
docker run --rm -v opengym-kit_api_storage:/data -v $(pwd):/backup alpine sh -c "rm -rf /data/* && tar xzf /backup/opengym-backup-<DATE>.tar.gz -C /"
docker compose up -d
```

> [!NOTE]
> The archive contains all profiles, workout history and, if enabled, the activity log. Users can also export their own data from **Settings → Data & backup**.

### Monitoring and Logs

```bash
# View logs for all services
docker compose logs -f

# View logs for specific service
docker compose logs -f api
docker compose logs -f web

# Check container health status
docker compose ps
```

### Resource Management

The Docker Compose file includes resource limits to prevent any single container from consuming all system resources:

*   **API**: Max 1 CPU, 1GB RAM
*   **Web**: Max 0.5 CPU, 256MB RAM
*   **Cloudflared**: Max 0.5 CPU, 512MB RAM

Adjust these limits in `docker-compose.yml` based on your system resources.

### Troubleshooting

#### Passkey sign-in fails

```bash
# Must print rpID=<HOST> and origin=https://<HOST>
docker compose logs api | grep 'gym-api on'

# After editing .env, recreate (restart does NOT re-read .env)
docker compose up -d
```

#### openGym won't start

```bash
# Check logs
docker compose logs api web

# Verify the API is healthy (web waits for it)
docker compose ps api

# Restart services
docker compose restart
```

#### AI coach test fails

```bash
# Look for the provider's error message
docker compose logs api | grep -i coach
```

Check the API key, the selected model, and for a local model raise `COACH_JOB_TIMEOUT_MS`.

#### openGym not reachable / Cloudflare tunnel not working

Since there is no local access, a broken tunnel means openGym is unreachable.

```bash
# All three containers must be healthy (cloudflared waits for web, web waits for api)
docker compose ps

# Check cloudflared logs
docker compose logs cloudflared

# Verify tunnel token is set correctly
docker compose config | grep TUNNEL_TOKEN
```

Make sure the tunnel route points to `http://web:80` and its public hostname matches `HOST`.

📜 License
----------

This project is licensed under the Apache License 2.0 - see the LICENSE file for details.

openGym itself is licensed under the [GNU AGPL v3.0](https://github.com/DuarteSantos8/openGym/blob/main/LICENSE). The exercise stills and animations shipped in the openGym images are © Aliaksandr Makatserchyk, [Gym visual](https://gymvisual.com/), licensed for use in openGym only.

💬 Support
----------

### openGym Community

*   [GitHub Discussions](https://github.com/DuarteSantos8/openGym/discussions)
*   [Discord Server](https://discord.gg/e62jY6fwVb)
*   [GitHub Issues](https://github.com/DuarteSantos8/openGym/issues)

### Getting Help

*   **Report Bugs**: Use GitHub Issues to report bugs or problems
*   **Feature Requests**: Share your ideas for new features
*   **Community Support**: Join Discord for real-time help and discussions
*   **Documentation**: Check the official docs for detailed guides

🤝 Contributing
---------------

Contributions are welcome! Please feel free to submit a Pull Request.

1.  Fork the repository
2.  Create your feature branch (`git checkout -b feature/AmazingFeature`)
3.  Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4.  Push to the branch (`git push origin feature/AmazingFeature`)
5.  Open a Pull Request

Acknowledgments
---------------

*   [openGym](https://github.com/DuarteSantos8/openGym) - The self-hosted gym and body-weight tracker
*   [Gym visual](https://gymvisual.com/) - Exercise stills and animations
*   [Cloudflare](https://www.cloudflare.com/) - Secure tunnel infrastructure

