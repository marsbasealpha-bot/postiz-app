# Self-host this fork on a Windows PC

This runs Postiz from **this repository** on your own computer. The site is `http://localhost:4007`. It does not use [platform.postiz.com](https://platform.postiz.com). Stripe is left empty, so the hosted-plan channel and post limits stay off.

The first start builds the app image from `Dockerfile.dev`. That build needs a lot of memory and time. Later starts reuse the image.

## What to install

1. **Windows 10 or 11**, 64-bit.
2. **Docker Desktop**, with the WSL2 backend.
   - https://docs.docker.com/desktop/setup/install/windows-install/
   - During setup, leave "Use WSL 2 instead of Hyper-V" turned on.
   - Start Docker Desktop and wait until the whale icon is steady.
   - Settings → Resources: give Docker at least **8 GB RAM** and **4 CPUs**. 10 GB RAM is more comfortable. The first image build uses about 4 GB of heap inside the container, plus Postgres, Redis, Temporal, and Elasticsearch.
   - Settings → General: "Start Docker Desktop when you sign in" so scheduled posts can run.
3. **Git for Windows** (includes Git Bash and OpenSSL).
   - https://git-scm.com/download/win
4. **Disk**: about **20 GB free**. The images and the build cache are large.
5. **Stay awake**. The PC and Docker must be running at the time a post is due. Sleep, shutdown, and a stopped Docker engine delay the post until the stack is up again.

You do not install Node, pnpm, Postgres, or Redis on Windows. They run inside Docker.

## Configure

Open **Git Bash** or **PowerShell**. Clone this fork and go into the folder:

```bash
git clone https://github.com/marsbasealpha-bot/postiz-app.git
cd postiz-app
```

Use a normal folder under your user profile, for example `C:\Users\you\projects\postiz-app`. Docker Desktop can mount that. A network drive or a path Docker is not allowed to share will fail at startup.

Copy the example environment file. It contains placeholders only.

Command Prompt:

```bat
copy .env.docker.example .env
```

PowerShell:

```powershell
Copy-Item .env.docker.example .env
```

Git Bash:

```bash
cp .env.docker.example .env
```

Edit `.env` and replace `JWT_SECRET` before the first launch. Git Bash:

```bash
openssl rand -base64 32
```

PowerShell, if `openssl` is not on `PATH`:

```powershell
[Convert]::ToBase64String((1..32 | ForEach-Object { Get-Random -Maximum 256 }))
```

Paste the result as the value of `JWT_SECRET`. One line, no quotes. Leave every `STRIPE_*` value empty.

`MAIN_URL`, `FRONTEND_URL`, and `NEXT_PUBLIC_BACKEND_URL` must be the address you open in the browser. The defaults are correct for this PC:

```env
MAIN_URL=http://localhost:4007
FRONTEND_URL=http://localhost:4007
NEXT_PUBLIC_BACKEND_URL=http://localhost:4007/api
```

Check that Compose can see the secret (the printed value is yours; do not commit `.env`):

```bash
docker compose config | grep JWT_SECRET
```

If `.env` is missing, that command stops with `Set JWT_SECRET in .env` instead of starting the stack with a shared default.

## Start

From the repository root, with Docker Desktop already running:

```bash
docker compose up -d --build
```

The first run builds `postiz-fork:local` and pulls Postgres, Redis, Temporal, and Elasticsearch. Expect a long first build. Later starts are:

```bash
docker compose up -d
```

Watch progress:

```bash
docker compose ps
```

Wait until `postiz`, `postiz-postgres`, `postiz-redis`, `temporal`, `temporal-postgresql`, `temporal-elasticsearch`, and `temporal-ui` show **healthy**. `temporal-admin-tools` stays **Up** without a health check. `(health: starting)` for a few minutes on the first boot is normal: Elasticsearch starts, then Temporal, then Postiz runs `prisma db push`.

If a service stays unhealthy:

```bash
docker compose logs postiz
docker compose logs temporal
docker compose logs temporal-elasticsearch
```

## Create the first admin

1. Open **http://localhost:4007** in a browser. Use `http`, not `https`.
2. Choose **Register**.
3. Enter an email, a password, and a company name, and submit.

No email server is configured. Postiz activates the account immediately when email is not set up, and the new user is the **SUPERADMIN** of that organization. That is the admin of this private instance. There is no separate CLI step and no license key.

Sign in with the same email and password. You should land on the calendar (`/launches`), with no billing wall.

After the account exists, close public registration. In `.env`:

```env
DISABLE_REGISTRATION=false
```

Change that line to:

```env
DISABLE_REGISTRATION=true
```

Apply it:

```bash
docker compose up -d
```

Your account stays in the Postgres volume.

## Confirm the instance is up

All of these should succeed.

1. `docker compose ps` shows the services above as healthy.
2. This URL prints `App is running!` (the trailing slash matters):

   http://localhost:4007/api/

   Git Bash: `curl -fsS http://localhost:4007/api/`

   PowerShell: `Invoke-WebRequest http://localhost:4007/api/ | Select-Object -ExpandProperty Content`

3. http://localhost:4007 loads the Postiz sign-in page, served by this stack, not by platform.postiz.com.
4. After you sign in, the calendar loads and the sidebar is usable. There is no plan-upgrade screen. Empty `STRIPE_*` values are what turns the hosted-plan limits off (`billingEnabled` stays false, and channel and post checks are skipped).
5. Optional: Temporal's own UI is at http://127.0.0.1:8080 while the stack is running.

## Connect a channel and schedule a post

Each social network needs an app that **you** create on that network. This repository cannot ship those secrets. Fill the matching variables in `.env`, then `docker compose up -d`.

The callback URL you register at the network is:

```text
http://localhost:4007/integrations/social/<provider>
```

Mastodon is the shortest path, because the instance issues a client id and secret immediately:

1. On the Mastodon site (the default in `.env` is https://mastodon.social): Preferences → Development → New application.
2. Redirect URI: `http://localhost:4007/integrations/social/mastodon`
3. Put the client id and secret in `.env` as `MASTODON_CLIENT_ID` and `MASTODON_CLIENT_SECRET`.
4. `docker compose up -d`
5. In Postiz: Add channel → Mastodon, and finish the prompt in the browser.
6. Create a post, pick a time a few minutes ahead, and schedule it.

The post is sent only while this PC and Docker are awake. X, Meta, TikTok, and YouTube need their own developer apps and, for some of them, a review or a paid API tier. That approval happens on their sites. It is not a switch in this repo.

Uploads are stored in the `postiz-uploads` volume (`STORAGE_PROVIDER=local`). You do not need a Cloudflare account to start.

## Day to day

```bash
docker compose ps
docker compose logs -f postiz
docker compose stop
docker compose up -d
docker compose restart postiz
```

`docker compose stop` keeps accounts, posts, and uploads. `docker compose down -v` deletes those volumes.

## If it does not start

**`Set JWT_SECRET in .env`**  
`.env` is missing or `JWT_SECRET` is empty. Copy `.env.docker.example` and set a secret.

**`docker: command not found` or the engine is not running**  
Start Docker Desktop and wait until it is idle. Open a new terminal so `PATH` includes Docker. Git Bash sometimes needs a full sign-out before it sees `C:\Program Files\Docker\Docker\resources\bin` or `%LOCALAPPDATA%\Programs\DockerDesktop\resources\bin`.

**The page loads, then register or login returns 502**  
nginx is up and the API process is not. Check `docker compose logs postiz` for `Backend failed to start`. Usual causes: Postgres or Temporal not healthy yet (wait and look at `docker compose ps`), or `DATABASE_URL` was overridden to `localhost`. Inside the container the database host is `postiz-postgres`, which the Compose file already sets.

**Elasticsearch exits, or logs say `vm.max_map_count`**  
In PowerShell:

```powershell
wsl -d docker-desktop sysctl -w vm.max_map_count=262144
```

Then `docker compose up -d`. Recent Docker Desktop versions already set this. Elasticsearch is required by this Compose file because Temporal is started with `ENABLE_ES=true`.

**`network temporal-network` already exists**  
An older Postiz Compose file created a network with that fixed name. This file uses a project-local network. Remove the leftover only if nothing else is using it: `docker network rm temporal-network`, then `docker compose up -d`.

**Port 4007, 7233, or 8080 is already in use**  
Stop the other program, or set `POSTIZ_PORT` in `.env` and change `MAIN_URL`, `FRONTEND_URL`, and `NEXT_PUBLIC_BACKEND_URL` to that same port.

**Build runs out of memory**  
Raise Docker Desktop's memory limit and run `docker compose up -d --build` again. The Dockerfile already sets the Node heap to 4 GB for the build.

**`/api/copilot/chat` timeouts in the logs**  
`OPENAI_API_KEY` is empty. Scheduling still works. Add a key only if you want the AI assistant.

## What this file does not do

- It does not install Postiz on a particular person's PC. You run the commands above on the machine that should host it.
- It does not publish to social networks by itself. You bring the developer-app credentials, and the computer has to be on at send time.
- The image `ghcr.io/marsbasealpha-bot/postiz-app` is built by CI on `main`, and an anonymous pull currently gets "not found" (the package is private). This Compose file builds `postiz-fork:local` from the clone instead, so a private registry login is not required.
