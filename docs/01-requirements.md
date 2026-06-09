# 01 — Requirements, Scope & Glossary

## 1.1 Problem

- Servers provisioned from ISOs built in **two layers**
- **Vanilla base** = hardened common Ubuntu 20.04
- **Adaptation layer** = per-team packages, config, IP/network, services, secrets
- Today: layers are opaque, failures invisible, no structured retry
- Want: PXE provisioning + operator UI + AD login + auditing + debuggability

## 1.2 Goals

- G1 — Reproducible, signed, versioned **vanilla** image (squashfs + ISO)
- G2 — Per-team **adaptation** images from a spec, on a pinned vanilla, signed/versioned
- G3 — PXE-boot any server; central control plane decides what it boots
- G4 — Simple operator UI: discover servers, select, pick image, provision/reimage/retry/debug
- G5 — Operator login via **Windows AD (LDAP)** with RBAC
- G6 — Audit every action + centralize build/provisioning logs
- G7 — Debuggable layers and boot: isolate broken layer, keep logs, rescue env
- G8 — Automatic retry + rollback (self-heal or park cleanly)

## 1.3 Non-goals (v1)

- Non-Ubuntu / non-x86 targets
- Post-first-boot config management (hand off to team's CM)
- Replacing production DHCP (use proxyDHCP)
- Multi-datacenter federation

## 1.4 Assumptions (confirm — see DECISIONS.md)

- Dedicated provisioning VLAN available
- Targets support PXE/UEFI; most have IPMI/Redfish
- AD reachable over LDAPS/StartTLS
- Can stand up: apt mirror, artifact store, Postgres, log stack, control plane
- Scale: tens–hundreds of servers, a few teams to start

## 1.5 Requirements

### Functional
- FR1 — Discover machines (DHCP leases, iPXE check-ins, optional ARP/LLDP)
- FR2 — Bind machine (MAC/UUID/serial) → image + version + action
- FR3 — Drive machine to that state on next boot; remote power via IPMI/Redfish
- FR4 — Stream live progress per machine to UI
- FR5 — Rescue/debug boot + per-machine retry/rollback from UI
- FR6 — Image catalog: list/promote/deprecate versions

### Non-functional
- NFR1 — Reproducible builds (pinned packages, snapshot repos)
- NFR2 — Integrity: signed artifacts, verified before boot
- NFR3 — Auditability: who/what/when/which-server/which-image, immutable
- NFR4 — Resilience: no silent bricking; logs survive reboot
- NFR5 — Least privilege: RBAC, segmented network, no secrets in shared images
- NFR6 — Coexist with existing network services

## 1.6 Glossary

- **Vanilla image** — shared hardened Ubuntu 20.04 base (squashfs + ISO)
- **Adaptation image** — per-team overlay on a pinned vanilla
- **squashfs** — compressed read-only filesystem; the layered unit
- **overlayfs** — kernel feature stacking writable/upper over read-only lower
- **iPXE** — scriptable network bootloader (HTTPS, menus, chainload)
- **proxyDHCP** — supplies only boot options; leaves IP assignment to existing DHCP
- **Control plane** — API + DB deciding boots and recording state
- **Operator** — human (AD-authenticated) driving provisioning
- **Rescue/debug image** — minimal live env with networking + tools
- **Image catalog** — registry of signed image versions
- **SBOM** — software bill of materials (exact package set)
