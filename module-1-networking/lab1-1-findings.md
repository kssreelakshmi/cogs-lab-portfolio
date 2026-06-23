# Lab 1.1 Findings

## My VM Details
- Provider: OCI (Oracle Cloud Infrastructure)
- Region: ap-hyderabad-1
- OS: Ubuntu 22.04

![SSH Connection](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/ssh_connection.png)

---

## Experiment Results

### Ping to 8.8.8.8

![Ping Test](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab1-1-1-image.png)

- Average RTT: **11.1 ms**
- TTL value observed: **118**
- What does TTL tell us about the path?  
  TTL starts at 128 on the server side. We received 118, so 128 − 118 = **10 routers** sit between my VM and Google's server. Each router decrements TTL by 1 as the packet passes through it.

---

### Traceroute to google.com

![Traceroute](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab1-1-2-image.png)

- Number of hops: **7**
- Any `* * *` hops? **Yes — at hop 5 and hop 6**  
  These routers did not reply to traceroute probes. This is normal — backbone routers are often configured to silently drop ICMP packets for security reasons. The destination (hop 7) was still reached successfully.

---

### DNS Comparison

![DNS Lookup](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab1-1-3-image.png)
![dig output](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-1-1-3-1-image.png)

- Result from default DNS (127.0.0.53): **142.251.43.142**
- Result from 1.1.1.1 (Cloudflare): **142.250.205.206**
- Are they different? **Yes**
- Why might they differ?  
  Google uses Anycast routing — the same domain name can map to different IP addresses depending on which DNS server you ask and where you are located. Each DNS server routes you to the nearest or least-busy Google server at that moment. Getting a different IP is completely normal and expected.

---

### google.com TLS Certificate
![TLS Certificate](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab1-1-6-image.png)

- Issuer: **Google Trust Services, CN = WE2**
- Expiry date: **Aug 17 08:36:27 2026 GMT**
- TLS version used: **TLSv1.3**
- Cipher suite: **TLS_AES_256_GCM_SHA384**
- Certificate subject: **CN = *.google.com** (wildcard — covers all google.com subdomains)
