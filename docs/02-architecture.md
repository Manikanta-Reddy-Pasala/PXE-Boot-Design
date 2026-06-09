# 02 — Architecture Overview

This is the whole system on one page. Deep dives live in docs 03–09.

## 2.1 Component map

```mermaid
flowchart TB
  subgraph Build["Build plane (CI)"]
    M[Snapshotted apt mirror<br/>aptly / pulp]
    VB[Vanilla builder<br/>debootstrap + chroot]
    AB[Adaptation builder<br/>per-team spec]
    SIGN[Sign + SBOM + checksum]
    M --> VB --> SIGN
    VB -- pinned vanilla --> AB --> SIGN
  end

  subgraph Registry["Artifact + catalog"]
    REG[(Image catalog<br/>squashfs + ISO + manifests)]
  end
  SIGN --> REG

  subgraph Control["Control plane"]
    API[Provisioning API]
    DB[(Postgres:<br/>machines, bindings,<br/>audit, sessions)]
    UI[Operator web console]
    AUTH[OIDC broker<br/>Keycloak/Dex]
    API <--> DB
    UI --> API
    UI --> AUTH
    AUTH <--> AD[(Active Directory<br/>LDAP/LDAPS)]
  end

  subgraph Net["Provisioning network services"]
    DHCP[proxyDHCP + TFTP<br/>dnsmasq]
    HTTP[HTTPS artifact + iPXE script server]
  end
  REG --> HTTP
  API --> HTTP

  subgraph Obs["Observability"]
    LOG[Log collector<br/>Loki/ELK + syslog]
    AUDITSTORE[(Write-once audit store)]
  end
  API --> AUDITSTORE
  API --> LOG

  subgraph Fleet["Target servers (bare metal)"]
    S1[Server A]
    S2[Server B]
    S3[Server N]
  end

  DHCP -. boot options .-> Fleet
  Fleet -- iPXE check-in --> API
  HTTP -- kernel/initrd/squashfs --> Fleet
  Fleet -- boot/install logs --> LOG
  API -- IPMI/Redfish power+nextboot --> Fleet
```

## 2.2 The two planes

- **Build plane** turns source specs into signed image artifacts. It runs in CI,
  is offline from the fleet, and is the *only* thing that produces images.
- **Run plane** (control plane + network services + fleet) consumes those
  artifacts to provision machines. It never builds images; it only selects,
  serves, boots, observes, and records.

Keeping these separate is what makes the system auditable and debuggable: an
image is a fixed, signed input; provisioning is a recorded transaction over it.

## 2.3 The two image layers

```mermaid
flowchart LR
  subgraph V["Vanilla layer (immutable, shared)"]
    VS[vanilla-20.04-1.4.0.squashfs<br/>+ kernel + initrd]
  end
  subgraph A["Adaptation layer (per team)"]
    AS[team-payments-2.1.0.squashfs<br/>delta only]
  end
  subgraph Boot["At boot on target"]
    LOWER[lower = vanilla squashfs]
    UPPER[upper = team squashfs]
    RW[writable = tmpfs / persistent]
    LOWER --> OVL[overlayfs merged root /]
    UPPER --> OVL
    RW --> OVL
  end
  VS --> LOWER
  AS --> UPPER
```

The team layer is a **delta** over a pinned vanilla version. They compose with
**overlayfs at boot**, and we *also* emit a fully-merged ISO per team for offline
/ USB use. Independent versioning + signing of each layer is what lets us answer
"is it vanilla or the team layer that broke?" (see [docs/08](docs/08-debuggability-retry.md)).

## 2.4 End-to-end provisioning flow

```mermaid
sequenceDiagram
  autonumber
  participant Op as Operator (AD user)
  participant UI as Operator console
  participant API as Control plane
  participant IPMI as IPMI/Redfish
  participant SRV as Target server
  participant DHCP as proxyDHCP/TFTP
  participant HTTP as HTTPS artifact server
  participant LOG as Log collector

  Op->>UI: Login (OIDC → AD)
  Op->>UI: Select server(s) + image + action (provision)
  UI->>API: POST /bindings (audited: who/what/when)
  API->>IPMI: Set next-boot=PXE, power cycle
  SRV->>DHCP: DHCP discover (PXE)
  DHCP-->>SRV: boot file = iPXE
  SRV->>HTTP: GET iPXE bootstrap script
  SRV->>API: GET /boot?mac&uuid&serial  (check-in)
  API-->>SRV: iPXE script → boot assigned image
  SRV->>HTTP: GET kernel + initrd + squashfs (verified)
  SRV->>LOG: stream boot/install progress
  SRV->>API: report stage transitions + health
  API-->>UI: live status (WebSocket/SSE)
  alt healthy
    API->>API: mark machine provisioned (audited)
  else failure
    API->>API: apply retry/rollback policy
  end
```

## 2.5 Machine state model

The control plane tracks each machine through an explicit state machine so the UI,
audit, and retry logic all share one source of truth.

```mermaid
stateDiagram-v2
  [*] --> Discovered
  Discovered --> Bound: operator assigns image+action
  Bound --> Booting: PXE check-in received
  Booting --> Installing: image fetched + verified
  Installing --> FirstBoot: layers applied
  FirstBoot --> Healthy: health checks pass
  FirstBoot --> Failed: health checks fail
  Installing --> Failed: error / timeout
  Booting --> Failed: verify/download error
  Failed --> Retrying: retry policy (backoff)
  Retrying --> Booting
  Failed --> Rescue: fall back to debug image
  Failed --> Held: max retries → park for human
  Healthy --> Bound: operator reimages
  Rescue --> Bound: operator re-assigns
  Held --> Bound: operator re-assigns
```

## 2.6 Technology choices (defaults — see DECISIONS.md)

| Concern | Default choice | Why |
| --- | --- | --- |
| Bootloader | **iPXE** (chainloaded from undionly/snponly) | Scriptable, HTTPS boot, menus, control-plane callback |
| DHCP strategy | **dnsmasq proxyDHCP** | Coexists with prod DHCP, no IP takeover |
| Base build | **debootstrap + chroot** (live-build optional) | Reproducible, scriptable, snapshot-friendly |
| Layer composition | **overlayfs at boot** + merged ISO artifact | True two-layer model, debuggable, fast team builds |
| apt determinism | **aptly / pulp snapshot mirror** | Reproducible builds, offline, rollback |
| Control plane API | **FastAPI (Python)** or **Go** | Fast to build, good async + WebSocket support |
| DB | **Postgres** | Relational state + audit, mature |
| Operator UI | **React + WebSocket/SSE** | Real-time fleet status |
| AuthN | **Keycloak/Dex (OIDC) federating AD** | SSO, MFA, keeps AD creds out of the app |
| Logs | **Loki + Grafana** (or ELK) | Centralized, label by machine/session |
| Remote power | **IPMI / Redfish** | Hands-off reimage from the UI |
| Image signing | **Secure Boot + signed kernels + cosign/GPG on artifacts** | Integrity from build to boot |
