# PXE Boot & Layered ISO Provisioning System

- Network-provisions bare-metal servers from a **two-layer image model**
- **Vanilla** = shared hardened Ubuntu 20.04 base; **Adaptation** = per-team layer on top
- Driven by **PXE boot** + an **operator UI** to pick servers and image them
- **LDAP/Active Directory** login, full **audit logging**, **debug + retry** built in
- **Status:** design proposal for review — nothing built yet. Read `docs/DECISIONS.md` first.

## What it solves

- Two OS layers → reproducible, signed, versioned squashfs + ISO ([docs/03](docs/03-iso-build-pipeline.md))
- Network boot → proxyDHCP + TFTP + iPXE + HTTPS, control plane decides each boot ([docs/04](docs/04-pxe-infrastructure.md))
- "Simple UI to choose servers" → web console: inventory, multi-select, image, provision/retry/debug ([docs/05](docs/05-control-plane-and-ui.md))
- Windows LDAP login → AD via OIDC broker, groups → roles ([docs/06](docs/06-authentication-ldap.md))
- Logging/auditing → immutable audit trail + central logs ([docs/07](docs/07-logging-auditing.md))
- "Black box" + retry → layer isolation, rescue boot, persistent logs, auto retry/rollback ([docs/08](docs/08-debuggability-retry.md))

## Docs

- [`01-requirements.md`](docs/01-requirements.md) — goals, scope, glossary
- [`02-architecture.md`](docs/02-architecture.md) — system on one page
- [`03-iso-build-pipeline.md`](docs/03-iso-build-pipeline.md) — two-layer build
- [`04-pxe-infrastructure.md`](docs/04-pxe-infrastructure.md) — boot chain
- [`05-control-plane-and-ui.md`](docs/05-control-plane-and-ui.md) — API + UI
- [`06-authentication-ldap.md`](docs/06-authentication-ldap.md) — AD/LDAP + RBAC
- [`07-logging-auditing.md`](docs/07-logging-auditing.md) — audit + logs
- [`08-debuggability-retry.md`](docs/08-debuggability-retry.md) — debug + self-heal
- [`09-security.md`](docs/09-security.md) — signing, secrets, network
- [`10-implementation-roadmap.md`](docs/10-implementation-roadmap.md) — phased plan
- [`DECISIONS.md`](docs/DECISIONS.md) — decisions + **open questions**

## In short

- CI builds an immutable **vanilla** rootfs (squashfs + ISO) from a snapshotted apt mirror
- CI builds a per-team **adaptation** delta on a pinned vanilla version
- Both signed and listed in an image catalog
- Server PXE-boots → iPXE → checks in to control plane with MAC/UUID/serial
- Control plane (driven by AD-authenticated UI) decides what it boots
- Every action audited; every machine streams logs; failures auto retry/rollback
