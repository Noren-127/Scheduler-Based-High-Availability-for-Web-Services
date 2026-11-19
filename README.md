# 🖥️ Scheduler-Based High Availability for Web Services

An experimental High-Performance Computing (HPC) mini-cluster demonstrating how **SLURM Workload Manager** can orchestrate persistent, always-on containerized web services with zero single points of failure.

Client traffic enters through a stable **Virtual IP (VIP)** managed by **Keepalived (VRRP)**, is distributed across active web replicas via **HAProxy**, and is served by **Dockerized NGINX** containers scheduled as long-running SLURM jobs backed by shared **NFS** storage.

---

## 🏗️ Architecture Overview

```text
                        ┌──────────────────────┐
                        │       Clients        │
                        └──────────┬───────────┘
                                   │ HTTP (:80)
                                   ▼
                        ┌──────────────────────┐
                        │  Virtual IP (VIP)    │
                        │    192.168.1.210     │
                        └──────────┬───────────┘
                                   │ VRRP (Keepalived)
                 ┌─────────────────┴─────────────────┐
                 ▼                                   ▼
    ┌─────────────────────────┐         ┌─────────────────────────┐
    │     lb1 (MASTER)        │         │     lb2 (BACKUP)        │
    │  HAProxy + Keepalived   │         │  HAProxy + Keepalived   │
    │     192.168.1.205       │         │     192.168.1.206       │
    └────────────┬────────────┘         └────────────┬────────────┘
                 │           Round-Robin             │
                 └─────────────────┬─────────────────┘
                       ┌───────────┴───────────┐
                       ▼                       ▼
          ┌─────────────────────────┐ ┌─────────────────────────┐
          │          web1           │ │          web2           │
          │  NGINX Container :8080  │ │  NGINX Container :8080  │
          │     192.168.1.202       │ │     192.168.1.203       │
          └────────────┬────────────┘ └────────────┬────────────┘
                       │                           │
                       └─────────────┬─────────────┘
                                     │ NFS Mount (/srv/www)
                                     ▼
                        ┌─────────────────────────┐
                        │      storage (NFS)      │
                        │      192.168.1.204      │
                        │ /srv/www                │
                        │ /srv/slurm-state/ctld   │
                        └────────────▲────────────┘
                                     │ Shared Controller State
                       ┌─────────────┴─────────────┐
                       │                           │
          ┌─────────────────────────┐ ┌─────────────────────────┐
          │   slurmctl1 (Primary)   │ │   slurmctl2 (Backup)    │
          │      192.168.1.200      │ │      192.168.1.201      │
          └─────────────────────────┘ └─────────────────────────┘
```

### Request & Control Flow
1. **Client Entry:** Clients send HTTP requests to the Virtual IP (`192.168.1.210`).
2. **VIP Ownership:** Keepalived uses VRRP (IP Protocol 112) to assign the VIP to the active load balancer (`lb1` by default, failing over to `lb2` if `lb1` or its HAProxy instance fails).
3. **Load Balancing:** HAProxy listens on port `80`, health-checks backend nodes (`GET /`), and distributes requests via round-robin to `web1:8080` and `web2:8080`.
4. **Workload Execution:** SLURM schedules two active-active exclusive batch jobs (`websvc-web1` and `websvc-web2`) running Dockerized NGINX containers with built-in restart loops.
5. **Shared State & Content:** `storage` exports `/srv/www` (web root) and `/srv/slurm-state/ctld` (SLURM controller state for active-passive controller failover).

---

## 🌐 Cluster Topology & Node Specifications

All nodes run **Ubuntu OS** with **2 vCPUs** and **2048 MB RAM** on a static subnet (`192.168.1.0/24`):

| Role | Hostname | IP Address | Description |
| :--- | :--- | :--- | :--- |
| **Primary Controller** | `slurmctl1` | `192.168.1.200` | Primary `slurmctld` daemon & job submission host |
| **Backup Controller** | `slurmctl2` | `192.168.1.201` | Standby `slurmctld` daemon (takes over on primary failure) |
| **Compute / Web Node 1** | `web1` | `192.168.1.202` | Runs `slurmd` + Docker NGINX container (`:8080`) |
| **Compute / Web Node 2** | `web2` | `192.168.1.203` | Runs `slurmd` + Docker NGINX container (`:8080`) |
| **Storage Server** | `storage` | `192.168.1.204` | NFS server hosting `/srv/www` and `/srv/slurm-state` |
| **Load Balancer 1** | `lb1` | `192.168.1.205` | HAProxy + Keepalived (`MASTER`, priority `150`) |
| **Load Balancer 2** | `lb2` | `192.168.1.206` | HAProxy + Keepalived (`BACKUP`, priority `100`) |
| **Virtual IP (VIP)** | — | `192.168.1.210` | Floating client endpoint managed by Keepalived |

---

## 🛠️ Tech Stack

- **Workload Manager:** SLURM (`slurm-wlm`)
- **Cluster Authentication:** MUNGE (`munge`)
- **Container Runtime & Web Server:** Docker (`docker.io`) + `nginx:stable`
- **Shared Storage:** NFS (`nfs-kernel-server` / `nfs-common`)
- **Load Balancer & Failover:** HAProxy + Keepalived (VRRP)
- **Time Synchronization:** Chrony

---

## 🚀 Setup & Deployment Guide

### 1. Base System & Time Sync
Configure static IPs and populate `/etc/hosts` across all 7 nodes so every hostname resolves cleanly. Install essential tools and enable `chrony` (critical for Munge token validation):

```bash
sudo apt update && sudo apt install -y curl vim net-tools chrony
sudo systemctl enable --now chrony
chronyc tracking
```

### 2. Shared Storage (NFS)
On **`storage` (`192.168.1.204`)**, install the NFS server and create export directories for web content and SLURM controller state:

```bash
sudo apt install -y nfs-kernel-server
sudo mkdir -p /srv/www /srv/slurm-state/ctld
echo "<h1>Hello from NFS Shared Web!</h1>" | sudo tee /srv/www/index.html
sudo chown -R root:root /srv/www /srv/slurm-state
sudo chmod -R 755 /srv/www /srv/slurm-state
```

Configure `/etc/exports`:
```text
/srv/www          web1(rw,sync,no_subtree_check) web2(rw,sync,no_subtree_check) slurmctl1(rw,sync,no_subtree_check) slurmctl2(rw,sync,no_subtree_check)
/srv/slurm-state  slurmctl1(rw,sync,no_subtree_check) slurmctl2(rw,sync,no_subtree_check)
```

Apply exports:
```bash
sudo exportfs -ra
sudo exportfs -v
```

On **`web1`, `web2`, `slurmctl1`, and `slurmctl2`**, install `nfs-common` and configure persistent `/etc/fstab` mounts:
```bash
sudo apt install -y nfs-common
sudo mkdir -p /srv/www
# Add to /etc/fstab on web1, web2, slurmctl1, slurmctl2:
# storage:/srv/www /srv/www nfs defaults,_netdev 0 0

# Additionally on slurmctl1 and slurmctl2:
sudo mkdir -p /srv/slurm-state
# Add to /etc/fstab on slurmctl1 and slurmctl2:
# storage:/srv/slurm-state /srv/slurm-state nfs defaults,_netdev 0 0

sudo mount -a
```

### 3. Authentication (Munge) & SLURM HA
On **`slurmctl1`, `slurmctl2`, `web1`, and `web2`**, install SLURM and Munge, and ensure the `slurm` user UID/GID matches across controllers and `storage`:

```bash
sudo apt install -y slurm-wlm munge
sudo mkdir -p /var/log/slurm
sudo chown -R slurm:slurm /var/log/slurm
```

Generate the Munge key on `slurmctl1` and distribute `/etc/munge/munge.key` (`0400`, owned by `munge:munge`) to `slurmctl2`, `web1`, and `web2`:
```bash
sudo /usr/sbin/mungekey --create
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 0400 /etc/munge/munge.key
sudo systemctl enable --now munge

# Verify authentication across nodes
munge -n | ssh web1 unmunge
```

Configure `/etc/slurm/slurm.conf` (and sync across all SLURM nodes):
```ini
ClusterName=mini-hpc-web
SlurmUser=slurm

# HA Controllers (order defines primary/backup)
SlurmctldHost=slurmctl1(192.168.1.200)
SlurmctldHost=slurmctl2(192.168.1.201)
SlurmctldPort=6817
SlurmdPort=6818

AuthType=auth/munge
CryptoType=crypto/munge

# Shared Controller State & Local Compute Spool
StateSaveLocation=/srv/slurm-state/ctld
SlurmdSpoolDir=/var/spool/slurmd

# Logs
SlurmctldLogFile=/var/log/slurm/slurmctld.log
SlurmdLogFile=/var/log/slurm/slurmd.log

# Scheduling & Recovery
SchedulerType=sched/backfill
SelectType=select/cons_tres
SelectTypeParameters=CR_Core
SlurmctldTimeout=60
SlurmdTimeout=120
ReturnToService=2
JobRequeue=1

# Compute Nodes & Partition
NodeName=web1 NodeAddr=192.168.1.202 CPUs=2 RealMemory=2048 State=UNKNOWN
NodeName=web2 NodeAddr=192.168.1.203 CPUs=2 RealMemory=2048 State=UNKNOWN
PartitionName=webnodes Nodes=web1,web2 MaxTime=INFINITE State=UP Default=YES
```

Ensure `slurmctld` waits for NFS mounts via systemd (`RequiresMountsFor=/srv/slurm-state`), then start `slurmctld` on controllers and `slurmd` on web nodes:
```bash
sinfo
scontrol show nodes
```

### 4. Containerized Web Service via SLURM
Install Docker on **`web1`** and **`web2`**:
```bash
sudo apt install -y docker.io
sudo systemctl enable --now docker
```

Create pinned batch job scripts on `slurmctl1` (`/opt/web-jobs/websvc_web1.sbatch` and `websvc_web2.sbatch`) to run NGINX in an active-active pattern with an automatic container restart loop:

```bash
#!/bin/bash
#SBATCH -J websvc-web1
#SBATCH -p webnodes
#SBATCH -N 1
#SBATCH --nodelist=web1
#SBATCH --exclusive
#SBATCH -o /opt/web-jobs/%x-%j.out
#SBATCH -e /opt/web-jobs/%x-%j.err

set -euo pipefail

CNAME="websvc-web1"
IMAGE="nginx:stable"

# Wait for NFS mount before starting service
until mountpoint -q /srv/www; do
  echo "Waiting for /srv/www mount..."
  sleep 2
done

# Restart loop for immediate container-level recovery
while true; do
  docker rm -f "$CNAME" >/dev/null 2>&1 || true
  docker run --name "$CNAME" --rm \
    -p 8080:80 \
    -v /srv/www:/usr/share/nginx/html:ro \
    "$IMAGE"
  echo "Container exited; restarting in 2 seconds..."
  sleep 2
done
```

Submit both jobs and verify endpoints:
```bash
sbatch /opt/web-jobs/websvc_web1.sbatch
sbatch /opt/web-jobs/websvc_web2.sbatch
squeue

curl http://web1:8080
curl http://web2:8080
```

### 5. Load Balancing & VIP Failover
Install HAProxy and Keepalived on **`lb1`** and **`lb2`**:
```bash
sudo apt install -y haproxy keepalived
```

Configure `/etc/haproxy/haproxy.cfg` identically on both load balancers:
```haproxy
global
    log /dev/log local0
    log /dev/log local1 notice
    daemon

defaults
    log     global
    mode    http
    timeout connect 5s
    timeout client  30s
    timeout server  30s

frontend fe_http
    bind *:80
    default_backend be_web

backend be_web
    balance roundrobin
    option httpchk GET /
    http-check expect status 200
    server web1 192.168.1.202:8080 check inter 2s fall 2 rise 1
    server web2 192.168.1.203:8080 check inter 2s fall 2 rise 1
```

Create the HAProxy health-check script `/etc/keepalived/check_haproxy.sh` (`chmod +x`) on both nodes:
```bash
#!/bin/bash
systemctl is-active --quiet haproxy
```

Configure `/etc/keepalived/keepalived.conf` (`state MASTER` / `priority 150` on `lb1`; `state BACKUP` / `priority 100` on `lb2`), binding VIP `192.168.1.210` and tracking `check_haproxy.sh`. Ensure firewall rules allow VRRP (IP protocol `112`), then restart services:
```bash
sudo systemctl restart haproxy keepalived
ip a | grep 192.168.1.210
curl -I [http://192.168.1.210](http://192.168.1.210)
```

---

## 🧪 High Availability & Failover Testing

| Failure Scenario | Action Simulated | Expected & Verified Behavior |
| :--- | :--- | :--- |
| **Container Crash** | `docker kill websvc-web1` on `web1` | SLURM job remains active; `while true` loop restarts NGINX within ~2 seconds. |
| **Web Node Outage** | Power off `web1` (`192.168.1.202`) | HAProxy marks `web1` DOWN within 4s (`inter 2s fall 2`) and routes 100% of traffic to `web2`. |
| **Primary Controller Down** | Stop `slurmctld` / power off `slurmctl1` | `slurmctl2` promotes itself via shared `/srv/slurm-state/ctld` and retains full job state. |
| **Load Balancer / HAProxy Failure** | Stop `haproxy` or power off `lb1` | Keepalived detects failure via `vrrp_script` and migrates VIP `192.168.1.210` to `lb2`. |

---

## 🔄 Zero-Downtime Rolling Updates

Because the web service runs in an **Active–Active** pattern across `web1` and `web2`, maintenance can be performed without dropping client requests:

```bash
# 1. Drain target node in SLURM
scontrol update NodeName=web1 State=DRAIN Reason="Rolling update"

# 2. Cancel and resubmit the node's service job
scancel -n websvc-web1
sbatch /opt/web-jobs/websvc_web1.sbatch

# 3. Verify health on port 8080, then resume node
curl -I http://web1:8080
scontrol update NodeName=web1 State=RESUME

# 4. Repeat for web2
```

---

## 💡 Key Lessons & Best Practices

- **Container Exit vs. Node Failure:** SLURM's `JobRequeue=1` handles node crashes, but an application exit inside a batch script would normally mark the job `COMPLETED`. Wrapping `docker run` in a `while true` loop provides instant application-level self-healing.
- **Systemd Mount Ordering:** Configuring `RequiresMountsFor=/srv/slurm-state` on `slurmctld` prevents split-brain or startup failures if NFS mounts slowly on boot.
- **UID/GID & Time Sync:** Identical `slurm` UID/GID values across controllers and NFS storage—paired with `chrony` time sync—are essential to prevent permission errors and Munge authentication failures.

---

## 🔮 Future Improvements

- **Observability:** Integrate Prometheus and Grafana exporters to monitor SLURM queue metrics, node health, and HAProxy request latency.
- **Job Accounting:** Deploy `slurmdbd` with MariaDB/MySQL for historical job tracking and resource auditing.
- **HPC-Native Containers:** Evaluate Singularity/Apptainer as a rootless alternative to Docker for tighter HPC security integration.
- **Shared Filesystem Scaling:** Migrate controller state from NFS to a low-latency distributed filesystem (e.g., Ceph or Lustre) for larger cluster deployments.

---

