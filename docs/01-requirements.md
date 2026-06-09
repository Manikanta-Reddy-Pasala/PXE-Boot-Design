# 01 — Requirements, Scope & Glossary

## 1.1 Problem statement

We provision bare-metal servers from ISO images built in **two layers**:

- **Vanilla base** — a hardened, common Ubuntu 20.04 OS shared by everyone.
- **Adaptation layer** — per-team customization stacked on the vanilla base
  (extra packages, configuration, IP/network settings, services, secrets).

Today the output is an ISO per combination, the two layers are opaque, and when
provisioning fails there is little visibility and no structured retry. We want a
**PXE-driven provisioning system** with an operator UI, AD/LDAP login, auditing,
and proper debuggability.

## 1.2 Goals

| # | Goal |
| --- | --- |
| G1 | Build a reproducible, signed, versioned **Vanilla** Ubuntu 20.04 image (squashfs + ISO). |
| G2 | Build per-team **Adaptation** images from a declarative spec, composed on a pinned vanilla version, independently signed/versioned. |
| G3 | PXE-boot any server on the network and have a central control plane decide what it boots. |
| G4 | Provide a **simple operator web UI** to discover servers, select one/many, choose a target image, and trigger provision / reimage / retry / debug. |
| G5 | Authenticate operators against **Windows Active Directory (LDAP)** with role-based access control. |
| G6 | **Audit** every operator action and **centralize** build + provisioning logs. |
| G7 | Make the layers and the boot process **debuggable**: isolate which layer broke, keep logs across reboots, offer a rescue environment. |
| G8 | Provide **automatic retry and rollback** so a failed machine self-heals or is parked cleanly for a human. |

## 1.3 Non-goals (for v1, revisit later)

- Provisioning non-Ubuntu / non-x86 targets (kept in mind in the design, not built).
- Full configuration-management lifecycle *after* first boot (we hand off to the
  team's existing CM; we own provisioning up to a healthy first boot).
- Replacing the organization's production DHCP (we use **proxyDHCP** to coexist).
- Multi-datacenter federation (single-site first; design doesn't preclude it).

## 1.4 Assumptions (please confirm — see DECISIONS.md)

- A dedicated **provisioning VLAN/subnet** is available (or can be created).
- Target servers support **PXE/UEFI network boot**; most support **IPMI/Redfish**
  for remote power + next-boot control.
- An **Active Directory** is reachable over LDAPS/StartTLS for operator auth.
- We may stand up supporting services: an apt mirror, artifact store, Postgres,
  a log stack, and the control plane (VMs or containers).
- Scale target is on the order of **tens–hundreds of servers** and a handful of
  teams initially (the design scales further, but this sets defaults).

## 1.5 Key requirements detail

### Functional
- FR1 Discover bootable machines on the provisioning network (DHCP leases, iPXE
  check-ins, optional ARP/LLDP scan) and present them in the UI.
- FR2 Bind a machine (by MAC/UUID/serial) to a **desired image + version + action**.
- FR3 Drive the machine to that state on next network boot, optionally power-cycling
  it remotely via IPMI/Redfish.
- FR4 Stream live provisioning progress per machine to the UI.
- FR5 Offer rescue/debug boot and per-machine retry/rollback from the UI.
- FR6 Image catalog: list/promote/deprecate vanilla and team image versions.

### Non-functional
- NFR1 **Reproducible** builds (pinned packages, snapshotted repos) → same inputs, same image hash.
- NFR2 **Integrity**: every artifact signed; boot verifies signature/checksum before executing.
- NFR3 **Auditability**: who did what, when, to which machines, with which image — immutable.
- NFR4 **Resilience**: a failed provision never silently bricks a box; logs survive reboot.
- NFR5 **Least privilege**: RBAC, segmented network, secrets never baked into shared images.
- NFR6 **Coexistence**: do not disrupt existing network services.

## 1.6 Glossary

| Term | Meaning |
| --- | --- |
| **Vanilla image** | Shared hardened Ubuntu 20.04 base rootfs (squashfs) + bootable ISO. |
| **Adaptation image** | Per-team overlay composed on a pinned vanilla version. |
| **squashfs** | Compressed read-only filesystem image; the unit we layer and ship. |
| **Overlay (overlayfs)** | Kernel feature to stack a writable/upper layer over a read-only lower layer. |
| **iPXE** | Scriptable open-source network bootloader (HTTP(S), menus, chainloading). |
| **proxyDHCP** | A DHCP helper that supplies *only* boot options, leaving IP assignment to existing DHCP. |
| **Control plane** | The API + DB that decides what each machine boots and records state. |
| **Operator** | A human (authenticated via AD) who drives provisioning from the UI. |
| **Rescue/debug image** | A minimal live environment with networking + tools for triage. |
| **Image catalog** | The registry of available signed image versions. |
| **SBOM** | Software Bill of Materials — the exact package set inside an image. |
