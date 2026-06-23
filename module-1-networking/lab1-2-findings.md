# Lab 1.2 Findings — Web Server + TLS Certificate

## My VM Details
- Provider: OCI (Oracle Cloud Infrastructure)
- Region: ap-hyderabad-1
- OS: Ubuntu 22.04
- VM IP: 140.245.192.248

---

## Step 1 — Nginx Serving HTTP and HTTPS

![HTTP and HTTPS responses](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab1-2-1-image.png)

Both commands returned the custom page successfully:

```
curl http://localhost
curl http://140.245.192.248
→ <h1>COGS Lab — TLS Demo</h1><p>Hello from InstaSafe Support Lab!</p>
```

- HTTP on port 80 is working ✓
- Custom HTML page is being served by nginx ✓
- nginx is configured to redirect HTTP → HTTPS automatically ✓

---

## Step 2 — Firewall Configuration

![UFW Status](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab1-2-2-image.png)

`sudo ufw status` shows all required ports are open:

| Port | Action | From |
|---|---|---|
| 80/tcp | ALLOW | Anywhere |
| 443/tcp | ALLOW | Anywhere |
| 22/tcp | ALLOW | Anywhere |

**Port scan from laptop confirms ports are publicly reachable:**

![nmap from laptop](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab1-2-3-image.png)

```
nmap -p 80,443 140.245.192.248
PORT    STATE  SERVICE
80/tcp  open   http
443/tcp open   https
```

**Windows connectivity test:**

![Test-NetConnection](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab1-2-5-image.png)

```
Test-NetConnection -ComputerName 140.245.192.248 -Port 80
TcpTestSucceeded : True
```

> **Note — RCA (Root Cause Analysis):** Even after configuring ufw and OCI Security List correctly, port 80 was still blocked. The root cause was a hidden iptables rule that OCI's default Ubuntu image ships with:
> ```
> sudo iptables -D INPUT -j REJECT --reject-with icmp-host-prohibited
> ```
> This rule sits underneath ufw and silently blocks all incoming traffic except SSH. Removing it fixed the issue.

---

## Step 3 — TLS Certificate Details

![Certificate details](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab1-2-4-image.png)

Certificate generated with:
```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/nginx/ssl/selfsigned.key \
    -out /etc/nginx/ssl/selfsigned.crt \
    -subj '/C=IN/ST=Karnataka/L=Bangalore/O=InstaSafe Lab/CN=140.245.192.248'
```

| Field | Value |
|---|---|
| Subject | C=IN, ST=Karnataka, L=Bangalore, O=InstaSafe Lab, CN=140.245.192.248 |
| Issuer | C=IN, ST=Karnataka, L=Bangalore, O=InstaSafe Lab, CN=140.245.192.248 |
| Valid From | Jun 23 12:11:14 2026 GMT |
| Expiry Date | Jun 23 12:11:14 2027 GMT (365 days) |
| Public Key Algorithm | rsaEncryption |
| Key Size | 2048 bit |
| Signature Algorithm | sha256WithRSAEncryption |
| TLS Version used | TLSv1.3 |
| Cipher Suite | TLS_AES_256_GCM_SHA384 |

---

## Step 4 — SSL Error Explanation

### curl -k (ignoring certificate warning) ✓
```bash
curl -k https://140.245.192.248
→ <h1>COGS Lab — TLS Demo</h1><p>Hello from InstaSafe Support Lab!</p>
```
Works fine — `-k` tells curl to skip certificate trust verification.

### curl without -k (strict check) ✗
```bash
curl https://140.245.192.248
→ curl: (60) SSL certificate problem: self signed certificate
```

**Why does curl without `-k` fail?**

Every browser and tool like curl maintains a list of trusted Certificate Authorities (CAs) — organisations like Google Trust Services, DigiCert, and Let's Encrypt that are globally recognised as trustworthy. When you connect to a website, the server presents its certificate and curl checks: "Was this certificate signed by someone I trust?"

Our certificate was signed by `InstaSafe Lab` — which is not on any trusted CA list. We signed it ourselves, which is exactly what "self-signed" means. curl sees this and refuses to connect because it cannot verify the identity of the server.

The data is still encrypted — the security is real. The problem is only trust, not encryption.

**What would need to change to make it succeed?**

Two options:
1. **Get a certificate from a trusted CA** — for example, Let's Encrypt issues free trusted certificates. If we ran `certbot` and got a Let's Encrypt certificate for our domain, curl and browsers would trust it automatically with no warnings.
2. **Add our certificate to the system's trusted CA store** — on Linux: `sudo cp selfsigned.crt /usr/local/share/ca-certificates/ && sudo update-ca-certificates`. This tells the system to trust our specific certificate, but only on that one machine.


---

## Step 5 — TLS Version Security Check

```bash
openssl s_client -connect 140.245.192.248:443 -tls1_1 2>&1 | grep 'Protocol\|Alert'
(empty output)
```

Empty output means TLS 1.1 was completely rejected — the connection was refused before any negotiation happened. nginx is configured to only accept TLSv1.2 and TLSv1.3.

**Why disable TLS 1.1?**
TLS 1.1 (released 2006) has known vulnerabilities and was officially deprecated in 2021. Allowing it would expose the server to downgrade attacks where an attacker tricks the connection into using the weaker older protocol. Only TLS 1.2 and 1.3 are considered secure today.

---

## Key Takeaways

| What I did | What I learned |
|---|---|
| Installed nginx and served a custom page | How a web server is set up and configured |
| Opened ports 80 and 443 in ufw and OCI | Two firewall layers exist — OS-level (ufw) and cloud-level (OCI Security List) |
| Fixed hidden iptables rule | OCI Ubuntu images have a default block rule underneath ufw |
| Generated a self-signed certificate | How TLS certificates are created and what fields they contain |
| Tested curl with and without -k | The difference between encryption and trust in HTTPS |
| Disabled TLS 1.1 | Why old protocol versions are a security risk |
