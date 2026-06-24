# Lab 1.3 — WireGuard VPN: Build Your Own Encrypted Tunnel

## My Setup
- **Cloud Provider:** Oracle Cloud Infrastructure (OCI)
- **Region:** ap-hyderabad-1
- **OS:** Ubuntu 22.04 on both VMs
- **VM1 (vpn-server):** Public IP `140.245.218.252` · Private IP `10.0.0.40`
- **VM2 (vpn-client):** Public IP `140.245.253.205` · Private IP `10.0.0.143`

---

## Part 1 — Both VMs Configured and WireGuard Running 

I installed WireGuard on both VMs, generated key pairs, and configured the tunnel. Here is the `wg show` output from both sides confirming the tunnel is active and the handshake was successful.

**VM1 (vpn-server) — wg show:**
- Interface wg0 is up, listening on UDP port 51820
- Peer (VM2) is connected with a recent handshake
- Data transfer is happening in both directions

![wg show VM1](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab1-3-1-image.png)

**VM2 (vpn-client) — wg show:**
- Interface wg0 is up and connected to VM1's endpoint
- Latest handshake confirmed within seconds
- Persistent keepalive active every 25 seconds

![wg show VM2](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab1-3-2-image.png)

---

## Part 2 — Successful Ping Through the Tunnel 

> **Note on IP address:** The lab guide uses `10.0.0.1` for the WireGuard server address. I had to use `10.8.0.1` instead because OCI had already assigned the `10.0.0.x` range to the VMs' physical network (VM1 was `10.0.0.40`, VM2 was `10.0.0.143`). Using `10.0.0.1` for WireGuard would have created a routing conflict and broken the tunnel. The tunnel itself works exactly as intended — only the IP range is different.

I ran `ping -c 5 10.8.0.1` from VM2. All 5 packets went through the encrypted WireGuard tunnel and came back with 0% packet loss.

![ping 10.8.0.1](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab1-3-3-image3.png)

---

## Part 3 — tcpdump Showing Encrypted UDP Traffic 

While the ping was running on VM2, I ran `sudo tcpdump -i ens3 -n udp port 51820` on VM1 to capture the traffic.

What we can see in the screenshot: packets flying between `140.245.253.205` (VM2) and `10.0.0.40` (VM1) on port 51820. The length is always 128 bytes — that's the WireGuard encrypted wrapper. The actual ping data (ICMP packets) is completely hidden inside. There is no readable content visible — no IP headers from inside the tunnel, no "PING" strings, nothing. That's the encryption working.

![tcpdump encrypted UDP](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab1-3-4-image.png)

---

## Part 4 — WireGuard → InstaSafe ZTNA Component Mapping 

### Why I Had to Deviate from the Lab Guide (OCI-Specific Issues)

Before the mapping, I want to document the real issues I ran into — because debugging these actually helped me understand how WireGuard works at a deeper level.

**Problem 1 — IP conflict with 10.0.0.x:**
The lab says to use `10.0.0.1/24` for the WireGuard tunnel. But OCI assigns VMs to the `10.0.0.x` subnet by default. If I added a WireGuard interface with `10.0.0.1/24`, the Linux kernel would have two routes for the same network — one pointing to the physical `ens3` interface and one pointing to `wg0`. This breaks routing completely and could lock you out of SSH. I switched to `10.8.0.x` which is the standard WireGuard convention and has no overlap with OCI's network.

**Problem 2 — eth0 vs ens3:**
The lab's iptables rules use `eth0` for the PostUp/PostDown masquerade rules. OCI Ubuntu names the main interface `ens3`. If I had left `eth0` in, the masquerade rule would silently fail — traffic would reach the tunnel but never get forwarded out. I verified the interface name with `ip link show` and updated the rules to use `ens3`.

**Problem 3 — DNS line causing startup failure:**
The client config in the lab includes `DNS = 1.1.1.1`. This requires a package called `resolvconf` to work. It's not installed on OCI Ubuntu by default. With that line in, `wg-quick` would fail to start and throw an error about `resolvconf` not being found. I removed the DNS line and the tunnel started fine.

**Problem 4 — OCI's double firewall:**
Opening UDP 51820 in OCI's Security List (the VCN-level firewall) was not enough. The VM itself also has iptables rules that block incoming traffic. Even with UFW allowing 51820, the tunnel wouldn't handshake until I added a direct iptables INPUT rule: `sudo iptables -I INPUT -p udp --dport 51820 -j ACCEPT`. Cloud providers have two firewall layers — always check both.

---

### The Mapping


| WireGuard Component | InstaSafe ZTNA Equivalent | Explanation |
|---|---|---|
| **VM2 — WireGuard Client** | **InstaSafe Agent** | VM2 is the "user device" that initiates the tunnel. The InstaSafe Agent on a user's laptop does the same thing — it reaches out, authenticates, and creates the encrypted connection. Neither can be reached from the outside; they always initiate outward. |
| **VM1 — WireGuard Server** | **InstaSafe Gateway** | VM1 accepts the tunnel and sits in front of the network. The InstaSafe Gateway does exactly this — it's the verified entry point that only lets authenticated devices in and routes their traffic to allowed resources. |
| **Public/Private Key Pair** | **Device Certificate (mTLS)** | Before any tunnel forms, both sides exchange public keys. WireGuard uses raw asymmetric key pairs; ZTNA uses mutual TLS certificates. The concept is identical — prove your identity cryptographically before any data flows. |
| **Encrypted WireGuard Tunnel (UDP 51820)** | **ZTNA Encrypted Session** | All traffic between VM2 and VM1 is wrapped in encryption. The tcpdump screenshot proves this — you can see packets but can't read what's inside. ZTNA works the same way; no traffic travels unencrypted between the Agent and Gateway. |
| **AllowedIPs = 10.8.0.2/32 on Server** | **Access Policy / Micro-segmentation** | The server only accepts packets from exactly one IP — `10.8.0.2`. Nothing else is allowed through. In ZTNA, the Controller enforces the same idea with access policies — users only reach the specific resources they are authorized for, nothing more. |
| **PersistentKeepalive = 25** | **Agent Heartbeat** | VM2 sends a small packet every 25 seconds to keep the tunnel alive through NAT. The InstaSafe Agent sends similar periodic signals to the Gateway to maintain the session. Without this, idle tunnels drop in cloud environments. |
| **wg genkey / wg pubkey (key enrollment)** | **Controller issuing certificates** | Before the tunnel works, both sides generate keys and share their public keys manually. In ZTNA, the Controller automates this — it issues and manages certificates for every Agent and Gateway during enrollment. Same trust-building step, just automated at scale. |

---

## What I Learned From This Lab

**1. Always check your existing network before picking a VPN IP range.**
I would have saved time if I'd run `ip addr` first and noticed OCI was using `10.0.0.x`. One minute of checking prevents an hour of debugging.

**2. Cloud providers have two firewall layers.**
Security List (OCI level) + iptables (VM level). Both need to allow the port.

**3. Interface names matter in iptables rules.**
`eth0` is not universal. On OCI Ubuntu it's `ens3`. Always confirm with `ip link show` before writing any iptables rules.

**4. tcpdump is the simplest way to prove a tunnel is encrypting.**
We don't need fancy tools. Just run tcpdump on the physical interface and check that the content is unreadable. If the ping data is in plaintext, the tunnel isn't working.

**5. WireGuard is simpler than I expected — and that's the point.**
No CA, no certificates, no complex config. Just keys and IPs. It's fast to set up, hard to misconfigure once we understand the IP routing, and the encryption is modern.
---

## Final Checklist

| Criteria | Status |
|---|---|
| WireGuard installed on both VMs | ✅ |
| Key pairs generated and exchanged correctly | ✅ |
| wg show on VM1 — active peer, handshake confirmed | ✅ |
| wg show on VM2 — active handshake, data transfer | ✅ |
| Ping through tunnel — 5/5 packets, 0% loss | ✅ |
| tcpdump — encrypted UDP only, no readable plaintext | ✅ |
| WireGuard → ZTNA component mapping written | ✅ |
| OCI deviation documented with explanation | ✅ |
