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

All configuration lives in environment variables, which `docker-compose.yml` reads from a `.env` file in this directory. `.env` is gitignored; `.env.example` is the committed template.

```sh
cp .env.example .env
```

The defaults work as-is for local development. See [Configuration](#configuration) below for what each variable does.

### 2. Start the server

```sh
docker compose up -d
```

This pulls `solidproject/community-server:${CSS_IMAGE_TAG}`, publishes it on `${HOST_PORT}`, and persists all pod data to `${DATA_DIR}` on your machine. Data survives container restarts/rebuilds since it's a bind-mounted volume.

Check it's running:

```sh
docker compose logs -f
```

### 3. Open it in a browser

Go to **http://localhost:3000/** (or whatever `HOST_PORT` you set).

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

Everything is driven by `.env`, which `docker-compose.yml` substitutes into the service definition. Copy `.env.example` → `.env` and edit; re-run `docker compose up -d` to apply changes.

| Variable | Default | What it does |
|---|---|---|
| `CSS_PORT` | `3000` | Port the server listens on *inside* the container. |
| `HOST_PORT` | `3000` | Port published on the host. **The one that matters in production** — change it if 3000 is taken, and point your reverse proxy here. |
| `HOST_BIND` | `0.0.0.0` | Host interface the port binds to. Set to `127.0.0.1` in production so the container is only reachable via your reverse proxy. |
| `CSS_BASE_URL` | `http://localhost:3000/` | Public URL the server is reached at, trailing slash included. Baked into every WebID and resource URL — see the warning below. |
| `CSS_CONFIG` | `/config/default.json` | Which built-in CSS config to run (`default.json`, `file-no-setup.json`, `memory.json`, …). |
| `CSS_LOGGING_LEVEL` | `info` | `error` \| `warn` \| `info` \| `verbose` \| `debug` \| `silly`. |
| `CSS_IMAGE_TAG` | `latest` | Image tag to run. Pin to a specific version in production. |
| `CONTAINER_NAME` | `solid-server` | Docker container name. |
| `DATA_DIR` | `./data` | Host path bind-mounted to `/data` for pod storage. |

> ⚠ **`CSS_BASE_URL` is not just cosmetic.** It's the origin the server mints every WebID and resource URL from. Changing it after pods exist invalidates those WebIDs and breaks links to existing data. Decide on the final public URL *before* creating pods you care about.

Note that `HOST_PORT` and `CSS_BASE_URL` are independent. Behind a reverse proxy terminating TLS on 443, you'd typically have `HOST_PORT=3000`, `HOST_BIND=127.0.0.1`, and `CSS_BASE_URL=https://pods.example.org/`.

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

**Apply a change to `.env`:**
```sh
docker compose up -d
```
Compose recreates the container when the resolved config changes. Use `docker compose config` first to see exactly what your `.env` resolves to.

**Upgrade the server image:**
```sh
docker compose pull
docker compose up -d
```
(Or bump `CSS_IMAGE_TAG` in `.env` if you've pinned a version.)

**Inspect a resource directly** (e.g. via curl, once logged in isn't required for public resources):
```sh
curl http://localhost:3000/my-pod/profile/card
```

## Notes on this setup

- The **defaults in `.env.example` are for local development** — plain HTTP on `localhost:3000`, no TLS, no external hostname. Fine for experimenting with the Solid protocol and building/testing apps locally; not hardened for the internet as-is.
- `.env` is gitignored and `.env.example` is not, so real values (hostnames, ports, anything secret you add later) stay out of the repo. When you add a new variable, add it to `.env.example` too — with a placeholder, never a real value.

### Running in production

Solid clients in practice require HTTPS, and CSS doesn't terminate TLS itself. Front it with a reverse proxy (Caddy, nginx, Traefik) and:

1. Set `CSS_BASE_URL` to the public HTTPS URL, e.g. `https://pods.example.org/` — do this *before* creating pods, since it's baked into WebIDs.
2. Set `HOST_BIND=127.0.0.1` so only the proxy on the same host can reach the container.
3. Set `HOST_PORT` to whatever free port the proxy should forward to (3000 is fine if unused; change it if something else on the box already has it).
4. Pin `CSS_IMAGE_TAG` to a released version rather than `latest`.
5. Point `DATA_DIR` at a path on durable, backed-up storage — everything users own lives there.
6. Make sure the proxy forwards the original `Host` and `X-Forwarded-*` headers, so CSS generates correct URLs.

## Further reading

- [Solid Project homepage](https://solidproject.org/)
- [Community Solid Server docs](https://communitysolidserver.github.io/CommunitySolidServer/)
- [CSS getting-started tutorial](https://github.com/CommunitySolidServer/tutorials/blob/main/getting-started.md) (the walkthrough this README is based on)
- [Solid specification](https://solidproject.org/TR/protocol)
