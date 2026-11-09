# Segmented Security Homelab

A 6U rack-mounted homelab built to learn network segmentation, firewalling and monitoring on real hardware. It runs a pfSense firewall, a Proxmox host for attack and defence VMs, a VLAN-capable switch and a Raspberry Pi for services, with out-of-band management through a JetKVM.

![All the parts](docs/images/parts.jpg)

## Goals

- Segment the network into zones with VLANs and firewall rules
- Run attacker and vulnerable VMs in an isolated lab
- Collect logs and detect attacks (SIEM)
- Remote access without port forwarding
- Document the whole build, including mistakes

## Status

| Step | Status |
|------|--------|
| Parts ordered and delivered | Done |
| Rack assembled | Done |
| HP BIOS setup (virtualization, boot, power) | Done |
| pfSense install, WAN on a tagged VLAN, LAN, DHCP and DNS | Done |
| Internal VLANs (10/20/30/99) and firewall rules | Next |
| Proxmox and lab VMs | Planned |
| Pi rebuild (Pi-hole, Plex, Tailscale, Vaultwarden) | Planned |
| Wazuh detection | Planned |

## Rack layout (6U, 10")

| U | Device |
|---|--------|
| 6 | HP EliteDesk 800 G3 Mini, pfSense firewall |
| 5 | HP EliteDesk 800 G3 Mini, Proxmox host |
| 4 | Patch panel (0.5U) + brush panel (0.5U) |
| 3 | TP-Link TL-SG108E switch + JetKVM (custom 3D-printed 1U plate) |
| 2 | Raspberry Pi 5 shelf |
| 1 | Spare |

The power board sits at the bottom or back. Power bricks are strapped at the back and the media drive is stuck to the rear.

## Network design

```
Internet -> Deco -> TL-SG108E (WAN VLAN 2) -> pfSense (one NIC, trunk) -> TL-SG108E (LAN) -> devices
```

The firewall has a single onboard NIC, so it runs as a "router on a stick". One cable carries both networks as VLANs:

| Switch port | Device | VLAN membership |
|-------------|--------|-----------------|
| 1 | Deco uplink | 2 (WAN), untagged |
| 2 | pfSense trunk | 1 untagged (LAN), 2 tagged (WAN) |
| 3-8 | Devices | 1 untagged for now |

pfSense maps the WAN to `em0.2` and the LAN to `em0` (192.168.1.0/24). Internal VLANs 10/20/30/99 get added on the same trunk next.

| Zone | VLAN ID | Purpose |
|------|---------|---------|
| Trusted | 10 | Personal devices |
| Lab | 20 | Proxmox VMs (Kali, vulnerable targets) |
| IoT / DMZ | 30 | Anything exposed or untrusted |
| Management | 99 | JetKVM and admin interfaces |

- The TL-SG108E is an "Easy Smart" managed switch. Unmanaged switches can't do VLANs.
- Cable colour coding uses white cables with coloured labels: green trusted, blue lab, orange IoT/DMZ, grey uplink.
- Firewall rules are documented in [`docs/security.md`](docs/security.md).

## Hardware

| Part | Role |
|------|------|
| 2 × HP EliteDesk 800 G3 Mini (i5-6500T, 8GB, 256GB) | Firewall and Proxmox host |
| TP-Link TL-SG108E | VLAN switch |
| Raspberry Pi 5 (4GB) | Plex, Pi-hole, Uptime Kuma, Tailscale, Vaultwarden |
| JetKVM | Remote console and BIOS access, on the management VLAN |
| Tecmojo 6U 10" rack + 12-port patch panel + brush panels | Enclosure and cable management |

## Software

- **Firewall:** pfSense (routing, NAT, rules, VLAN interfaces)
- **Hypervisor:** Proxmox VE (Kali, Metasploitable/DVWA, SIEM)
- **Pi:** Raspberry Pi OS Lite (64-bit), Docker Compose, Plex, Pi-hole, Uptime Kuma, Vaultwarden
- **Remote access:** Tailscale subnet router with ACLs, no port forwarding
- **Planned:** Wazuh for detection, 16GB RAM upgrade for Proxmox

## Problems I hit

**1. A USB Ethernet adapter made the WAN unreliable.**
- The plan was a USB3 adapter as the second NIC. After a pfSense update, the WAN lost its address and the link kept flapping.
- I first blamed the update. Checking `dmesg` and `usbconfig` showed the link going up and down on the adapter itself.
- Different USB ports, forced 100 Mbps and disabling offload didn't help. Reinstalling the older pfSense version still flapped, which ruled out the update.
- **Fix:** drop the adapter and use router-on-a-stick, with the WAN as a tagged VLAN on the onboard Intel NIC. Zero errors since.
- **Lesson:** change one variable at a time, and test the hardware before blaming the software.

## What it demonstrates

- Network segmentation with VLANs and inter-zone firewall rules
- Safe, isolated attack and defence practice
- Self-hosting with remote access that doesn't expose the home network
- Documenting a build honestly, mistakes included
