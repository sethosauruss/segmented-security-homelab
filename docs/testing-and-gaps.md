# Testing and known gaps

Isolation test results and what's still open. For the network design behind these tests, see `network-security.md`. For DVWA-specific tests, see `dvwa-pentest.md`.

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
| From Kali: reach the JetKVM (Management) | Blocked | Pass (timeout) |
| From the DVWA VM: reach Proxmox and pfSense | Blocked | Pass (timeout) |
| From the DVWA VM: reach the internet | Works | Pass |
| From Kali: reach the DVWA web app (same VLAN) | Works, by design | Pass |
| Proxmox SSH: key login | Works | Pass |
| Proxmox SSH: forced password login | Refused | Pass (`Permission denied (publickey)`) |
| Switch audit: each port's VLAN membership matches the design | Match | Fixed two leftovers (Proxmox port in VLAN 1, test port in Management), then Pass |
| Vaultwarden backup restored into a clean container | Data intact | Pass |
| Pi reboot: drive, Docker, Plex and Samba return on their own | Yes | Pass |

Note: Management also blocks ping to everything, because the `PRIVATE_NETS` block comes before any pass rule. That is intended.

## Known gaps and next steps

- **Home Wi-Fi is outside the firewall.** The Deco is upstream of pfSense, so phones, TVs and guests are not segmented by it.
- **Trusted is allow-all.** It is the most trusted zone, but a compromised device there could reach everything. Outbound limits may follow.
- **Lab VMs can reach the internet.** The vulnerable DVWA VM could make outbound connections. A tighter rule, or a separate attack-only network, may follow.
- **Trusted can start connections into the Lab.** That is intended, because I work on the lab from my PC, but a compromised PC could reach the lab.
- **Same-VLAN VMs trust each other.** Kali and DVWA talk directly across the Proxmox bridge without passing through the firewall.
- **The switch's web page uses plain HTTP.** The TL-SG108E has no HTTPS, and its page is reachable from Trusted through the firewall, so its login could be sniffed by a compromised device on the path.
- **Port 6 is an open VLAN 1 port.** It is the recovery port. Physical access is out of scope, but I leave it unplugged.
- **Proxmox web login has no second factor yet.** Two-factor on the root login is planned.
- **Uptime Kuma is published on all of the Pi's interfaces.** Only the Trusted VLAN can reach it, but binding it to one address is planned.
- **pfSense is on 2.7.2.** An upgrade is planned at a quiet time, after saving a configuration backup.
- **No detection yet.** Wazuh is planned to collect logs from the Lab.
- **Backups are on the Pi's drive only.** Copies to another machine are planned.
- **Plex forward** exposes TCP 32400 to the Deco subnet. It is limited by source, but it is still a service on the Pi reachable from Wi-Fi devices.

## Lessons

- Change one variable at a time, and test before blaming the software.
- The error code tells you which layer is at fault (`REFUSED` pointed at Unbound, not the firewall).
- Read a verification error before bypassing it, and keep any exception as narrow as it can be.
