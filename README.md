# Homelab

Architecture map of my homelab: a 2-node Proxmox VE cluster (`donlabnetwork`) running self-hosted services, dev sandboxes, a data pipeline, and a GPU inference server.

This repo documents the *shape* of the lab — nodes, guests, resource allocation, network, storage, and security design. It is not a config export: real IPs, MAC addresses, and hardware/VM identifiers are excluded.

## Cluster

| Node | Hardware | CPU | RAM | GPU | Role |
|------|----------|-----|-----|-----|------|
| pve1 | Lenovo ThinkCentre M80q | Intel Core i5-10500T (6C/12T) | 16 GB | — | Primary node: services, data, agents |
| pve2 | HP Z4 workstation | Intel Xeon W-2123 (4C/8T) | 64 GB | RTX 2070 (8 GB) | Inference node, GPU passthrough |

- **Proxmox VE:** 9.1.0 (kernel 6.17.2-1-pve, pve-manager 9.1.1) on both nodes
- **Clustering:** corosync/knet over a Tailscale overlay (see [Network](#network) for why)

## Guests

### Active

| VMID | Node | Name | Type | vCPU | RAM | Disk | Purpose |
|------|------|------|------|------|-----|------|---------|
| 106 | pve1 | postgres | LXC | 2 | 1 GB | 8 GB | Shared Postgres instance; backs the markets database |
| 111 | pve1 | hermeslab | VM (Linux) | 10 | 14 GB | 52 GB | Hermes: autonomous agent framework and its eval harness |
| 201 | pve2 | inflab | VM (Linux) | 8 | 49 GB | 104 GB SSD | LLM inference: llama-server with llama-swap for on-demand model swapping; GPU passthrough, ballooning disabled |
| 1000 | pve1 | firecrawl | LXC | 3 | 4 GB | 20 GB | Self-hosted Firecrawl: web crawling and scraping API |
| 1001 | pve1 | camoufox | LXC | 3 | 3 GB | 16 GB | Camoufox browser-automation service for scraping pipelines |
| 1002 | pve1 | monitoring | LXC | 2 | 1 GB | 8 GB | Prometheus + Grafana + Loki; TUN device enabled for VPN access |

### Stopped / pending decommission

| VMID | Node | Name | Type | Notes |
|------|------|------|------|-------|
| 102 | pve2 | mcserver | VM (Linux) | 4 vCPU / 20 GB / 64 GB SSD; game server (Minecraft), not running |
| 100 | pve1 | win11 | VM (Windows 11) | 4 vCPU / 4 GB / 64 GB; unused |
| 101 | pve1 | linux-sandbox | VM (Linux) | 2 vCPU / 4 GB / 32 GB; unused |
| 999 | pve1 | pihole1 | LXC | Former network DNS/ad-block; not running |

## Network

- **vmbr0** on each node bridges the physical NIC to a static LAN address. Guests use DHCP off that bridge unless noted.
- **Why Tailscale for cluster traffic:** the two nodes sit on physically separate networks (one behind a Wi-Fi extender), and keeping those networks separate is a hard requirement. Tailscale gives the cluster a single flat management plane without bridging the two LANs.
- **Tradeoff:** corosync is latency-sensitive, so cluster stability depends on the overlay's latency and on Tailscale itself being up.

## Storage

| Storage | Type | Backs |
|---------|------|-------|
| `local` | Directory (`/var/lib/vz`) | ISOs, container templates, backups |
| `local-lvm` | LVM-thin (pool `data` on VG `pve`) | VM disks, container rootfs |

## Observability

The `monitoring` container runs Prometheus for metrics, Loki for logs, and Grafana for dashboards across both nodes and their guests.

## Security

- **Management access:** how the Proxmox UI (8006) and SSH are reached, e.g. Tailscale-only, not exposed on the LAN or WAN
- **Authentication:** Proxmox realm, 2FA/TOTP on `root@pam`, SSH key-only auth, password auth disabled
- **Tailscale ACLs:** which devices can reach which nodes and guests
- **Firewall:** Proxmox datacenter/node/guest firewall rules, and default-deny inbound
- **Segmentation:** which guests can reach `postgres`; scraping containers isolated from management
- **Least privilege:** unprivileged LXCs where possible; which containers are privileged, and why
- **Patching:** update cadence for PVE hosts and guests
- **Secrets:** how DB credentials and API keys are stored; none are committed to this repo

## Known limitations

- **Two-node quorum:** with no QDevice, losing either node drops the cluster below quorum. TODO: add a QDevice or document the recovery procedure.
- **Backups are node-local:** `local` lives on the same disks as the guests. TODO: set up an off-node target (PBS or NAS) and schedule restore tests.
- **pve1 memory overcommit:** active guests are allocated 23 GB against 16 GB physical. `hermeslab` is allocated 14 GB but peaks around 8 GB under load. TODO: cut it to ~10 GB, which brings the total to 19 GB. The remaining overage is the LXC ceilings on `firecrawl`/`camoufox`, which fill up only if both scrape heavily at once.
- **pve2 headroom:** `inflab` pins 49 GB (ballooning disabled) of 64 GB. Restarting `mcserver` at its current 20 GB would overcommit the node, so cut it to ~8 GB before bringing it back.
- **Postgres sizing:** 1 GB RAM and 8 GB disk for a shared DB is tight. Watch disk growth.
- **Thin provisioning:** `local-lvm` can be overcommitted. Monitor pool usage.
