# Segmented Security Homelab

A rack-mounted security homelab in a 6U 10-inch rack, built to learn network segmentation, virtualisation and Linux administration. It runs on my desk and is designed for practising segmentation, firewalling, monitoring and attack-and-defend labs.

> **Status:** in progress. Parts are on order. The status table below shows what is built and what is planned.

## Goals
- Build a segmented network with a real firewall (pfSense) and VLANs
- Run a Proxmox host for safe attack-and-defend labs
- Practise secure remote access without exposing services to the internet
- Document the build, including mistakes and fixes

## Architecture

```
Internet
   |
Deco Wi-Fi router
   |
pfSense firewall (HP EliteDesk 800 G3 Mini)
   |
TP-Link TL-SG108E managed switch (802.1Q VLANs)
   |-- Trusted VLAN   : my PC, laptop
   |-- Lab VLAN       : Proxmox host and lab VMs
   |-- IoT/DMZ VLAN   : Raspberry Pi 5 services (media, DNS, monitoring)
   |-- Management     : JetKVM
```

VLAN IDs: _to be decided._ Firewall rules will be documented in `docs/security.md`.

## Rack layout (Tecmojo 6U, 10")

| Unit | Device | Role |
|------|--------|------|
| U6 | HP EliteDesk 800 G3 Mini | pfSense firewall |
| U5 | HP EliteDesk 800 G3 Mini | Proxmox host |
| U4 | TP-Link TL-SG108E | Managed switch, VLANs |
| U3 | Raspberry Pi 5 + JetKVM | Services; remote keyboard/video/mouse |
| U2 | Cable brush panel | Cable management |
| U1 | 10" rack PDU | Power |

## Status

| Part | Status |
|------|--------|
| pfSense CE 2.7.2 installed and booting (on a test machine) | Done |
| Plex backup and Pi rebuild plan | Done |
| Hardware ordered | Planned |
| pfSense on the HP firewall | Planned |
| Proxmox on the HP host | Planned |
| VLANs on the switch, firewall rules | Planned |
| Pi rebuilt (Plex, Pi-hole, Uptime Kuma, Tailscale) | Planned |
| Remote access via Tailscale | Planned |
| Wazuh and attack simulation (CALDERA, Atomic Red Team) | Later (needs 16GB RAM) |

## What it demonstrates
- **Networking:** VLAN segmentation, routing, DHCP and DNS
- **Firewalling:** rule design between zones (default deny, one-way access from Trusted)
- **Virtualisation:** Proxmox VMs and containers
- **Linux administration:** a headless Pi 5 managed from the command line, Docker Compose, Samba, fstab
- **Secure remote access:** Tailscale instead of port forwarding
- **Documentation:** a dated build log and a record of problems and fixes

## Problems I hit (and fixed)
- **Copied the installer to the USB instead of flashing it:** a `.img.gz` has to be written with an imaging tool (balenaEtcher), not copied as a file.
- **Pulled the USB before pressing Reboot:** the system lost its install media and threw I/O errors. Fix: power-cycle, boot from the internal disk, and press Reboot first next time.
- **Wrong installer variant:** the serial-console image shows nothing on a monitor. I needed the VGA memstick image.

## Security note
This repository contains no credentials, tokens, firewall configuration backups or real network addresses. Example addresses are placeholders.

## Author
Seth Davies-Patrick, Bachelor of Information Technology student at La Trobe University.
