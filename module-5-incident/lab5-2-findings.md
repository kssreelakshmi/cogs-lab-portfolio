# Lab 5.2 — Change Management Simulation: Safe Change on a Live System

## My Setup
- **VM:** vpn-server — OCI ap-hyderabad-1 — Public IP `140.245.218.252`
- **Web Server:** Nginx with self-signed TLS certificate
- **Change ID:** CHANGE-001
- **Change Type:** Normal Change
- **Change:** Upgrade TLS from TLS 1.2+1.3 to TLS 1.3 only
- **Portfolio Repo:** github.com/kssreelakshmi/cogs-lab-portfolio

---

## Pre-Change Setup

Before starting the change, I needed a working TLS setup on Nginx. The original VM from Lab 1.2 was terminated due to an IP issue, so I set up TLS fresh on the current vpn-server:

```bash
sudo mkdir -p /etc/nginx/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/nginx/ssl/selfsigned.key \
    -out /etc/nginx/ssl/selfsigned.crt \
    -subj '/C=IN/ST=Karnataka/L=Bangalore/O=InstaSafe Lab/CN=140.245.218.252'
```

Created the Nginx TLS site config at `/etc/nginx/sites-available/lab-tls` with `ssl_protocols TLSv1.2 TLSv1.3` as the starting state — this is what the change will upgrade from.

Verified the baseline before touching anything:
```bash
openssl s_client -connect localhost:443 -tls1_3 2>/dev/null | grep 'Protocol'
# Protocol: TLSv1.3 ✅
```

---

## Step 1 — Change Plan Committed Before Execution

The first rule of change management: **write the plan before touching anything**. I created the change plan at `changes/CHANGE-001-TLS-Upgrade.md` and committed it to GitHub before running a single command on the server.

```
git commit -m 'CHANGE-001: Change plan — TLS upgrade — PENDING APPROVAL'
```

The git timestamp on this commit proves the plan existed before execution — that's what the grading criteria checks. In a real environment, this commit would trigger a review workflow before anyone is allowed to proceed.

The plan included:
- What the change is and why
- Exact execution steps in order
- Rollback trigger conditions
- Rollback steps with commands
- Estimated rollback time: 2 minutes

📄 **Change Plan:** [CHANGE-001-TLS-Upgrade.md](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/changes/CHANGE-001-TLS-Upgrade.md)

---

## Step 2 — Executing the Change

### Backup First

```bash
sudo cp /etc/nginx/sites-available/lab-tls /etc/nginx/sites-available/lab-tls.backup
```

This is the most important step. Without a backup, rollback requires manually rewriting the config from memory under pressure. With a backup, rollback is one `cp` command.

### Config Change

Updated `ssl_protocols` from `TLSv1.2 TLSv1.3` to `TLSv1.3` only:

```bash
sudo sed -i 's/ssl_protocols TLSv1.2 TLSv1.3;/ssl_protocols TLSv1.3;\n    ssl_prefer_server_ciphers off;/' /etc/nginx/sites-available/lab-tls
```

### Config Test

```bash
sudo nginx -t
# nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
# nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Always test before reloading. `nginx -t` catches syntax errors without touching the running service.

### Apply and Verify

```bash
sudo systemctl restart nginx
```

**TLS 1.3 verification:**
```bash
openssl s_client -connect localhost:443 -tls1_3 2>/dev/null | grep 'Protocol\|Cipher'
# New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384 ✅
```

**TLS 1.2 rejection verification:**
```bash
openssl s_client -connect localhost:443 -tls1_2 2>&1 | tail -15
# Cipher: 0000 — no cipher negotiated
# closed — connection rejected ✅
```

**TLS 1.3 confirmed working, TLS 1.2 rejected:**

<p align="center">
  <img src="https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-5-2-1-image.png" alt="TLS 1.3 confirmed — New TLSv1.3 Cipher is TLS_AES_256_GCM_SHA384" width="800"/>
</p>

**TLS 1.2 rejected — connection closed with no cipher:**

<p align="center">
  <img src="https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-5-2-2-image.png" alt="TLS 1.2 rejected — connection closed no cipher negotiated" width="800"/>
</p>

---

## Step 3 — Rollback Simulation

### Introducing the Bad Config

To simulate a failed change, I deliberately introduced a typo in the config:

```bash
sudo sed -i 's/ssl_protocols/ssl_protocolz/' /etc/nginx/sites-available/lab-tls
sudo nginx -t
```

Result:
```
nginx: [emerg] unknown directive "ssl_protocolz" in /etc/nginx/sites-enabled/lab-tls:6
nginx: configuration file /etc/nginx/nginx.conf test failed
```

This is the rollback trigger — `nginx -t` failed. In a real environment this would be caught before reloading, meaning the running service stays up while you fix the config.

**nginx -t failure — rollback triggered:**

<p align="center">
  <img src="https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-5-2-3-image.png" alt="nginx -t failing with unknown directive ssl_protocolz" width="800"/>
</p>

### Executing the Rollback

```bash
sudo cp /etc/nginx/sites-available/lab-tls.backup /etc/nginx/sites-available/lab-tls
sudo nginx -t && sudo systemctl reload nginx
curl -k https://localhost
```

Result:
```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
<h1>COGS Lab — TLS Demo</h1><p>Hello from InstaSafe Support Lab!</p>
```

Service restored in under 2 minutes. ✅

---

## Git Commit Trail

The full change trail in git:

<p align="center">
  <img src="https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-5-2-5-image.png" alt="Git log showing full commit trail for CHANGE-001" width="800"/>
</p>

- `3d12490` — Change plan committed before execution
- `83e5426` — Execution and rollback documented

---

## What I Learned

**Always test config before reloading.** `nginx -t` is a one-second check that prevents a bad config from taking down a live service. The rollback simulation proved this — I introduced a deliberate typo and nginx -t caught it before the running service was touched. In a real P1 situation, skipping this step could turn a config change into a full outage.

**The backup is your rollback plan.** Writing "restore from backup" in the rollback steps only works if the backup actually exists and was taken right before the change. Taking the backup as Step 1 means rollback is always a single command.

**Git timestamps enforce process.** Committing the change plan before execution isn't just documentation — it's proof of process. In a real change management system, that commit would be the approval trigger. Without the pre-execution commit, you have no evidence the plan existed before the change happened.

**TLS 1.2 rejection confirmation is subtle.** The `openssl s_client -tls1_2` output doesn't show a dramatic error — it shows `Cipher: 0000` and `closed`. That's the rejection. Understanding what a successful rejection looks like is important — if you don't know what to look for, you might think TLS 1.2 is still working when it isn't.

---

## Final Checklist

| Criteria | Status |
|---|---|
| Change plan written before execution | ✅ git commit `3d12490` |
| Backup taken before config change | ✅ lab-tls.backup created |
| nginx -t passed after TLS change | ✅ syntax ok |
| TLS 1.3 confirmed working | ✅ TLS_AES_256_GCM_SHA384 |
| TLS 1.2 confirmed rejected | ✅ Cipher 0000 — closed |
| Bad config introduced for rollback test | ✅ ssl_protocolz typo |
| nginx -t caught bad config before reload | ✅ test failed |
| Rollback executed from backup | ✅ under 2 minutes |
| Service verified after rollback | ✅ curl returning COGS Lab page |
| Full git commit trail | ✅ Plan → Execution → Rollback |
