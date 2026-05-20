<div align="center">

# 🛡️ FlowGuard

**Intelligent network security platform built on Software Defined Networking**

Real-time intrusion detection · Automated mitigation · Live topology monitoring

![Python](https://img.shields.io/badge/Python-3.11-blue?style=flat-square&logo=python&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-22-green?style=flat-square&logo=node.js&logoColor=white)
![OpenFlow](https://img.shields.io/badge/OpenFlow-1.3-orange?style=flat-square)
![React](https://img.shields.io/badge/React-19-61DAFB?style=flat-square&logo=react&logoColor=black)
![License](https://img.shields.io/badge/license-MIT-lightgrey?style=flat-square)

</div>

---

## Overview

FlowGuard is a research-grade network security platform that combines machine learning-based intrusion detection with Software Defined Networking (SDN) to detect and block attacks automatically — from capturing raw packets to installing OpenFlow rules on a switch, the entire pipeline runs in under a second.

The platform was built around four cooperating modules, each in its own repository:

| Module | Role | Port | Repo |
|---|---|---|---|
| [SDN-ML-Guard](#-sdn-ml-guard--detection-engine) | ML detection engine | `:8000` | [→](https://github.com/FlowGuard-platform/Flowguard-ML-Engine) |
| [FlowGuard-Mitigating-Engine](#-flowguard-mitigating-engine--decision-center) | Security decision center | `:9000` | [→](https://github.com/FlowGuard-platform/FlowGuard-Mitigating-Engine) |
| [SDN-controller](#-sdn-controller--data-plane) | Ryu OpenFlow controller | `:8080` | [→](https://github.com/FlowGuard-platform/FlowGuard-controller) |
| [FlowGuard-Dashboard](#-flowguard-dashboard--soc-interface) | React SOC dashboard | `:5173` | [→](https://github.com/FlowGuard-platform/FlowGuard-Dashboard) |

---

## Architecture

```
Network traffic (mirror interface)
         │
         ▼
┌─────────────────────┐
│   SDN-ML-Guard      │  :8000   NFStreamer capture → RF + Autoencoder fusion
│   detection engine  │          → verdict (ATTACK / SUSPECT / ANOMALY / BENIGN)
└──────────┬──────────┘
           │  POST /alert (non-benign verdicts only)
           ▼
┌─────────────────────┐
│  Mitigating Engine  │  :9000   normalize → decide → persist → dedup → act
│  decision center    │          → block / ratelimit / isolate / log_only
└──────────┬──────────┘
           │  POST /firewall/rules
           ▼
┌─────────────────────┐
│   SDN Controller    │  :8080   Ryu + OpenFlow 1.3
│   (Ryu + OVS)       │          table 0: security rules  table 1: L2 forwarding
└──────────┬──────────┘
           │  OpenFlow :6653
           ▼
     Open vSwitch (br0)
           │
    ┌──────┴──────┐
  Host A        Host B

           ▲
           │  REST (aggregated)
┌─────────────────────┐
│  FlowGuard Dashboard│  :5173 (dev) / :80 (prod)
│  React + Express    │  Express gateway (:3000) → all three services
└─────────────────────┘
```

The frontend never talks directly to internal services. All requests go through the Express backend, which is the sole JWT-protected entry point. The ML module communicates with the mitigation engine over a private interface; the dashboard only observes.

---

## Modules

### 🧠 SDN-ML-Guard — Detection engine

> [github.com/Anisayz/SDN-ML-Guard](https://github.com/FlowGuard-platform/Flowguard-ML-Engine)

Captures live traffic passively via NFStreamer, extracts 72 CICFlowMeter-compatible statistical features per flow, and runs a four-source detection fusion:

1. **Whitelist** — immediately clears known service protocols (DHCP, mDNS, SSDP, broadcasts)
2. **SYN flood heuristic** — catches unidirectional SYN bursts before ML inference
3. **Random Forest** — classifies flows across 14 attack families with a confidence score
4. **Autoencoder** — flags anomalous reconstruction error for flows the RF misses

**Fusion logic:**
```
SYN flood detected       → ATTACK  (heuristic)
RF attack, conf ≥ 0.70   → ATTACK  (RF)
RF attack + AE flagged   → ATTACK  (RF + AE)
RF attack, conf < 0.70   → SUSPECT (RF)
AE flagged, RF conf ≥ 0.50 → ANOMALY (AE)
otherwise                → BENIGN
```

**Performance on 749 647 test flows (CIC-IDS2018):**

| Model | Metric | Score |
|---|---|---|
| Random Forest | Overall accuracy | 0.900 |
| Random Forest | Weighted F1 | 0.906 |
| Autoencoder | AUC-ROC | 0.854 |
| Autoencoder | AUC-PR | 0.909 |
| RF + AE fusion | Recovery of RF misses | +26.8% |

**Stack:** FastAPI · scikit-learn (RF, 200 trees) · PyTorch (autoencoder 72→32→16→32→72) · nfstream · SMOTE · httpx

---

### 🛡️ FlowGuard-Mitigating-Engine — Decision center

> [github.com/Anisayz/FlowGuard-Mitigating-Engine](https://github.com/FlowGuard-platform/FlowGuard-Mitigating-Engine)

Sits between the ML module and the SDN controller. On receiving a verdict it runs a deterministic five-step pipeline:

1. **Normalize** — Pydantic validation, HTTP 422 on bad input
2. **Decide** — map attack label + confidence to an action
3. **Persist** — write to PostgreSQL *before* contacting Ryu (no lost alerts if the controller is down)
4. **Deduplicate** — skip rule installation if the same source IP was already handled within the cache window
5. **Act** — call `POST /firewall/rules` on Ryu and link the returned `rule_id` to the alert

**Decision table (excerpt):**

| Attack category | Action |
|---|---|
| DDoS volumetric (HOIC, LOIC), DoS Hulk, SQL Injection | `block` |
| Slow DoS (Slowloris, GoldenEye), Web brute force | `ratelimit` |
| Bot, Infiltration | `isolate` |
| SSH/FTP brute force (low confidence) | `ratelimit` |
| Unknown label | `log_only` |

**Stack:** FastAPI · SQLAlchemy async · PostgreSQL · asyncpg · httpx

---

### 🔀 SDN Controller — Data plane

> [github.com/Anisayz/SDN-controller](https://github.com/FlowGuard-platform/FlowGuard-controller)

A Ryu application that programs Open vSwitch via OpenFlow 1.3. Three cooperating apps share a single `state_store`:

- **FirewallApp** — installs `block`, `ratelimit`, and `isolate` rules in table 0 at priority 200/150
- **L2Switch** — learns MAC→port mappings and installs unicast forwarding in table 1
- **Topology** — discovers switches and links via LLDP; serves a live graph to the dashboard

**Two-table pipeline:**
```
Packet in → Table 0 (Firewall)
              ├── priority 200: block     → DROP
              ├── priority 150: ratelimit → meter → goto table 1
              └── priority   1: default  → goto table 1
                                                │
                                           Table 1 (L2 Switch)
                                                │
                                           unicast / flood
```

Security rules in table 0 are independent of forwarding in table 1 — adding or removing a firewall rule never disrupts legitimate traffic flows.

**Stack:** Ryu SDN Framework · Open vSwitch · Python 3.11

---

### 📊 FlowGuard Dashboard — SOC interface

> [github.com/Anisayz/FlowGuard-Dashboard](https://github.com/FlowGuard-platform/FlowGuard-Dashboard)

A React 19 + TypeScript single-page application behind an Express 5 gateway. The gateway aggregates all three internal services and is the only externally reachable endpoint, protected by JWT (HMAC-SHA256).

**Pages:**

| Route | Description |
|---|---|
| `/` | Summary indicators, alert timeline, attack distribution chart |
| `/firewall-rules` | Active rules, add/remove manually |
| `/alerts` | Paginated, filterable ML alert log |
| `/infrastructure` | Live SVG topology (switches + hosts) |
| `/ryu-health` | Controller uptime, protocol, connected switches |
| `/mitigation-engine` | DB status, dedup cache, recent signals |

**Stack:** React 19 · TypeScript · Vite · Recharts · Express 5 · JWT · Docker (multi-stage → Nginx)

---

## Getting started

Each module has its own installation guide. To run the full platform locally, start services in this order:

```bash
# 1. SDN controller (requires Open vSwitch on the host)
cd SDN-controller && ryu-manager main.py app/l2_switch.py app/topology.py app/firewall.py api/ofctl_rest.py --observe-links

# 2. Mitigation engine (requires PostgreSQL)
cd FlowGuard-Mitigating-Engine && python3 -m src.capture --interface mirror0

# 3. ML detection engine
cd SDN-ML-Guard && python3 -m src.capture --interface mirror0

# 4. Dashboard backend
cd FlowGuard-Dashboard/backend && npm start

# 5. Dashboard frontend (dev)
cd FlowGuard-Dashboard/frontend && npm run dev
```

Each module reads from a `.env` file — copy `.env.example` and adjust the URLs before starting.

> ⚠️ All default API keys and JWT secrets must be replaced before any non-lab deployment.

---

## CI / security posture

| Check | Tool | Modules |
|---|---|---|
| Linting | ESLint / flake8 | All |
| Type checking | tsc --noEmit | Dashboard |
| Unit tests | pytest | ML Guard · Mitigating Engine · SDN Controller |
| Static analysis (SAST) | GitHub CodeQL | SDN Controller |
| Dependency audit (SCA) | Dependabot | SDN Controller |
| AI code review | CodeRabbit | SDN Controller |

Known gaps: DAST is not yet integrated in any module. Mutual TLS between services is not implemented — inter-service authentication relies on shared API keys.

---

## Dataset

Models are trained on **CIC-IDS2018** (Canadian Institute for Cybersecurity). Class imbalance is handled with SMOTE on the training split only. The 72 features retained after cleaning are listed in `data/feature_list.json` in the ML Guard repository.

> **Distribution shift note:** Models were trained on 2018 data. Traffic generated in a 2026 Docker environment may carry slightly different statistical signatures. The multi-source fusion (heuristic + autoencoder) was designed precisely to handle this case.

---

<div align="center">

Built as a research platform for SDN-based network security · Contributions and feedback welcome

</div>
