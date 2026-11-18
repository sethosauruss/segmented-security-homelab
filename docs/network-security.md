# Network security

How the homelab is locked down: zones, firewall rules, DNS and remote access, then hardening per host. For test results and open gaps, see `testing-and-gaps.md`. For the DVWA attack log, see `dvwa-pentest.md`.

![Network diagram](docs/images/network-diagram.png)

## Threat model

This is a home network with a lab inside it. The risks I designed for:

- A compromised IoT device or lab VM reaching my real devices
- An attack VM (Kali) or a vulnerable target escaping the Lab
- Management interfaces (Proxmox, firewall, JetKVM) being reachable from untrusted zones
- Services on the Pi being exposed to the internet
- Losing the password vault

Out of scope: physical access to the rack, and the home Wi-Fi devices behind the Deco, which sit upstream of the firewall (see `testing-and-gaps.md`).

## Switch and patch panel

| Switch port | Device | VLAN membership |
|-------------|--------|-----------------|
| 1 | Deco uplink | 2 (WAN), untagged |
| 2 | pfSense trunk | 1 untagged (LAN), 2 tagged (WAN) |
| 2 (also) | pfSense trunk | 10, 20, 30, 99 tagged |
| 3 | Proxmox host | 99 (Management) untagged, 20 (Lab) tagged |
| 4 | Raspberry Pi 5 | 10 (Trusted), untagged |
| 5 | JetKVM | 99 (Management), untagged |
| 6 | Recovery port | 1 (LAN) untagged, my way back in if a VLAN change locks me out |
| 7 | Test port | Parked in VLAN 999 (no routing). Moved into a zone VLAN to test it, then back |
| 8 | My PC | 10 (Trusted), untagged |

pfSense maps the WAN to `em0.2` and the LAN to `em0` (192.168.1.0/24). The internal VLANs run as tagged sub-interfaces on the same trunk.

Every device except the Pi goes through the 12-port patch panel. That way I can trace or swap any link from the front without touching the switch's VLAN settings. Both ends of each cable are labelled with the panel port and the device.

| Panel port | Device | Switch port |
|------------|--------|-------------|
| 1 | Deco uplink | 1 |
| 2 | pfSense trunk | 2 |
| 3 | Proxmox host | 3 |
| 4 | JetKVM | 5 |
| 12 | My PC | 8 |

Panel ports 5 to 11 are spare. The Pi plugs straight into switch port 4 from its shelf. VLAN membership belongs to the switch port, not the panel port, so a device must stay on its switch port or it lands on the wrong VLAN.

The Proxmox host has its own switch port: Management (99) untagged and Lab (20) tagged. Its VLAN-aware bridge tags each VM onto the right VLAN, so one cable carries both the host and the lab VMs.

## Zones

| Zone | VLAN | Subnet | Trust level |
|------|------|--------|-------------|
| Trusted | 10 | 192.168.10.0/24 | High: personal devices and the Pi |
| Lab | 20 | 192.168.20.0/24 | Low: attack and vulnerable VMs |
| IoT / DMZ | 30 | 192.168.30.0/24 | Low: anything untrusted or exposed |
| Management | 99 | 192.168.99.0/24 | High: admin interfaces, tightly limited outbound |

## Firewall rules (pfSense)

pfSense evaluates rules top to bottom and the first match wins. A zone with no matching pass rule is denied by default, and traffic is only allowed in the direction of the rule (nothing outside can start a connection back into a zone).

`PRIVATE_NETS` is a single alias: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`. Every "block private" rule uses it, so changes happen in one place.

### Lab and IoT

| # | Action | Destination | Why |
|---|--------|-------------|-----|
| 1 | Pass | Interface address, DNS | Clients need name resolution |
| 2 | Block | `PRIVATE_NETS` | Cannot reach any other internal network |
| 3 | Pass | Any | Internet only, because the block above already removed internal ranges |

### Management

| # | Action | Destination | Why |
|---|--------|-------------|-----|
| 1 | Pass | Interface address, DNS | Name resolution |
| 2 | Block | `PRIVATE_NETS` | No lateral movement |
| 3 | Pass | UDP 123 | Time sync |
| 4 | Pass | TCP 443 | Updates over HTTPS |
| 5 | Pass | TCP 80 from `192.168.99.10` to alias `PROXMOX_REPO` only | Exception, explained below |

### Trusted

- Pass any. Nothing in another zone can initiate a connection into Trusted.
- Samba on the Pi also uses `hosts allow` for my own subnets, so the Lab and IoT zones couldn't reach the file shares even if a firewall rule changed.

### The one exception: Proxmox repository over HTTP

- Proxmox's package CDN answered `download.proxmox.com` with a certificate for `enterprise.proxmox.com`, so `apt` over HTTPS failed with `certificate verify failed`.
- I did not disable certificate checking. Packages and the repository index are signed with Proxmox's GPG key and `apt` verifies that signature, so HTTP is acceptable for this one source.
- The rule is limited to a single source host and a single hostname alias on port 80. Everything else in Management remains HTTPS-only.
- Review this if Proxmox fixes the certificate: switch the source back to HTTPS and delete the rule and alias.

### Inbound from the internet

- No port forwards from the internet. The WAN is a private address behind the Deco.
- The only forward on the firewall is **Plex**: TCP 32400 from the home Wi-Fi subnet (Deco side) to the Pi. It is source-restricted, so it does not accept traffic from anywhere else.

## DNS

- Unbound serves each VLAN, with an explicit access-list entry per subnet. Without it, clients got `REFUSED` even though the firewall rule allowed DNS.
- Lab, IoT and Management clients can only query the firewall's own interface address.

## Remote access

- Tailscale on the Pi and my PC. No router ports are opened for it.
- Vaultwarden listens on `127.0.0.1` only and is published to the tailnet over HTTPS with `tailscale serve`.

## Hardening by host

### Raspberry Pi (Trusted)

- SSH key-only. Drop-in file `/etc/ssh/sshd_config.d/00-hardening.conf` sets: `PermitRootLogin no`, `MaxAuthTries 3`, `LoginGraceTime 30`, `AllowUsers` limited to my user, `X11Forwarding no`. It is named `00-` so it is read first and wins (sshd uses the first value it sees).
- Verified with `sshd -T`, not just by reading the file.
- Plex media folders are mounted read-only into the container.
- Samba defines only the two media shares. The default home, printer and print$ shares are removed.
- Secrets live in `.env` files that are never committed, and each project commits a `.env.example`.
- The data drive uses `nofail`, so a missing drive cannot stop the Pi booting.

### Vaultwarden

- Sign-ups disabled.
- Nightly backup at 03:30: SQLite `.backup`, then GPG AES-256 encryption.
- The backup was restore-tested into a throwaway container before real passwords went in.

### Proxmox (Management)

- Fixed address on the Management VLAN, VLAN-aware bridge, enterprise repository disabled.
- System fully updated.
- SSH key-only. Drop-in file `/etc/ssh/sshd_config.d/00-hardening.conf` sets: `PasswordAuthentication no`, `KbdInteractiveAuthentication no`, `PermitRootLogin prohibit-password`, `MaxAuthTries 3`, `LoginGraceTime 30`, `X11Forwarding no`. Verified with `sshd -T` and by testing both a key login and a forced password login.
- The change was applied with `reload`, with the JetKVM console open as a way back in.

### Switch (TL-SG108E)

- Each port is a member only of the VLANs it needs. I compare the 802.1Q table against the design after any change, because the firewall tests don't catch layer-2 leaks.
- Proxmox (port 3) carries Management untagged and Lab tagged, and is not in VLAN 1. The JetKVM (port 5) is Management only. The Pi (port 4) and my PC (port 8) are Trusted only.
- Unused and test ports are parked in VLAN 999 (`PARKED`), which has no firewall interface, so a device plugged in there can't reach any network.
- Port 6 stays on VLAN 1 as a recovery port, in case a VLAN change locks me out of the switch's own page.
- I save a switch config backup before VLAN changes, and keep it out of the repo.

### Lab VMs

- Kali sits on the Lab VLAN with a `clean-install` snapshot to roll back after experiments.
- Same-VLAN VMs talk directly across the Proxmox bridge without passing through the firewall, so VMs that must not trust each other should go on different VLANs.
- The DVWA target VM and everything done to it is covered in `dvwa-pentest.md`.
