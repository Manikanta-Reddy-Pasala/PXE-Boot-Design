# PXE Boot & Layered ISO Provisioning System

A design for network-provisioning a fleet of bare-metal servers from a **two-layer
image model** (a shared hardened **Vanilla** Ubuntu 20.04 base + a per-team
**Adaptation** layer), driven from a **PXE boot** pipeline with an **operator UI**,
**LDAP/Active Directory** login, full **audit logging**, and first-class
**debuggability + retry**.

> **Status:** Design proposal — for review. No infrastructure is built yet.
> Read [`docs/DECISIONS.md`](docs/DECISIONS.md) first if you only have five minutes;
> it lists the choices that need your sign-off before we implement.

## What this solves

| Need | How the design addresses it |
| --- | --- |
| Two OS layers (vanilla base + team adaptation) | Reproducible build pipeline producing independently **signed, versioned** squashfs layers + ISOs ([docs/03](docs/03-iso-build-pipeline.md)) |
| Boot machines over the network | proxyDHCP + TFTP + **iPXE** + HTTPS artifact serving, with a control plane that decides what each machine boots ([docs/04](docs/04-pxe-infrastructure.md)) |
| "Simple UI to choose servers on the network" | Web operator console: inventory, multi-select, pick image, provision/reimage/retry/debug, live status ([docs/05](docs/05-control-plane-and-ui.md)) |
| Login via Windows LDAP | AD bind via OIDC broker (Keycloak/Dex), AD groups → RBAC roles ([docs/06](docs/06-authentication-ldap.md)) |
| Logging / auditing | Immutable operator audit trail + centralized provisioning/build logs ([docs/07](docs/07-logging-auditing.md)) |
| "Not debuggable when something goes wrong" + retry | Layer isolation, rescue boot target, persistent boot logs, health checks, automatic retry/rollback ([docs/08](docs/08-debuggability-retry.md)) |

## How to read these docs

1. [`docs/01-requirements.md`](docs/01-requirements.md) — goals, scope, assumptions, glossary
2. [`docs/02-architecture.md`](docs/02-architecture.md) — the whole system on one page (diagrams + flows)
3. [`docs/03-iso-build-pipeline.md`](docs/03-iso-build-pipeline.md) — the two-layer build (vanilla + adaptation)
4. [`docs/04-pxe-infrastructure.md`](docs/04-pxe-infrastructure.md) — DHCP / TFTP / iPXE / HTTP boot chain
5. [`docs/05-control-plane-and-ui.md`](docs/05-control-plane-and-ui.md) — API + operator console
6. [`docs/06-authentication-ldap.md`](docs/06-authentication-ldap.md) — AD/LDAP auth and RBAC
7. [`docs/07-logging-auditing.md`](docs/07-logging-auditing.md) — audit + observability
8. [`docs/08-debuggability-retry.md`](docs/08-debuggability-retry.md) — debugging and self-healing
9. [`docs/09-security.md`](docs/09-security.md) — signing, secrets, network, hardening
10. [`docs/10-implementation-roadmap.md`](docs/10-implementation-roadmap.md) — phased delivery plan
11. [`docs/DECISIONS.md`](docs/DECISIONS.md) — decisions made, alternatives, **open questions for you**

## One-paragraph summary

A CI pipeline builds an immutable **Vanilla** Ubuntu 20.04 root filesystem
(`vanilla-<ver>.squashfs` + ISO) from a snapshotted apt mirror, and a per-team
**Adaptation** overlay (`team-<name>-<ver>.squashfs` + ISO) from a declarative spec.
Both are signed and registered in an image catalog. A target server PXE-boots,
chainloads **iPXE**, and calls back to a **control plane** with its MAC/UUID/serial.
The control plane — driven by an **operator web console** authenticated against
**Active Directory** — decides what that machine boots: a team image, vanilla,
a previous good version, or a **rescue/debug** environment. Every action is
**audited** against the operator's AD identity; every machine streams boot logs to
a central collector so failures are visible, and a **retry/rollback** policy makes
provisioning self-healing instead of a black box.
