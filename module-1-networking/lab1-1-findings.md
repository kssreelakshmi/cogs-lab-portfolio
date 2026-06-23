# Lab 1.1 — Findings

## VM Details

| Field | Details |
|---|---|
| Cloud Service | OCI (Oracle Cloud Infrastructure) |
| Region | ap-hyderabad-1 |
| OS | Ubuntu |

I connected to this cloud VM securely using SSH from my local machine.

![SSH Connection](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/ssh_connection.png)

---

## Experiment 1 — Can my VM reach the internet?

The goal here was simple: check if my VM can send and receive data over the internet.  
I used the `ping` command, which sends small test packets to a server and waits for a reply.

![Ping Test](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab1-1-1-image.png)

---

### Test 1: `ping -c 5 8.8.8.8`

I sent 5 packets to **8.8.8.8**, which is Google's public DNS server — a reliable, always-online address good for testing.

**What `-c 5` means:** Without this flag, ping runs forever. `-c 5` tells it to stop after 5 packets.

**What came back:**
- All 5 packets were returned — **0% packet loss** → internet is working
- Each reply took about **11.1 ms** — that's fast and consistent
- **TTL = 118** — TTL (Time To Live) is a number that counts down by 1 at every router the packet passes through. It starts at 128 (Google's server uses Windows-style TTL). So 128 - 118 = **10 routers** between my VM and Google's server.

> TTL exists as a safety measure — if a packet gets stuck in a loop, TTL eventually hits 0 and the packet is discarded so it doesn't loop forever.

---

### Test 2: `ping -c 3 google.com`

This time I used a **domain name** instead of an IP address.  
Before the ping could even go out, my VM had to figure out: *"What is the IP address of google.com?"*  
This automatic translation is called **DNS resolution**.

**What came back:**
- google.com resolved to **142.250.182.78**
- The reply came from a server named `lcmaaa-ax-in-f14.1e100.net` — this is Google's internal network. (`1e100` = 10^100 = a googol, Google's namesake)
- TTL = 119 → **9 hops** — slightly different route than 8.8.8.8
- **0% packet loss** again ✓

---

### What I learned from Experiment 1

| What I observed | What it means |
|---|---|
| 0% packet loss | My VM has a healthy internet connection |
| TTL = 118–119 | About 10 routers sit between my VM and Google |
| ~11 ms round-trip | Fast connection — VM in Hyderabad is close to Google's India servers |
| google.com → IP automatically | DNS is working correctly |

---

## Experiment 2 — What path does my data take to reach Google?

Ping tells me *if* I can reach a server. **Traceroute** tells me *exactly which routers my packet passes through* to get there.

It works by sending packets with TTL = 1, then 2, then 3... Each router that hits TTL = 0 sends back a "time exceeded" message, revealing its IP. By collecting all these replies, traceroute builds a map of the full route.

![Traceroute](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab1-1-2-image.png)

---

### Test 1: `traceroute google.com`

**The route my packet took:**

| Hop | IP Address | What it is |
|---|---|---|
| 1 | 140.91.228.25 | OCI's internal gateway — first exit point from my VM |
| 2 | 182.72.202.2 | Airtel's network backbone |
| 3 | 182.72.202.1 | Another Airtel router |
| 4 | 116.119.164.82 | Where Airtel hands off traffic to Google |
| 5–6 | `* * *` | Router didn't reply — firewall is blocking probes |
| 7 | 142.251.222.142 | Google's server — destination reached ✓ |

**About the `* * *`:**  
This does NOT mean the connection failed. It means those routers are configured to silently ignore traceroute probes — this is common on large internet backbones. The packet still passed through them, as proven by hop 7 successfully reaching Google.

**The full path:** `My VM → OCI → Airtel → Google`

---

### Test 2: `traceroute -T -p 443 google.com`

This tries to do the same traceroute but using **TCP on port 443** (the same port used for HTTPS/secure websites) instead of ICMP.

Why? Because some firewalls block ICMP (causing `* * *`) but allow HTTPS traffic. Using port 443 would reveal those hidden hops.

**Result:** `Operation not permitted`

This failed because TCP traceroute needs **administrator (root) privileges** on Linux.  
Fix: run it with `sudo traceroute -T -p 443 google.com`

---

### What I learned from Experiment 2

| What I observed | What it means |
|---|---|
| 7 hops total | My data travels through OCI → Airtel → Google |
| `* * *` at hops 5–6 | Those routers block ICMP probes — normal firewall behaviour |
| TCP traceroute failed | Needs `sudo` — Linux restricts raw socket access to root |
| Google gave a different IP than Experiment 1 | Google routes you to the nearest available server (Anycast) |

---

## Experiment 3 — How does the internet know where google.com is?

When we type `google.com`, our computer doesn't understand names — it only understands numbers (IP addresses). **DNS (Domain Name System)** is the Internet's phonebook that does this translation automatically.

Every DNS query goes through **port 53** — that's the universal standard port for DNS.

![DNS Lookup](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab1-1-3-image.png)
![dig output](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-1-1-3-1-image.png)

---

### How DNS resolution actually works (step by step)

```
You type: google.com
          ↓
Our VM asks its local resolver (127.0.0.53)
          ↓  "Is the answer cached?"
          ↓  If not cached:
Root nameserver → "Who handles .com?"
          ↓
.com nameserver → "Who handles google.com?"
          ↓
Google's own nameserver → "The IP is 142.251.222.142"
          ↓
Answer returned to you in milliseconds ✓
```

---

### Test 1: `nslookup google.com` — using my VM's default DNS

| What was returned | Value | What it means |
|---|---|---|
| DNS server used | 127.0.0.53 | My VM's local resolver — it asks on my behalf and caches answers |
| Port | 53 | Standard DNS port |
| IPv4 address | 142.251.43.142 | google.com's IP address |
| IPv6 address | 2404:6800:4007:832::200e | google.com's newer IPv6 address |

**"Non-authoritative answer"** means this resolver **doesn't own** the google.com records. It fetched the answer from Google's real nameserver and cached it. Like getting directions from someone who looked it up on their phone rather than someone who lives there.

---

### Test 2: `nslookup google.com 1.1.1.1` — using Cloudflare's DNS

I manually told `nslookup` to use **1.1.1.1** (Cloudflare's public DNS) instead of my VM's default.

| What was returned | Value | What it means |
|---|---|---|
| DNS server used | 1.1.1.1 | Cloudflare's public DNS server |
| IPv4 address | 142.250.205.206 | **Different IP than Test 1!** |
| Multiple IPs returned | 5 different IPv6 addresses | Google spreading load across many servers |

**Why is the IP different?** Same domain, different answer — this is completely normal. Google uses **Anycast routing**, meaning different DNS servers return different IPs based on which Google server is closest and least busy at that moment.

---

### Test 3: `dig google.com` — same as nslookup but more detailed

`dig` is a more powerful DNS tool. It shows everything `nslookup` shows, plus extra details:

```
google.com.    42    IN    A    142.251.222.142
               ↑               ↑
             TTL=42s          IP address
```

- **TTL = 42 seconds** — the cached answer expires in 42s, then a fresh lookup happens
- **Query time = 1ms** — the answer was already cached locally, no network trip needed
- **MSG SIZE = 55 bytes** — the entire DNS response was just 55 bytes, extremely lightweight

---

### Test 4: `dig google.com @8.8.8.8 +trace` — watching every DNS step live

This bypasses all caches and replays the full DNS lookup from scratch, step by step.

**What it showed:**

| Step | Who was asked | What they said |
|---|---|---|
| 1 | Google's DNS (8.8.8.8) | Here are the 13 root servers that know everything |
| 2 | A root server | I don't know google.com — ask the .com servers |
| 3 | A .com server | I don't know the IP — ask Google's own nameservers |
| 4 | ns2.google.com | The IP is 142.251.222.142 ✓ |

The **RRSIG lines** in the output are **DNSSEC signatures** — cryptographic proof that these DNS records were not tampered with on their way to me. This is the security layer of DNS.

---

### nslookup vs dig — which to use?

| Feature | `nslookup` | `dig` |
|---|---|---|
| Output style | Simple, readable | Detailed, technical |
| Shows TTL | No | Yes |
| Shows query time | No | Yes |
| Can trace full DNS chain | No | Yes (`+trace`) |
| Best for | Quick lookups | Debugging DNS problems |

---

### What I learned from Experiment 3

| What I observed | What it means |
|---|---|
| 127.0.0.53 answered first | My VM uses a local DNS cache before going to the internet |
| Different IPs from different DNS servers | Google uses Anycast — nearest server answers |
| TTL = 42 on the cached answer | DNS answers expire and get refreshed automatically |
| 1ms query time | Local cache hit — DNS doesn't go to the internet every time |
| 13 root servers in the trace | The entire internet's DNS starts from just 13 server addresses |
| RRSIG signatures present | DNSSEC is active — DNS responses are cryptographically verified |
