[فارسی](README-persian.md) | [English](README.md)

# cAdvisor (Container Advisor) Setup

cAdvisor (Container Advisor) is an open-source tool by Google that analyzes and exposes resource usage and performance data from running containers (like Docker, containerd, etc.). Since containers don't have their own internal Node Exporter, cAdvisor acts as the "Node Exporter for Containers".

## Install cAdvisor as a System Service (Binary)

While running cAdvisor in a Docker container is the most common approach, you can also run it as a standalone binary (systemd service) but this method its not recommended.

### 1. Download and Install the Binary

```bash
# Note: Ensure you download the correct architecture (e.g. amd64 vs arm64)
VERSION=$(curl -s https://api.github.com/repos/google/cadvisor/releases/latest | grep '"tag_name"' | cut -d'"' -f4)
wget -O cadvisor https://github.com/google/cadvisor/releases/download/${VERSION}/cadvisor-${VERSION}-linux-amd64
chmod +x cadvisor
sudo mv cadvisor /usr/local/bin/
```

### 2. Create a systemd service

> [!WARNING]
> **Why we run cAdvisor as `root`:** Unlike Node Exporter or Prometheus, you should not create a dedicated/isolated user for cAdvisor. cAdvisor requires deep system privileges to read container metrics from restricted paths like `/var/lib/docker/` and `/sys/fs/cgroup/`. If run as a normal user, it will encounter "Permission Denied" errors and fail to collect container metrics.

Create a file named `/etc/systemd/system/cadvisor.service`:

```bash
sudo nano /etc/systemd/system/cadvisor.service
```

Add the following configuration:

```ini
[Unit]
Description=cAdvisor
Wants=network-online.target
After=network-online.target

[Service]
# cAdvisor needs root access to read container metrics from /var/lib/docker, /sys, etc.
User=root
ExecStart=/usr/local/bin/cadvisor
Restart=always

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable cadvisor
sudo systemctl start cadvisor
```

cAdvisor should now be accessible on port **8080**:

- `http://{IP_ADDRESS}:8080/metrics`

## Set up cAdvisor with Docker Compose (Recommended)

Since cAdvisor's main job is to monitor containers, running it as a container makes perfect sense.

Example `docker-compose.yml`:

```yaml
services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.49.1
    container_name: cadvisor
    restart: unless-stopped
    devices:
      - /dev/kmsg:/dev/kmsg
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    ports:
      - "8080:8080"
```

Start the container:

```bash
docker compose up -d
```

## Configure Prometheus to scrape cAdvisor (port 8080)

Once cAdvisor is running, you need to tell Prometheus to scrape it. Open your `prometheus.yml` and add a new job:

```yaml
scrape_configs:
  - job_name: "cadvisor"
    static_configs:
      - targets: ["{NODE_IP}:8080"]
```

> [!IMPORTANT]
> **Firewall Note**: For Prometheus (the central server) to scrape cAdvisor, port `8080/tcp` must be open and accessible on the machine running cAdvisor.
>
> If you are on a restricted network where you cannot open inbound ports, you should look into using **Prometheus Agent** (we will cover this in a separate repository later).

## Grafana Dashboards for cAdvisor

To visualize the metrics collected by cAdvisor, you can import community dashboards in Grafana. Here are a few excellent examples you can try:

- [cAdvisor Exporter (14282)](https://grafana.com/grafana/dashboards/14282-cadvisor-exporter/)
- [cAdvisor Dashboard (19792)](https://grafana.com/grafana/dashboards/19792-cadvisor-dashboard/)
- [Docker Container Monitoring (19908)](https://grafana.com/grafana/dashboards/19908-docker-container-monitoring-with-prometheus-and-cadvisor/)

> [!WARNING]
> **cgroup v1 vs. cgroup v2**
>
> Some older Grafana dashboards for cAdvisor are explicitly written for **cgroup v1** metrics, while newer ones are optimized for **cgroup v2**. If you import a dashboard and some panels are empty, it's highly likely because of a mismatch between the dashboard's queries and your system's cgroup version.

### How to check your cgroup version:

You can easily check which cgroup version your Linux system is using by running:

```bash
stat -fc %T /sys/fs/cgroup/
```

- If the output is `cgroup2fs`, your system is running **cgroup v2** (common on modern OS like Ubuntu 22.04+, Debian 11+).
- If the output is `tmpfs` or `cgroup`, your system is likely running **cgroup v1**.

If your dashboard doesn't work, try importing a different dashboard or manually edit the queries (e.g. changing `container_memory_usage_bytes` to match whatever metric name cAdvisor is currently exporting based on your cgroup version).
