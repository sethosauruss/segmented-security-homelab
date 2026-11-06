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
| HP BIOS setup (virtualization, boot, power) | In progress |
| pfSense install | Next |
| VLANs and firewall rules | Planned |
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
Internet -> Deco Wi-Fi -> pfSense -> patch panel -> TL-SG108E (802.1Q) -> devices
```

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
| USB3 Ethernet adapter | Second NIC for the firewall |

## Software

- **Firewall:** pfSense (routing, NAT, rules, VLAN interfaces)
- **Hypervisor:** Proxmox VE (Kali, Metasploitable/DVWA, SIEM)
- **Pi:** Raspberry Pi OS Lite (64-bit), Docker Compose, Plex, Pi-hole, Uptime Kuma, Vaultwarden
- **Remote access:** Tailscale subnet router with ACLs, no port forwarding
- **Planned:** Wazuh for detection, 16GB RAM upgrade for Proxmox

## What it demonstrates

- Network segmentation with VLANs and inter-zone firewall rules
- Safe, isolated attack and defence practice
- Self-hosting with remote access that doesn't expose the home network
- Documenting a build honestly, mistakes included

## Problems I hit

- Copied an `.img.gz` instead of the extracted image when flashing
- Pulled the USB installer before the reboot finished
- Downloaded the wrong
