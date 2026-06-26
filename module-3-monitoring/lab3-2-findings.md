# Lab 3.2 — Prometheus + Grafana: Build Your Own APM Dashboard

## My Setup
- **Cloud Provider:** Oracle Cloud Infrastructure (OCI)
- **Region:** ap-hyderabad-1
- **VM:** vpn-server — Public IP `140.245.218.252`
- **Stack:** Prometheus + Grafana + Node Exporter (Docker Compose)
- **Grafana version:** Latest
- **Prometheus version:** Latest
- **Node Exporter version:** Latest

---

## Part 1 — Deploying the Stack

I created a `docker-compose.yml` with three services — Prometheus (metrics storage and querying), Grafana (visualization), and Node Exporter (system metrics collector). The whole stack came up with a single command:

```bash
docker-compose up -d
```

I had to open ports 9090 (Prometheus), 3000 (Grafana), and 9100 (Node Exporter) in both the OCI Security List and via iptables on the VM — the same double firewall issue I ran into in previous labs.

**All 3 containers running:**
![docker-compose ps](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-3-2-1-image.png)

---

## Part 2 — Prometheus Queries

I opened Prometheus at `http://140.245.218.252:9090` and ran four PromQL queries. Each one represents a metric that support engineers read during incidents.

There was a "Server time is out of sync" warning — this is a minor clock difference between my browser and the OCI VM. It doesn't affect the query results, just means the graph timestamps may be slightly offset.

**Query 1 — CPU Usage (%):**
```
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```
This calculates CPU usage by taking the idle CPU percentage and subtracting from 100. The VM was showing around 37% CPU usage when I first ran this, which dropped as the Docker containers settled.

![CPU Usage graph](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-3-2-2-image.png)

**Query 2 — Memory Usage (%):**
```
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
```
Memory was running at ~73-80%. This is expected — the OCI Always Free VM has 1GB RAM and running Prometheus, Grafana, Node Exporter, and Keycloak (from Lab 2.2) together takes up most of it.

![Memory Usage graph](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-3-2-3-image.png)

**Query 3 — Disk Usage (%):**
```
(1 - node_filesystem_avail_bytes{fstype="ext4"} / node_filesystem_size_bytes{fstype="ext4"}) * 100
```
Note: The lab guide uses `mountpoint="/"` but on this setup node-exporter runs inside Docker and only sees container mounts (`/etc/hostname`, `/etc/hosts`, `/etc/resolv.conf`). I used `fstype="ext4"` instead to get the actual disk data — showing ~16.6% disk usage.

![Disk Usage graph](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-3-2-4-image.png)

**Query 4 — Network Traffic (bytes/sec):**
```
rate(node_network_receive_bytes_total{device="eth0"}[5m])
```
Shows network bytes received per second on eth0 — around 42 bytes/sec at idle, which makes sense for a VM that's just running monitoring tools.

![Network Traffic graph](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-3-2-5-image.png)

---

## Part 3 — Grafana Dashboard

I opened Grafana at `http://140.245.218.252:3000` and logged in with `admin / Admin@Grafana123`.

First I added Prometheus as a data source:
- Go to **Connections → Data Sources → Add → Prometheus**
- URL: `http://prometheus:9090` (uses Docker internal networking — containers talk to each other by service name)
- Save & Test → "Successfully queried the Prometheus API"

Then I built a custom dashboard with 3 panels using the same PromQL queries from Step 2. I switched each query to **Code** mode to enter raw PromQL instead of using the visual builder. Setting the time range to **Last 15 minutes** made the panels load much faster than the default 6 hours.

**Custom dashboard — CPU Usage, Memory Usage, Network Traffic:**
![Grafana custom dashboard](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-3-2-6-image.png)

I also imported the pre-built **Node Exporter Full** dashboard (ID: 1860) from Grafana.com. This is a professional full-VM monitoring dashboard used in real enterprise environments — it shows dozens of metrics including CPU breakdown by mode, memory breakdown, disk I/O, network traffic, and system load, all in one view.

**Node Exporter Full dashboard (ID: 1860):**
![Node Exporter Full dashboard](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-3-2-7-image.png)

---

## Part 4 — Prometheus/Grafana vs SigNoz: When to Use Each

This is the question that connects the lab to real support work.

### What Prometheus + Grafana shows

Prometheus with Node Exporter collects **infrastructure metrics** — CPU, memory, disk, network, system load. These are numbers about the machine itself. Grafana visualizes them as time-series graphs. What you see is:

- Is the CPU spiking?
- Is memory being exhausted?
- Is disk running out?
- Is network bandwidth saturated?

These metrics tell you the **health of the server**. They don't tell you anything about what's happening inside the applications running on that server.

### What SigNoz shows

SigNoz is an **Application Performance Monitoring (APM)** tool. It collects telemetry from inside applications — traces, logs, and application-level metrics. What you see is:

- Which API endpoint is slow?
- Which database query is taking too long?
- Where exactly in the code is the bottleneck?
- What error rate is a specific service seeing?
- What is the latency distribution across microservices?

SigNoz works with OpenTelemetry — applications need to be instrumented to send traces and spans to SigNoz. Without instrumentation, SigNoz has nothing to show.

### When to use each during a P1 incident

**Start with Prometheus/Grafana** when the alert says something is slow or down but you don't know why:

- CPU at 100% → something is consuming all processing power → identify the process
- Memory at 95% → possible memory leak → check which service is growing
- Disk at 99% → logs or data filling up → find and clear the large files
- Network saturated → possible DDoS or runaway process → check traffic by process

Prometheus tells you **the system is unhealthy**. It narrows down which resource is the problem.

**Switch to SigNoz** once you know the system is healthy but users are still complaining:

- CPU and memory are fine, but API responses are slow → look at traces to find which function is slow
- No resource exhaustion, but error rate is high → look at traces to find which service is throwing errors
- Everything looks green on Prometheus, but a specific user workflow is broken → trace that specific request end-to-end

SigNoz tells you **which application component is failing** and exactly where in the code path.

### The practical combination during a real P1

```
Alert fires → Check Prometheus/Grafana first
    ↓
Resource issue found (CPU/memory/disk)?
    → Fix the infrastructure issue
    ↓
No resource issue?
    → Open SigNoz → find the slow trace → find the bad service
    → Escalate to the dev team with the specific span/function that's failing
```

For monitoring an **InstaSafe Gateway** specifically:

Prometheus/Grafana would monitor the gateway VM's resources — if the gateway process is consuming 100% CPU, memory is exhausted, or disk is full, the Prometheus dashboard catches it. SigNoz would monitor the gateway application itself — connection success rates, authentication latency, tunnel setup time, and error rates per user or policy. Both are needed for complete observability.

---

## Issues I Ran Into

**Disk query returning no data with `mountpoint="/"`:** Node Exporter running inside Docker only exposes the container's filesystem mounts, not the host VM's. The standard query from the lab guide returns empty results. Switching to `fstype="ext4"` worked because all three container mount points use ext4, which maps to the underlying VM disk.

**Prometheus "Server time is out of sync" warning:** The OCI VM's clock and my browser clock differ by about 1m 33s. This doesn't break anything but causes the graph x-axis to show timestamps that are slightly off from what I expect. In a real environment, fixing this would require enabling NTP sync on the VM: `sudo timedatectl set-ntp true`.

**Node Exporter Full dashboard (1860) not loading via grafana.com initially:** The first attempt returned an unrelated example dashboard JSON. A second attempt connected to grafana.com correctly and imported the full Node Exporter dashboard successfully.

---

## What I Learned

**The difference between infrastructure metrics and application metrics.** Before this lab I thought monitoring was just one thing. After building both Uptime Kuma (Lab 3.1) and Prometheus/Grafana, I can see that there are actually three layers: availability monitoring (is it up?), infrastructure monitoring (is the machine healthy?), and APM (is the application behaving correctly?). Each layer answers a different question.

**PromQL is SQL for metrics.** Once I understood that `rate()` calculates per-second change and `avg by()` groups results, the queries made intuitive sense. The disk query fix — using `fstype="ext4"` instead of `mountpoint="/"` — required understanding what labels node-exporter attaches to each metric, which is exactly the kind of debugging real SREs do.

**Docker networking makes multi-service setups elegant.** Grafana connects to Prometheus using `http://prometheus:9090` — no IP addresses, no hardcoding. Docker Compose creates a virtual network where services find each other by name. This is how microservices work in production too.

**Pre-built dashboards are powerful starting points.** The Node Exporter Full dashboard (ID 1860) has dozens of panels that would take days to build from scratch. In real enterprise environments, teams import community dashboards and customize them rather than starting from zero.

---

## Final Checklist

| Criteria | Status |
|---|---|
| docker-compose.yml created | ✅ |
| All 3 containers running (Prometheus, Grafana, Node Exporter) | ✅ |
| CPU query returning data | ✅ ~37% |
| Memory query returning data | ✅ ~73-80% |
| Disk query returning data | ✅ ~16.6% (using fstype filter) |
| Network query returning data | ✅ ~42 bytes/sec |
| Grafana data source configured | ✅ |
| Custom dashboard with 3 panels built | ✅ |
| Node Exporter Full dashboard imported | ✅ |
| Prometheus vs SigNoz comparison written | ✅ |
