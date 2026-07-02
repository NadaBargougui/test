# RUNBOOK — Install MicroCloud on `newhomeserver`

> **Target**
>
> - Ubuntu 24.04 LTS
> - Single-node MicroCloud
> - Components: LXD + MicroOVN + MicroCloud
> - Storage: local ZFS pool `disks`
> - **No MicroCeph** (single-node homelab)

---

# Phase -1 — Verify IPv6 is NOT disabled

Before installing MicroCloud, check the GRUB configuration.

```bash
cat /etc/default/grub
```

Look for a kernel parameter similar to:

```text
ipv6.disable=1
```

or

```text
GRUB_CMDLINE_LINUX="... ipv6.disable=1 ..."
```

## Why?

During bootstrap, MicroCloud, LXD and MicroOVN perform several operations that rely on IPv6 internally.

Even if your infrastructure only uses IPv4, **IPv6 must not be disabled** during installation.

If `ipv6.disable=1` is present:

1. Remove it from `/etc/default/grub`.
2. Save the file.
3. Regenerate the bootloader:

```bash
sudo update-grub
sudo reboot
```

Failure to do this can cause `microcloud init` to fail or hang.

---

# Phase 0 — Inspect server resources

## Check OS

```bash
hostnamectl
lsb_release -a
uname -r
```

## Check CPU / RAM

```bash
lscpu
free -h
nproc
```

Recommended:
- ≥8 vCPU
- ≥32 GB RAM
- ≥200 GB storage

## Identify the disk for the ZFS pool

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT
sudo fdisk -l
```

### Selection criteria

Choose a disk that:

- is **not** the operating system disk;
- contains no important data;
- is dedicated to VM storage;
- has enough free capacity for future virtual machines.

Example:

```
sda -> Ubuntu operating system (DO NOT USE)
sdb -> Empty 1 TB SSD (GOOD CHOICE)
```

The selected disk will later become the ZFS storage pool named **disks**.

## Identify the uplink NIC

```bash
ip -br addr
ip route
```

### Selection criteria

The uplink interface should:

- be physically connected to your LAN;
- have Internet connectivity;
- be the interface that reaches your default gateway.

Use:

```bash
ip route
```

Example:

```
default via 192.168.1.1 dev eno1
```

Here `eno1` is the correct uplink interface.

---

# Phase 1 — Prepare the server

Update packages:

```bash
sudo apt update
sudo apt upgrade -y
```

Install required utilities:

```bash
sudo apt install -y curl jq bridge-utils net-tools zfsutils-linux
```

## Why these packages?

| Package | Purpose |
|---------|----------|
| curl | Download scripts and test HTTP APIs |
| jq | Read and manipulate JSON output |
| bridge-utils | Manage Linux bridges used by virtualization |
| net-tools | Legacy networking tools (`ifconfig`, `netstat`...) |
| zfsutils-linux | Required to create and manage ZFS pools |

If Docker is installed but unused:

```bash
sudo systemctl disable --now docker
```

---

# Phase 2 — Install MicroCloud

## Choose the Snap channels

Before installing the components, check the available Snap channels.

Canonical publishes several channels (stable, candidate, beta, edge) and multiple release tracks.

For production or reproducible development environments, always install the components from an explicit stable channel instead of relying on the default channel.

Display the available channels:

```bash
snap info lxd
snap info microcloud
snap info microovn
```

For this runbook, use:

| Component | Channel |
|-----------|----------|
| LXD | `5.21/stable` |
| MicroCloud | `2/stable` |
| MicroOVN | `24.03/stable` |

Installing from explicit channels ensures that every server uses the same feature set and avoids unexpected behavior caused by automatic migration to a newer major release.

## Install the components

```bash
sudo snap install lxd --channel=5.21/stable
sudo snap install microovn --channel=24.03/stable
sudo snap install microcloud --channel=2/stable
```

Verify the installation:

```bash
snap list | grep -E 'lxd|microovn|microcloud'
```

Expected output (versions may differ slightly):

```text
Name        Version                 Tracking
lxd         5.21.x                  5.21/stable
microcloud  2.x                     2/stable
microovn    24.03.x                 24.03/stable
```

---

## Prevent automatic updates

For a reproducible AuroraIQ development environment, it is recommended to disable automatic updates for these three components.

This ensures that all developers and deployment servers continue using the exact validated versions until the team explicitly decides to upgrade.

Hold the three snaps:

```bash
sudo snap refresh --hold lxd
sudo snap refresh --hold microcloud
sudo snap refresh --hold microovn
```

Verify:

```bash
snap refresh --time
```

When the team decides to upgrade, simply remove the hold:

```bash
sudo snap refresh --unhold lxd
sudo snap refresh --unhold microcloud
sudo snap refresh --unhold microovn
```

Then refresh manually:

```bash
sudo snap refresh lxd
sudo snap refresh microcloud
sudo snap refresh microovn
```

---

# Why no MicroCeph?

MicroCeph provides distributed replicated storage.

It becomes useful with three or more physical servers.

On one server it only increases complexity and resource usage while providing no real high availability.

---

# Phase 3 — Create the ZFS storage pool

**Replace `/dev/sdb` with the disk you selected during Phase 0.**

### Step 1

```bash
sudo wipefs -a /dev/sdb
```

What does it do?

- removes filesystem signatures;
- removes previous partition metadata;
- prepares the disk for a new storage configuration.

⚠️ This permanently erases metadata on the selected disk.

### Step 2

```bash
sudo zpool create disks /dev/sdb
```

What does it do?

- creates a new ZFS storage pool;
- names the pool `disks`;
- uses the selected disk as the storage backend.

Later, LXD stores VM root disks inside this pool.

Verify:

```bash
zpool status
zfs list
```

---

# Phase 4 — Bootstrap MicroCloud

Start:

```bash
sudo microcloud init
```

## Cluster members?

Answer:

```
No
```

Because this deployment has only one physical server.

## Local storage?

Answer:

```
Yes
```

Select:

```
disks
```

## Distributed storage?

Answer:

```
No
```

Because we do not install MicroCeph.

## Distributed networking?

Answer:

```
Yes
```

Because AuroraIQ will later create isolated OVN networks such as `portal-net`.

## Select the UPLINK

Choose the interface identified in Phase 0 (example: `eno1`).

Traffic flow:

```
VM
 ↓
OVN network
 ↓
UPLINK
 ↓
LAN
 ↓
Internet
```

## IPv4 Gateway

Example:

```
192.168.1.1/24
```

Usually your router.

## Address allocation range

Example:

```
First IP : 192.168.1.200
Last IP  : 192.168.1.220
```

### Why reserve addresses outside the DHCP scope?

Your router already distributes IP addresses using DHCP.

Example:

```
Router DHCP:
192.168.1.100
      ↓
192.168.1.180
```

If OVN also allocates addresses inside this range, two devices could receive the same IP address, causing conflicts.

Instead, reserve a separate block:

```
Router DHCP:
192.168.1.100 → 192.168.1.180

OVN external addresses:
192.168.1.200 → 192.168.1.220
```

Since the router never assigns addresses in this reserved range, OVN can safely use them without collisions.

## IPv6

Press Enter if unused.

## DNS

Example:

```
1.1.1.1,8.8.8.8
```

## Dedicated underlay?

Answer:

```
No
```

A single-node homelab does not require a dedicated underlay network.