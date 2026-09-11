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

The host port is read from a `.env` file in this directory. `.env` is gitignored; `.env.example` is the committed template.

```sh
cp .env.example .env
```

The default (`PORT=3000`) works as-is for local development. In production, change it if port 3000 is already taken on the host.

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

The only setting exposed via `.env` is the host port:

| Variable | Default | What it does |
|---|---|---|
| `PORT` | `3000` | Port published on the host. Change it in production if 3000 is already in use, and point your reverse proxy here. |

Copy `.env.example` → `.env`, edit, and re-run `docker compose up -d` to apply.

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

- This is configured for **local development only** — it's served over plain HTTP on `localhost:3000`, with no TLS, and no external hostname. It's fine for experimenting with the Solid protocol and building/testing apps locally, but isn't hardened for exposing to the internet as-is.
- If you want to expose this pod to others or to real apps (not just localhost testing), you'll need a public hostname + HTTPS (Solid strongly prefers/requires this in practice for interop with most clients), which typically means fronting it with a reverse proxy (e.g. Caddy/nginx) and changing the server's configured base URL.
- `.env` is gitignored and `.env.example` is not, so keep real values out of `.env.example` — it should only ever hold placeholders.

## Further reading

- [Solid Project homepage](https://solidproject.org/)
- [Community Solid Server docs](https://communitysolidserver.github.io/CommunitySolidServer/)
- [CSS getting-started tutorial](https://github.com/CommunitySolidServer/tutorials/blob/main/getting-started.md) (the walkthrough this README is based on)
- [Solid specification](https://solidproject.org/TR/protocol)
