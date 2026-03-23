# Cisco Catalyst SD-WAN Deep Dive

> 아키텍처부터 정책까지 완전 정복  
> 작성일: 2026-03-22 | 카테고리: Networking / SD-WAN

---

## 목차

1. [SD-WAN 등장 배경](#1-sd-wan-등장-배경)
2. [아키텍처 개요 — 4개 Plane 분리](#2-아키텍처-개요--4개-plane-분리)
3. [핵심 컴포넌트 상세](#3-핵심-컴포넌트-상세)
4. [OMP (Overlay Management Protocol)](#4-omp-overlay-management-protocol)
5. [TLOC (Transport Locator)](#5-tloc-transport-locator)
6. [제어 연결과 보안 메커니즘](#6-제어-연결과-보안-메커니즘)
7. [Policy Framework](#7-policy-framework)
8. [Cloud OnRamp](#8-cloud-onramp)
9. [Multi-Region Fabric](#9-multi-region-fabric)
10. [Security & SASE 통합](#10-security--sase-통합)
11. [Analytics & AIOps](#11-analytics--aiops)
12. [설계 Best Practice](#12-설계-best-practice)

---

## 1. SD-WAN 등장 배경

전통적인 MPLS 중심의 Hub-and-Spoke 아키텍처는 모든 트래픽을 데이터센터로 백홀(backhaul)하는 구조였다. 그러나 멀티클라우드 전환, SaaS 도입 확산, 하이브리드 근무 체제가 보편화되면서 이 모델은 한계에 도달했다.

### 전통 WAN vs SD-WAN 핵심 차이

| 구분 | 전통 WAN (MPLS) | SD-WAN |
|------|----------------|--------|
| **Transport** | MPLS 전용선 단일 의존 | MPLS + Broadband + 5G/LTE + Satellite (Transport Agnostic) |
| **설정 방식** | CLI 기반 Box-by-Box | Manager GUI 기반 중앙 정책 관리 |
| **앱 인식** | 없음 (IP/Port 기반) | DPI 기반 1,400+개 애플리케이션 인식 |
| **배포 속도** | 수 주 소요 | ZTP(Zero-Touch Provisioning) — 분 단위 배포 |
| **보안** | 별도 방화벽/VPN 장비 | NGFW, IPS, URL Filtering, AMP 내장 |

> **Point:** Cisco Catalyst SD-WAN은 Fortune 100 기업 중 70% 이상이 도입, 전 세계 48,000건 이상 배포 사례를 보유한 업계 최대 규모의 SD-WAN 솔루션이다.

---

## 2. 아키텍처 개요 — 4개 Plane 분리

Cisco Catalyst SD-WAN의 핵심 설계 원칙은 **Management, Control, Orchestration, Data Plane의 완전한 분리**이다.

```
┌─────────────────────────────────────────────────────┐
│                  Management Plane                    │
│               (SD-WAN Manager / vManage)             │
│         설정, 모니터링, Day 0/1/2 운용               │
├─────────────────────────────────────────────────────┤
│                   Control Plane                      │
│              (SD-WAN Controller / vSmart)             │
│         OMP Route Reflector, 정책 배포               │
├─────────────────────────────────────────────────────┤
│                Orchestration Plane                    │
│             (SD-WAN Validator / vBond)                │
│         인증, 자동 온보딩, NAT Traversal             │
├─────────────────────────────────────────────────────┤
│                    Data Plane                         │
│                (WAN Edge Router)                      │
│          패킷 포워딩, 암호화, QoS 적용              │
└─────────────────────────────────────────────────────┘
```

> **아키텍트 Tip:** 4개 Plane 분리 덕분에 Control Plane 장애 시에도 Data Plane은 OMP Graceful Restart Timer(기본 12시간) 동안 마지막 정책·경로·IPsec 키를 유지하며 정상 포워딩한다.

---

## 3. 핵심 컴포넌트 상세

### 3-1. SD-WAN Manager (구 vManage)

전체 SD-WAN 패브릭의 **Single Pane of Glass**. GUI 기반으로 Day 0(ZTP), Day 1(Template/Config Group), Day 2(모니터링·트러블슈팅)를 통합 제공한다.

- **배포 옵션:** On-Premise VM / Cloud-Hosted (AWS, Azure)
- **HA:** 최소 3노드 클러스터 (Raft Consensus)
- **API:** REST API 완전 지원 → 자동화·CI/CD 연동
- **Config Group (20.12+):** 기존 Feature Template을 대체하는 새로운 설정 모델

### 3-2. SD-WAN Controller (구 vSmart)

OMP 기반 **중앙 집중형 Route Reflector**. WAN Edge로부터 수신한 경로·정책·암호화 키를 연산하여 최적 경로를 재배포한다.

- **스케일:** Controller 1대당 WAN Edge 최대 5,500대
- **HA:** Active-Active 이중화 (최소 2대 권장)
- **정책 적용 지점:** Control Policy는 Controller에서 직접 적용 (Route Filtering, Route Manipulation)

### 3-3. SD-WAN Validator (구 vBond)

최초 인증·NAT Traversal·오케스트레이션을 담당하는 **첫 번째 접점(First Point of Contact)**.

- WAN Edge가 부팅되면 Validator에게 자신을 등록
- 유효한 인증서 확인 후 Manager/Controller 정보 전달
- **NAT Traversal:** STUN/TURN 유사 메커니즘으로 NAT 환경에서도 DTLS/TLS 터널 수립
- **배포:** 반드시 Public IP 필요 (모든 컴포넌트 중 유일)

### 3-4. WAN Edge Router

실제 트래픽을 처리하는 **Data Plane 장비**.

- **하드웨어:** Catalyst 8300/8200/8500, ISR 1100, C1101-4P 등
- **가상:** Catalyst 8000V (AWS/Azure/GCP Marketplace)
- **주요 기능:** IPsec 데이터 터널, AppQoE(TCP Opt, DRE, FEC, Pkt Dup), DPI, NGFW, IPS
- **Crypto 성능:** 하드웨어 가속(QFP) 기반 최대 수십 Gbps 암호화 처리

---

## 4. OMP (Overlay Management Protocol)

OMP는 Cisco Catalyst SD-WAN의 **독자적인 Control Plane 프로토콜**로, BGP와 유사한 Path-Vector 기반이지만 SD-WAN 오버레이에 최적화되어 있다.

### OMP Route 3가지 유형

| Route Type | 설명 | 비유 |
|------------|------|------|
| **vRoute** | 사이트 내부 Prefix (Connected, Static, OSPF/BGP Redistribute) | BGP NLRI |
| **TLOC Route** | WAN Edge의 Transport 연결 정보 (System-IP + Color + Encap) | BGP Next-Hop |
| **Service Route** | Firewall, IDS 등 서비스 체이닝 대상 | BGP Community + Next-Hop |

### OMP 동작 흐름

```
WAN Edge (Site A)                Controller               WAN Edge (Site B)
     │                              │                           │
     │── OMP Update (vRoute) ──────▶│                           │
     │── OMP Update (TLOC) ────────▶│                           │
     │                              │── Best Path 연산 ──▶      │
     │                              │── OMP Update ────────────▶│
     │                              │   (vRoute + TLOC)         │
     │◀─────────── IPsec Key (AES-256-GCM) ───────────────────▶│
     │                              │                           │
     │◀═══════════ IPsec Data Tunnel (Direct) ════════════════▶│
```

### OMP 핵심 특성

- **Graceful Restart:** Controller 장애 시 기본 12시간 동안 기존 경로 유지
- **Best Path Selection:** Preference → TLOC Preference → Origin → OMP Tag → Site-ID → System-IP
- **Route Limit:** WAN Edge당 기본 OMP vRoute 수신 한도 존재 (플랫폼별 상이)
- **Send-Path-Limit:** Controller가 동일 Prefix에 대해 전송하는 경로 수 (기본 4, ECMP용)

---

## 5. TLOC (Transport Locator)

TLOC은 SD-WAN 오버레이에서 WAN Edge의 물리적 위치를 식별하는 **3-Tuple 구조**이다.

### TLOC 3-Tuple

```
TLOC = (System-IP, Color, Encapsulation)
         │           │        │
         │           │        └─ IPsec or GRE
         │           └─ Transport 유형 (biz-internet, mpls, lte, etc.)
         └─ WAN Edge 고유 식별자 (Loopback 0)
```

### Color 종류 및 특성

| Color | 용도 | Restrict 기본값 |
|-------|------|-----------------|
| `default` | 범용 | No |
| `mpls` | 사설 MPLS | Yes |
| `biz-internet` | 비즈니스 인터넷 | No |
| `public-internet` | 공용 인터넷 | No |
| `lte` | LTE/5G 셀룰러 | No |
| `metro-ethernet` | 메트로 이더넷 | Yes |
| `private1~6` | 사설 커스텀 | Yes |

> **Restrict 속성:** `restrict`가 설정된 Color끼리만 터널 형성 가능. 예: `mpls ↔ mpls`는 되지만, `mpls ↔ biz-internet`은 불가 (Policy Override 가능).

### TLOC Extension

WAN Edge에 직접 WAN 회선이 연결되지 않는 경우 (예: L2 스위치 뒤에 위치), 인접 WAN Edge의 TLOC을 확장하여 사용하는 메커니즘이다.

---

## 6. 제어 연결과 보안 메커니즘

### 제어 연결 (Control Connection)

모든 컨트롤 플레인 통신은 **DTLS(UDP 12346) 또는 TLS(TCP 12346)** 터널을 통해 암호화된다.

```
WAN Edge ──DTLS/TLS──▶ Validator ──DTLS/TLS──▶ Controller
                                   ──DTLS/TLS──▶ Manager
```

### IKE-less 대칭 키 배포

Cisco SD-WAN은 전통적인 IKE 협상 없이 **Controller를 통한 대칭 키 배포** 방식을 사용한다.

1. WAN Edge A가 자신의 TLOC + 공개키를 OMP로 Controller에 전송
2. Controller가 WAN Edge B의 TLOC + 공개키를 WAN Edge A에 전달
3. 양측이 공유된 키 정보로 **AES-256-GCM** IPsec 터널을 즉시 수립
4. 키 로테이션은 Controller가 중앙에서 관리

> **장점:** IKE 협상 지연 제거, 대규모 Full-Mesh에서도 빠른 터널 수립, 중앙 집중형 키 관리로 보안 강화.

### 인증 체계

- **자동 인증서:** Symantec/DigiCert 루트 CA 기반 (하드웨어 Edge)
- **Enterprise CA:** 자체 PKI 연동 가능
- **허용 목록:** Manager에서 디바이스 시리얼/Chassis-ID 사전 등록 필수

---

## 7. Policy Framework

Cisco Catalyst SD-WAN의 정책은 4가지 유형으로 분류된다.

### 정책 유형 비교

| 정책 유형 | 적용 위치 | 동작 | 주요 용도 |
|-----------|----------|------|----------|
| **Control Policy** | Controller (Centralized) | OMP Route 조작 | Route Filtering, Topology 제어, Traffic Engineering |
| **Data Policy** | WAN Edge (Centralized Push) | 패킷 단위 처리 | App-Aware Routing, DSCP Marking, Service Chaining |
| **AAR (App-Aware Routing)** | WAN Edge (Centralized Push) | SLA 기반 동적 경로 | Loss/Latency/Jitter 임계치 기반 경로 전환 |
| **Localized Policy** | WAN Edge (Local) | 장비 레벨 기능 | ACL, QoS, Route-Map, SNMP, Logging |

### Control Policy 예시 — Hub-and-Spoke 강제

```
policy
  control-policy HUB-SPOKE
    sequence 10
      match route
        site-list BRANCH-SITES
      action accept
        set
          tloc-list HUB-TLOC
          preference 100
```

> Hub 사이트의 TLOC을 Branch 경로의 Next-Hop으로 강제 → 모든 Branch 간 통신이 Hub을 경유

### AAR Policy 예시 — 음성 트래픽 SLA 보장

```
policy
  app-aware-routing AAR-VOICE
    sequence 10
      match
        app-list VOICE-APPS
      action
        sla-class VOICE-SLA strict
          preferred-color mpls biz-internet
    default-action sla-class BEST-EFFORT
!
sla-class VOICE-SLA
  latency 150
  loss 1
  jitter 30
```

### Policy 불일치 트러블슈팅

Policy 불일치는 Overlay 라우팅 불안정의 주요 원인이다.

- `show sdwan policy from-vsmart` — Controller로부터 수신한 정책 확인
- `show sdwan omp routes` — OMP 라우팅 테이블 검증
- `show sdwan app-route statistics` — 애플리케이션별 경로 품질 확인
- `show sdwan policy service-path` — 실제 서비스 경로 확인

---

## 8. Cloud OnRamp

### Cloud OnRamp for SaaS

- **1,400+개 SaaS 앱** 자동 인식 (Microsoft 365, Salesforce, Webex 등)
- **vQoE (Quality of Experience) Score:** 0~10점으로 각 경로의 SaaS 품질을 실시간 측정
- **동작:** 주기적으로 각 Transport를 통해 SaaS Probe를 전송 → 최적 경로로 자동 전환
- **Direct Internet Access (DIA):** Branch에서 직접 SaaS로 Local Breakout

### Cloud OnRamp for IaaS

- **AWS Transit Gateway / Azure vWAN / GCP NCC** 자동 연동
- Manager GUI에서 클릭 몇 번으로 클라우드 WAN Edge 자동 배포
- **Multi-Cloud Mesh:** AWS ↔ Azure ↔ On-Prem 간 Full-Mesh 오버레이

### Cloud OnRamp for Colocation (Interconnect)

- Equinix, Megaport 등 Colocation 환경에서 SD-WAN Gateway 역할
- 여러 클라우드 프로바이더로의 단일 연결 허브

---

## 9. Multi-Region Fabric

대규모 글로벌 네트워크를 위한 **계층적 SD-WAN 아키텍처**.

### 구조

```
┌──────────────────────────────────────────────┐
│              Core Region                      │
│     (글로벌 Controller + Border Router)       │
│                                              │
│   Border Router ◀══▶ Border Router           │
│        │                    │                │
├────────┼────────────────────┼────────────────┤
│  Access Region 1      Access Region 2        │
│  (지역 Controller)    (지역 Controller)      │
│  [Edge] [Edge]        [Edge] [Edge]          │
└──────────────────────────────────────────────┘
```

### 핵심 개념

- **Core Region:** 글로벌 백본. Border Router가 리전 간 트래픽을 중계
- **Access Region:** 지역 사이트 그룹. 자체 Controller로 로컬 정책 처리
- **Border Router:** Core ↔ Access 간 OMP 경로 교환 및 트래픽 포워딩
- **Secondary Region:** 하나의 사이트가 여러 리전에 소속 가능 (이중화)
- **Route Aggregation:** Access Region 내부 경로를 요약하여 Core에 전달 → 스케일 확보

> **스케일:** 단일 리전 최대 ~5,500 Edge → Multi-Region Fabric으로 수만 대 Edge 지원

---

## 10. Security & SASE 통합

### 내장 보안 스택 (WAN Edge)

| 기능 | 설명 |
|------|------|
| **Enterprise Firewall (NGFW)** | Zone-Based Firewall, Stateful Inspection |
| **IPS/IDS** | Snort 엔진 기반 침입 탐지/방지 |
| **URL Filtering** | 카테고리 기반 웹 필터링 (80+ 카테고리) |
| **AMP (Advanced Malware Protection)** | 파일 해시 기반 악성코드 탐지 |
| **DNS Security** | Cisco Umbrella 연동 |
| **SSL/TLS Decryption** | 암호화 트래픽 검사 |

### SASE 통합 (Cisco Secure Access)

- **SSE (Security Service Edge):** SWG, CASB, ZTNA, DLP를 클라우드에서 제공
- **SD-WAN + SSE = SASE:** 네트워크와 보안의 완전한 통합
- **SIG (Secure Internet Gateway):** Branch → Umbrella SIG 자동 터널링
- **ZTNA:** 사용자 ID + 디바이스 포스처 기반 Zero Trust 접근 제어

---

## 11. Analytics & AIOps

- **vAnalytics:** 네트워크 전체 트래픽·성능·이상 탐지 대시보드
- **Predictive Path Recommendations:** AI/ML 기반 경로 추천
- **Anomaly Detection:** 트래픽 패턴 이상 자동 감지·알림
- **ThousandEyes 연동:** End-to-End 인터넷 경로 가시성
- **Mean Time to Repair (MTTR):** 자동 Root Cause Analysis로 장애 복구 시간 단축

---

## 12. 설계 Best Practice

### 컨트롤 컴포넌트 HA

| 컴포넌트 | 최소 권장 | HA 방식 |
|----------|----------|---------|
| Manager | 3노드 | Raft Cluster |
| Controller | 2대 | Active-Active |
| Validator | 2대 | Active-Standby |

### 설계 체크리스트

- **System-IP:** 사이트별 고유 /32, 절대 중복 불가 (Loopback 0)
- **Site-ID:** 사이트별 고유 번호, OMP Split-Brain 방지를 위해 동일 사이트 내 모든 Edge 동일 값
- **Color 체계:** Transport 유형별 Color 매핑 표준화 (mpls, biz-internet, lte 등)
- **Organization Name:** 모든 컴포넌트에서 동일해야 OMP 인증 통과
- **DNS/NTP:** Validator FQDN 해석을 위한 DNS 필수, 인증서 유효성을 위한 NTP 동기화 필수
- **MTU 설계:** IPsec 오버헤드(~58 bytes) 감안한 Path MTU 설정
- **BFD Timer:** 기본 1초, 대규모 환경에서는 튜닝 필요

### Day 2 운용 필수 명령어

```bash
# OMP 상태 확인
show sdwan omp summary
show sdwan omp peers
show sdwan omp routes

# Control Connection 확인
show sdwan control connections
show sdwan control local-properties

# Data Plane 확인
show sdwan bfd sessions
show sdwan ipsec inbound-connections
show sdwan tunnel statistics

# 정책 확인
show sdwan policy from-vsmart
show sdwan running-config
```

---

## 마무리

Cisco Catalyst SD-WAN은 단순한 WAN 오버레이가 아닌, 멀티클라우드·보안·AI 기반 운용까지 포괄하는 **엔터프라이즈급 네트워크 트랜스포메이션 플랫폼**이다.

4-Plane 분리 아키텍처, OMP 기반 지능형 Control Plane, IKE-less 암호화, Policy 프레임워크, Cloud OnRamp, Multi-Region Fabric까지 이해하면 어떤 규모의 고객 요구에도 최적의 설계를 제안할 수 있다.

### Reference

- [Cisco SD-WAN Design Guide (CVD)](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/SDWAN/cisco-sdwan-design-guide.html)
- [OMP Routing Configuration Guide](https://www.cisco.com/c/en/us/td/docs/routers/sdwan/configuration/routing/ios-xe-17/routing-book-xe/omp-routing-protocol.html)
- [Policy Configuration Guide](https://www.cisco.com/c/en/us/td/docs/routers/sdwan/configuration/policies/ios-xe-17/policies-book-xe/policy-overview.html)
- [Multi-Region Fabric Guide](https://www.cisco.com/c/en/us/td/docs/routers/sdwan/configuration/hierarchical-sdwan/hierarchical-sdwan-guide/h-sd-wan-basics.html)
