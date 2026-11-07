# Security design

Firewall rules and zone policy will be documented here as they are built.

## Zones (planned)

| Zone | VLAN ID | Purpose |
|------|---------|---------|
| Trusted | 10 | Personal devices |
| Lab | 20 | Proxmox VMs (Kali, vulnerable targets) |
| IoT / DMZ | 30 | Anything exposed or untrusted |
| Management | 99 | JetKVM and admin interfaces |

## Principles

- Default deny between zones, with explicit allow rules only
- Lab and IoT/DMZ cannot initiate connections to Trusted
- Remote access over Tailscale only, no port forwarding
- No secrets committed to this repository
