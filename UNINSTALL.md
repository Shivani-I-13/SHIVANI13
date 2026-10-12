# Uninstall Ceph (cephadm) from Rocky Linux 9.8

This removes the Ceph cluster, its containers, packages, repository and configuration from the VM.

> **Warning:** This deletes the cluster and **all data stored in it**. `--zap-osds` also wipes the OSD disks. Run this only when you want to start over or decommission the VM.

---

## 1. Install the Ceph CLI (if not already installed)

```bash
sudo dnf install -y ceph-common
```

## 2. Remove the cluster

Get the cluster FSID:

```bash
sudo cephadm ls | grep fsid
FSID=$(sudo grep fsid /etc/ceph/ceph.conf | awk '{print $3}')
echo $FSID
```

Remove the cluster and wipe the OSD disks:

```bash
sudo cephadm rm-cluster --fsid $FSID --force --zap-osds
```

## 3. Remove leftover containers and images

If the containers ran on Podman:

```bash
sudo podman rm -af
sudo podman rmi -af
```

If the containers ran on Docker (as in the installation guide):

```bash
sudo docker rm -f $(sudo docker ps -aq)
sudo docker rmi -f $(sudo docker images -aq)
```

> If a command prints an error such as `requires at least 1 argument`, there is nothing left to remove for that runtime. Continue to the next step.

## 4. Remove packages and repository

```bash
sudo dnf remove -y cephadm ceph-common centos-release-ceph-squid
sudo dnf autoremove -y
```

## 5. Remove leftover files

```bash
sudo rm -rf /etc/ceph /var/lib/ceph /var/log/ceph /var/run/ceph
sudo rm -f /usr/sbin/cephadm /usr/local/sbin/cephadm
```

## 6. Verify nothing is left

```bash
sudo systemctl list-units 'ceph*'
sudo podman ps -a
sudo docker ps -a
which cephadm ceph
```

Expected:

- `systemctl list-units 'ceph*'` shows `0 loaded units listed`
- `podman ps -a` and `docker ps -a` show no containers
- `which cephadm ceph` prints nothing
