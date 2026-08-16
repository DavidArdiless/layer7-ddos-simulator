# 🛡️ Layer 7 HTTP Flood Simulation & Real-Time Monitoring Lab

![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-009639?style=for-the-badge&logo=nginx&logoColor=white)
![Alpine Linux](https://img.shields.io/badge/Alpine_Linux-0D597F?style=for-the-badge&logo=alpine-linux&logoColor=white)
![Security](https://img.shields.io/badge/Category-DevSecOps_Lab-red?style=for-the-badge)

An isolated, containerized security laboratory built with Docker Compose to simulate application-layer Denial of Service (DoS) attacks (Layer 7 HTTP Flood) and monitor hardware performance metrics in real time.

> ⚠️ **Disclaimer:** This repository is strictly for educational, research, and authorized testing purposes. Do not execute these tests against unauthorized infrastructure.

---

## � Table of Contents

- [📁 Repository Structure](#-repository-structure)
- [🏗️ Architecture Overview](#️-architecture-overview)
- [📋 Prerequisites & Requirements](#-prerequisites--requirements)
- [🚀 Quick Start](#-quick-start)
- [⚡ Simulation Execution](#-simulation-execution)
- [📊 Results & Metric Analysis](#-results--metric-analysis)
- [🛡️ DevSecOps Mitigation Strategies](#️-devsecops-mitigation-strategies)
- [🔧 Troubleshooting](#-troubleshooting)
- [🧹 Teardown](#-teardown)

---

## �📁 Repository Structure

```text
.
├── docker-compose.yml       # Docker Compose configuration for all services
├── README.md               # Project documentation
└── assets/
    └── dashboard-demo.png  # Screenshot of Glances monitoring dashboard
```

---

## 🏗️ Architecture Overview

The lab provisions a private, isolated Docker virtual bridge network containing three core services:

### 🎯 **victim_server** (nginx:alpine)
The target web server handling HTTP requests on **port 8080**. Runs Alpine Linux-based Nginx to serve as the attack target.

### 🔫 **attacker_node** (alpine)
The traffic generator running **wrk** (HTTP benchmarking tool) to generate high-concurrency HTTP flood attacks. Deployed as a standalone Alpine container to isolate attack traffic.

### 📊 **glances_dashboard** (nicolargo/glances:latest-full)
Real-time monitoring dashboard accessing `/var/run/docker.sock` to report container CPU, RAM, and network I/O metrics on **port 61208**. Provides live performance visualization during attack execution.

---

## 📋 Prerequisites & Requirements

### **Software Requirements**
- **Docker Desktop** (with running daemon)
  - Windows: [Docker Desktop for Windows](https://www.docker.com/products/docker-desktop)
  - macOS: [Docker Desktop for Mac](https://www.docker.com/products/docker-desktop)
  - Linux: [Docker Engine](https://docs.docker.com/engine/install/)
- **Git** (for cloning the repository)
- **Web Browser** (Firefox, Chrome, or Edge for dashboard access)

### **Hardware Recommendations**
| Component | Minimum | Recommended |
|-----------|---------|-------------|
| CPU Cores | 2       | 4+          |
| RAM       | 2 GB    | 4+ GB       |
| Disk      | 500 MB  | 1 GB        |
| Network   | 10 Mbps | 100 Mbps+   |

---

## 🚀 Quick Start

### **1. Prerequisites**

✅ Docker Desktop (with running daemon)  
✅ Git


### **2. Deployment**

Clone the repository and spin up the containers:

```bash
git clone https://github.com/yourusername/layer7-ddos-simulation-lab.git
cd layer7-ddos-simulation-lab
docker compose up -d
```

Expected output:
```
[+] Building 0.0s (0/0)                                                           
[+] Creating docker network layer7-ddos_lab_network
[+] Creating layer7-ddos-victim_server-1
[+] Creating layer7-ddos-attacker_node-1
[+] Creating layer7-ddos-glances_dashboard-1
```

### **3. Verification**

Verify that both endpoints are accessible in your browser:

- **Target Web Server:** http://localhost:8080
- **Glances Monitoring Dashboard:** http://localhost:61208

---

## ⚡ Simulation Execution

### **Step 1: Access the Attacker Node**

```bash
docker exec -it attacker_node sh
```

### **Step 2: Install wrk (HTTP Benchmarking Tool)**

**wrk** is a modern HTTP benchmarking tool capable of generating significant load with multiple concurrent connections and threads.

```bash
apk add wrk
```

### **Step 3: Launch the Layer 7 HTTP Flood Attack**

```bash
wrk -c 500 -d 40s -t 4 http://victim/
```

#### **Parameter Breakdown:**
- `-c 500`: Maintains **500 concurrent HTTP connections**
- `-t 4`: Utilizes **4 worker threads** for request generation
- `-d 40s`: Sustains the attack for **40 seconds**
- `http://victim/`: Targets the victim container's internal hostname on the Docker bridge network

#### **Expected wrk Output:**

```
Running 40s test @ http://victim/
  4 threads and 500 connections
  Thread Stats   Avg      Stdev    Max   +/- Stdev
    Latency    45.32ms   32.21ms 410.22ms  89.42%
    Req/Sec     2.87k    345.23   4.12k    65.33%
  Latency Distribution
     50%    38.11ms
     75%    52.42ms
     90%    78.33ms
     99%   210.55ms
  459203 requests in 40.01s, 182.34MB read
  Socket errors: connect 0, read 125, write 0, timeout 342
Requests/sec:  11476.23
Transfer/sec:    4.56MB
```

### **Step 4: Monitor in Real-Time**

While the attack is running, open a new terminal and monitor the Glances dashboard:

```bash
open http://localhost:61208
```

Or access via: **http://localhost:61208** in your browser


## 📊 Results & Metric Analysis

During the attack execution, key metrics demonstrate the impact of an HTTP flood:

### **🔥 CPU Exhaustion (>700% CPU Utilization)**
As Nginx processes thousands of concurrent HTTP requests, multi-core CPU usage spikes significantly to handle request parsing and connection queues. This is the primary bottleneck in Layer 7 attacks.

**Observable in Glances Dashboard:**
- CPU cores running at 99-100% utilization
- System load average spikes (e.g., 4.5+ on a 4-core system)
- CPU steal time increases on shared cloud infrastructure

### **💾 Low Memory Impact (~19 MB RAM)**
Memory consumption remains flat, proving that Layer 7 HTTP flood attacks cause service degradation through **CPU clock cycle starvation** rather than memory leaks or RAM exhaustion. This distinguishes Layer 7 attacks from Layer 3/4 amplification attacks.

**Observable in Glances Dashboard:**
- RAM usage static (~19-25 MB)
- Swap memory unchanged
- No OOM (Out of Memory) killer events

### **❌ Service Denial**
Refreshing `http://localhost:8080` during execution results in:
- **High latency** (>1000ms response time)
- **Connection timeouts** (TCP RST packets)
- **HTTP 503 Service Unavailable** responses
- Browser hang or complete unresponsiveness


## 🛡️ DevSecOps Mitigation Strategies

To harden application servers against Layer 7 DoS attacks, implement the following security controls:

### **1. Nginx Rate Limiting (limit_req_zone)**

Restrict maximum allowed request rates per client IP address:

```nginx
http {
  limit_req_zone $binary_remote_addr zone=general:10m rate=10r/s;
  limit_req_zone $binary_remote_addr zone=api:10m rate=5r/s;

  server {
    listen 80;
    server_name localhost;

    location / {
      limit_req zone=general burst=20 nodelay;
      proxy_pass http://backend;
    }

    location /api/ {
      limit_req zone=api burst=10 nodelay;
      proxy_pass http://backend;
    }
  }
}
```

### **2. Docker Resource Limits**

Enforce CPU quotas in `docker-compose.yml` to prevent single containers from saturating host resources:

```yaml
services:
  victim_server:
    image: nginx:alpine
    ports:
      - "8080:80"
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 256M
        reservations:
          cpus: '1.0'
          memory: 128M
```

**Effect:** Limits container to max 2 CPU cores and 256MB RAM, protecting host from total saturation.

### **3. Web Application Firewall (WAF)**

Deploy Cloudflare, ModSecurity, or AWS WAF to inspect incoming HTTP headers, identify bot patterns, and block automated floods before reaching origin servers:

- **Cloudflare:** DDoS Protection, Rate Limiting, Bot Management
- **ModSecurity:** Open-source WAF with OWASP CRS rules
- **AWS WAF:** AWS Shield integration for enterprise environments
- **Nginx ModSecurity:** `nginx-modsecurity` module for local protection

### **4. Connection Timeout Optimization**

Adjust kernel and Nginx parameters to handle connection floods:

```nginx
http {
  keepalive_timeout 30s;
  client_body_timeout 10s;
  client_header_timeout 10s;
  send_timeout 10s;
  
  # TCP backlog queue
  listen 80 backlog=2048;
}
```

---

## 🔧 Troubleshooting

### **Port Already in Use (8080 or 61208)**

**Error:** `bind: address already in use`

**Solution:**
```bash
# Find process using port 8080
lsof -i :8080

# Kill the process (adjust PID as needed)
kill -9 <PID>

# Or use different ports in docker-compose.yml:
# Change "8080:80" to "8081:80"
# Change "61208:61208" to "61209:61208"
```

### **Docker Daemon Not Running**

**Error:** `Cannot connect to Docker daemon`

**Solution:**
- **Windows/Mac:** Start Docker Desktop from Applications
- **Linux:** `sudo systemctl start docker`

### **wrk Command Not Found**

**Error:** `sh: wrk: not found`

**Solution:**
```bash
docker exec -it attacker_node sh
apk add wrk  # Ensure this command completes successfully
which wrk    # Verify installation
```

### **Victim Server Returns 503 Service Unavailable**

**Normal behavior during attack.** This is the intended denial of service effect.

To test server responsiveness after attack stops:
```bash
docker exec -it victim_server sh
wget -O- http://localhost/  # Should return 200 OK after attack ends
```

### **Glances Dashboard Not Loading (Port 61208)**

**Solution:**
```bash
# Verify glances_dashboard container is running
docker ps | grep glances

# Check container logs
docker logs glances_dashboard

# Restart the container
docker restart glances_dashboard
```

### **High Socket Errors in wrk Output**

Socket errors during wrk execution are expected and indicate successful denial of service:
- **connect errors:** Connection refused (service overwhelmed)
- **read errors:** Connection reset by server
- **timeout errors:** Request timeout due to CPU exhaustion

These demonstrate the attack's effectiveness.

---

## 🧹 Teardown

To stop and clean up all resources:

```bash
# Stop and remove all containers
docker compose down

# Remove associated volumes (if any)
docker compose down -v

# Remove the entire project directory (optional)
rm -rf layer7-ddos-simulation-lab
```

---

## 📚 Additional Resources

- [Nginx Documentation - Rate Limiting](http://nginx.org/en/docs/http/ngx_http_limit_req_module.html)
- [wrk GitHub Repository](https://github.com/wg/wrk)
- [Glances Official Documentation](https://glances.io/)
- [OWASP DDoS Protection Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Denial_of_Service_Prevention_Cheat_Sheet.html)
- [Docker Resource Limits](https://docs.docker.com/compose/compose-file/05-services/#resources)

---

**Last Updated:** August 2026  
**License:** MIT  
**Educational Purpose Only** 🎓