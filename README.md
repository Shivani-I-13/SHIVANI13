# Ceph Single-Node Installation (cephadm + Docker) on Rocky Linux 9.8

Manual installation guide for a single-node Ceph **Squid (19.2.x)** cluster using `cephadm` on Rocky Linux 9.8.

---

## Requirements

| Item      | Requirement                                                   |
|-----------|---------------------------------------------------------------|
| OS        | Rocky Linux 9.8 (x86_64)                                      |
| CPU / RAM | 2+ vCPU, 8 GB+ RAM recommended                                |
| Disk      | OS disk + at least one **unused raw disk** for an OSD         |
| Network   | Static IP, internet access to download.ceph.com and docker.com |
| Access    | Root or sudo user                                             |

> The extra raw disk is needed to add an OSD. Bootstrap works without it, but the cluster shows `HEALTH_WARN` until an OSD is added.

---

## 1. Prerequisites

```bash
sudo dnf install -y python3 chrony
sudo systemctl enable --now chronyd
python3 --version
```

## 2. Install Docker

```bash
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
docker --version
sudo systemctl status docker
```

Allow the current user to run Docker without sudo:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

## 3. Install cephadm

```bash
CEPH_RELEASE=19.2.5
curl --silent --remote-name --location https://download.ceph.com/rpm-${CEPH_RELEASE}/el9/noarch/cephadm
chmod +x cephadm

sudo ./cephadm add-repo --release squid
sudo ./cephadm install
which cephadm
```

## 4. Install the Ceph CLI

> Run this **after** cephadm is installed (Step 3).

```bash
sudo cephadm install ceph-common
ceph --version
```

## 5. Find the VM IP

```bash
ip r
```

Note the IP of the interface used by the default route (the `src` address), e.g. `192.168.1.50`.

## 6. Bootstrap the cluster

Replace `<VM-IP>` with the IP from Step 5.

```bash
sudo cephadm bootstrap --mon-ip <VM-IP>
```

On completion, the output shows the dashboard URL, username and password:

```
Ceph Dashboard is now available at:

             URL: https://<hostname>:8443/
            User: admin
        Password: xxxxxxxxxx
```

## 7. Verify

```bash
sudo ceph -s
sudo cephadm ls
sudo docker ps
```

Open the dashboard in a browser: `https://<VM-IP>:8443` and log in with the credentials shown at the end of the bootstrap output (a password change is requested on first login).

## 8. (Optional) Add storage

```bash
sudo ceph orch device ls
sudo ceph orch apply osd --all-available-devices
```

---

## Troubleshooting

| Problem                          | Fix                                                                              |
|----------------------------------|----------------------------------------------------------------------------------|
| `docker: permission denied`      | Log out and back in (or `newgrp docker`) after `usermod -aG docker`              |
| `ceph: command not found`        | Run `sudo cephadm install ceph-common` (after installing cephadm)                |
| Bootstrap fails on time sync     | `sudo systemctl enable --now chronyd`, then check `chronyc tracking`             |
| Bootstrap fails on hostname      | Use a short hostname: `sudo hostnamectl set-hostname ceph-node1`                 |
| `HEALTH_WARN` - no OSDs          | Add a raw disk, then `sudo ceph orch apply osd --all-available-devices`          |
| Dashboard unreachable            | Run `sudo ceph mgr services` and confirm the URL and port 8443                   |

## Uninstall / Cleanup

```bash
FSID=$(sudo ls /var/lib/ceph | head -1)
sudo cephadm rm-cluster --force --zap-osds --fsid $FSID
```

## References

- [cephadm documentation](https://docs.ceph.com/en/latest/cephadm/)
- [Ceph Squid release notes](https://docs.ceph.com/en/latest/releases/squid/)
- [Docker Engine on RHEL-family](https://docs.docker.com/engine/install/rhel/)
