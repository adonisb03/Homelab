# Homelab
A map of my homelab: a 2-node Proxmox VE cluster (donlabnetwork) running a mix of self-hosted services, dev sandboxes, and an inference server. This repo documents the architecture, not raw config exports — no MACs, UUIDs, or internal LAN IPs are included.
Cluster

Node	LAN	Role
pve1	<node1-lan-ip>/24	primary node
pve2	<node2-lan-ip>/24	secondary node

Both nodes run Proxmox VE 9.1.0 (kernel 6.17.2-1-pve, pve-manager 9.1.1) and are joined via corosync/knet, with a Tailscale overlay (100.88.xx.xx / 100.86.xxx.xx) used for cluster/management traffic.

Guests

VMID	Node	Name	Type	vCPU	RAM	Disk	Notes
100	pve1	—	VM (Windows 11)	4	4 GB	64 GB	
101	pve1	—	VM (Linux)	2	4 GB	32 GB	
102	pve2	mcserver	VM (Linux)	4	20 GB	64 GB SSD	
106	pve1	postgres	LXC	2	1 GB	8 GB	shared Postgres instance
111	pve1	hermeslab	VM (Linux)	10	14 GB	52 GB	
201	pve2	inflab	VM (Linux)	8	49 GB	104 GB SSD	GPU passthrough
999	pve1	pihole1	LXC	2	512 MB	4 GB	static LAN IP, network-wide DNS/ad-block
1000	pve1	firecrawl	LXC	3	4 GB	20 GB	
1001	pve1	camoufox	LXC	3	3 GB	16 GB	
1002	pve1	monitoring	LXC	2	1 GB	8 GB	monitoring stack, TUN device for VPN access

Network

	•	vmbr0 on each node bridges the physical NIC to a static LAN address; guests default to DHCP off that bridge unless noted otherwise (e.g. pihole1 has a static LAN IP so DNS is reachable at a fixed address).
	•	Cluster/management traffic between nodes rides a Tailscale overlay rather than the raw LAN.

Storage

	•	local — directory storage on each node (/var/lib/vz) for ISOs, container templates, and backups.
	•	local-lvm — LVM-thin pool (data on volume group pve) backing VM disks and container rootfs.

Repo scope

This repo intentionally documents the shape of the homelab (nodes, guests, resource allocation, network/storage design) rather than exporting live configs. Real IPs, MAC addresses, and hardware/VM identifiers are excluded — they're operational details specific to my LAN, not the architecture.
hows this for repo