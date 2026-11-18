# Segmented Security Homelab

A 6U rack-mounted homelab built to learn network segmentation, firewalling and monitoring on real hardware. It runs a pfSense firewall, a Proxmox host for attack and defence VMs, a VLAN-capable switch and a Raspberry Pi for services, with out-of-band management through a JetKVM.

![All the parts](docs/images/parts.jpg)

## Goals

- Segment the network into zones with VLANs and firewall rules
- Run attacker and vulnerable VMs in an isolated lab
- Collect logs and detect attacks (SIEM)
- Remote access without opening the home network to the internet
- Document the whole build, including mistakes

## Status

| Step | Status |
|------|--------|
| Parts ordered and delivered | Done |
| Rack assembled | Done |
| HP BIOS setup (virtualization, boot, power) | Done |
| pfSense install, WAN on a tagged VLAN, LAN, DHCP and DNS | Done |
| Internal VLANs (10/20/30/99), DHCP and default-deny firewall rules | Done, tested per zone |
| Pi rebuild: Docker, Plex, Samba on the Trusted VLAN | Done, reboot-tested |
| Tailscale, Vaultwarden (restore-tested backup), Uptime Kuma, SSH hardening | Done |
| My PC moved onto the Trusted VLAN | Done |
| JetKVM moved to the Management VLAN | Done, Lab isolation tested |
| Proxmox on the Management VLAN, updates working | Done |
| Proxmox SSH hardened (key-only, tested both ways) | Done |
| Kali VM in the Lab (VLAN 20), isolation tested | Done |
| DVWA target VM in the Lab, isolation tested | Done |
| Attack tests from Kali against DVWA | In progress (brute force done) |
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

![The rack](docs/images/rack_final_pixelated.jpg)

## Network design

![Network diagram](docs/images/network-diagram.png)

```
Internet -> Deco -> TL-SG108E (WAN VLAN 2) -> pfSense (one NIC, trunk) -> TL-SG108E (LAN) -> devices
```

The firewall has a single onboard NIC, so it runs as a "router on a stick", carrying all VLANs as tagged sub-interfaces on one trunk. pfSense maps the WAN to `em0.2` and the LAN to `em0` (192.168.1.0/24). Switch port layout, the patch panel, and the full zone breakdown are in [`docs/network-security.md`](docs/network-security.md).

### Firewall policy

Each zone is default-deny. pfSense reads rules top to bottom and the first match wins, so every zone follows the same pattern:

| Zone | Rules, in order |
|------|-----------------|
| IoT, Lab | Allow DNS to pfSense, block everything to private networks, allow the rest (internet only) |
| Management | Allow DNS, block private networks, allow NTP (UDP 123) and HTTPS (TCP 443) out, plus one narrow exception: TCP 80 from the Proxmox host to a single hostname (see problem 5) |
| Trusted | Allow all. Nothing in other zones can start a connection back in |

The "private networks" rule uses one alias (`PRIVATE_NETS`: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16), so it's edited in one place. I tested each zone from a laptop on the test port: the internet works, and ping to other zones and to the firewall's own address fails.

- The TL-SG108E is an "Easy Smart" managed switch. Unmanaged switches can't do VLANs.
- Cable colour coding uses white cables with coloured labels: green trusted, blue lab, orange IoT/DMZ, grey uplink.
- Firewall rules are documented in [`docs/network-security.md`](docs/network-security.md).

## Hardware

| Part | Role |
|------|------|
| 2 × HP EliteDesk 800 G3 Mini (i5-6500T, 8GB, 256GB) | Firewall and Proxmox host |
| TP-Link TL-SG108E | VLAN switch |
| Raspberry Pi 5 (4GB) | Plex, Samba, Uptime Kuma, Tailscale, Vaultwarden |
| JetKVM | Remote console and BIOS access, on the Management VLAN |
| Tecmojo 6U 10" rack + 12-port patch panel + brush panels | Enclosure and cable management |

## Software

- **Firewall:** pfSense (routing, NAT, rules, VLAN interfaces)
- **Hypervisor:** Proxmox VE 9 (Kali and DVWA running; a SIEM planned)
- **Pi:** Raspberry Pi OS Lite (64-bit), Docker Compose, Plex, Samba, Vaultwarden, Uptime Kuma
- **Remote access:** Tailscale to reach the Pi's services, nothing exposed to the internet. A subnet router with ACLs is planned.
- **Planned:** Wazuh for detection, 16GB RAM upgrade for Proxmox

## Services on the Pi

The Pi sits on the Trusted VLAN with a static DHCP reservation, so its address never changes.

- **Storage:** a 1.8 TB ext4 drive is mounted by UUID in `/etc/fstab` with `nofail`, so the Pi still boots if the drive is unplugged.
- **Plex:** runs in Docker Compose with host networking. The media folders are mounted into the container read-only (`:ro`), so Plex can't change or delete anything. Plex's database lives on the drive, which avoids wearing out the SD card.
- **Samba:** shares only the movies and shows folders, with a user login. The default home and printer shares are removed. `hosts allow` limits access to my own subnets, so the Lab and IoT zones can't reach the shares.
- **Secrets:** tokens and passwords are in `.env` files that are never committed. Each project commits a `.env.example` instead.
- **Reboot test:** after a restart the drive, Docker, Plex and Samba all come back on their own.
- **Tailscale:** installed on the Pi and my PC for remote access. No router ports are opened for it.
- **Vaultwarden:** self-hosted password manager in Docker. It listens on localhost only and is published to my Tailscale network over HTTPS with `tailscale serve`. Sign-ups are off. A nightly script makes an encrypted backup (SQLite `.backup`, then GPG AES-256), and I restore-tested it into a throwaway container before putting real passwords in.
- **Uptime Kuma:** checks pfSense, the Deco, internet reachability, DNS, Plex, Samba and Proxmox every minute. Its data lives on the drive instead of the SD card to avoid wear.
- **SSH:** key-only login, password and root login off, 3 attempts per connection, only my own user allowed.
- **Plex port forward:** the only forward on the firewall. It accepts TCP 32400 from the home Wi-Fi (Deco) subnet only and sends it to the Pi.

## Proxmox and the lab

- **Host:** Proxmox VE 9 on the second EliteDesk, in the Management zone with a fixed address. Its bridge is VLAN-aware, so each VM is tagged onto a VLAN from its network settings.
- **Kali VM:** installed from the official ISO, which Proxmox verified against its SHA-256 checksum. It has 2 cores, 4 GB RAM and a 40 GB disk, sits on the Lab VLAN, and runs the QEMU guest agent. I took a `clean-install` snapshot so I can roll back after experiments.
- **Isolation test from Kali:** the internet and DNS work. The firewall's own address, the Pi (Trusted), the Proxmox web interface (Management) and the Deco all time out. A VM in the Lab can reach the internet but nothing in my real network.
- **Updates:** Debian's repositories use HTTPS. Proxmox's free repository needed a narrow exception, documented in `docs/network-security.md`.
- **SSH:** key-only. Password and keyboard-interactive login are off, root can only log in with a key, and there are 3 attempts per connection. I tested both halves: the key works, and a forced password attempt gets `Permission denied (publickey)` with no prompt.
- **DVWA target:** a Debian 13 VM (2 GB RAM, 20 GB disk, no desktop) on the Lab VLAN, running Apache, PHP and MariaDB. The database listens on localhost only. I took `clean-install` and `dvwa-ready` snapshots so I can roll back after attacks.
- **Lab tests:** from the DVWA VM, Proxmox and pfSense time out while the internet works. Kali can reach DVWA across the Lab VLAN, which is intended. From Kali, the JetKVM (Management) times out.

## Docs

- [`docs/network-security.md`](docs/network-security.md): threat model, zones, switch/patch panel, firewall rules, DNS, remote access and hardening by host
- [`docs/testing-and-gaps.md`](docs/testing-and-gaps.md): isolation test results, known gaps and lessons
- [`docs/dvwa-pentest.md`](docs/dvwa-pentest.md): DVWA target setup and the attack log (brute force so far, more to come)
- [`docs/parts-list.md`](docs/parts-list.md): everything I bought for the build

## What it demonstrates

- Network segmentation with VLANs and inter-zone firewall rules
- Least privilege, with the one exception documented and tightly scoped
- Safe, isolated attack and defence practice
- Self-hosting with remote access that doesn't expose the home network
- Documenting a build honestly, mistakes included
