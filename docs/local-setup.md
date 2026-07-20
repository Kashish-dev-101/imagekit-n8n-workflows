# Local Setup — Self-Hosted n8n

How the workflows in this repo were actually developed and tested: n8n self-hosted locally via Docker/OrbStack, exposed to the internet with ngrok so ImageKit's webhook servers can reach it.

## Why self-hosted (not n8n Cloud)

n8n Cloud works fine with the `@imagekit/n8n-nodes-imagekit` community node too — it's verified and installs the same way there. Self-hosting was chosen here specifically for:

- **No cost** while iterating on workflow designs (n8n Cloud is a paid trial/subscription past the free tier limits)
- **Full control over the instance** for this kind of experimentation — installing/reinstalling nodes, resetting data, etc.

If you just want to *use* a workflow from this repo rather than build/modify one, n8n Cloud is the faster path — skip everything below and import the `.json` directly.

## Prerequisites

- Docker-compatible engine — see the OrbStack note below if you're on a resource-constrained Mac
- [ngrok](https://ngrok.com/) account (only needed if self-hosting and using webhook-triggered workflows, e.g. the ImageKit Trigger node)

## Docker Desktop vs OrbStack

Docker Desktop works, but on a low-RAM Mac (8GB) it crashed mid-image-pull here (`docker: unexpected EOF`, daemon socket dropped) under memory pressure from other running apps. It succeeded on retry after freeing up memory, but its Electron GUI + full Linux VM overhead isn't worth carrying for a lightweight, occasional-use case like this.

[OrbStack](https://orbstack.dev/) was used instead — a Mac-native, Docker-compatible engine with a much smaller footprint, drop-in replacement for the same `docker` CLI. Free for personal/non-commercial use.

```bash
# after installing OrbStack and launching it once
docker context ls
# should show orbstack as the active context
```

## Start n8n

```bash
docker volume create n8n_data
docker run -d --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
```

Then open `http://localhost:5678` — first run asks you to create a local owner account (email + password), entirely local to this instance.

For subsequent sessions, once the container exists:

```bash
docker start n8n
```

## Expose it publicly (for webhook-based workflows)

The ImageKit Trigger node needs ImageKit's real servers to reach your n8n instance, so `localhost:5678` alone isn't enough. Tunnel it with ngrok:

```bash
ngrok http 5678
# or, with a reserved custom domain:
ngrok http --domain=your-reserved-domain.ngrok.io 5678
```

Use the resulting public HTTPS URL (swap in for `http://localhost:5678`) as the webhook endpoint you configure in ImageKit's dashboard.

## Stopping / starting everything

| Component | Start | Stop |
|---|---|---|
| OrbStack (Docker engine) | Open the app | Quit the app |
| n8n container | `docker start n8n` | `docker stop n8n` |
| ngrok tunnel | `ngrok http --domain=... 5678` | Ctrl+C / kill the process |

Stopping the container just pauses it — workflows and credentials persist in the `n8n_data` volume. Note: test-webhook listeners ("Test this trigger") are one-shot and don't survive a restart — re-arm them (or hit "Execute workflow" to arm the whole workflow) after bringing things back up.

## Reference

- [n8n official docs](https://docs.n8n.io/)
- [ImageKit + n8n integration docs](https://imagekit.io/docs/integration/n8n)
- [`@imagekit/n8n-nodes-imagekit` node source](https://github.com/imagekit-developer/n8n-nodes-imagekit)
