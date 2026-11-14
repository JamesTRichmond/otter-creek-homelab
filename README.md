# Otter Creek Homelab

A small self-hosted lab in Vermont where my AI agents, local models, and test sandboxes run. It's described here by **role**: no addresses, hostnames, or configs. That's on purpose.

## Why it exists

- Run open-weight models locally, so long agent loops cost electricity, not API credit.
- Give coding agents somewhere disposable to run untrusted code, away from my laptop and my accounts.
- Practice the operations side: virtualization, networking, backups, runbooks, incident cleanup.

## Architecture

```mermaid
flowchart LR
  L["Laptop<br/>review + merge<br/>key-only access"] -->|tasks| R
  subgraph NET["Segmented home network"]
    subgraph AH["Agent host · trusted"]
      R["Relay loop<br/>bounded, redacted results"]
      M["Local models<br/>localhost only"]
    end
    subgraph HV["Hypervisor · firewalled management"]
      subgraph SB["Isolated sandbox bridge · untrusted"]
        S["Ephemeral sandbox VMs<br/>no egress, destroyed after each run"]
      end
      W["Egress worker VM<br/>allowlisted outbound"]
      C["CI runner VM"]
      HH["Household VMs<br/>internet-only"]
    end
  end
  R --> M
  R -->|"code + tests, one SSH path, secrets excluded"| S
  S -->|results| R
  C --> G["GitHub<br/>protected main, required review"]
  K["Scheduled health checks"] -->|failures open issues| G
  classDef trusted fill:#5B54C7,stroke:#403A96,color:#FFFFFF
  classDef untrusted fill:#C3512C,stroke:#93391C,color:#FFFFFF
  classDef ops fill:#15866B,stroke:#0C6450,color:#FFFFFF
  classDef neutral fill:#646C76,stroke:#4B535C,color:#FFFFFF
  class R,M trusted
  class S,W,C untrusted
  class K ops
  class L,HH,G neutral
  style NET fill:transparent,stroke:#768390
  style HV fill:transparent,stroke:#768390
  style AH fill:transparent,stroke:#7A73D6
  style SB fill:transparent,stroke:#D46A42
  linkStyle default stroke:#768390,stroke-width:1.5px
```

| Role | What it does |
|---|---|
| Agent host | Serves open-weight models (MLX and Ollama) on localhost only and runs the relay loop |
| Hypervisor | Proxmox VE. Management is reachable only from the laptop and the agent host, SSH is key-only, and every VM is backed up nightly |
| Sandbox VMs | Built from a hardened template with CPU, memory, and disk caps. No internet, no route to the rest of the lab, destroyed after each run. They never see my keys |
| Egress worker | The one VM allowed outbound, and only to an allowlist |
| CI runner | Self-hosted GitHub Actions runner in its own VM, off the hypervisor host |
| Network | Segmented home network: fixed addresses for lab hosts, IoT and guest devices on their own segments, household VMs limited to the internet |

## Featured: the relay loop

A tag-team coding loop that runs entirely on local models:

1. A **builder** model writes code and tests for a task.
2. The code goes to a fresh sandbox VM and the tests run there under a hard timeout, enforced on the worker with a local backstop.
3. Two **reviewer** models critique the result.
4. Repeat until the tests pass and both reviewers are clean, or the round budget runs out.

What makes it trustworthy:

- **Secrets never leave the host.** `.env*`, keys, `.pem` files and `.git` are excluded from every copy, and there's no agent forwarding.
- **Sandbox output is treated as hostile.** Results are size-bounded and stripped of anything that looks like a credential before a reviewer model reads them.
- **Infrastructure failures aren't blamed on the model.** A dropped connection, a timeout, or the wrong worker answering is retried once, then the run stops with a distinct exit code. The builder only gets a failed round for its own code.
- **Every outcome is classified.** Pass, failing tests, timeout, no tests collected, missing dependency, nothing saved. Each lands in a per-round log.
- **Runaway tests get killed.** A deliberately hanging test is part of the examples, and it must be killed by the worker timeout.

## Operations

- **A living runbook.** The lab's runbook is a live page that a scheduled agent sweeps every night. It folds in what changed, restores anything missing, and trims stale steps. A static runbook goes stale in a week.
- **Checks that fail loud.** Scheduled checks cover backups, disk, exposed ports, sandbox egress, and credential scope. Any failure opens a GitHub issue, so nothing breaks quietly.
- **Nightly backups.** Every VM is snapshotted nightly with a rolling retention window.
- **Guardrails on the repos agents touch.** Branch protection on main, required review, no force pushes, linear history. Agents open PRs; I merge.
- **Incident practice.** I've already had to rotate an exposed gateway token and track down a runaway API-cost leak (wrong model tier, low cache hit rate, a chatty heartbeat).

## In progress

Moving from "hardened homelab" toward a lab where every trust boundary is physical, built the cheap way where the cheap way is also the right way: single-board computers, e-waste, and owned gear. Where it's headed:

```mermaid
%%{init: {"flowchart": {"nodeSpacing": 30, "rankSpacing": 40}}}%%
flowchart TB
  subgraph ACC["Access"]
    direction LR
    L["Laptop<br/>hardware key,<br/>device posture"]
    ID["Identity gateway<br/>SSO, mutual TLS"]
    OOB["Out-of-band<br/>console<br/>break-glass path"]
    L --> ID
    ID ~~~ OOB
  end
  ACC --> NET
  subgraph NET["Network · default deny"]
    direction TB
    FW["Zone firewall<br/>no inbound,<br/>egress allowlist"]
    SW["Segmentation<br/>unknown devices<br/>quarantined"]
    DNS["Private DNS<br/>filtered, own<br/>resolver"]
    HOME["Household + IoT<br/>isolated,<br/>never captured"]
  end
  NET --> TS
  NET -.-> SEN
  subgraph TS["Trust services"]
    direction TB
    CA["Internal CA<br/>offline root,<br/>short-lived certs"]
    V["Secrets vault<br/>hardware-backed,<br/>short-lived"]
    MIR["Artifact mirror<br/>pinned, verified<br/>code + models"]
  end
  TS --> CP
  subgraph CP["Control plane · trusted"]
    O["Orchestrator<br/>durable state,<br/>queue"]
    SG["Sandbox gate<br/>secrets stay out,<br/>results validated"]
    GA["GitHub App<br/>scoped tokens,<br/>signed commits"]
    O <--> SG
    O ~~~ GA
  end
  O ---> IG
  SG <--> S
  subgraph IP["Inference plane · trusted"]
    IG["Inference gateway<br/>local-only,<br/>mutual TLS"]
    MH["Model hosts<br/>agent host +<br/>GPU node"]
  end
  subgraph EP["Execution plane · untrusted"]
    S["MicroVM sandboxes<br/>no egress,<br/>single-use"]
  end
  EP -.-> DF
  subgraph DF["Forensics · evidence zone"]
    direction LR
    ACQ["Acquisition<br/>write-blocked,<br/>hashed"]
    EV["Evidence vault<br/>immutable,<br/>encrypted"]
    AN["Analysis<br/>offline,<br/>read-only"]
    ACQ --> EV --> AN
  end
  subgraph DET["Detection + recovery"]
    SEN["Sensors<br/>IDS tap,<br/>canaries"]
    SIEM["SIEM<br/>tamper-evident<br/>logs, paging"]
    OPS["Ops agent<br/>read-only checks,<br/>opens issues"]
    BK["Backups<br/>encrypted, pulled,<br/>immutable off-site"]
    SEN --> SIEM
    OPS ~~~ BK
  end
  SIEM ~~~ LR
  subgraph RNG["Practice range · air-gapped, logs out one-way"]
    LR["Lab router<br/>own network,<br/>no uplink"]
    ATK["Attack box +<br/>vulnerable<br/>targets"]
    HW["Hardware bench<br/>serial, USB, RF"]
    LR --> ATK
    LR --> HW
  end
  classDef trusted fill:#5B54C7,stroke:#403A96,color:#FFFFFF
  classDef untrusted fill:#C3512C,stroke:#93391C,color:#FFFFFF
  classDef evidence fill:#15866B,stroke:#0C6450,color:#FFFFFF
  classDef neutral fill:#646C76,stroke:#4B535C,color:#FFFFFF
  class CA,V,MIR,O,SG,GA,IG,MH trusted
  class S,LR,ATK,HW untrusted
  class SEN,SIEM,OPS,BK,ACQ,EV,AN evidence
  class L,ID,OOB,FW,SW,DNS,HOME neutral
  style ACC fill:transparent,stroke:#768390
  style NET fill:transparent,stroke:#768390
  style TS fill:transparent,stroke:#7A73D6
  style CP fill:transparent,stroke:#7A73D6
  style IP fill:transparent,stroke:#7A73D6
  style EP fill:transparent,stroke:#D46A42
  style RNG fill:transparent,stroke:#D46A42
  style DET fill:transparent,stroke:#2FA183
  style DF fill:transparent,stroke:#2FA183
  linkStyle default stroke:#768390,stroke-width:1.5px
```

- **Dedicated hardware per plane.** Orchestration, inference, and execution on separate machines, so untrusted code never shares a host with anything that holds state or credentials.
- **Default-deny network.** Managed switching with per-plane segments behind a zone firewall, unknown devices quarantined on arrival, and a mirror port feeding intrusion-detection sensors and canary services that page when touched.
- **Private DNS.** A filtering resolver doing its own recursion on a single-board computer, so lookups stay in the lab and ads, trackers, and known-bad domains die at the edge.
- **Trust services.** An internal certificate authority with an offline root issuing short-lived certs, a secrets vault backed by a hardware key, and an internal artifact mirror so builds install pinned, hash-verified packages and models instead of reaching for the internet.
- **A real control plane.** An orchestrator with durable state and a queue, a sandbox gate that keeps secrets out and validates results on the way back, and a scoped GitHub App in place of personal tokens.
- **An inference gateway.** One front door for every model, local-only with mutual TLS, plus a GPU node for models too large for the agent host.
- **MicroVM sandboxes.** Faster, smaller throwaway runners behind the sandbox gate.
- **An air-gapped practice range.** A lab router with no uplink fencing the attack box, deliberately vulnerable targets, and a hardware bench (serial, USB, RF — powered by a recycled ATX supply), with logs leaving one-way only.
- **A forensics pipeline with chain of custody.** Write-blocked, hashed acquisition — retired phones and e-waste drives are the practice corpus — into an immutable encrypted evidence vault, analyzed offline.
- **Access done right.** Hardware-key sign-in with device posture checks, and an out-of-band console as the break-glass path for when the network itself is down.
- **Observability.** A SIEM with tamper-evident logs and paging, fed by the sensors and the range; an ops agent that runs read-only checks and opens issues when something fails.
- **3-2-1 backups.** Pull-based so a compromised host can't reach them, encrypted, with an immutable off-site copy, automated restore tests, and battery backup for the rack.
- **Ephemeral CI.** A fresh runner per job, gone when the job ends.
- **Infrastructure as code** for the sandbox VMs, so a rebuild is one command.

— [James Richmond](https://github.com/JamesTRichmond)
