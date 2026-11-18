# Security Policy

This is a personal homelab project, built for learning network segmentation and penetration testing — it isn't a production system or a service anyone else deploys.

## Scope

- The code and configs in this repo (compose files, firewall rule descriptions, scripts) are fair game to report issues on.
- `docs/dvwa-pentest.md` documents attacks against a *deliberately vulnerable* training app (DVWA) running isolated in my own lab VLAN. Those "vulnerabilities" are intentional and not something to report — that's the point of the exercise.

## Reporting a vulnerability

If you spot something unsafe in the repo itself (e.g. a real secret accidentally committed, a script that could do something harmful if someone else ran it, a genuinely bad config recommendation), please open a GitHub issue or reach out directly rather than a public PR with details attached.

I'll acknowledge reports within a few days — this is a side project I maintain in my spare time, not a monitored product, so response times won't be enterprise-grade.

## Not in scope

- Anything inside the isolated Lab VLAN (Kali, DVWA) — that's an intentionally vulnerable practice environment, fully documented in `docs/dvwa-pentest.md`.
- My actual home network, which this repo describes but does not expose.
