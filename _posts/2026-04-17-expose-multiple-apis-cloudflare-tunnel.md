---
layout: post
title: "Expose Multiple APIs from a Single Server Using Cloudflare Tunnel (No Open Ports)"
categories: devops
---

Run multiple APIs on one EC2 instance and expose each one to the internet on its own subdomain — no open firewall ports, no SSL configuration, no nginx reverse proxy.

## Introduction

In the [previous post](/devops/2025/01/01/expose-local-server-to-internet.html), we used FRP to expose a Raspberry Pi over SSH. That approach requires a relay server and open ports. For HTTP APIs, there's a cleaner option: [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/).

The idea is simple: instead of opening inbound ports on your server, a lightweight `cloudflared` process on your server makes an outbound connection to Cloudflare's network. Cloudflare handles SSL termination, DDoS protection, and traffic routing — and your EC2 instance never needs a public IP or an open firewall rule.

This post walks through how we run multiple APIs on a single EC2 instance, each exposed on its own subdomain, all routed through a single Cloudflare Tunnel.

## Architecture

```
Public internet (HTTPS)
        ↓
Cloudflare network
  api1.yourdomain.com → EC2 → api1:5000
  api2.yourdomain.com → EC2 → api2:5001
        ↓
EC2 Instance (no inbound ports open)
├── api1          (port 5000, internal only)
├── api2          (port 5001, internal only)
├── redis         (internal only)
└── cloudflared   (outbound tunnel — the only thing touching the internet)
```

The `cloudflared` container uses an ingress configuration to route traffic by hostname to the correct internal service. The EC2 security group can block all inbound traffic — only outbound port 443 is needed for the tunnel.

## Prerequisites

- A domain managed by Cloudflare (free plan works)
- An EC2 instance (or any server) with Docker and Docker Compose installed
- A Cloudflare account with Zero Trust enabled (free tier is sufficient)

## Step 1: Create a Cloudflare Tunnel

1. Go to the [Cloudflare Zero Trust dashboard](https://one.dash.cloudflare.com/) → **Networks** → **Tunnels**.
2. Click **Create a tunnel**, name it (e.g. `my-server`), and save.
3. Copy the tunnel token — you'll need it as an environment variable.
4. Under **Public Hostname**, add a route for each API:
   - Subdomain: `api1`, Domain: `yourdomain.com`, Service: `http://api1:5000`
   - Subdomain: `api2`, Domain: `yourdomain.com`, Service: `http://api2:5001`

Cloudflare automatically provisions SSL certificates for these subdomains.

## Step 2: Configure Docker Compose

Create a `docker-compose.yml` with your services and a `cloudflared` container.

```yaml
services:
  api1:
    image: your-registry/api1:latest
    restart: unless-stopped

  api2:
    image: your-registry/api2:latest
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    restart: unless-stopped

  cloudflared:
    image: cloudflare/cloudflared:latest
    command: tunnel --no-autoupdate run
    environment:
      - TUNNEL_TOKEN=${TUNNEL_TOKEN}
    depends_on:
      - api1
      - api2
    restart: unless-stopped
```

`TUNNEL_TOKEN` is the token from Step 1. The `cloudflared` container reads the ingress rules you configured in the dashboard — no local config file needed.

> **Note:** Service names in Docker Compose become internal DNS names. The hostname `api1` in the Cloudflare dashboard routes to the `api1` container. Make sure they match.

## Step 3: Run It

Create a `.env` file with your token:

```bash
TUNNEL_TOKEN=your-tunnel-token-here
```

Then start everything:

```bash
docker compose up -d
```

Check the tunnel is connected:

```bash
docker compose logs cloudflared
```

You should see `Registered tunnel connection` in the logs. Your APIs are now live at `https://api1.yourdomain.com` and `https://api2.yourdomain.com`.

## Step 4: CI/CD Deployment (GitHub Actions)

Here's a minimal GitHub Actions workflow that builds, pushes, and deploys on every push to `main`.

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build and push Docker images
        run: |
          docker build -t ${{ secrets.ECR_REGISTRY }}/api1:${{ github.sha }} ./api1
          docker push ${{ secrets.ECR_REGISTRY }}/api1:${{ github.sha }}

      - name: Deploy to EC2
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ubuntu
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            echo "TUNNEL_TOKEN=${{ secrets.TUNNEL_TOKEN }}" > .env
            docker compose pull
            docker compose up -d
            docker image prune -f
```

The `TUNNEL_TOKEN` is stored as a GitHub secret and written to `.env` on the server at deploy time — it never lives in the repo.

## Rollback

To roll back to a previous version, re-run the deploy workflow with an earlier image tag, or add a manual `rollback.yml` workflow that accepts a version input and pulls that specific image from your registry.

## FAQs

### Does this work on a Raspberry Pi or home server?

Yes. `cloudflared` runs on ARM and x86. Your server only needs outbound internet access — no static IP or open ports required. This makes it a great fit for home labs.

### Can I route to services on different ports or different machines?

Yes. In the Cloudflare dashboard, each hostname can point to any `host:port` — including services on other machines on the same network, or even `localhost` if you're running without Docker.

### What's the difference between the dashboard ingress config and a local config file?

The dashboard approach (used here) stores routing rules in Cloudflare's control plane. A local `config.yml` stores them on the server. Both work — the dashboard approach is easier to update without redeploying.

### Is Cloudflare Tunnel free?

The tunnel itself is free. Cloudflare Zero Trust has a free tier for up to 50 users, which is more than enough for running personal or small team APIs.

## Conclusion

Cloudflare Tunnel is the cleanest way to expose multiple APIs from a single server. One `cloudflared` container handles all public traffic, each service stays fully internal, and you get SSL, DDoS protection, and global CDN for free. Combined with a simple GitHub Actions pipeline, the full deploy workflow — from `git push` to live API — takes under two minutes.
