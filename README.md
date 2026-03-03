# telco-projects-
Few Telco Cloud Native Projects
# Telco Cloud Architect — Real-World Deployment Coaching Roadmap

**Profile:** AWS SA Associate/Pro | Senior Telecom Architect (Ericsson/Nokia, Verizon VBG/VCG)  
**Target Roles:** Solutions Architect, Cloud Network Architect (Telecom/Enterprise)  
**Differentiator:** Hands-on cloud-native telecom deployments — not just theory, actually built and operated

---

## Why This Roadmap Is Different

Generic cloud architect projects (3-tier web apps, serverless APIs) don't differentiate you. Every candidate has those.

What NO generic candidate can demonstrate:
- Deploying 5G Core Network Functions as CNFs on Kubernetes
- Connecting simulated gNBs to a cloud-hosted core
- Building the NMS/OSS observability layer around network functions
- Explaining VNF → CNF migration with real deployment artifacts
- Network slicing implementation using cloud-native constructs

**You already understand these systems from the Ericsson/Nokia side.** Now you prove you can architect them on cloud infrastructure — which is exactly what operators like Verizon, AT&T, and Dish are doing.

---

## Open-Source Telecom Stack We'll Use

| Component | Tool | What It Simulates |
|-----------|------|-------------------|
| 5G Core (AMF, SMF, UPF, NRF, UDM, AUSF, PCF) | **Open5GS** | Ericsson Packet Core, Nokia CBIS, Athonet Core |
| 5G RAN (gNB + UE simulator) | **UERANSIM** | Ericsson RAN / Nokia AirScale gNB |
| Container Orchestration | **Amazon EKS** | Telco cloud platform (equivalent to Ericsson CCD, Nokia NFVI) |
| VNF Hosting | **EC2 instances** | Traditional VM-based NF deployment (legacy mode) |
| Service Mesh | **Istio** (optional) | SBI (Service-Based Interface) traffic management |
| Monitoring / NMS | **Prometheus + Grafana** | Ericsson ENM, Nokia NetAct equivalent |
| Log Management / OSS | **EFK Stack (Elasticsearch, Fluentd, Kibana)** | Fault Management, Event Correlation |
| Infrastructure as Code | **Terraform + Helm Charts** | Automated NF lifecycle management |
| CI/CD for NFs | **GitHub Actions + ArgoCD** | CNF upgrade/rollback orchestration |

All open-source. All deployable on your AWS account. Total cost: under $80 across all projects if managed carefully.

---

## PROJECT 1: 5G Core CNF Deployment on EKS

**Estimated Time:** 12–15 hours  
**Estimated AWS Cost:** $15–25 (use t3.medium nodes, tear down daily)  
**Interview Value:** Demonstrates you can deploy and operate cloud-native network functions, not just talk about them

### The Scenario
A telecom operator wants to deploy their 5G Standalone Core as Cloud-Native Functions on AWS. They need the core network functions (AMF, SMF, UPF, NRF, UDM, AUSF, PCF, UDR) running as microservices on Kubernetes with proper network segmentation, service discovery, and observability.

### What You Deploy

```
AWS Region: us-east-1

EKS Cluster: "telco-5g-core"
├── Node Group: control-plane-nfs (t3.medium × 3)
│   ├── Namespace: open5gs-cp
│   │   ├── Pod: NRF (Network Repository Function)
│   │   │   └── Service discovery for all NFs — the "DNS of 5G Core"
│   │   ├── Pod: AMF (Access & Mobility Management Function)
│   │   │   └── N2 interface toward RAN (SCTP)
│   │   ├── Pod: SMF (Session Management Function)
│   │   │   └── N4 interface toward UPF (PFCP)
│   │   ├── Pod: UDM (Unified Data Management)
│   │   ├── Pod: AUSF (Authentication Server Function)
│   │   ├── Pod: PCF (Policy Control Function)
│   │   ├── Pod: UDR (Unified Data Repository)
│   │   │   └── Backed by MongoDB (StatefulSet or Amazon DocumentDB)
│   │   └── Pod: NSSF (Network Slice Selection Function)
│   └── Namespace: open5gs-db
│       └── StatefulSet: MongoDB (persistent storage via EBS gp3)
│
├── Node Group: user-plane-nfs (t3.medium × 2)
│   ├── Namespace: open5gs-up
│   │   └── Pod: UPF (User Plane Function)
│   │       ├── N3 interface (GTP-U from RAN)
│   │       ├── N4 interface (PFCP from SMF)
│   │       ├── N6 interface (toward internet/data network)
│   │       └── Requires: NET_ADMIN capability, TUN/TAP device access
│   └── Note: UPF needs special networking — this is where it gets real
│
└── Node Group: monitoring (t3.small × 1)
    └── Namespace: observability
        ├── Prometheus (scraping all NF metrics)
        ├── Grafana (dashboards per NF)
        └── Fluentd → CloudWatch Logs

VPC Architecture (Purpose-built for telco):
├── VPC: 10.0.0.0/16
│   ├── Subnet: EKS Control Plane (10.0.1.0/24, 10.0.2.0/24) — 2 AZs
│   ├── Subnet: CP Network Functions (10.0.10.0/24, 10.0.11.0/24)
│   ├── Subnet: UP Network Functions (10.0.20.0/24, 10.0.21.0/24)
│   │   └── Why separate? UPF handles subscriber data plane traffic — must be isolated
│   ├── Subnet: Database/Storage (10.0.30.0/24, 10.0.31.0/24)
│   ├── Subnet: Management/OAM (10.0.40.0/24)
│   └── Subnet: Public (10.0.100.0/24) — NAT GW, bastion
├── Security Groups:
│   ├── SG-SBI: TCP 80/443 (HTTP/2 for Service-Based Interface between CP NFs)
│   ├── SG-N2: SCTP 38412 (AMF ↔ gNB, the NGAP interface)
│   ├── SG-N3: UDP 2152 (GTP-U: UPF ↔ gNB, user plane tunnel)
│   ├── SG-N4: UDP 8805 (PFCP: SMF ↔ UPF, session control)
│   ├── SG-DB: TCP 27017 (MongoDB access, only from CP NFs)
│   └── SG-OAM: TCP 9090/3000 (Prometheus/Grafana, management only)
└── EBS Volumes: gp3 for MongoDB persistent storage
```

### Kubernetes Resource Design

```yaml
# Example: AMF Deployment (you'll create similar for each NF)
# This is what you'll walk through in interviews

apiVersion: apps/v1
kind: Deployment
metadata:
  name: open5gs-amf
  namespace: open5gs-cp
  labels:
    app: open5gs
    nf: amf
spec:
  replicas: 2                    # HA — interview talking point
  selector:
    matchLabels:
      nf: amf
  template:
    spec:
      nodeSelector:
        node-role: control-plane  # Pin CP NFs to CP nodes
      containers:
      - name: amf
        image: openverso/open5gs:latest
        command: ["open5gs-amfd"]
        ports:
        - containerPort: 38412    # N2 (NGAP/SCTP toward RAN)
          protocol: SCTP
        - containerPort: 7777     # SBI (HTTP/2 toward other NFs)
          protocol: TCP
        - containerPort: 9090     # Metrics endpoint for Prometheus
          protocol: TCP
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        livenessProbe:            # Critical for CNF — auto-restart unhealthy NFs
          httpGet:
            path: /nnrf-nfm/v1/nf-instances
            port: 7777
          initialDelaySeconds: 15
          periodSeconds: 10
        readinessProbe:           # Don't send traffic until NF is registered with NRF
          httpGet:
            path: /nnrf-nfm/v1/nf-instances
            port: 7777
          initialDelaySeconds: 5
          periodSeconds: 5
        volumeMounts:
        - name: amf-config
          mountPath: /etc/open5gs/amf.yaml
          subPath: amf.yaml
      volumes:
      - name: amf-config
        configMap:
          name: amf-config
---
apiVersion: v1
kind: Service
metadata:
  name: amf-ngap
  namespace: open5gs-cp
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"  # NLB for SCTP
spec:
  type: LoadBalancer
  selector:
    nf: amf
  ports:
  - port: 38412
    targetPort: 38412
    protocol: SCTP       # SCTP support is critical — this is a real deployment challenge
```

### Key Decisions to Document (Interview Gold)

| Decision | What You Chose | Why | Ericsson/Nokia Parallel |
|----------|---------------|-----|------------------------|
| Separate node groups for CP vs UP | Dedicated node groups with taints/tolerations | UPF needs NET_ADMIN caps, different scaling profile, data plane isolation | Like separating CP and UP blades in Ericsson Cloud Core |
| NLB for N2 interface (not ALB) | Network Load Balancer | SCTP protocol support required for NGAP; ALB only does HTTP/HTTPS | Similar to how SCTP load balancing works in traditional signaling |
| MongoDB on K8s vs DocumentDB | Start with K8s StatefulSet, document migration path to DocumentDB | StatefulSet gives you hands-on storage management experience; DocumentDB is the production recommendation | UDR backend — equivalent to HSS/HLR database decisions |
| Multus CNI vs single CNI | Document Multus need but deploy with VPC CNI + service routing | Full Multus requires bare-metal/custom AMIs; use service-level separation as practical alternative | In production, Multus provides dedicated N2/N3/N4/N6 interfaces per NF |
| ConfigMaps for NF configuration | Externalized configuration, not baked into images | Allows config changes without redeploying NFs; mirrors CM (Configuration Management) in telecom | Equivalent to day-1/day-2 configuration in Ericsson ENM |
| Horizontal Pod Autoscaler for AMF | HPA on AMF based on active UE connections metric | AMF load scales with connected UEs, not CPU | Maps to AMF capacity dimensioning (SAUs) you do for Verizon |

### Hands-On Challenges

1. **NF Registration Chain:** Deploy NRF first. Then AMF. Watch AMF register with NRF via SBI. Check NRF's NF instance list. This is the 5G service discovery mechanism — explain it like you built it.

2. **UPF Networking Challenge:** UPF needs to create TUN interfaces (for N3/N6). On EKS, this requires privileged containers or specific security contexts. Document the exact capabilities needed (`NET_ADMIN`, `NET_RAW`) and why this is the #1 challenge in telco CNF deployments.

3. **AMF Scaling Test:** Generate simulated UE registrations. Watch AMF pod CPU/memory. Trigger HPA scale-out. Verify new AMF pod registers with NRF. Verify gNB discovers new AMF via NRF.

4. **NF Failure & Recovery:** Kill the SMF pod. Observe: existing PDU sessions continue (UPF has the forwarding rules). New sessions fail. SMF restarts, re-registers with NRF, new sessions work. Measure recovery time.

5. **Rolling Update of AMF:** Update AMF config (add a new PLMN ID). Apply rolling update. Verify zero-downtime — existing UE connections maintained during update. This is the core value proposition of CNF over VNF.

### Case Study Narrative

```
Title: "Cloud-Native 5G Core Deployment on AWS EKS"
The Ask: Deploy a 3GPP-compliant 5G SA core as containerized network functions on AWS
Architecture: [Your diagram showing all NFs, interfaces, and AWS components]
Key Challenge: UPF data plane networking on Kubernetes (TUN devices, SCTP, GTP-U tunneling)
What Worked: NRF-based service discovery, HPA-driven AMF scaling, rolling updates
What Surprised Me: [Real issues you encountered]
Production Gap Analysis: What's missing for carrier-grade (Multus, DPDK, SR-IOV, real SCTP LB)
```

---

## PROJECT 2: RAN-to-Cloud Core Connectivity (UERANSIM + Open5GS)

**Estimated Time:** 10–12 hours  
**Estimated AWS Cost:** $10–15  
**Interview Value:** End-to-end 5G call flow — from UE attach through gNB to cloud core and out to internet

### The Scenario
A gNB (simulated with UERANSIM) located at a cell site needs to connect to the 5G Core running on AWS. This involves N2 (control plane signaling) and N3 (user plane GTP-U tunneling) connectivity, subscriber provisioning, and end-to-end data path verification.

### What You Deploy

```
Architecture: Simulated RAN → Cloud Core

"Cell Site" (EC2 instance or separate VPC simulating edge/site)
├── UERANSIM gNB
│   ├── N2 connection → AMF (SCTP, port 38412)
│   │   └── Carries: NG Setup, Initial UE Message, PDU Session Resource Setup
│   └── N3 connection → UPF (GTP-U, UDP port 2152)
│       └── Carries: Encapsulated user plane packets
├── UERANSIM UE (simulated subscriber)
│   ├── SUPI: 208930000000001 (IMSI equivalent)
│   ├── Key/OPc configured (milenage authentication)
│   └── Requested NSSAI: SST=1 (eMBB slice)
└── Connectivity:
    └── Site-to-Site VPN or VPC Peering to Core VPC
        (Simulates: dedicated transport / IPsec backhaul from cell site to core)

Cloud Core (EKS Cluster from Project 1)
├── AMF receives N2: NG Setup Request from gNB
│   └── AMF verifies PLMN, TAC, selects serving area
├── AMF processes: UE Registration
│   ├── AMF → AUSF: Authentication (5G-AKA)
│   ├── AUSF → UDM: Get authentication vectors
│   ├── UDM → UDR: Subscriber data lookup
│   └── AMF → UE: Registration Accept
├── SMF processes: PDU Session Establishment
│   ├── AMF → SMF: SM Context Create (via SBI)
│   ├── SMF → PCF: Policy lookup (QoS, charging rules)
│   ├── SMF → UPF: Session Establishment (N4/PFCP)
│   │   └── UPF installs forwarding rules (PDR, FAR, QER)
│   └── SMF → AMF → gNB: PDU Session Resource Setup
└── UPF: Data path active
    ├── N3 (from gNB): Decapsulate GTP-U
    ├── N6 (to internet): NAT + route to internet via AWS NAT Gateway
    └── Subscriber can now "browse the internet"

Subscriber Database (Pre-provisioned):
├── MongoDB (UDR backend)
│   ├── Subscriber profile: IMSI, Ki, OPc, SST, DNN
│   ├── QoS profile: 5QI=9 (default internet), AMBR: 100 Mbps DL / 50 Mbps UL
│   └── APN/DNN: "internet"
└── This is equivalent to provisioning in Ericsson HSS-FE or Nokia UDM
```

### Network Connectivity Design

```
Option A: VPC Peering (Simple, for lab)
┌─────────────────┐         VPC Peering         ┌─────────────────┐
│  RAN VPC         │◄──────────────────────────►│  Core VPC        │
│  172.16.0.0/16   │                             │  10.0.0.0/16     │
│  ├── gNB (EC2)   │    N2 (SCTP/38412)         │  ├── AMF         │
│  └── UE (EC2)    │────────────────────────────►│  ├── SMF         │
│                   │    N3 (GTP-U/UDP 2152)     │  ├── UPF         │
│                   │────────────────────────────►│  └── ...         │
└─────────────────┘                             └─────────────────┘

Option B: Site-to-Site VPN (Realistic, simulates IPsec backhaul)
┌─────────────────┐      IPsec VPN Tunnel       ┌─────────────────┐
│  "Cell Site" VPC │◄══════════════════════════►│  Core VPC        │
│  StrongSwan VPN  │    BGP for route exchange   │  AWS VPN GW      │
│  ├── gNB         │                             │  Transit GW      │
│  └── UE          │                             │  ├── AMF (EKS)   │
│                   │    Encrypted N2 + N3       │  ├── UPF (EKS)   │
└─────────────────┘    through tunnel            └─────────────────┘

Interview Talking Point: "In production, this would be AWS Wavelength
for edge UPF placement, or AWS Outposts for on-prem UPF, with
Direct Connect for backhaul. The VPN simulates the IPsec tunnel
that operators use for RAN transport security."
```

### End-to-End Call Flow You'll Demonstrate

```
Step 1: gNB → AMF: NG Setup Request
        AMF → gNB: NG Setup Response
        ✓ RAN connected to Core

Step 2: UE → gNB → AMF: Registration Request (NAS)
        AMF → AUSF: Nausf_UEAuthentication (SBI/HTTP2)
        AUSF → UDM: Nudm_UEAuthentication_Get
        UDM → UDR: Nudr_DataRepository (subscriber lookup)
        AUSF → AMF: Authentication Response (RAND, AUTN, HXRES*)
        AMF → UE: Authentication Request
        UE → AMF: Authentication Response (RES*)
        AMF → AUSF: Verification → Success
        AMF → UE: Registration Accept
        ✓ UE registered on network

Step 3: UE → AMF: PDU Session Establishment Request (DNN="internet")
        AMF → NSSF: Slice selection (SST=1)
        AMF → SMF: Nsmf_PDUSession_CreateSMContext
        SMF → PCF: Npcf_SMPolicyControl (get QoS/charging policy)
        SMF → UPF: PFCP Session Establishment (N4)
              UPF installs: PDR (match on TEID), FAR (forward to N6), QER (rate limit)
        SMF → AMF: N1N2MessageTransfer (PDU Session Resource Setup)
        AMF → gNB: PDU Session Resource Setup Request
        gNB → AMF: PDU Session Resource Setup Response
        ✓ Data session established

Step 4: UE → gNB → UPF (GTP-U tunnel over N3) → Internet (N6 via NAT GW)
        ping 8.8.8.8 from UE → SUCCESS
        ✓ End-to-end data path verified
```

### Key Decisions to Document

| Decision | Rationale | Interview Angle |
|----------|-----------|----------------|
| UERANSIM vs srsRAN | UERANSIM is lighter, purpose-built for core testing, runs in userspace | srsRAN is a real SDR-based RAN — mention it as "production next step" |
| VPN for RAN transport (not just VPC peering) | Demonstrates understanding of real-world RAN backhaul security requirements | "In production, this would be Direct Connect or Wavelength for latency-sensitive UPF" |
| UPF placement in cloud vs edge | Cloud-hosted UPF works for eMBB; edge UPF (Wavelength/Outposts) needed for URLLC | Shows you understand latency-driven NF placement decisions |
| Subscriber provisioning via MongoDB | Direct DB provisioning mirrors how real operators provision UDM/HSS for lab/testing | "In production, this flows through BSS/OSS provisioning chains" |
| N6 egress via NAT Gateway | UPF needs internet breakout; NAT GW provides managed SNAT | Equivalent to Gi-LAN/N6 breakout design in production cores |

### Hands-On Challenges

1. **Full UE Attach + Data Session:** Execute the complete flow. Capture logs from every NF. Build a sequence diagram from actual log timestamps. This becomes your interview whiteboard material.

2. **PLMN/TAC Mismatch Test:** Misconfigure the gNB PLMN ID. Watch the NG Setup fail. Read the AMF rejection cause. Fix it. This teaches you exactly what breaks during real deployments.

3. **Subscriber Authentication Failure:** Provision wrong Ki/OPc in UDR. Watch authentication fail at AUSF. Read the error. Fix it. Now you can explain 5G-AKA authentication from experience, not textbooks.

4. **GTP-U Tunnel Verification:** Use tcpdump on the UPF pod to capture N3 packets. See the GTP-U encapsulation (TEID, QFI). Verify the inner IP packet is the UE's traffic. This is exactly what you'd do troubleshooting real subscriber issues.

5. **Latency Measurement:** Measure round-trip latency: UE → gNB → UPF → Internet. Then move UPF to a "closer" subnet. Compare. This demonstrates why UPF placement matters for URLLC use cases.

6. **UPF Failover:** Kill the UPF pod. Observe: PDU session drops. SMF detects via PFCP heartbeat failure. New UPF pod starts, but session is lost (no session continuity without redundancy protocol). Document this — it's a real production gap.

---

## PROJECT 3: VNF to CNF Migration — The Transformation Story

**Estimated Time:** 8–10 hours  
**Estimated AWS Cost:** $10–15  
**Interview Value:** Shows you understand BOTH legacy and modern deployment models, and can plan the migration

### The Scenario
An operator is running their EPC/5G core as VNFs on VMs (the legacy Ericsson/Nokia deployment model). They want to migrate to cloud-native CNFs on Kubernetes. You architect and execute this migration, documenting the challenges, trade-offs, and operational differences.

### Phase 1: VNF Deployment (The "Before")

```
EC2-Based VNF Deployment (simulates traditional NFVI):

VPC: 10.0.0.0/16
├── EC2: VNF-AMF (t3.medium)
│   ├── Open5GS AMF running directly on Ubuntu (systemd service)
│   ├── Dedicated ENI for N2 interface (10.0.10.x)
│   ├── Dedicated ENI for SBI interface (10.0.20.x)
│   └── CloudWatch Agent for monitoring
├── EC2: VNF-SMF (t3.medium)
│   ├── Open5GS SMF as systemd service
│   ├── Dedicated ENI for SBI (10.0.20.x)
│   └── Dedicated ENI for N4 (10.0.30.x)
├── EC2: VNF-UPF (t3.medium)
│   ├── Open5GS UPF as systemd service
│   ├── Dedicated ENI for N3 (10.0.10.x)
│   ├── Dedicated ENI for N4 (10.0.30.x)
│   └── Dedicated ENI for N6 (10.0.40.x)
├── EC2: VNF-NRF/UDM/AUSF/PCF (t3.medium — co-located)
│   └── Multiple NFs on single VM (common in legacy deployments)
├── EC2: MongoDB (t3.medium, EBS gp3)
└── Manual scaling: Must launch new EC2, configure, register

Characteristics of VNF Model (document these):
- Fixed resource allocation (VM sized for peak, wasted at low load)
- Manual scaling (launch VM, install, configure, test, add to LB)
- Update = maintenance window (stop NF, update, restart, verify)
- Network interfaces: dedicated ENIs per NF interface (good for isolation)
- Monitoring: per-VM metrics (CloudWatch), NF-level metrics manual
- Recovery: if EC2 dies, manual or ASG-based replacement (minutes)
```

### Phase 2: CNF Deployment (The "After" — from Project 1)

```
Refer to Project 1 EKS deployment, but document the specific
differences you observe:

Characteristics of CNF Model:
- Dynamic resource allocation (K8s requests/limits, bin-packing)
- Auto-scaling (HPA based on NF-specific metrics)
- Update = rolling update (zero downtime, canary possible)
- Network interfaces: shared CNI (challenge for multi-interface NFs like UPF)
- Monitoring: built-in Prometheus scraping, pod-level metrics
- Recovery: if pod dies, K8s restarts in seconds (not minutes)
```

### Phase 3: Migration Execution & Comparison

```
Migration Steps (document each):
1. Deploy CNF core alongside VNF core (parallel operation)
2. Provision test subscribers on CNF core
3. Point test gNB to CNF AMF
4. Verify registration + data session on CNF core
5. Compare: performance, resource usage, recovery time, update speed
6. Document migration runbook for remaining subscribers
7. Decommission VNF core

Comparison Matrix (your case study centerpiece):
┌─────────────────────┬──────────────────────┬──────────────────────┐
│ Dimension            │ VNF (EC2)            │ CNF (EKS)            │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ Deployment Time      │ ~30 min (manual)     │ ~5 min (helm install)│
│ Scaling Time         │ ~10 min (new EC2)    │ ~30 sec (new pod)    │
│ Update Strategy      │ Maintenance window   │ Rolling / Canary     │
│ Resource Efficiency  │ ~40% utilization     │ ~70% utilization     │
│ Recovery Time (NF)   │ 2–5 min (EC2 repl.)  │ 10–30 sec (pod)     │
│ Network Isolation    │ Dedicated ENIs       │ Network policies     │
│ Multi-interface      │ Native (multi-ENI)   │ Needs Multus CNI     │
│ Cost (monthly est.)  │ $XX (fixed VMs)      │ $YY (shared nodes)   │
│ Operational Overhead │ Patching VMs, OS mgt │ K8s cluster ops      │
│ Skill Requirement    │ Linux/VM admin       │ K8s/container skills │
└─────────────────────┴──────────────────────┴──────────────────────┘
Fill in actual measurements from YOUR deployment.
```

### Key Decisions to Document

| Decision | Rationale | Maps To Real World |
|----------|-----------|-------------------|
| Parallel operation during migration | Zero-downtime migration; ability to rollback to VNF if CNF has issues | Exactly how operators migrate from Ericsson vEPC to Cloud Native Packet Core |
| UPF migration is last | UPF has the most networking challenges on K8s (GTP-U, TUN, performance) | UPF is always the hardest NF to containerize — operators often keep it as VNF initially |
| Helm charts for CNF deployment | Reproducible, version-controlled, parameterized NF deployment | Equivalent to Ericsson VNFM / Nokia CBAM functionality |
| Resource requests/limits per NF | Prevent noisy neighbor, ensure guaranteed resources for critical NFs (AMF) | Maps to VNF flavor sizing in NFVI deployment |

### Hands-On Challenges

1. **Side-by-side benchmark:** Run the same UE attach flow on VNF and CNF core. Measure and compare latency, CPU usage, memory usage.
2. **Failure recovery race:** Kill AMF on both platforms simultaneously. Measure which recovers first and by how much. This number is your interview weapon.
3. **Rolling update vs VNF update:** Update AMF config on both platforms. Measure downtime for each approach. On VNF (restart required) vs CNF (rolling update, zero downtime).
4. **Resource efficiency:** Compare total EC2 cost (VNF: dedicated VMs) vs EKS cost (CNF: shared nodes). Calculate the efficiency gain.

---

## PROJECT 4: NMS & OSS Platform — Observability for Telco Cloud

**Estimated Time:** 10–12 hours  
**Estimated AWS Cost:** $10–15  
**Interview Value:** Shows you think beyond deployment — you build the operational layer that keeps it running

### The Scenario
The 5G Core from Projects 1-2 is deployed. Now build the NMS/OSS layer that provides: real-time NF health monitoring, fault detection and alerting, performance KPIs (KPIs that operators actually care about), subscriber-level troubleshooting capability, and capacity planning dashboards.

### What You Deploy

```
NMS/OSS Architecture on AWS:

┌─────────────────────────────────────────────────────────────────┐
│  Grafana Dashboards (NMS Console)                                │
│  ├── Dashboard: Network Overview                                 │
│  │   ├── Panel: NF Health Matrix (all NFs, green/red/amber)    │
│  │   ├── Panel: Active UE Count (from AMF metrics)              │
│  │   ├── Panel: Active PDU Sessions (from SMF metrics)          │
│  │   └── Panel: Throughput (from UPF metrics: N3 in/out)        │
│  ├── Dashboard: Per-NF Deep Dive                                │
│  │   ├── AMF: Registration success/fail rate, SCTP associations │
│  │   ├── SMF: Session setup success/fail, N4 PFCP sessions     │
│  │   ├── UPF: Packet rate, byte rate, dropped packets, GTP-U   │
│  │   └── NRF: Registered NF count, discovery request rate       │
│  ├── Dashboard: Subscriber Troubleshooting                      │
│  │   ├── UE registration timeline (log correlation)             │
│  │   ├── PDU session events                                     │
│  │   └── Error code analysis                                    │
│  └── Dashboard: Capacity Planning                               │
│      ├── CPU/Memory trends per NF (predict scaling needs)       │
│      ├── UE growth trend                                        │
│      └── Throughput growth trend                                │
└─────────────────────────────────────────────────────────────────┘
        ▲                    ▲                    ▲
        │                    │                    │
┌───────┴───────┐   ┌───────┴───────┐   ┌───────┴───────┐
│  Prometheus    │   │  EFK Stack     │   │  CloudWatch    │
│  (Metrics)     │   │  (Logs/Events) │   │  (AWS Infra)   │
│                │   │                │   │                │
│ Scrapes:       │   │ Collects:      │   │ Monitors:      │
│ - NF /metrics  │   │ - NF stdout    │   │ - EKS health   │
│ - Node metrics │   │ - K8s events   │   │ - EC2 metrics  │
│ - K8s metrics  │   │ - Audit logs   │   │ - VPC Flow Logs│
│                │   │                │   │ - RDS metrics  │
│ AlertManager:  │   │ Kibana:        │   │                │
│ - NF down      │   │ - Log search   │   │ Alarms:        │
│ - High latency │   │ - Correlation  │   │ - Node pressure│
│ - Auth failures│   │ - Dashboards   │   │ - EBS issues   │
└────────────────┘   └────────────────┘   └────────────────┘

Alert Rules (Prometheus AlertManager):
├── CRITICAL: NF pod down for > 30 seconds
├── CRITICAL: AMF registration success rate < 95%
├── WARNING: UPF packet drop rate > 1%
├── WARNING: SMF session setup latency > 200ms
├── WARNING: MongoDB replication lag > 5 seconds
├── INFO: UE count approaching HPA threshold
└── Notification: SNS → Email/Slack/PagerDuty
```

### Telco-Specific KPIs to Implement

```
These are the KPIs operators ACTUALLY monitor (not generic CPU/memory):

AMF KPIs:
- Registration Success Rate = (successful_registrations / total_attempts) × 100
- Service Request Success Rate
- Paging Success Rate
- Active UE connections (gauge)
- N2 SCTP association count

SMF KPIs:
- PDU Session Establishment Success Rate
- Session Setup Latency (p50, p95, p99)
- Active PDU Sessions (gauge)
- N4 PFCP session count

UPF KPIs:
- Throughput: bytes/sec on N3 (uplink/downlink)
- Packet rate: packets/sec
- Drop rate: dropped_packets / total_packets
- GTP-U tunnel count
- Latency: N3-to-N6 forwarding delay

NRF KPIs:
- Registered NF instances (by NF type)
- NF discovery request rate
- NF status notifications sent

System KPIs:
- Pod restart count (by NF — indicates instability)
- Container resource utilization vs limits
- Persistent volume usage (MongoDB)
```

### Hands-On Challenges

1. **Build a "War Room" Dashboard:** Create a single Grafana dashboard that an NOC operator would use during a major incident. Include NF health, active subscribers, error rates, and resource utilization — all on one screen.

2. **Fault Injection → Alert Verification:** Kill SMF pod. Verify: Prometheus alert fires within 30 seconds, AlertManager sends notification, Grafana dashboard shows red. Measure time from fault to notification. This is your MTTD (Mean Time To Detect).

3. **Log Correlation Exercise:** Trigger a UE registration failure. Using Kibana, trace the failure across AMF, AUSF, UDM logs using a correlation ID (SUPI). Document the troubleshooting workflow. This is exactly what L3 support does.

4. **Capacity Planning Simulation:** Generate steadily increasing UE registrations over 2 hours. Use Grafana's prediction feature to estimate when you'll hit AMF pod resource limits. Create a capacity report.

5. **CloudWatch Integration:** Forward critical Prometheus alerts to CloudWatch. Set up CloudWatch Composite Alarms. Demonstrate cross-platform alerting (K8s metrics + AWS infrastructure health).

### Case Study Narrative

```
Title: "Telco NMS/OSS Platform on AWS — Observability for Cloud-Native 5G Core"
Context: 5G Core running as CNFs needs carrier-grade monitoring and operations support
Architecture: Prometheus (metrics) + EFK (logs) + Grafana (visualization) + CloudWatch (infra)
Key KPIs: Registration success rate, session setup latency, UPF throughput, NF availability
Alert Design: Tiered (Critical/Warning/Info) with telecom-specific thresholds
Demo: Fault injection → detection → troubleshooting → resolution in < 5 minutes
Comparison: How this maps to Ericsson ENM / Nokia NetAct functionality
```

---

## PROJECT 5: Network Slicing on AWS — Multi-Tenant Telco Cloud

**Estimated Time:** 10–12 hours  
**Estimated AWS Cost:** $10–20  
**Interview Value:** 5G network slicing is the hottest topic in telecom cloud — and almost nobody has actually implemented it

### The Scenario
An operator needs three network slices:
- **Slice 1 (eMBB):** High bandwidth consumer broadband (SST=1)
- **Slice 2 (URLLC):** Low-latency enterprise/industrial (SST=2)
- **Slice 3 (MIoT):** Massive IoT with millions of low-throughput devices (SST=3)

Each slice needs isolation at the network function level, QoS enforcement, and independent scaling.

### What You Deploy

```
Slice Architecture on EKS:

EKS Cluster
├── Namespace: slice-embb (SST=1)
│   ├── Dedicated SMF instance (smf-embb)
│   │   └── Config: default QoS (5QI=9), high AMBR (1 Gbps DL)
│   ├── Dedicated UPF instance (upf-embb)
│   │   └── Resource limits: generous CPU/memory for throughput
│   │   └── N6 breakout: standard internet via NAT GW
│   └── ResourceQuota: CPU=4, Memory=8Gi (prevents slice from hogging)
│
├── Namespace: slice-urllc (SST=2)
│   ├── Dedicated SMF instance (smf-urllc)
│   │   └── Config: low-latency QoS (5QI=1), guaranteed bit rate
│   ├── Dedicated UPF instance (upf-urllc)
│   │   └── Resource limits: prioritized CPU (node affinity to fast nodes)
│   │   └── N6 breakout: direct to enterprise network (VPC peering)
│   │   └── Note: In production, this UPF would be at edge (Wavelength)
│   └── ResourceQuota: CPU=2, Memory=4Gi
│   └── PriorityClass: high-priority (K8s scheduling priority)
│
├── Namespace: slice-miot (SST=3)
│   ├── Dedicated SMF instance (smf-miot)
│   │   └── Config: lightweight QoS (5QI=9), very low AMBR (64 Kbps)
│   ├── Dedicated UPF instance (upf-miot)
│   │   └── Resource limits: minimal (low throughput per device)
│   │   └── N6 breakout: to IoT platform (AWS IoT Core)
│   └── ResourceQuota: CPU=1, Memory=2Gi (efficiency over performance)
│
├── Namespace: shared-cp (Common Control Plane)
│   ├── AMF (shared across slices — handles NSSAI selection)
│   ├── NRF (shared — all slice NFs register here)
│   ├── NSSF (Network Slice Selection Function — routes UE to correct slice)
│   ├── UDM / AUSF / UDR (shared — subscriber data includes slice subscription)
│   └── PCF (shared but with per-slice policy rules)
│
└── Network Policies (K8s):
    ├── slice-embb pods can ONLY talk to shared-cp and own namespace
    ├── slice-urllc pods can ONLY talk to shared-cp and own namespace
    ├── slice-miot pods can ONLY talk to shared-cp and own namespace
    └── No cross-slice traffic allowed (east-west isolation)

Subscriber Provisioning:
├── UE-1 (consumer phone):
│   ├── Subscribed NSSAI: [SST=1] (eMBB only)
│   └── Expects: high bandwidth, standard latency
├── UE-2 (factory robot):
│   ├── Subscribed NSSAI: [SST=2] (URLLC only)
│   └── Expects: ultra-low latency, guaranteed bit rate
├── UE-3 (IoT sensor):
│   ├── Subscribed NSSAI: [SST=3] (MIoT only)
│   └── Expects: long battery life (eDRX), low throughput
└── UE-4 (enterprise phone):
    ├── Subscribed NSSAI: [SST=1, SST=2] (multi-slice)
    └── Expects: eMBB for browsing + URLLC for enterprise app
```

### Key Decisions to Document

| Decision | Rationale | Interview Gold |
|----------|-----------|---------------|
| Shared AMF, dedicated SMF/UPF per slice | AMF handles initial access for all UEs; SMF/UPF provide slice-specific session handling | "This is the 3GPP-recommended architecture — shared CCNF, dedicated session management" |
| K8s Namespaces for slice isolation | Logical isolation with NetworkPolicies, ResourceQuotas | "In production, you might use separate node groups or even separate clusters per slice for harder isolation" |
| K8s ResourceQuotas per slice | Prevents one slice from consuming all cluster resources | "This is the cloud equivalent of capacity reservation per slice in traditional networks" |
| PriorityClass for URLLC | K8s scheduler prefers URLLC pods during resource contention | "Mirrors the QoS priority in 5G — URLLC traffic always gets priority over eMBB" |
| NetworkPolicy for slice isolation | East-west traffic control; slice-embb cannot reach slice-urllc pods | "Defense in depth — even if a slice NF is compromised, it can't reach other slices" |

### Hands-On Challenges

1. **Slice Selection Flow:** Register UE-1 (eMBB). Watch NSSF select Slice 1. Register UE-2 (URLLC). Watch NSSF select Slice 2. Verify each UE gets routed to the correct SMF/UPF.

2. **Cross-Slice Isolation Test:** From a pod in slice-embb namespace, try to reach a pod in slice-urllc namespace. Must fail (NetworkPolicy enforcement). From shared-cp, reach both — must succeed.

3. **QoS Differentiation Demo:** Generate traffic from UE-1 (eMBB) and UE-2 (URLLC) simultaneously. Show that URLLC slice maintains low latency even when eMBB slice is fully loaded. This requires UPF-level QoS enforcement.

4. **Slice-Specific Scaling:** Generate massive IoT registrations on Slice 3. Watch HPA scale miot-smf pods. Verify that Slice 1 and Slice 2 are completely unaffected (resource isolation).

5. **Multi-Slice UE:** Register UE-4 with NSSAI=[SST=1, SST=2]. Verify it gets TWO PDU sessions — one via smf-embb/upf-embb, another via smf-urllc/upf-urllc. This is multi-slice operation in action.

---

## Interview Question Mapping — Telecom Cloud Edition

| Interview Question | Project | Your Answer Anchored In |
|-------------------|---------|------------------------|
| "How would you deploy a 5G core on AWS?" | Project 1 | Walk through EKS cluster, NF deployment, UPF networking challenges |
| "Explain the 5G call flow" | Project 2 | Draw the exact UE → gNB → AMF → AUSF → UDM → SMF → UPF flow from your logs |
| "What's the difference between VNF and CNF?" | Project 3 | Show your comparison matrix with actual measured numbers |
| "How do you monitor network functions?" | Project 4 | NF-specific KPIs, Prometheus alerting, fault injection results |
| "Explain network slicing" | Project 5 | Walk through your 3-slice deployment with isolation proof |
| "How does Kubernetes fit in telecom?" | All | EKS for CNFs, node affinity for UP/CP separation, Multus for multi-NIC |
| "What are the challenges of cloud-native telecom?" | All | UPF networking, SCTP on K8s, stateful NFs, real-time performance requirements |
| "How do you connect RAN to cloud core?" | Project 2 | N2/N3 connectivity, VPN for transport security, latency considerations |
| "Tell me about a migration you've done" | Project 3 | VNF→CNF migration with parallel operation, rollback plan, measured improvements |
| "How do you handle carrier-grade availability?" | Projects 1+4 | Pod anti-affinity, HPA, NF re-registration, measured recovery times |
| "What's different about telecom vs IT workloads on cloud?" | All | SCTP, GTP-U, real-time processing, multi-interface NFs, regulatory requirements |

---

## Execution Timeline

| Week | Project | Deliverable |
|------|---------|------------|
| 1–2 | Project 1: 5G Core CNFs on EKS | Running core with all NFs, architecture diagram |
| 2–3 | Project 2: RAN-to-Core Connectivity | End-to-end call flow, UE attach success |
| 3–4 | Project 3: VNF→CNF Migration | Side-by-side comparison with measured data |
| 4–5 | Project 4: NMS/OSS Platform | Grafana dashboards, alerting, fault injection results |
| 5–6 | Project 5: Network Slicing | 3-slice deployment with isolation proof |
| 7 | Portfolio Polish | Architecture diagrams, case study writeups, GitHub repo |

---

## Tools & Prerequisites

| Tool | Purpose | Install |
|------|---------|---------|
| AWS CLI v2 | AWS resource management | aws.amazon.com/cli |
| kubectl | Kubernetes management | kubernetes.io |
| eksctl | EKS cluster lifecycle | eksctl.io |
| Helm 3 | Kubernetes package management | helm.sh |
| Open5GS | 5G Core Network Functions | open5gs.org |
| UERANSIM | gNB + UE Simulator | github.com/aligungr/UERANSIM |
| Terraform | Infrastructure as Code | terraform.io |
| Docker | Container image management | docker.com |
| Wireshark/tcpdump | Protocol analysis (GTP-U, SCTP, PFCP) | wireshark.org |
| draw.io | Architecture diagrams | app.diagrams.net |

---

## Cost Management

- **EKS Cluster:** $0.10/hr (~$72/mo). Run only during lab hours. `eksctl delete cluster` when done.
- **EC2 Worker Nodes:** t3.medium × 3–6 nodes. Use Spot Instances for non-production.
- **NAT Gateway:** Silent killer. Delete after each session.
- **Budget Alert:** Set $100/month hard limit in AWS Budgets.
- **Tag Everything:** `Project=Telco-Cloud`, `Owner=Mandeep` for cost tracking.
- **Estimated Total (6 weeks):** $60–100 if you tear down resources after each session.

---

*This roadmap turns your Ericsson/Nokia/Verizon experience into demonstrable cloud architecture expertise. Every project produces artifacts that no generic cloud candidate can replicate — because they don't understand the telecom domain.*
