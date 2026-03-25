토폴로지 분석 및 CCIE-EI v1.1 출제포인트

---

## 1. 토폴로지 전체 구조 요약

이 토폴로지는 **Enterprise + SP + SD-WAN + SDA**가 융합된 대규모 Multi-Site, Multi-AS 환경이다.
총 **10개 이상의 사이트**가 다양한 기술로 상호 연결되어 있으며, CCIE Enterprise Infrastructure v1.1 Lab에서 요구하는 거의 모든 기술 영역을 포함한다.

---

## 2. 사이트별 상세 분석

### 2.1 Datacenter (DC)
| 항목 | 내용 |
|------|------|
| **BGP AS** | 65002 |
| **IGP** | IPv4 OSPF Area 0 |
| **서브넷** | 10.2.0.0/16 |
| **주요 장비** | xrv211, xrv212, xrv221, xrv222 (IOS-XR), vIOS 다수, DHCP Server |
| **특수 장비** | vBond, vSmart, vManage (SD-WAN 컨트롤러), cEdge (x4) |
| **Overlay** | EVPN/VXLAN Fabric 추정 (Spine-Leaf 구조) |
| **SD-WAN** | vManage, vSmart, vBond가 DC 내 위치 → SD-WAN Control Plane의 중심 |
| **DMVPN** | DMVPN Hub이 DC 상단에 위치 |

**핵심 포인트:**
- DC 내부는 **VXLAN/EVPN Fabric** (IOS-XR 기반 Spine-Leaf)
- SD-WAN 컨트롤러 3종이 모두 DC에 집중 배치
- DMVPN Hub 역할 → Branch와의 DMVPN 터널 종단
- DHCP Server가 DC 내에 존재 → 중앙 집중 DHCP
- cEdge 장비들이 SD-WAN Edge 역할 수행

---

### 2.2 HeadQuarters (HQ)
| 항목 | 내용 |
|------|------|
| **BGP AS** | 65001 |
| **IGP** | IPv4 OSPF Area 0 + **IPv6 EIGRP 1** |
| **서브넷** | 10.1.0.0/16 |
| **주요 장비** | xrv101 (IOS-XR), xrv110, 다수의 L2/L3 스위치 |
| **구조** | Full-Mesh Core (4대 스위치 크로스 연결) |

**핵심 포인트:**
- **Dual-Stack 환경**: IPv4 OSPF + IPv6 EIGRP → Route Redistribution 가능성
- IOS-XR 장비 포함 → XR 기반 설정 필수
- HQ ↔ DC 간 직접 연결 (eBGP)
- HQ ↔ Global SP#1 연결 → Multi-homed Internet 접속
- HQ Core는 Full-Mesh L3 스위칭

---

### 2.3 IaaS (Cloud/외부 서비스)
| 항목 | 내용 |
|------|------|
| **BGP AS** | 65003 |
| **서브넷** | 10.3.0.0/16 |
| **주요 장비** | DHCP Server, DHCP Client (HostS1) |

**핵심 포인트:**
- HQ 우측에 위치, HQ와 직접 연결
- DHCP Server/Client 관계 → DHCP Relay, Helper-Address 시나리오
- IaaS를 통한 클라우드 서비스 시뮬레이션

---

### 2.4 Global SP#1 (Service Provider Core)
| 항목 | 내용 |
|------|------|
| **BGP AS** | 10000 |
| **IGP** | IPv4 OSPF Area 0 |
| **서브넷** | 100.0.0.0/8 |
| **주요 장비** | R1, R2, R3, R4, R5 (ISP Core 라우터) |

**핵심 포인트:**
- **MPLS Core** 환경 → LDP, MPLS VPN (VPNv4), MP-BGP
- 모든 사이트의 WAN Transit 역할
- HQ, DC, Branch들, SDA 사이트와 연결
- eBGP Peering을 통한 Multi-AS 환경 구성
- **100.0.0.0/8** → Public IP 대역 사용

---

### 2.5 Internet SP (ISP)
| 항목 | 내용 |
|------|------|
| **BGP AS** | 15959 |
| **역할** | DC와 HQ의 Internet 접속 제공 |

**핵심 포인트:**
- DC ↔ Internet SP, HQ ↔ Internet SP 이중 연결
- eBGP Peering, Default Route 광고, NAT 시나리오

---

### 2.6 Internet SP#2
| 항목 | 내용 |
|------|------|
| **BGP AS** | 10001 |
| **역할** | SD-WAN Transport 및 Branch 연결용 보조 ISP |

**핵심 포인트:**
- SD-WAN Overlay의 Transport 역할 (빨간 점선 = SD-WAN Overlay)
- Branch #1, #2와 연결
- Dual ISP 환경 구성

---

### 2.7 Branch #1
| 항목 | 내용 |
|------|------|
| **BGP AS** | 65004 |
| **서브넷** | 10.4.0.0/16 |
| **주요 장비** | cEdge40 (SD-WAN), xrv400, HostR41, HostR42 |

**핵심 포인트:**
- SD-WAN cEdge → DC의 vSmart/vManage와 Overlay 연결
- Internet SP#2를 통한 WAN Transport
- Branch 로컬 라우팅 + SD-WAN OMP

---

### 2.8 Branch #2
| 항목 | 내용 |
|------|------|
| **BGP AS** | 65005 |
| **서브넷** | 10.5.0.0/16 |
| **주요 장비** | cEdge51, cEdge52 (Dual cEdge), xrv501, xrv502, HostR51 등 |

**핵심 포인트:**
- **Dual cEdge** → SD-WAN HA (High Availability)
- Internet SP#2 + Global SP#1 Dual Transport
- cEdge 이중화를 통한 Active/Standby 또는 Active/Active

---

### 2.9 Branch #3
| 항목 | 내용 |
|------|------|
| **BGP AS** | 65006 |
| **서브넷** | 10.6.0.0/16 |
| **주요 장비** | xrv601, xrv602, L2 스위치 다수, Host 다수 |

**핵심 포인트:**
- Full-Mesh Core 구성 (4대 스위치)
- Global SP#1과 직접 연결
- SD-WAN이 아닌 **전통적 WAN (DMVPN/MPLS VPN)** 연결 가능성
- 가장 큰 Branch → Enterprise Branch 시뮬레이션

---

### 2.10 Branch #4
| 항목 | 내용 |
|------|------|
| **BGP AS** | 65007 |
| **서브넷** | 10.7.0.0/16 |
| **주요 장비** | xrv702, xrv700, HostR71 |
| **특수** | **SDA (Software-Defined Access)** 레이블 |

**핵심 포인트:**
- **SDA Fabric** 사이트 → Cisco DNA Center/ISE 연동
- Global SP#1과 연결
- LISP + VXLAN + CTS(TrustSec) 기반 Fabric

---

## 3. 연결 유형 분석

### 3.1 파란 실선 (Physical/Underlay 연결)
- 각 사이트 내부의 물리적 연결
- IGP (OSPF, EIGRP) 기반 Underlay

### 3.2 빨간 점선 (SD-WAN Overlay / DMVPN Tunnel)
- DC의 cEdge ↔ Branch cEdge 간 SD-WAN Overlay (OMP)
- DMVPN Hub (DC) ↔ Branch 간 DMVPN Tunnel
- IPsec 기반 Secure Overlay

### 3.3 분홍 점선 (Inter-Site WAN)
- Global SP#1을 경유하는 MPLS VPN 연결
- eBGP Peering 연결

---

## 4. IP 주소 체계

| 사이트 | 서브넷 | BGP AS |
|--------|--------|--------|
| HeadQuarters | 10.1.0.0/16 | 65001 |
| Datacenter | 10.2.0.0/16 | 65002 |
| IaaS | 10.3.0.0/16 | 65003 |
| Branch #1 | 10.4.0.0/16 | 65004 |
| Branch #2 | 10.5.0.0/16 | 65005 |
| Branch #3 | 10.6.0.0/16 | 65006 |
| Branch #4 (SDA) | 10.7.0.0/16 | 65007 |
| Global SP#1 | 100.0.0.0/8 | 10000 |
| Internet SP#2 | - | 10001 |
| Internet SP | - | 15959 |

---

## 5. CCIE-EI v1.1 출제포인트 매핑

### 5.1 Network Infrastructure (인프라)

#### 5.1.1 Layer 2 Technologies
| 출제포인트 | 토폴로지 적용 위치 | 상세 |
|-----------|-------------------|------|
| **VLAN / Trunking** | HQ, Branch #3 Core 스위치 | 802.1Q, DTP, VTP |
| **STP / RSTP / MST** | HQ Full-Mesh, Branch #3 Full-Mesh | Loop Prevention, Root Bridge 선출 |
| **EtherChannel** | HQ Core, DC Fabric 간 연결 | LACP, PAgP, L3 EtherChannel |
| **VXLAN** | DC (Spine-Leaf), Branch #4 (SDA) | VXLAN Encapsulation, VNI Mapping |

#### 5.1.2 Layer 3 Technologies
| 출제포인트 | 토폴로지 적용 위치 | 상세 |
|-----------|-------------------|------|
| **OSPFv2 Single Area** | DC (Area 0), Global SP#1 (Area 0) | Cost, Timer, Network Type |
| **OSPFv2 Multi-Area** | HQ ↔ DC 간 Area 설계 가능 | ABR, LSA Type 3/4/5/7, Stub/NSSA |
| **OSPFv3 (IPv6)** | HQ (IPv6 EIGRP → OSPFv3 Redistribution 가능) | AF 지원, LSA 차이 |
| **EIGRP (IPv6)** | HQ (IPv6 EIGRP 1) | Named Mode, Stub, Summarization |
| **BGP (eBGP Multi-AS)** | 모든 사이트 간 eBGP | AS-Path, Community, Route Filtering |
| **MP-BGP (VPNv4)** | Global SP#1 MPLS Core | VRF, RD, RT, PE-CE Routing |
| **Route Redistribution** | HQ (OSPF ↔ EIGRP) | Mutual Redistribution, Loop Prevention |
| **Policy-Based Routing** | 다수 사이트 가능 | Route-Map, ACL 기반 |
| **VRF-Lite** | DC, HQ (Multi-tenant 시나리오) | Inter-VRF Routing |
| **BFD** | SP Core, HQ-DC 간 | Fast Failover Detection |

---

### 5.2 SD-WAN (Cisco SD-WAN / Viptela)

| 출제포인트 | 토폴로지 적용 위치 | 상세 |
|-----------|-------------------|------|
| **Controller 배치** | DC (vManage, vSmart, vBond) | Orchestration Plane 이해 |
| **OMP (Overlay Management Protocol)** | cEdge ↔ vSmart | OMP Route, TLOC, Service Route |
| **Transport / Tunnel** | Internet SP#2, Global SP#1 경유 | IPsec, GRE, Dual Transport |
| **cEdge 설정** | Branch #1 (cEdge40), Branch #2 (cEdge51/52) | Template, Policy |
| **SD-WAN HA** | Branch #2 (Dual cEdge) | Active/Standby cEdge |
| **Application-Aware Routing** | SD-WAN Overlay 전체 | SLA Class, App-Route Policy |
| **Centralized/Localized Policy** | vSmart → cEdge | Data Policy, Control Policy |
| **SD-WAN + MPLS 공존** | Branch #2 (Dual Transport) | Hybrid WAN |

---

### 5.3 SDA (Software-Defined Access)

| 출제포인트 | 토폴로지 적용 위치 | 상세 |
|-----------|-------------------|------|
| **Fabric 구성요소** | Branch #4 (SDA 레이블) | Control Plane Node, Border Node, Edge Node |
| **LISP** | Branch #4 | Map-Server/Resolver, EID/RLOC |
| **VXLAN (SDA)** | Branch #4 Fabric | Fabric Encapsulation |
| **CTS / TrustSec** | Branch #4 + ISE 연동 | SGT, SGACL |
| **DNA Center** | 중앙 관리 (DC 또는 HQ) | Assurance, Provisioning |
| **ISE 연동** | HQ/DC (Cisco ISE) | Authentication, Authorization |
| **Host Onboarding** | Branch #4 Host | Wired/Wireless Fabric |
| **Fabric WAN (Transit)** | Branch #4 ↔ Global SP#1 | IP Transit, SD-Access Transit |

---

### 5.4 DMVPN

| 출제포인트 | 토폴로지 적용 위치 | 상세 |
|-----------|-------------------|------|
| **DMVPN Phase 1/2/3** | DC (Hub) ↔ Branch (Spoke) | mGRE, NHRP |
| **IPsec over DMVPN** | Overlay 전체 | IKEv2, Transform Set, Profile |
| **DMVPN + EIGRP/BGP** | Hub-Spoke Routing | Split-Horizon, Next-Hop 처리 |
| **Dual DMVPN** | DC Hub Dual-Hub 가능 | Redundancy, Front-door VRF |
| **DMVPN ↔ SD-WAN Migration** | DC에서 양쪽 공존 | 병행 운영, 마이그레이션 전략 |

---

### 5.5 MPLS / SP Technologies

| 출제포인트 | 토폴로지 적용 위치 | 상세 |
|-----------|-------------------|------|
| **LDP** | Global SP#1 Core | Label Distribution |
| **MPLS VPN (L3VPN)** | Global SP#1 ↔ 각 사이트 PE-CE | VRF, MP-BGP VPNv4 |
| **PE-CE Routing** | SP ↔ HQ/DC/Branch | eBGP, OSPF, Static PE-CE |
| **Route Target / Route Distinguisher** | VPN 구성 | Import/Export RT |
| **MPLS TE** | SP Core (가능) | RSVP, Explicit Path |

---

### 5.6 Security (Infrastructure Security)

| 출제포인트 | 토폴로지 적용 위치 | 상세 |
|-----------|-------------------|------|
| **AAA / RADIUS / TACACS+** | HQ, DC (ISE 연동) | Authentication, Authorization |
| **802.1X** | HQ/Branch Access Layer | Port-Based NAC |
| **ACL** | 모든 사이트 | Standard, Extended, Named |
| **CoPP (Control Plane Policing)** | SP Core, DC Core | Rate-Limit, Policy-Map |
| **IPsec VPN** | DMVPN, SD-WAN Tunnel | IKEv2, GRE+IPsec |
| **Zone-Based Firewall** | Branch Router (가능) | Zone Pair, Inspect |
| **MACsec** | DC Fabric, HQ Core | 802.1AE |
| **uRPF** | SP Edge | Unicast RPF (Strict/Loose) |

---

### 5.7 Services (Network Services)

| 출제포인트 | 토폴로지 적용 위치 | 상세 |
|-----------|-------------------|------|
| **DHCP Server/Relay** | DC (DHCP Server), IaaS (DHCP Server/Client) | Helper-Address, Option 82 |
| **NAT/PAT** | Internet 접속 구간 (DC/HQ ↔ ISP) | Static, Dynamic, PAT |
| **NTP** | 전체 네트워크 | Stratum, Authentication |
| **SNMP** | 모든 장비 | v2c, v3 |
| **Syslog** | 모든 장비 | Severity Level, Remote Logging |
| **IP SLA** | SD-WAN SLA, WAN Backup | ICMP Echo, Jitter, Track |
| **QoS** | WAN 구간, SD-WAN Policy | Classification, Marking, Queuing |
| **Multicast** | HQ/DC (가능) | PIM SM/SSM, RP, MSDP |
| **FHRP** | HQ Core, Branch Core | HSRP, VRRP, GLBP |
| **DNS** | 전체 | Name Resolution |

---

### 5.8 Automation & Programmability

| 출제포인트 | 토폴로지 적용 위치 | 상세 |
|-----------|-------------------|------|
| **NETCONF/YANG** | IOS-XR (DC), IOS-XE (cEdge) | YANG Model, RPC |
| **RESTCONF** | IOS-XE 장비 | REST API |
| **Python Scripting** | 전체 자동화 | Paramiko, Netmiko, NAPALM |
| **DNA Center API** | SDA 사이트 | Intent API |
| **vManage API** | SD-WAN 관리 | REST API, Template Push |
| **Ansible** | 전체 장비 프로비저닝 | Playbook, Inventory |
| **EEM (Embedded Event Manager)** | 모든 IOS-XE/XR | Applet, Script |

---

## 6. DOO (Deploy, Optimize, Operate) 시나리오별 출제 예상

### 6.1 Deploy (배포)
1. **DC VXLAN/EVPN Fabric 구축** → Spine-Leaf, BGP EVPN, VNI/VLAN 매핑
2. **SD-WAN Zero-Touch Provisioning** → vBond → cEdge 자동 등록
3. **MPLS VPN PE-CE 설정** → VRF 생성, RT Import/Export, MP-BGP Peering
4. **DMVPN Hub-Spoke 구축** → Phase 2/3, IPsec Profile
5. **SDA Fabric 구축** → LISP CP, VXLAN DP, Border/Edge Node 배포
6. **Dual-Stack (IPv4+IPv6)** → HQ에서 OSPF + EIGRP 동시 운영
7. **AAA/ISE 통합** → RADIUS 서버 등록, 802.1X Policy

### 6.2 Optimize (최적화)
1. **Route Redistribution Tuning** → HQ에서 OSPF ↔ EIGRP 상호 재분배, Loop 방지
2. **BGP Policy (Community, AS-Path Filter)** → Multi-AS 환경에서 경로 제어
3. **SD-WAN Application-Aware Routing** → SLA Class 튜닝, BFD Probe
4. **QoS End-to-End** → WAN 구간 DSCP Marking, Queuing Policy
5. **OSPF Optimization** → Summarization, Stub Area, Timer 조정
6. **STP Optimization** → Root Bridge 배치, PortFast, BPDU Guard

### 6.3 Operate (운영/트러블슈팅)
1. **OSPF Neighbor 장애** → MTU Mismatch, Timer, Area ID, Authentication
2. **BGP Peering Down** → AS Mismatch, Update-Source, eBGP Multihop
3. **SD-WAN Control Connection 실패** → vBond 연결, Certificate, NAT Traversal
4. **DMVPN Tunnel 장애** → NHRP Registration, IPsec SA, Crypto Map
5. **DHCP 할당 실패** → Relay, Helper-Address, VLAN/Subnet Mismatch
6. **VPN Overlay 경로 손실** → RT Mismatch, BGP Next-Hop, MPLS Label
7. **SDA Host Onboarding 실패** → ISE 인증, VXLAN Encap, LISP Registration

---

## 7. 기술 영역 간 상호 연관성 (Cross-Domain)

이 토폴로지의 가장 큰 특징은 **기술 영역이 분리되지 않고 상호 연결**되어 있다는 점이다.

| Cross-Domain 시나리오 | 관련 기술 |
|----------------------|----------|
| DC EVPN Fabric → MPLS VPN → Branch | VXLAN + MP-BGP + MPLS L3VPN |
| SD-WAN Overlay → Underlay MPLS/Internet | OMP + BGP + IPsec + OSPF |
| SDA Fabric → WAN Transit → DC | LISP + VXLAN + BGP + MPLS |
| HQ Dual-Stack → SP IPv4 Transit | OSPFv2 + EIGRPv6 + Redistribution + BGP |
| DMVPN + SD-WAN 공존 (DC) | mGRE + IPsec + OMP + TLOC |
| ISE/DNAC → SDA + HQ AAA | RADIUS + 802.1X + TrustSec + SGT |

---

## 8. 시험 전략 요약

### 최우선 숙달 영역 (이 토폴로지 기준)
1. **BGP (eBGP Multi-AS + MP-BGP VPNv4)** → 전체 사이트 연결의 핵심
2. **OSPF (Multi-Area + IPv6)** → DC/HQ/SP 내부 IGP
3. **SD-WAN (Controller + cEdge + Policy)** → DC-Branch 연결의 주축
4. **VXLAN/EVPN** → DC Fabric + SDA Fabric
5. **Route Redistribution** → HQ Dual-Protocol 환경
6. **DMVPN Phase 2/3 + IPsec** → 전통 WAN Overlay
7. **MPLS VPN** → SP Core 통한 L3VPN
8. **Troubleshooting Methodology** → Layer별 체계적 접근

### 시간 배분 권장 (8시간 Lab 기준)
| 영역 | 비중 | 시간 |
|------|------|------|
| L2/L3 Infrastructure | 25% | 2시간 |
| BGP / MPLS / VPN | 25% | 2시간 |
| SD-WAN | 15% | 1시간 12분 |
| SDA / Automation | 15% | 1시간 12분 |
| Services (DHCP, QoS, NAT 등) | 10% | 48분 |
| Troubleshooting | 10% | 48분 |

---
 CCIE Enterprise Infrastructure v1.1 Blueprint에 매핑하였습니다. 실제 Lab 시나리오는 DOO(Deploy/Optimize/Operate) 섹션으로 구분되며, 각 섹션에서 위 기술들이 복합적으로 출제됩니다.
