# Lab 3.1 – Uptime Kuma (Build Your Own Site24x7)



## Lab Objective

The objective of this lab was to learn how to monitor servers and services using **Uptime Kuma**. In this lab, I installed Uptime Kuma using Docker, created different types of monitors, simulated a service failure, configured email notifications, and observed how the monitoring system detected failures and recovery.

---

# Lab Environment

| Component | Details |
|-----------|---------|
| Cloud Platform | Oracle Cloud Infrastructure (OCI) |
| Monitoring Server | VM1 (vpn-server) |
| Target Server | VM2 (vpn-client) |
| Monitoring Tool | Uptime Kuma |
| Web Server | Nginx |
| Container Platform | Docker |
| Email Notification | Mailtrap SMTP |

---

# VM Details

| Virtual Machine | Purpose |
|-----------------|---------|
| VM1 | Installed Docker and Uptime Kuma |
| VM2 | Installed Nginx server to monitor |

Private IP used for monitoring:

```
10.0.0.143
```

---

# Step 1 – Deploy Uptime Kuma

I installed Docker on VM1 and deployed Uptime Kuma using the official Docker image.

Docker command used:

```bash
docker pull louislam/uptime-kuma:1

docker run -d \
--restart=always \
--name uptime-kuma \
-p 3001:3001 \
-v uptime-kuma:/app/data \
louislam/uptime-kuma:1
```

After the container started successfully, I opened the web interface.

```
http://<VM1_Public_IP>:3001
```

I created the administrator account and logged into the dashboard.

---

## Screenshot – Uptime Kuma Dashboard

![Uptime Kuma Dashboard](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-3-1-1-image.png)

---

# Step 2 – Creating Monitors

I created three different monitors to monitor VM2.

---

## Monitor 1 – HTTP Monitor

Purpose:

Monitor whether the Nginx web server is available.

Configuration

- Monitor Type: HTTP(S)
- Name: Lab Nginx Server
- URL

```
http://10.0.0.143
```

- Heartbeat Interval

```
60 seconds
```

---

## Monitor 2 – TCP Port Monitor

Purpose:

Monitor whether the SSH service is accepting connections.

Configuration

- Monitor Type

```
TCP Port
```

- Host

```
10.0.0.143
```

- Port

```
22
```

---

## Monitor 3 – Ping Monitor

Purpose:

Check whether VM2 is reachable through the network.

Configuration

- Monitor Type

```
Ping
```

- Host

```
10.0.0.143
```

---

# Dashboard after Creating Monitors

After waiting for a few minutes, all three monitors became healthy.

The dashboard showed:

- HTTP Monitor – Up
- SSH Monitor – Up
- Ping Monitor – Up

---

## Screenshot – Three Healthy Monitors

![Healthy Monitors](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-3-1-2-image.png)

---

# Step 3 – Simulating a Failure

To understand how monitoring tools detect problems, I intentionally stopped the Nginx service.

Command used:

```bash
sudo systemctl stop nginx
```

After approximately one minute, the HTTP monitor detected that the website was unavailable.

The dashboard changed:

- HTTP Monitor → Down
- SSH Monitor → Up
- Ping Monitor → Up

Uptime Kuma also recorded the incident in its event history.

---

## Screenshot – HTTP Monitor Down

![HTTP Down](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-3-1-3-image.png)

---

# Step 4 – Recovery

After confirming the failure detection, I restarted the Nginx service.

Command used:

```bash
sudo systemctl start nginx
```

Within a short time, Uptime Kuma detected that the service was available again.

The monitor status changed from **Down** to **Up**, and the incident log showed the recovery.

---

## Screenshot – Recovery

![Recovery](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-3-1-4-image.png)

---

# Step 5 – Configuring Email Notifications

Initially I tried to configure Gmail SMTP.

However, my Google account did not allow App Passwords, even though Two-Step Verification was enabled.

Therefore, I used **Mailtrap Email Sandbox** as the SMTP server.

SMTP configuration:

| Setting | Value |
|----------|-------|
| SMTP Host | sandbox.smtp.mailtrap.io |
| Port | 587 |
| Security | STARTTLS |
| SMTP Provider | Mailtrap Email Sandbox |

I configured the SMTP settings in Uptime Kuma and successfully sent a test notification.

The email appeared in the Mailtrap Inbox.

---

## Screenshot – SMTP Configuration

![SMTP Configuration](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-3-1-5-image.png)

---

## Screenshot – Mailtrap Email Received

![Mailtrap Email](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-3-1-6-image.png)

---

# Problems Faced

During this lab, I encountered several issues.

## Issue 1

The HTTP monitor was showing **Down** even though Nginx was running.

### Cause

The monitor was configured using the public IP.

### Solution

I changed the monitor to use the private IP:

```
10.0.0.143
```

This solved the issue.

---

## Issue 2

The Ping monitor failed initially.

### Cause

ICMP traffic was blocked.

### Solution

I updated the OCI networking configuration and verified connectivity using the private IP.

---

## Issue 3

Gmail SMTP could not be configured.

### Cause

Google App Passwords were unavailable for my account.

### Solution

I used Mailtrap Email Sandbox instead.

---

# Monitor Type Mapping

| Uptime Kuma Monitor | Site24x7 Equivalent |
|----------------------|---------------------|
| HTTP(S) | Website Monitor |
| TCP Port | Port Monitor |
| Ping | Server Availability Monitor |

---

# Which Monitor Would I Use for an InstaSafe Gateway?

For monitoring an InstaSafe Gateway, I would use all three monitor types.

### HTTP(S) Monitor

To verify that the management portal is available.

### TCP Port Monitor

To check whether important ports such as SSH, HTTPS, or VPN ports are accepting connections.

### Ping Monitor

To verify that the gateway server is reachable over the network.

Using these three monitor types together provides better monitoring because they check both the network connectivity and the application availability.

---

# What I Learned

Through this lab, I learned:

- How to deploy Uptime Kuma using Docker.
- How to create HTTP, TCP, and Ping monitors.
- How uptime monitoring tools continuously check server health.
- How monitoring tools detect service failures.
- How recovery is automatically detected.
- How SMTP notifications work.
- How to configure Mailtrap for email testing.
- Why private IP addresses are useful for monitoring servers within the same cloud network.
- The difference between HTTP monitoring, TCP monitoring, and Ping monitoring.

---

# Conclusion

In this lab, I successfully deployed Uptime Kuma on an OCI virtual machine using Docker.

I created multiple monitors for a web server and verified that Uptime Kuma correctly detected service failures and recoveries.

I also configured SMTP notifications using Mailtrap and successfully received test alert emails.

This lab helped me understand how enterprise monitoring tools such as Site24x7 continuously monitor server health, detect failures, and notify administrators automatically.
