# solid-server

A local [Community Solid Server](https://github.com/CommunitySolidServer/CommunitySolidServer) (CSS) instance, run via Docker. This is the reference implementation of a [Solid](https://solidproject.org/) server — it hosts one or more **Pods** and acts as an identity provider for logging into Solid apps.

## What is a Solid Pod?

Solid decouples *your data* from *the apps that use it*. Normally, an app (a to-do list, a photo gallery, a chat client) stores your data on its own server, in its own format, and you're stuck with it.

With Solid:

- Your data lives in a **Pod** ("Personal Online Datastore") — basically your own personal web server / folder tree of files (documents, photos, contacts, calendar entries, app settings, etc.), which you control.
- You have a **WebID** — a URL that identifies *you*, the same way an email address does, but for the whole web. It's just an RDF document living in your pod (e.g. `http://localhost:3000/my-pod/profile/card#me`).
- Apps don't get a copy of your data. Instead, you log in with your WebID, and the app reads/writes data directly in your pod, with permission you control.
- Access control is per-resource, via `.acl` files — you decide who (which WebID, or the public) can read/write/append/control each file or folder.

So this server is both:
1. A place to store your data (the Pod itself), and
2. An **Identity Provider (IdP)** — the thing Solid apps redirect you to in order to log in and prove who you are.

## Prerequisites

- [Docker](https://www.docker.com/) and Docker Compose installed and running.

## Getting started

### 1. Create your `.env`

Configuration is read from a `.env` file in this directory. `.env` is gitignored; `.env.example` is the committed template.

```sh
cp .env.example .env
```

The defaults (`PORT=3000`, `CSS_BASE_URL=http://localhost:3000/`) work as-is for local development.

> ⚠ **Deploying this anywhere other than `localhost`?** You *must* set `CSS_BASE_URL` to your real public URL (e.g. `https://solid.example.org/`). CSS rejects any request that doesn't match its configured base URL — get this wrong and you'll see errors like *"Request received for an unsupported path"* / *"This server appears to be misconfigured"* on **every** request, not just sign-up. See [Configuration](#configuration) below.

### 2. Start the server

```sh
docker compose up -d
```

This pulls `solidproject/community-server:latest`, publishes it on `${PORT}` (default `3000`), and persists all pod data to `./data` on your machine. Data survives container restarts/rebuilds since it's a bind-mounted volume.

Check it's running:

```sh
docker compose logs -f
```

### 3. Open it in a browser

Go to **http://localhost:3000/** (or whatever `PORT` you set).

### 4. Create an account

Click **Sign up**, and register with an email + password. This account is *not* your pod — it's your login credentials for this server (you can own multiple pods with one account).

### 5. Create your Pod

After signing up you'll land on your account page. Click **Create pod**, give it a name (e.g. `my-pod`), and **leave "Create a WebID for me" checked** — this makes the server generate a profile document for you automatically.

This gives you:
- Pod root: `http://localhost:3000/my-pod/`
- Profile/WebID: `http://localhost:3000/my-pod/profile/card#me`

Your WebID is the identity you'll hand to any Solid app to log in.

### 6. Poke around your pod

- Visit `http://localhost:3000/my-pod/` — you'll see a basic file browser (it's just a container/resource listing).
- Visit `http://localhost:3000/my-pod/profile/` without being logged in — you should get a `401 Unauthorized`. That's Web Access Control (WAC) at work: profile data is private by default except the public card itself.

### 7. Try a Solid app against your pod

You don't need to write any code to try this out. Use an existing Solid client, e.g. [Penny](https://penny.vincenttunru.com/) (a simple pod file browser/editor):

1. Open Penny, click **Connect Pod** (or similar).
2. Enter your server URL: `http://localhost:3000/`
3. Log in with the email/password you created.
4. You'll now be able to browse, create, and edit resources in your pod as your authenticated WebID — and set access rules on them.

Other apps to try: [Solid OS](https://solidos.solidcommunity.net/), [PodBrowser](https://podbrowser.inrupt.com/), or anything listed at [solidproject.org/apps](https://solidproject.org/apps).

## Configuration

| Variable | Default | What it does |
|---|---|---|
| `PORT` | `3000` | Port published on the host. Change it in production if 3000 is already in use, and point your reverse proxy here. |
| `CSS_BASE_URL` | `http://localhost:3000/` | The server's real public URL, trailing slash included. **Required to be correct in production** — see the warning below. |
| `CSS_CONFIG` | `config/file.json` | Which server config to run. The default (built into the image) has registration enabled. Set to `/config/no-registration.json` to disable it — see [First-time production setup](#first-time-production-setup-create-your-account-then-lock-it-down). |

Copy `.env.example` → `.env`, edit, and re-run `docker compose up -d` to apply.

> ⚠ **Why `CSS_BASE_URL` matters:** CSS treats every incoming request's Host/URL as either inside or outside this configured base URL, and rejects anything outside it — including its own account/sign-up API. If you're running behind a reverse proxy (nginx, Caddy, Traefik) on a real domain:
> 1. Set `CSS_BASE_URL` to that domain, e.g. `CSS_BASE_URL=https://solid.example.org/`.
> 2. Make sure the reverse proxy forwards the original request info via a `Forwarded` header (or `X-Forwarded-Proto` / `X-Forwarded-Host`), so CSS can see the real scheme and host rather than assuming `localhost`.
>
> Symptoms of getting this wrong: a "Registration is disabled" message even though registration is enabled, or a full-page CSS error reading *"Request received for an unsupported path"* — both mean the base URL (or the proxy headers feeding it) don't match the address you're actually visiting.

## Key concepts, quick reference

| Term | Meaning |
|---|---|
| **Pod** | Your personal storage space on a Solid server — a tree of resources (files/folders), each with its own URL. |
| **WebID** | A URL identifying you, pointing to an RDF "profile card" document. Used to log into apps instead of a username. |
| **Identity Provider (IdP)** | The server that authenticates you and issues proof of your WebID to apps. This CSS instance plays this role. |
| **WAC / ACL** | Web Access Control — `.acl` resources next to your data that define who (by WebID) can Read/Write/Append/Control it. |
| **Resource** | Any item in your pod — a file, an image, an RDF document (Turtle, JSON-LD, etc.), or a container (like a folder). |
| **Container** | A "folder" in your pod — itself a resource, listing the resources inside it. |

## Common operations

**Stop the server:**
```sh
docker compose down
```

**Reset everything (⚠ deletes all pod data):**
```sh
docker compose down
rm -rf ./data
```

**Upgrade the server image:**
```sh
docker compose pull
docker compose up -d
```

**Inspect a resource directly** (e.g. via curl, once logged in isn't required for public resources):
```sh
curl http://localhost:3000/my-pod/profile/card
```

## Notes on this setup

- The defaults in this repo are for **local development** — plain HTTP on `localhost:3000`, no TLS, no external hostname. See [Running in production](#running-in-production) below before exposing this to the internet.
- `.env` is gitignored and `.env.example` is not, so keep real values out of `.env.example` — it should only ever hold placeholders.

## Running in production

Solid clients expect HTTPS in practice, and CSS doesn't terminate TLS itself, so production means putting a reverse proxy in front of it. Checklist:

### 1. Point a domain at your host

You need a real hostname (e.g. `solid.example.org`) with DNS pointing at the machine running Docker.

### 2. Put a reverse proxy in front, terminating TLS

The proxy must forward the original request's scheme/host so CSS can validate it against `CSS_BASE_URL` (see the warning in [Configuration](#configuration) — get this wrong and *everything* breaks, not just sign-up).

**Caddy** (simplest — handles HTTPS certificates automatically):
```
solid.example.org {
    reverse_proxy localhost:3000
}
```
Caddy sends `X-Forwarded-*` headers by default, so no extra config is needed.

**nginx** (needs the forwarded headers set explicitly):
```nginx
server {
    listen 443 ssl;
    server_name solid.example.org;

    ssl_certificate     /etc/letsencrypt/live/solid.example.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/solid.example.org/privkey.pem;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        # WebSocket support (CSS uses this for live notifications)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

### 3. Set your production `.env`

On the production host (this file is *not* the one committed to the repo):

```sh
PORT=3000
CSS_BASE_URL=https://solid.example.org/
```

`CSS_BASE_URL` must exactly match the public address, trailing slash included.

### 4. Keep the raw port off the public internet

The reverse proxy should be the only public entry point. Either firewall port `3000` so it's only reachable from `localhost`, or bind Docker's port mapping to loopback only by changing `docker-compose.yml`:

```yaml
ports:
  - "127.0.0.1:${PORT:-3000}:3000"
```

### 5. Deploy

```sh
docker compose up -d
docker compose logs -f
```

### 6. Back up `./data`

Every pod's data lives in `./data` as plain files. Back it up like you would any other stateful volume — there's no separate database to worry about.

### 7. Upgrading

Consider pinning the image to a specific version instead of `latest` (e.g. `solidproject/community-server:7.2.0` in `docker-compose.yml`) so an upstream release can't change your server under you. Bump it deliberately, then:
```sh
docker compose pull
docker compose up -d
```

### First-time production setup: create your account, then lock it down

By default, anyone who can reach the server can sign up and create their own pod. If this server is meant to host only *your* account, run through this once, in order — registration has to be open long enough for you to create your own account, then gets switched off:

1. **Deploy with registration enabled.** This is the default (`CSS_CONFIG=config/file.json`) — don't set `CSS_CONFIG` yet.
   ```sh
   docker compose up -d
   ```
2. **Go to the registration page**, e.g. `https://solid.example.org/.account/login/password/register/` (or click **Sign up** on the root page).
3. **Create your one account** — the email/password login for this server (see [Create an account](#4-create-an-account) above).
4. **Create your Pod/WebID** — click **Create pod**, leave "Create a WebID for me" checked (see [Create your Pod](#5-create-your-pod) above).
5. **Verify you can log in and access the Pod** — log out, log back in, confirm your WebID and pod resources load correctly. Do this *before* locking anything down, since registration UI won't be reachable afterwards if something needs fixing.
6. **Disable registration.** In your production `.env`:
   ```sh
   CSS_CONFIG=/config/no-registration.json
   ```
   This points at [`config/no-registration.json`](config/no-registration.json) in this repo (bind-mounted into the container as `/config`) — it's identical to the default config except account/pod creation is switched off. Your existing account, pod, and login are completely unaffected; only *new* registrations are blocked.
7. **Restart CSS** to apply it:
   ```sh
   docker compose up -d
   ```
8. **Confirm registration is now closed** — reload the root page; it should show *"Registration is disabled on this server"* instead of a sign-up link. Then confirm your own account still logs in and your pod still loads.

## Further reading

- [Solid Project homepage](https://solidproject.org/)
- [Community Solid Server docs](https://communitysolidserver.github.io/CommunitySolidServer/)
- [CSS getting-started tutorial](https://github.com/CommunitySolidServer/tutorials/blob/main/getting-started.md) (the walkthrough this README is based on)
- [Solid specification](https://solidproject.org/TR/protocol)
