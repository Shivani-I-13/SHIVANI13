# Ceph Squid (19.2.5) Single-Node Deployment using cephadm and Docker on Rocky Linux 9.8

## Table of Contents

1. [Overview](#1-overview)
2. [Software Versions](#2-software-versions)
3. [Pre-Installation Checks](#3-pre-installation-checks)
4. [Installation Steps](#4-installation-steps)
5. [Post-Installation Verification](#5-post-installation-verification)
6. [Accessing the Ceph Dashboard](#6-accessing-the-ceph-dashboard)
7. [Adding Storage (OSD)](#7-adding-storage-osd)
8. [Useful Commands](#8-useful-commands)
9. [Troubleshooting](#9-troubleshooting)
10. [References](#10-references)

To remove Ceph from the VM, see [UNINSTALL.md](UNINSTALL.md).

---

## 1. Overview

This document describes the manual installation of a **single-node Ceph cluster** on a Rocky Linux 9.8 virtual machine using **cephadm**, the official Ceph deployment tool.

cephadm deploys every Ceph daemon (MON, MGR, OSD, etc.) as a container. The installation in this guide uses **Docker** as the container runtime.

**What gets deployed after bootstrap:**

| Component              | Purpose                                         |
|------------------------|-------------------------------------------------|
| MON (Monitor)          | Maintains the cluster map and quorum            |
| MGR (Manager)          | Cluster management, metrics, orchestrator       |
| Ceph Dashboard         | Web UI for managing the cluster (port 8443)     |
| Prometheus / Grafana / Alertmanager / Node Exporter | Monitoring stack deployed by cephadm |
| OSD (Object Storage)   | Added manually in [Section 7](#7-adding-storage-osd) |

**Installation flow:**

```
Prerequisites  ->  Docker  ->  cephadm  ->  ceph-common (CLI)  ->  Bootstrap  ->  Verify  ->  Add OSD
```

---

## 2. Software Versions

| Software          | Version / Source                                              |
|-------------------|---------------------------------------------------------------|
| Operating System  | Rocky Linux 9.8 (x86_64)                                      |
| Ceph Release      | **Squid 19.2.5** (`CEPH_RELEASE=19.2.5`, `--release squid`)   |
| cephadm           | 19.2.5 (downloaded from download.ceph.com, el9 build)         |
| Container Runtime | Docker CE (latest stable from the Docker RHEL repository)     |
| Python            | Python 3 (default `python3` of Rocky Linux 9, 3.9.x)          |
| Time Sync         | chrony (from Rocky Linux repositories)                        |

> Check the exact versions installed on your VM with the commands in [Section 5](#5-post-installation-verification).

---

## 3. Pre-Installation Checks

Run these before starting and confirm the output matches your expectations.

```bash
# OS version (should show Rocky Linux 9.x)
cat /etc/os-release

# Hostname
hostnamectl

# CPU and memory
nproc
free -h

# Disks (the OSD disk must show no partitions and no mountpoint)
lsblk

# IP addresses and default route
ip a
ip r
```

---

## 4. Installation Steps

Run all steps on the Ceph VM.

### Step 1: Install prerequisites (Python 3 and chrony)

Ceph requires Python 3, and all cluster nodes must have synchronized time. chrony provides time synchronization.

```bash
sudo dnf install -y python3 chrony
sudo systemctl enable --now chronyd
python3 --version
```

Expected: a Python 3.x version is printed, for example `Python 3.9.x`.

### Step 2: Install Docker

cephadm needs a container runtime. Add the Docker repository and install Docker CE.

```bash
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Start Docker and enable it at boot:

```bash
sudo systemctl enable --now docker
docker --version
sudo systemctl status docker
```

Expected: `docker --version` prints the version, and the status shows `active (running)`. Press `q` to exit the status view.

Allow your user to run Docker without `sudo`:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

### Step 3: Download and install cephadm

Set the Ceph release and download the cephadm script for Rocky Linux 9 (el9):

```bash
CEPH_RELEASE=19.2.5
curl --silent --remote-name --location https://download.ceph.com/rpm-${CEPH_RELEASE}/el9/noarch/cephadm
chmod +x cephadm
```

Add the Ceph Squid repository and install cephadm as a system package:

```bash
sudo ./cephadm add-repo --release squid
sudo ./cephadm install
which cephadm
```

Expected: `which cephadm` prints `/usr/sbin/cephadm`.

### Step 4: Install the Ceph CLI (ceph-common)

This provides the `ceph`, `rados` and `rbd` commands on the host.

> Run this **after** Step 3. It fails if cephadm is not installed yet.

```bash
sudo cephadm install ceph-common
ceph --version
```

Expected: a version string similar to `ceph version 19.2.5 (...) squid (stable)`.

### Step 5: Identify the VM IP address

The bootstrap command needs the IP address the Ceph monitor will listen on.

```bash
ip r
```

In the output, find the line starting with `default` and note the interface name. Then note the `src` IP address of that interface. Example:

```
default via 192.168.1.1 dev ens192 proto static metric 100
192.168.1.0/24 dev ens192 proto kernel scope link src 192.168.1.50 metric 100
```

In this example the VM IP is `192.168.1.50`.

### Step 6: Bootstrap the cluster

Replace `<VM-IP>` with the IP from Step 5:

```bash
sudo cephadm bootstrap --mon-ip <VM-IP>
```

Bootstrap takes a few minutes. It pulls the container images and then:

- creates the first MON and MGR daemons on this host
- generates `/etc/ceph/ceph.conf` and the admin keyring
- enables the Ceph Dashboard and the monitoring stack

At the end of the output, cephadm prints the dashboard URL, username and password:

```
Ceph Dashboard is now available at:

             URL: https://<hostname>:8443/
            User: admin
        Password: xxxxxxxxxx
```

> Copy the username and password from this output. The password is only shown here.

---

## 5. Post-Installation Verification

```bash
# Ceph version
ceph --version
cephadm version

# Cluster status
sudo ceph -s

# Daemons managed by cephadm
sudo ceph orch ps
sudo cephadm ls

# Running containers
sudo docker ps

# Host list
sudo ceph orch host ls
```

Expected after bootstrap on a new single node:

- `ceph -s` shows 1 MON and 1 MGR
- `HEALTH_WARN` is normal until an OSD is added (no OSDs / not enough replicas)
- `docker ps` lists containers for mon, mgr, prometheus, grafana, alertmanager and node-exporter

---

## 6. Accessing the Ceph Dashboard

1. Open a browser and go to `https://<VM-IP>:8443`
2. Accept the self-signed certificate warning
3. Log in with the username and password shown at the end of the bootstrap output (Step 6)
4. Ceph asks you to set a new password on the first login

To find the dashboard URL again at any time:

```bash
sudo ceph mgr services
```

---

## 7. Adding Storage (OSD)

An OSD is needed before the cluster can store data. The disk must be empty, with no partitions and no filesystem.

List the devices cephadm can use (look for `Available = Yes`):

```bash
sudo ceph orch device ls
```

Create OSDs on all available devices:

```bash
sudo ceph orch apply osd --all-available-devices
```

Or create an OSD on one specific device (replace the host and device):

```bash
sudo ceph orch daemon add osd <hostname>:/dev/sdb
```

Check the result:

```bash
sudo ceph osd tree
sudo ceph -s
```

> On a single node, the default replicated pools expect 3 copies, so the cluster may still show `HEALTH_WARN`. This is expected on a one-node setup.

---

## 8. Useful Commands

| Task                         | Command                                  |
|------------------------------|------------------------------------------|
| Cluster status               | `sudo ceph -s`                           |
| Detailed health              | `sudo ceph health detail`                |
| List all daemons             | `sudo ceph orch ps`                      |
| List services                | `sudo ceph orch ls`                      |
| List OSDs                    | `sudo ceph osd tree`                     |
| Disk usage                   | `sudo ceph df`                           |
| List pools                   | `sudo ceph osd pool ls detail`           |
| Open the Ceph shell          | `sudo cephadm shell`                     |
| Dashboard URL                | `sudo ceph mgr services`                 |
| Container list               | `sudo docker ps`                         |
| Config file location         | `/etc/ceph/ceph.conf`                    |
| Admin keyring location       | `/etc/ceph/ceph.client.admin.keyring`    |

---

## 9. Troubleshooting

| Problem                                     | Cause / Fix                                                                           |
|---------------------------------------------|---------------------------------------------------------------------------------------|
| `docker: permission denied`                 | Log out and back in, or run `newgrp docker`, after `usermod -aG docker`               |
| `ceph: command not found`                   | Run `sudo cephadm install ceph-common` (cephadm must be installed first)              |
| `cephadm: command not found`                | Run `sudo ./cephadm add-repo --release squid` and `sudo ./cephadm install`           |
| Bootstrap fails with a time sync error      | `sudo systemctl enable --now chronyd`, then check `chronyc tracking`                  |
| Bootstrap fails on hostname                 | Use a short hostname: `sudo hostnamectl set-hostname ceph-node1`                      |
| Image pull fails                            | Check internet access to `quay.io` and DNS resolution                                 |
| `HEALTH_WARN` with no OSDs                  | Add a raw disk and run `sudo ceph orch apply osd --all-available-devices`             |
| Disk not listed as available                | The disk must have no partitions or filesystem; check with `lsblk`                    |
| Dashboard not reachable                     | Run `sudo ceph mgr services` and confirm the URL and port 8443                        |
| Forgot dashboard password                   | Reset: `echo 'NewPass@123' > /tmp/pw && sudo cephadm shell --mount /tmp/pw -- ceph dashboard ac-user-set-password admin -i /mnt/pw` |

---

## 10. References

- [cephadm documentation](https://docs.ceph.com/en/latest/cephadm/)
- [cephadm install guide](https://docs.ceph.com/en/latest/cephadm/install/)
- [Ceph Squid release notes](https://docs.ceph.com/en/latest/releases/squid/)
- [Ceph Dashboard documentation](https://docs.ceph.com/en/latest/mgr/dashboard/)
- [Docker Engine on RHEL-family](https://docs.docker.com/engine/install/rhel/)
