# Decisions & Open Questions

- Records defaults the design assumes + questions needing your answer
- Read this first

## Part A — Key decisions

- D1 — Layer composition: **overlayfs at boot** + merged ISO (vs full ISO rebuild) — tiny builds, file-level diff, vanilla-only debug
- D2 — Bootloader: **iPXE** (vs GRUB/pxelinux) — scripting, HTTPS, control-plane callback
- D3 — DHCP: **proxyDHCP (dnsmasq)** (vs take over DHCP) — coexists, no disruption
- D4 — Operator auth: **OIDC broker federating AD** (vs direct LDAPS bind) — SSO + MFA, creds never touch app
- D5 — Base build: **debootstrap + chroot** (vs live-build/Packer) — reproducible, scriptable
- D6 — apt determinism: **snapshot mirror (aptly/pulp)** (vs live apt) — reproducible/offline + SBOM
- D7 — Control plane: **FastAPI/Go + Postgres + React** (vs MAAS/Foreman) — exact fit; see Q1
- D8 — Secrets/IP: **injected at provision time (Vault/IPAM)** (vs baked in) — keeps shared images signable
- D9 — Remote reimage: **IPMI/Redfish next-boot** (vs manual) — hands-off
- D10 — Logs vs audit: **separate** (WORM/SIEM audit + Loki/ELK logs) — different retention/immutability

## Part B — Open questions for you

- **Q1 — Build vs adopt (highest impact):** build bespoke control plane + UI, or adopt/extend MAAS/Foreman and add the two-layer overlay + custom UI/audit?
- **Q2 — Scale & sites:** how many servers + teams at launch and 12 months? One site or several?
- **Q3 — Disk vs diskless:** install-to-disk, diskless/live, or a mix per team?
- **Q4 — BMC:** do servers have IPMI/Redfish for power + next-boot? (No BMC → manual PXE for those)
- **Q5 — AD specifics:**
  - Read-only service account + LDAPS CA cert + LDAPS/StartTLS reachable from prov network?
  - OK to stand up OIDC broker (Keycloak/Dex), or must do direct LDAPS bind (no SSO/MFA)?
  - Which AD groups → admin / operator / team-operator / auditor?
- **Q6 — Network/VLAN:** dedicated provisioning VLAN + proxyDHCP allowed (with relay where needed)? Net-sec constraints on TFTP/HTTP boot?
- **Q7 — Secure Boot & secrets:**
  - Secure Boot required for v1 (need signing key/process), or HTTPS + checksum pinning enough?
  - Have/standup Vault + IPAM for provision-time secrets, or integrate existing system?
- **Q8 — Retry defaults:** confirm `max_retries=3`, backoff, on-exhaustion → rescue (vs hold). Auto-rollback new versions on repeated failure?
- **Q9 — Pilot:** which team is pilot? Can we get 2–3 lab servers for Phases 1–5?
- **Q10 — CI platform:** GitHub Actions / GitLab CI / Jenkins? Drives `ci/` + artifact publishing

## Part C — Feedback

- Comment inline or reply with answers
- Once Q1–Q9 settled → I'll turn docs/10 into a concrete Phase 0/1 task breakdown and start the vanilla pipeline
