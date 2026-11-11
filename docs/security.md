# Security notes

How the homelab is locked down, what I tested, and what is still open. The aim is least privilege: every zone gets only what it needs, and anything else is denied.

## Threat model

This is a home network with a lab inside it. The risks I designed for:

- A compromised IoT device or lab VM reaching my real devices
- An attack VM (Kali) or a vulnerable target escaping the Lab
- Management interfaces (Proxmox, firewall, JetKVM) being reachable from untrusted zones
- Services on the Pi being exposed to the internet
- Losing the password vault

Out of scope: physical access to the rack, and the home Wi-Fi devices behind the Deco, which sit upstream of the firewall (see Known gaps).

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
- Secrets live in `.env` files that are never committed, and each project commits a `.env.example`.
- The data drive uses `nofail`, so a missing drive cannot stop the Pi booting.

### Vaultwarden

- Sign-ups disabled.
- Nightly backup at 03:30: SQLite `.backup`, then GPG AES-256 encryption.
- The backup was restore-tested into a throwaway container before real passwords went in.

### Proxmox (Management)

- Fixed address on the Management VLAN, VLAN-aware bridge, enterprise repository disabled.
- System fully updated.

### Lab VMs

- Kali sits on the Lab VLAN with a `clean-install` snapshot to roll back after experiments.
- Isolation was tested from inside the VM (below).
- Same-VLAN VMs talk directly across the Proxmox bridge without passing through the firewall, so VMs that must not trust each other should go on different VLANs.

## Testing

I tested each zone from a laptop on a test switch port, then retested the Lab from the Kali VM itself.

| Test | Expected | Result |
|------|----------|--------|
| Internet and DNS from Lab, IoT, Management | Works | Pass |
| Ping to other zones from Lab, IoT, Management | Blocked | Pass |
| Ping to the firewall's own address from restricted zones | Blocked | Pass |
| From Kali: reach the Pi (Trusted) | Blocked | Pass (timeout) |
| From Kali: reach the Proxmox web UI (Management) | Blocked | Pass (timeout) |
| From Kali: reach the Deco | Blocked | Pass (timeout) |
| Vaultwarden backup restored into a clean container | Data intact | Pass |
| Pi reboot: drive, Docker, Plex and Samba return on their own | Yes | Pass |

Note: Management also blocks ping to everything, because the `PRIVATE_NETS` block comes before any pass rule. That is intended.

## Known gaps and next steps

- **Home Wi-Fi is outside the firewall.** The Deco is upstream of pfSense, so phones, TVs and guests are not segmented by it.
- **Trusted is allow-all.** It is the most trusted zone, but a compromised device there could reach everything. Outbound limits may follow.
- **JetKVM is still on the LAN.** It is planned to move to the Management VLAN.
- **My PC is not on the Trusted VLAN yet.**
- **Proxmox SSH is not hardened yet** (key-only login to follow).
- **pfSense is on 2.7.2.** An upgrade is planned at a quiet time, after saving a configuration backup.
- **No detection yet.** Wazuh is planned to collect logs from the Lab.
- **Backups are on the Pi's drive only.** Copies to another machine are planned.
- **Plex forward** exposes TCP 32400 to the Deco subnet. It is limited by source, but it is still a service on the Pi reachable from Wi-Fi devices.
