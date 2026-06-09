# Decisions & Open Questions

Read this first. It records the **defaults the design assumes** and the **questions
that need your answer** before/while we implement. Each open question notes how the
answer changes the build.

## Part A — Key design decisions (with rationale)

| # | Decision | Choice | Main alternative | Why this default |
| --- | --- | --- | --- | --- |
| D1 | Layer composition | **overlayfs at boot** (vanilla lower + team delta upper) **and** a merged ISO artifact | Full ISO rebuild per team | Tiny team builds, file-level diff of "what the team changed," independent signing, vanilla-only debugging. Still get a single ISO when needed. |
| D2 | Bootloader | **iPXE** (chainloaded) | GRUB netboot / pxelinux | Scripting, HTTPS boot, menus, and the control-plane callback that wires the UI to boots. |
| D3 | DHCP strategy | **proxyDHCP (dnsmasq)** | Take over DHCP | Coexists with production DHCP; no disruption; lower blast radius. |
| D4 | Operator auth | **OIDC broker (Keycloak/Dex) federating AD** | Direct LDAPS bind in app | SSO + MFA, AD creds never touch our app, clean token RBAC. |
| D5 | Base build tool | **debootstrap + chroot** | live-build / Packer | Reproducible, scriptable, snapshot-friendly. |
| D6 | apt determinism | **snapshotted mirror (aptly/pulp)** | Live-internet apt | Reproducible/offline builds + rollback + SBOM. |
| D7 | Control plane | **FastAPI/Go + Postgres + React** | Off-the-shelf MAAS/Foreman | Fits the exact two-layer + UI + audit + retry requirements; see D11. |
| D8 | Secrets / per-machine IP | **Injected at provision time (Vault/IPAM)** | Baked into team image | Keeps shared images signable/cacheable; real identity per machine. |
| D9 | Remote reimage | **IPMI/Redfish next-boot + power** | Manual PXE selection | Hands-off reimage from the UI (where BMC exists). |
| D10 | Logs vs audit | **Separate**: WORM/SIEM audit + Loki/ELK logs | One store | Different retention, immutability, and volume needs. |

## Part B — Open questions for you (please answer to finalize)

> These genuinely change implementation choices. Grouped; the most impactful first.

### Q1 — Build vs adopt the control plane (highest impact)
Do we **build** the bespoke control plane + UI as designed, or **adopt/extend an
existing tool** like **Canonical MAAS** or **Foreman** and add the two-layer overlay +
custom UI/audit on top?
- *Build*: exact fit for the two-layer model, UI, audit, retry; more engineering.
- *Adopt*: faster bare-metal/PXE/IPMI plumbing, but the two-layer overlay model and the
  specific UI/audit still need custom work and may fight the tool's assumptions.

### Q2 — Scale & sites
Roughly how many **servers** and **teams** at launch and at 12 months, and is it
**one site** or several? Drives HA, per-rack caches, and DHCP-relay topology.

### Q3 — Disk vs diskless
Are targets **install-to-disk** (persistent OS on local disk — steady state) or
**diskless/live** (run from squashfs each boot — appliance style), or a mix per team?
Affects the initrd, persistence, and the writable-layer design.

### Q4 — BMC availability
Do the servers have **IPMI/Redfish** BMCs we can drive for power + next-boot? If not
(or partially), the UI degrades to "set PXE + reboot manually" for those machines.

### Q5 — Active Directory specifics
- Can we get a **read-only AD service account** + the **LDAPS CA cert**, and is
  **LDAPS/StartTLS** reachable from the provisioning network?
- Is standing up an **OIDC broker (Keycloak/Dex)** acceptable, or must we do a
  **direct LDAPS bind** (no SSO/MFA)?
- What **AD groups** should map to admin / operator / team-scoped-operator / auditor?

### Q6 — Network policy / VLAN
Can we get a dedicated **provisioning VLAN** and run **proxyDHCP** on it (with DHCP
relay where needed)? Any constraints from net-sec on running TFTP/HTTP boot?

### Q7 — Secure Boot & secrets store
- Is **UEFI Secure Boot** required (need a signing key/process), or is HTTPS +
  checksum pinning sufficient for v1?
- Do we have (or can we stand up) **HashiCorp Vault** / an IPAM for provision-time
  secrets and per-machine IPs, or is there an existing system to integrate?

### Q8 — Retry policy defaults
Confirm defaults: `max_retries = 3`, exponential backoff, and **on exhaustion →
rescue** (vs `hold`). Should new versions **auto-rollback** to previous-good on
repeated failure?

### Q9 — Pilot team
Which **team** is the pilot for the adaptation layer, and can we get **2–3 lab
servers** for Phases 1–5?

### Q10 — CI platform
What CI runs the image builds (GitHub Actions / GitLab CI / Jenkins)? Drives where the
`ci/` pipelines live and how artifacts publish to the catalog.

## Part C — How to give feedback
Comment inline on these docs or reply with answers to the Qs above. Once Q1–Q9 are
settled I'll turn [docs/10](10-implementation-roadmap.md) into a concrete Phase 0/1
task breakdown and we can start building the vanilla pipeline.
