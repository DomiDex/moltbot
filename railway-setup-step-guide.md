# Moltbot Railway + Cloudflare Tunnel — Step-by-Step Setup Guide

Complete walkthrough: deploy Moltbot on Railway and expose it through a Cloudflare Tunnel with your custom domain.

**Architecture:**

```
User ─── HTTPS ──→ Cloudflare Edge ──→ cloudflared (Railway) ──→ moltbot-gateway:8080 (Railway)
```

---

## Prerequisites

- A [Railway](https://railway.com) account (Hobby plan or above for volumes)
- A [Cloudflare](https://cloudflare.com) account (free plan works)
- A domain with DNS managed by Cloudflare
- (Optional) Telegram / Discord bot tokens ready

---

## Part 1 — Deploy moltbot-gateway on Railway

### Step 1: Create the Railway project

1. Go to https://railway.com/deploy/moltbot-railway-template
2. Click **Deploy Now**
3. Railway creates a project with the `moltbot-gateway` service from the GitHub repo

> If you prefer a manual setup: create a new project, add a service from the `moltbot/moltbot` GitHub repo.

### Step 2: Attach a volume

1. In your Railway project, click the `moltbot-gateway` service
2. Go to **Settings → Volumes**
3. Click **Add Volume**
4. Set the mount path to `/data`
5. Click **Save**

This is where Moltbot stores config, credentials, sessions, and workspace data. Without a volume, everything is lost on redeploy.

### Step 3: Set environment variables

Go to **Variables** in the `moltbot-gateway` service and add:

| Variable | Value | Required? |
|----------|-------|-----------|
| `SETUP_PASSWORD` | A strong password (you'll use this to access `/setup`) | Yes |
| `PORT` | `8080` | Yes |
| `CLAWDBOT_STATE_DIR` | `/data/.clawdbot` | Recommended |
| `CLAWDBOT_WORKSPACE_DIR` | `/data/workspace` | Recommended |
| `CLAWDBOT_GATEWAY_TOKEN` | A random secret string (admin API access) | Recommended |

Leave `MOLTBOT_NODE` and `MOLTBOT_CONFIG_PATH` blank — they are not needed for this setup.

### Step 4: Enable networking

1. Go to **Settings → Networking**
2. Enable **Private Networking** (this gives the service an internal hostname like `moltbot-gateway.railway.internal`)
3. Optionally enable **Public Networking** on port `8080` for initial testing — you can disable this later once the Cloudflare Tunnel is working

### Step 5: Deploy

Railway auto-deploys when you save settings. Check the **Deploy Logs** tab — wait until you see:

```
Gateway listening on 0.0.0.0:8080
```

If the service restarts or crashes, check the logs for missing env vars.

---

## Part 2 — Create the Cloudflare Tunnel

### Step 6: Create a tunnel in Cloudflare

1. Open https://one.dash.cloudflare.com/
2. Select your account
3. Go to **Networks → Tunnels** (left sidebar)
4. Click **Create a tunnel**
5. Select **Cloudflared** as the connector type
6. Name it `moltbot-railway` (or whatever you prefer)
7. Click **Save tunnel**
8. On the "Install and run a connector" page, find the **token** — it's the long string after `--token` in the install command. **Copy this token.**

> You only need the token value, not the full install command. Railway will run cloudflared for you.

### Step 7: Configure the public hostname

Still in the tunnel configuration page in Cloudflare:

1. Go to the **Public Hostname** tab
2. Click **Add a public hostname**
3. Fill in:

| Field | Value |
|-------|-------|
| Subdomain | `bot` (or your choice) |
| Domain | Select your Cloudflare-managed domain |
| Type | `HTTP` |
| URL | `moltbot-gateway.railway.internal:8080` |

4. Expand **Additional application settings → TLS**
   - Set **No TLS Verify** to **enabled**
5. Expand **Additional application settings → HTTP Settings** (if available)
   - Set **WebSockets** to **enabled**
6. Click **Save hostname**

> The URL `moltbot-gateway.railway.internal` is Railway's internal DNS name for your service. If you renamed the service in Railway, use that name instead.

Cloudflare auto-creates a DNS CNAME record: `bot.yourdomain.com → <tunnel-id>.cfargotunnel.com` (proxied).

---

## Part 3 — Add cloudflared to Railway

### Step 8: Add a new service in Railway

1. In your Railway project, click **+ New Service**
2. Choose **Docker Image**
3. Enter: `cloudflare/cloudflared:latest`
4. Name the service `cloudflared` (optional but clearer)

### Step 9: Configure the cloudflared service

**Environment variable:**

| Variable | Value |
|----------|-------|
| `TUNNEL_TOKEN` | Paste the token from Step 6 |

**Start command override** (set in Settings → Deploy → Custom Start Command):

```
cloudflared tunnel --no-autoupdate run
```

**No volume** needed — cloudflared is stateless.

**No public networking** needed — it only makes outbound connections to Cloudflare.

### Step 10: Deploy and verify the tunnel

1. Railway auto-deploys the cloudflared service
2. Check the deploy logs — you should see:
   ```
   INF Starting tunnel
   INF Connection registered
   INF Registered tunnel connection connIndex=0
   ```
3. Go back to Cloudflare Zero Trust → Tunnels — your tunnel should show as **Healthy**

---

## Part 4 — Configure Moltbot Gateway

### Step 11: Run the setup wizard

1. Open `https://bot.yourdomain.com/setup` in your browser
2. Enter the `SETUP_PASSWORD` you set in Step 3
3. The setup wizard walks you through:
   - Choosing a model provider (e.g. Anthropic, OpenAI) and pasting your API key
   - (Optional) Adding Telegram / Discord / Slack tokens
4. Click **Run setup**

### Step 12: Gateway config adjustments

After setup, access the Control UI at `https://bot.yourdomain.com/moltbot` and update the config:

**Enable insecure auth** (required — the internal hop is HTTP, TLS terminates at Cloudflare):

```json5
{
  gateway: {
    controlUi: {
      allowInsecureAuth: true
    }
  }
}
```

**Trusted proxies (optional):** Only needed if you want real client IP addresses in logs. Since Railway internal IPs can change on redeploy, skip this initially. Token auth works without it.

```json5
{
  gateway: {
    trustedProxies: ["<cloudflared-internal-ip>"]
  }
}
```

To find the IP: check cloudflared's Railway logs for its internal IP, or run a test request and look at the gateway logs.

---

## Part 5 — Disable Railway Public Domain (optional)

### Step 13: Lock down to Cloudflare only

Once you've confirmed everything works through your custom domain:

1. Go to the `moltbot-gateway` service in Railway
2. Go to **Settings → Networking**
3. Remove or disable the **Public Domain** (the `.up.railway.app` URL)

All traffic now goes exclusively through the Cloudflare Tunnel, giving you DDoS protection, WAF, and zero-trust access controls.

---

## Verification Checklist

- [ ] Cloudflare Zero Trust dashboard shows tunnel as **Healthy**
- [ ] Railway logs: `moltbot-gateway` shows `Gateway listening on 0.0.0.0:8080`
- [ ] Railway logs: `cloudflared` shows `Connection registered`
- [ ] `https://bot.yourdomain.com/setup` loads and accepts password
- [ ] `https://bot.yourdomain.com/moltbot` loads and WebSocket connects
- [ ] Send a test message through a configured channel (Telegram, Discord, etc.)

---

## Troubleshooting

### Tunnel shows "Down" in Cloudflare

- Check cloudflared Railway logs for errors
- Verify `TUNNEL_TOKEN` is correct (no extra spaces or quotes)
- Make sure the start command is exactly: `cloudflared tunnel --no-autoupdate run`

### "502 Bad Gateway" when accessing your domain

- The moltbot-gateway service might not be running — check Railway logs
- Verify the hostname URL in Cloudflare is `moltbot-gateway.railway.internal:8080` (exact service name matters)
- Confirm **Private Networking** is enabled on the moltbot-gateway service

### Setup page loads but shows auth errors

- Set `allowInsecureAuth: true` in the gateway config (see Step 12)
- The internal connection is HTTP; auth requires this flag when TLS isn't end-to-end

### WebSocket doesn't connect in Control UI

- Verify **WebSockets** is enabled in the Cloudflare tunnel hostname settings
- Check browser console for connection errors

### Railway service keeps restarting

- Check logs for missing env vars (`SETUP_PASSWORD`, `PORT`)
- Ensure the volume is mounted at `/data`
- Verify `PORT` is set to `8080`

---

## Summary of all services

| Service | Image | Port | Networking | Volume | Key env vars |
|---------|-------|------|------------|--------|-------------|
| `moltbot-gateway` | From GitHub repo (Dockerfile) | 8080 | Private (internal) | `/data` | `SETUP_PASSWORD`, `PORT`, `CLAWDBOT_STATE_DIR`, `CLAWDBOT_WORKSPACE_DIR` |
| `cloudflared` | `cloudflare/cloudflared:latest` | — | None needed | — | `TUNNEL_TOKEN` |
