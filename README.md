# VPN Gate SSTP Auto-Pilot 🚀

**Fully automated VPN Gate SSTP node detection & subscription system — Cloudflare Workers + GitHub Actions, zero maintenance after setup.**

> Inspired by a Chinese blog post; hardened with real-world pitfalls discovered during an actual deployment.
> 100% browser-based setup — no local tooling required.

---

## What This System Does

[VPN Gate](https://www.vpngate.net/) (an academic anti-censorship project by University of Tsukuba) publishes thousands of volunteer-run VPN servers. Among them, **residential broadband nodes** are gold: real home IPs that pass risk checks. The pain: they churn constantly — nodes appear and disappear every 30 minutes, and manual testing is torture.

This system automates everything:

```
┌─────────────────────────────────────────────────────────────┐
│ 1. SOURCE                                                   │
│ VPN Gate public API: volunteer VPN servers worldwide,       │
│ including precious residential-broadband nodes              │
└────────────────────────┬────────────────────────────────────┘
                         ↓ every 30 min (GitHub Actions cron)
┌─────────────────────────────────────────────────────────────┐
│ 2. DETECTION (fork of gate repo + your Check Worker)        │
│ Fetch full node list → filter SSTP-capable (TCP) nodes      │
│ → test each one live via your Cloudflare Check Worker       │
│ → keep survivors, tag residential/datacenter, group by      │
│   country, sort by latency → publish nodes.txt to           │
│   GitHub Pages                                              │
└────────────────────────┬────────────────────────────────────┘
                         ↓ fetched live on every subscription request
┌─────────────────────────────────────────────────────────────┐
│ 3. DISTRIBUTION (edgetunnel Worker on your custom domain)   │
│ Parses nodes.txt lines into:                                │
│   entry  = optimized Cloudflare CDN edge (fast, unblocked)  │
│   chain  = SSTP node credentials (encoded in /video/ path)  │
│ Outputs vless links to your client's subscription           │
└────────────────────────┬────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. YOUR TRAFFIC                                             │
│ Client → optimized edge → your edgetunnel Worker            │
│        → decodes SSTP chain from path → residential node    │
│        → destination website                                │
│ Exit IP = volunteer's home broadband (that's the point)     │
└─────────────────────────────────────────────────────────────┘
```

### Why this design resists blocking

1. The client **never connects to a sensitive domain** — it targets "optimized edges" (low-latency, unblocked Cloudflare CDN entries)
2. The TLS SNI/Host is **your own custom domain** on a fresh zone (never use raw `*.workers.dev` — heavily blocked in some regions; this cost us a full redeployment)
3. **Exit IPs rotate every 30 minutes** automatically — dead residential nodes get replaced in the next detection round
4. Every layer is swappable: entries, exits, and even the domain

---

## Prerequisites

- A **Cloudflare account** (free plan is enough)
- A **GitHub account** (public repos = unlimited Actions minutes)
- A **custom domain** you control (free subdomains work — e.g., from providers that let you set NS records; see Pitfall #2 for why this matters)
- A client supporting **SSTP chained proxy** (v2rayN 7.x+ recommended)

---

## Deployment Steps (browser only)

### Step 1 — Deploy the Check Worker (CheckSocks5)

1. Cloudflare Dashboard → **Workers & Pages** → **Create** → **Create Worker**, name it `check-sstp` → Deploy (placeholder code is fine)
2. Edit code → paste the full content of [`_worker.js` from lsh8848/cm-Workers-CheckSocks5](https://github.com/lsh8848/cm-Workers-CheckSocks5) → Deploy
3. Note the URL: `https://check-sstp.<your-subdomain>.workers.dev`

**Verify**: open `https://check-sstp.<your-subdomain>.workers.dev/check?sstp=vpn:vpn@anything:443` — it should return JSON (either success or failure is fine).

### Step 2 — Deploy edgetunnel (main proxy)

1. Create another Worker named `edgetunnel` → paste [cmliu/edgetunnel `_worker.js`](https://github.com/cmliu/edgetunnel) → Deploy
2. Worker → **Settings** → **Variables and Secrets**: add variable `ADMIN` = your admin password
3. **Settings** → **Bindings**: add a **KV namespace** binding, variable name `KV` (create the namespace when prompted)
4. (Recommended) Bind a **custom domain** — see Step 8
5. Visit `https://<your-worker-domain>/admin`, log in with the ADMIN password, complete the OTP binding, and **note down your UUID and node domain**

> **Don't confuse**: `ADMIN` is the panel login password; `UUID` is the node identifier shown on the panel home. The UUID never goes into any Python code.

### Step 3 — Fork the gate repo

Fork [hezhanleiok/gate](https://github.com/hezhanleiok/gate) to your account.

### Step 4 — Modify 2 files in your fork

**File ①: `vpngate.py`** (~line 98)

```python
NODES_URL = os.environ.get("NODES_URL", "https://YOUR_GITHUB_USERNAME.github.io/gate/nodes.txt")
```

Replace `YOUR_GITHUB_USERNAME` with your GitHub username. (Cosmetic — it's only used in a log hint.)

**File ②: `.github/workflows/check.yml`**

1. Point the check at your own Worker via a secret:

```yaml
env:
  CHECK_WORKER: ${{ secrets.DOMAIN }}
```

2. Add a schedule (the upstream file has **no cron** — you must add it):

```yaml
on:
  workflow_dispatch:
  schedule:
    - cron: "*/30 * * * *"   # every 30 min; hourly = "0 * * * *"
```

### Step 5 — Add the GitHub Secret

Repo → **Settings** → **Secrets and variables** → **Actions** → New repository secret:

- Name: `DOMAIN` (exact)
- Value: `https://check-sstp.<your-subdomain>.workers.dev/check?sstp=vpn:vpn@`

### Step 6 — Enable GitHub Pages

Repo → **Settings** → **Pages** → Source = **GitHub Actions**.

### Step 7 — Enable Actions & first run

1. Repo → **Actions** → if you see *"Workflows on this fork have been disabled"* → click **"I understand my workflows, go ahead and enable them"**
2. Open the **VPN Gate Node Check** workflow → if it shows *Disabled* → **Enable workflow**
3. **Run workflow** → Run workflow
4. Wait ~1 min for a green check

### Step 8 — Bind a custom domain to edgetunnel ⭐ important

1. Add your domain as a zone in Cloudflare (free plan works — subdomains of PSL-listed suffixes count as registrable domains), point its NS to the two Cloudflare nameservers assigned to your zone
2. edgetunnel Worker → **Settings** → **Domains & Routes** → add e.g. `edgetunnel.<your-domain>`
3. **Update the KV config**: edgetunnel stores `HOST`/`HOSTS` in KV on first request and never auto-updates them. If you switch domains, you must update the KV `config.json` value (via dashboard KV editor or API): set `HOST` and `HOSTS` to the new domain
4. Your subscription URL changes (the token is derived from the hostname!): new URL = `https://edgetunnel.<your-domain>/sub?token=<new-token>`, where `<new-token> = MD5(MD5(hostname + UUID).hex[7:27])`

### Step 9 — Verify & connect

1. Open `https://<your-github-username>.github.io/gate/nodes.txt` — you should see lines like:
   ```
   optimized.domain:443#Japan-Residential-01$sstp://vpn:vpn@vpnXXXXXX.opengw.net:port
   ```
2. In edgetunnel admin → **Subscription mode = Custom subscription** → paste your `nodes.txt` URL into the **Custom optimized IPs** box → Save
3. In v2rayN: add subscription `https://edgetunnel.<your-domain>/sub?token=<token>` → update → test latency → pick a node → enable system proxy

---

## Pitfalls We Hit (learn from our pain)

| # | Pitfall | Fix |
|---|---------|-----|
| 1 | **GitHub auto-disables fork Actions** after inactivity (~10 days for us). Cron silently stops. | When data goes stale: Actions → re-enable → Run workflow. 30-second fix. |
| 2 | **`*.workers.dev` is DNS-poisoned AND SNI-blocked** in some regions. Subscription unreachable without an existing proxy (chicken-and-egg). | Bind a custom domain to the Worker. This is THE fix. Free subdomain providers (e.g., KataBump, DigitalPlat) work if they let you set NS records. |
| 3 | **Free DNS providers can have broken authoritative servers.** Our first provider's second NS server served SERVFAIL for the delegation (its own infra migration). Every public resolver failed while their panel claimed "delegated". | Diagnose: query each authoritative server directly over TCP (`nslookup -vc -type=NS <domain> <server-ip>`). File a ticket with the evidence, or switch providers. |
| 4 | **KV config `HOST`/`HOSTS` never auto-update** when you change the Worker's domain — generated links keep the stale SNI. | Update KV `config.json` manually (dashboard KV editor or API). |
| 5 | **Subscription token changes with the hostname** (`MD5(MD5(host+uuid).hex[7:27])`). Old token invalid on new domain. | Recompute or copy the new subscription URL from the admin panel. |
| 6 | The blog tells you to edit `vpngate.yml` — the actual file is **`check.yml`**. | Don't get lost. |
| 7 | edgetunnel's **"Optimized subscription generator" mode is NOT for raw node-list URLs** — it probes another edgetunnel instance and 404s. | Use **Custom subscription mode** → paste the `nodes.txt` URL into the Custom optimized IPs box. |
| 8 | GitHub OAuth inside embedded browsers (webviews) often fails with **Error 400**. | Log in with username/password directly, or use an external browser. |

---

## Maintenance

| Symptom | Cause | Fix |
|---------|-------|-----|
| `nodes.txt` stale for days | GitHub disabled fork Actions (Pitfall #1) | Re-enable + run workflow |
| Nodes all dead | Optimized edge entries blocked | Refresh `EDGE_HOSTS` in `vpngate.py` with fresh speed-tested entries |
| Subscription unreachable | workers.dev blocked (Pitfall #2) | Custom domain (Step 8) |
| Links still point to old domain | KV config not updated (Pitfall #4) | Edit KV `config.json` |

---

## Security Notes

- **VPN Gate is a public volunteer network.** Your traffic exits through strangers' home connections. Fine for browsing; do not use it for banking or sensitive logins.
- Never share your subscription URL — it contains your UUID.
- The `DOMAIN` secret is only readable by your own repo's Actions.

---

## Credits

| Project | Role |
|---------|------|
| [hezhanleiok/gate](https://github.com/hezhanleiok/gate) | Node fetch/detect/publish pipeline |
| [cmliu/edgetunnel](https://github.com/cmliu/edgetunnel) | VLESS proxy + chained proxy + subscription |
| [lsh8848/cm-Workers-CheckSocks5](https://github.com/lsh8848/cm-Workers-CheckSocks5) | SSTP node liveness checker |
| [VPN Gate](https://www.vpngate.net/) | Node data source |
| [Original blog post (Chinese)](https://blog.imxiaohe.com/2026/10/ipcloudflare.html) | The idea this guide is based on |

*This guide documents a real deployment, including every pitfall we actually hit. Good luck.*
