# Lab 3.1 — Uptime Kuma: Build Your Own Site24x7

## My Setup
- **Cloud Provider:** Oracle Cloud Infrastructure (OCI)
- **Region:** ap-hyderabad-1
- **Monitoring VM (VM1):** vpn-server — Public IP `140.245.218.252`
- **Target VM (VM2):** vpn-client — Public IP `140.245.253.205` · Private IP `10.0.0.143`
- **Tool:** Uptime Kuma (Docker container)
- **Web Server:** Nginx (on VM2)
- **Email Notifications:** Mailtrap Email Sandbox (SMTP)

---

## Part 1 — Deploying Uptime Kuma

I deployed Uptime Kuma on VM1 using Docker. The setup itself was straightforward — one command and it was running:

```bash
docker pull louislam/uptime-kuma:1

docker run -d --restart=always --name uptime-kuma \
  -p 3001:3001 \
  -v uptime-kuma:/app/data \
  louislam/uptime-kuma:1
```

I then opened port 3001 in OCI's Security List and added an iptables rule on VM1 to allow the traffic through. Accessing `http://140.245.218.252:3001` showed the Uptime Kuma setup page where I created the admin account.

**Uptime Kuma dashboard after first login:**
![Uptime Kuma dashboard](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-3-1-1-image.png)

---

## Part 2 — Creating the Three Monitors

I installed Nginx on VM2 first:

```bash
sudo apt install -y nginx
sudo systemctl start nginx
```

Then I created three monitors in Uptime Kuma, each testing a different aspect of VM2's health.

**Monitor 1 — HTTP(S) Monitor**
This checks whether the Nginx web server is actually serving requests. If the web server crashes but the VM is still up, ping and SSH would still show green — but this monitor would catch the application failure.

- Type: HTTP(S)
- Name: Lab Nginx Server
- URL: `http://10.0.0.143`
- Interval: 60 seconds

I used the private IP `10.0.0.143` here rather than the public IP. When I first tried the public IP, the monitor kept timing out even though Nginx was running fine. The issue was that both VMs are in the same OCI subnet, so traffic going out to the public IP and back in gets blocked by OCI's routing. Using the private IP routes traffic directly within the subnet and works reliably.

**Monitor 2 — TCP Port Monitor**
This checks whether port 22 (SSH) is accepting connections. Even if Nginx is down, SSH being up means the VM is alive and accessible for troubleshooting.

- Type: TCP Port
- Name: Lab SSH Port
- Hostname: `10.0.0.143`
- Port: 22
- Interval: 60 seconds

**Monitor 3 — Ping Monitor**
This is the most basic check — is the VM reachable at all? If this goes red, it means the VM itself is down or unreachable, not just a specific service.

- Type: Ping
- Name: Lab VM Reachability
- Hostname: `10.0.0.143`
- Interval: 30 seconds

The Ping monitor had a rocky start. ICMP traffic was initially blocked and the monitor showed Down even though the VM was running fine. I had to allow ICMP through OCI's Security List and verify connectivity separately before it stabilized.

**All 3 monitors showing green — Up: 3, Down: 0:**
![Three green monitors](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-3-1-2-image.png)

---

## Part 3 — Triggering a Failure

With all three monitors green, I SSH'd into VM2 and stopped Nginx:

```bash
sudo systemctl stop nginx
```

Within about 60 seconds, Uptime Kuma detected the failure. The HTTP monitor turned red while SSH and Ping stayed green — which makes perfect sense. The VM is still alive and reachable, only the web server is down.

The dashboard showed:
- **Lab Nginx Server** → Down (`connect ECONNREFUSED 140.245.253.205:80`)
- **Lab SSH Port** → Up
- **Lab VM Reachability** → Up
- **Quick Stats:** Up: 2, Down: 1

The event timeline at the bottom also logged the exact timestamp of the failure with the error message — `connect ECONNREFUSED` means the connection was actively refused, confirming Nginx stopped listening on port 80.

**Lab Nginx Server showing red — HTTP monitor down:**
![HTTP monitor down](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-3-1-3-image.png)

---

## Part 4 — Recovery

I restarted Nginx on VM2:

```bash
sudo systemctl start nginx
```

Within the next check cycle (about 60 seconds), Uptime Kuma detected the recovery. The HTTP monitor went back to green and the incident timeline updated with the recovery timestamp. This is exactly how a real monitoring tool would show an incident — down at a specific time, up at a specific time, and the duration in between.

**Recovery — all monitors back to green:**
![Recovery](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-3-1-4-image.png)

---

## Part 5 — Email Notifications via Mailtrap

I initially tried to configure Gmail SMTP for alert notifications. This didn't work because Google requires App Passwords, and my account's security settings didn't allow generating one even with 2FA enabled.

I switched to Mailtrap Email Sandbox instead — a service designed exactly for testing email flows without actually sending emails to real inboxes. The SMTP configuration was:

| Setting | Value |
|---|---|
| SMTP Host | `sandbox.smtp.mailtrap.io` |
| Port | 587 |
| Security | None / STARTTLS |
| Username | `8989d245d3952c` |
| From | `Uptime Kuma <uptime@example.com>` |
| To | `kssreelakshmi2211@gmail.com` |

**SMTP notification configured in Uptime Kuma:**
![SMTP config](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-3-1-5-image.png)

I sent a test notification and it arrived in Mailtrap's inbox within seconds.

**Test alert email received in Mailtrap:**
![Mailtrap email](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-3-1-6-image.png)

---

## Issues I Ran Into

**HTTP monitor timing out with public IP:** When I configured the HTTP monitor using VM2's public IP, it kept timing out. Both VMs are in the same OCI VCN, so outbound traffic to a public IP from within the same subnet gets handled differently. Switching to the private IP `10.0.0.143` fixed it immediately — traffic stays within the subnet and doesn't need to go through OCI's internet gateway.

**Ping monitor failing initially:** ICMP traffic was blocked at the OCI Security List level. I had to add an ICMP ingress rule to allow ping from within the VCN. Once that was done, the Ping monitor came up and stayed stable.

**Gmail SMTP not working:** Google's App Passwords feature wasn't available for my account configuration. Mailtrap was a better solution for this lab anyway — it's designed for exactly this kind of testing and shows the full email content including headers.

---

## Monitor Type Mapping: Uptime Kuma → Site24x7

| Uptime Kuma Monitor | Site24x7 Equivalent | What it checks |
|---|---|---|
| **HTTP(S)** | **Website Monitor** | Makes an HTTP request and checks the response code. Detects if a web server or API is returning errors, even if the server is technically running. `200 OK` = healthy, `5xx` or timeout = problem. |
| **TCP Port** | **Port Monitor** | Attempts a TCP connection to a specific port. Doesn't check HTTP responses — just checks if something is listening on that port. Useful for SSH (22), HTTPS (443), database ports, VPN ports. |
| **Ping** | **Server Availability Monitor / Ping Monitor** | Sends ICMP echo requests. Checks raw network reachability. If this fails, the server is completely unreachable — firewall block, VM crash, or network outage. |
| **DNS** | **DNS Monitor** | Checks if a domain resolves to the expected IP. Catches DNS misconfigurations or hijacking. |
| **Docker Container** | **Process Monitor** | Checks if a specific Docker container is running. Site24x7 has a similar process/service check. |
| **Keyword** | **Content Check / Web Page Speed Monitor** | Checks that a specific word or phrase appears in the HTTP response body. Useful for detecting when a page loads but shows an error message instead of real content. |

---

## Which Monitor Would I Use for an InstaSafe Gateway?

For an InstaSafe Gateway, I wouldn't rely on a single monitor type — I'd use all three in combination, because each one catches a different class of failure.

**TCP Port Monitor on port 443** would be my primary monitor. The InstaSafe Gateway communicates over HTTPS/443 for the management plane and the agent connections. If this port stops responding, agents can't connect and users lose access — even if the VM is still up and pingable. This is the most business-critical check.

**Ping Monitor** would be my secondary monitor. If the gateway becomes completely unreachable (VM crash, network partition, OCI maintenance), ping fails first. This gives me an early warning before I even know which service is affected. It also helps distinguish between "the gateway service crashed" (ping up, TCP down) vs "the whole VM is gone" (both down).

**HTTP(S) Monitor on the gateway's management portal** (if it exposes one) would be my third monitor. This validates that not only is the port open, but the application is actually responding correctly. A gateway that hangs and stops serving requests would pass the TCP check but fail the HTTP check.

The combination gives me three layers of visibility:
- Is the VM reachable? (Ping)
- Is the VPN/HTTPS port accepting connections? (TCP Port)
- Is the application actually responding? (HTTP)

In a real on-call scenario, if I get a Ping alert, I know the whole VM is down and need to check OCI console. If I get a TCP alert but Ping is fine, I know the gateway service crashed and need to restart it. If I get an HTTP alert but TCP is fine, I know the service is running but something is wrong at the application layer. This tiered approach is exactly how Site24x7 is used in enterprise environments to quickly narrow down the root cause of an outage.

---

## What I Learned

**Private IP vs public IP for internal monitoring.** When both the monitoring server and the target are in the same cloud network, always use the private IP. Traffic between VMs in the same VCN doesn't need to go through the internet gateway, and some cloud providers block or route this traffic differently when you use the public IP.

**Three monitor types catch three different failure modes.** Ping tells you the VM is alive. TCP tells you the service port is open. HTTP tells you the application is actually working. You need all three to fully understand an outage.

**ECONNREFUSED vs timeout are different errors.** When I stopped Nginx, Uptime Kuma showed `connect ECONNREFUSED` — not a timeout. This is an important distinction: ECONNREFUSED means the VM actively rejected the connection (nothing listening on port 80), while a timeout means the VM isn't responding at all (firewall, network issue, VM down). In real support situations, reading the error message tells you immediately which problem you're dealing with.

**Mailtrap is better than Gmail for testing email alerts.** Gmail's security restrictions make it hard to use for SMTP in lab environments. Mailtrap is designed for exactly this — it accepts any email sent to it and shows the full content without actually delivering to real inboxes. In production, you'd use a real SMTP relay, but for testing alert flows, Mailtrap is the right tool.

**Uptime Kuma's incident timeline is as important as the dashboard.** The green/red status tells you the current state. The timeline tells you when it went down, when it came back, and how long the outage lasted — which is what you actually need for incident reports and postmortems.

---

## Final Checklist

| Criteria | Status |
|---|---|
| Uptime Kuma deployed via Docker | ✅ |
| All 3 monitors created | ✅ |
| All 3 monitors showing green | ✅ |
| HTTP monitor failure triggered | ✅ |
| Red alert captured with incident timeline | ✅ |
| Recovery captured | ✅ |
| Email notification configured (Mailtrap) | ✅ |
| Test email received | ✅ |
| Monitor type mapping written | ✅ |
| InstaSafe Gateway monitoring rationale written | ✅ |
