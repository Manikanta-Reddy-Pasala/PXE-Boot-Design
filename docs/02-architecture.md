# 02 — Architecture Overview

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

## 2.2 Two planes

- **Build plane** — CI only; turns specs into signed images; offline from fleet
- **Run plane** — control plane + network services + fleet; consumes images, never builds
- Separation = auditable (image is a fixed input, provisioning is a recorded transaction)

## 2.3 Two image layers

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

- Team layer = **delta** over a pinned vanilla version
- Compose via **overlayfs at boot**; also emit merged ISO per team
- Independent versioning/signing → can tell "vanilla broke" vs "team layer broke"

## 2.4 Provisioning flow

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
  Op->>UI: Select server(s) + image + action
  UI->>API: POST /bindings (audited)
  API->>IPMI: Next-boot=PXE, power cycle
  SRV->>DHCP: DHCP discover (PXE)
  DHCP-->>SRV: boot file = iPXE
  SRV->>HTTP: GET iPXE bootstrap
  SRV->>API: GET /boot?mac&uuid&serial
  API-->>SRV: iPXE script → boot assigned image
  SRV->>HTTP: GET kernel + initrd + squashfs (verified)
  SRV->>LOG: stream progress
  SRV->>API: report stages + health
  API-->>UI: live status
  alt healthy
    API->>API: mark provisioned (audited)
  else failure
    API->>API: apply retry/rollback policy
  end
```

## 2.5 Machine state model

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

## 2.6 Tech choices (defaults — see DECISIONS.md)

- Bootloader — **iPXE** (scriptable, HTTPS, control-plane callback)
- DHCP — **dnsmasq proxyDHCP** (coexists with prod DHCP)
- Base build — **debootstrap + chroot** (live-build optional)
- Layer composition — **overlayfs at boot** + merged ISO
- apt determinism — **aptly/pulp snapshot mirror**
- API — **FastAPI (Python)** or **Go**
- DB — **Postgres**
- UI — **React + WebSocket/SSE**
- AuthN — **Keycloak/Dex (OIDC)** federating AD
- Logs — **Loki + Grafana** (or ELK)
- Remote power — **IPMI / Redfish**
- Signing — **Secure Boot + signed kernels + cosign/GPG**
