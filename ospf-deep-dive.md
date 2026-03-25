# OSPF Deep Dive — 예제 중심 완전 정복

> **대상:** 네트워크 엔지니어 / 아키텍트 / CCIE 준비자  
> **범위:** OSPF Fundamentals → Advanced Design → Real-World Troubleshooting  
> **플랫폼:** Cisco IOS-XE / IOS-XR / NX-OS 기반 예제

---

## 목차

1. [OSPF 기본 개념 리뷰](#1-ospf-기본-개념-리뷰)
2. [Neighbor & Adjacency 상세](#2-neighbor--adjacency-상세)
3. [LSA Types 완전 정복](#3-lsa-types-완전-정복)
4. [Area Design & Stub Variations](#4-area-design--stub-variations)
5. [OSPF Network Types](#5-ospf-network-types)
6. [Route Summarization & Filtering](#6-route-summarization--filtering)
7. [OSPF Authentication](#7-ospf-authentication)
8. [Virtual Link & GRE Workarounds](#8-virtual-link--gre-workarounds)
9. [OSPFv3 (IPv6)](#9-ospfv3-ipv6)
10. [OSPF + MPLS / Segment Routing 연동](#10-ospf--mpls--segment-routing-연동)
11. [OSPF Convergence & Performance Tuning](#11-ospf-convergence--performance-tuning)
12. [OSPF & BGP Redistribution](#12-ospf--bgp-redistribution)
13. [Troubleshooting Playbook](#13-troubleshooting-playbook)

---

## 1. OSPF 기본 개념 리뷰

### 1.1 OSPF의 본질

OSPF는 **Link-State Protocol**이다. 모든 라우터가 동일한 LSDB(Link-State Database)를 구축하고, SPF(Dijkstra) 알고리즘으로 최단 경로를 독립적으로 계산한다.

| 특성 | 설명 |
|------|------|
| Protocol Type | Link-State |
| Algorithm | Dijkstra SPF |
| Transport | IP Protocol 89 (TCP/UDP 아님!) |
| Admin Distance | 110 |
| Metric | Cost (Reference BW / Interface BW) |
| Hello/Dead | 10/40 (Broadcast/P2P), 30/120 (NBMA) |
| Multicast | 224.0.0.5 (AllSPFRouters), 224.0.0.6 (AllDRRouters) |
| Area 0 | Backbone — 모든 Area는 Area 0에 연결 필수 |
| Hierarchical | 2-Level (Backbone + Non-Backbone) |

### 1.2 OSPF vs IS-IS vs EIGRP

| 항목 | OSPF | IS-IS | EIGRP |
|------|------|-------|-------|
| Type | Link-State | Link-State | Advanced DV |
| Transport | IP 89 | CLNS (L2) | IP 88 |
| Hierarchy | Area 기반 | Level 기반 | 없음 |
| Convergence | Fast (SPF) | Fast (SPF) | Very Fast (DUAL) |
| Scalability | 중-대 | 대규모 (ISP) | 소-중 |
| Multi-Vendor | 표준 (RFC 2328) | 표준 (ISO 10589) | Cisco 전용 → RFC 7868 |
| IPv6 | OSPFv3 | Native | Named Mode |

### 1.3 OSPF Packet Types

```
┌──────┬────────────────┬──────────────────────────────────┐
│ Type │ Name           │ 용도                              │
├──────┼────────────────┼──────────────────────────────────┤
│  1   │ Hello          │ Neighbor 발견/유지, DR/BDR 선출    │
│  2   │ DBD (DD)       │ LSDB 요약 교환 (ExStart/Exchange) │
│  3   │ LSR            │ 부족한 LSA 요청                    │
│  4   │ LSU            │ LSA 전달 (실제 데이터)              │
│  5   │ LSAck          │ LSU 수신 확인                      │
└──────┴────────────────┴──────────────────────────────────┘
```

### 1.4 Cost 계산

```
Cost = Reference Bandwidth / Interface Bandwidth

기본 Reference BW: 100 Mbps (100,000,000 bps)

┌──────────────────┬───────────────┬────────────────────────┐
│ Interface        │ Default Cost  │ auto-cost ref 100000   │
│                  │ (ref 100M)    │ (ref 100G 설정 시)      │
├──────────────────┼───────────────┼────────────────────────┤
│ 10 Mbps          │ 10            │ 10000                  │
│ 100 Mbps         │ 1             │ 1000                   │
│ 1 Gbps           │ 1 ← 문제!    │ 100                    │
│ 10 Gbps          │ 1 ← 문제!    │ 10                     │
│ 25 Gbps          │ 1 ← 문제!    │ 4                      │
│ 40 Gbps          │ 1 ← 문제!    │ 2                      │
│ 100 Gbps         │ 1 ← 문제!    │ 1                      │
└──────────────────┴───────────────┴────────────────────────┘
```

> **필수 설정:** 현대 네트워크에서 `auto-cost reference-bandwidth`를 반드시 올려야 한다!

```
router ospf 1
 auto-cost reference-bandwidth 100000   ! 100 Gbps 기준
 ! ⚠️ 모든 OSPF 라우터에서 동일하게 설정해야 함
```

---

## 2. Neighbor & Adjacency 상세

### 2.1 OSPF FSM (Neighbor States)

```
Down → Attempt → Init → 2-Way → ExStart → Exchange → Loading → Full
                   │        │
                   │        └─ DR/BDR 선출 (Broadcast/NBMA)
                   │           DROther끼리는 2-Way에서 멈춤
                   │
                   └─ Hello 수신, 자신의 RID가 안 보임
```

**각 상태별 의미:**

| State | 설명 | 정상 여부 |
|-------|------|----------|
| Down | 초기/Hello 미수신 | 시작 상태 |
| Attempt | NBMA에서 수동 neighbor로 Hello 전송 | NBMA only |
| Init | Hello 수신했으나 양방향 미확인 | ⚠️ 여기서 멈추면 문제 |
| 2-Way | 양방향 확인, DR/BDR 선출 | DROther 간 정상 |
| ExStart | Master/Slave 결정 (높은 RID=Master) | 일시적 |
| Exchange | DBD 교환 | 일시적 |
| Loading | LSR/LSU로 부족한 LSA 수신 | 일시적 |
| Full | 완전 동기화 | ✅ 정상 |

### 2.2 Adjacency 조건 (이것 안 맞으면 Neighbor 안 됨!)

```
┌──────────────────────────────────────────────────────────────┐
│  OSPF Neighbor 성립 조건 체크리스트:                            │
│                                                              │
│  ✅ 1. Hello/Dead Timer 일치                                 │
│  ✅ 2. Area ID 일치                                          │
│  ✅ 3. Subnet/Mask 일치 (P2P 제외)                            │
│  ✅ 4. Authentication 일치 (설정된 경우)                       │
│  ✅ 5. Stub Flag 일치 (Stub/NSSA 설정)                        │
│  ✅ 6. MTU 일치 (DBD 교환 시) — ip ospf mtu-ignore로 우회     │
│  ✅ 7. Unique Router-ID                                      │
│                                                              │
│  ⚠️ 일치하지 않아도 되는 것:                                   │
│  - Process ID (local significance only)                      │
│  - Router-ID (unique이면 됨)                                  │
│  - Cost (각자 독립 계산)                                      │
└──────────────────────────────────────────────────────────────┘
```

### 2.3 기본 OSPF 설정

**토폴로지:**
```
        Area 0
R1 ─────────── R2 ─────────── R3
.1  10.1.12.0/30  .2  .2  10.1.23.0/30  .3
Lo0: 1.1.1.1      Lo0: 2.2.2.2       Lo0: 3.3.3.3
```

**R1 설정 (Interface 방식):**
```
interface Loopback0
 ip address 1.1.1.1 255.255.255.255
 ip ospf 1 area 0

interface GigabitEthernet0/0
 ip address 10.1.12.1 255.255.255.252
 ip ospf 1 area 0
 ip ospf network point-to-point    ← P2P로 변경 (DR 선출 불필요)

router ospf 1
 router-id 1.1.1.1
 auto-cost reference-bandwidth 100000
```

**R2 설정 (Network 방식):**
```
router ospf 1
 router-id 2.2.2.2
 auto-cost reference-bandwidth 100000
 network 2.2.2.2 0.0.0.0 area 0
 network 10.1.12.0 0.0.0.3 area 0
 network 10.1.23.0 0.0.0.3 area 0
```

> **Interface 방식 vs Network 방식:**  
> - Interface 방식 (`ip ospf 1 area 0`)이 명시적이고 권장  
> - Network 방식은 wildcard mask 실수 가능  
> - IOS-XR은 Interface 방식만 지원

### 2.4 DR/BDR 선출

**선출 규칙:**
```
1. 가장 높은 OSPF Priority → DR
2. Priority 동일 시 → 가장 높은 Router-ID → DR
3. 차순위 → BDR
4. Priority 0 → 선출 후보에서 제외
5. 선출 후 변경 안 됨 (Non-preemptive!) — 새 라우터가 더 높은 Priority여도
```

**Priority 설정:**
```
! R1을 반드시 DR로
interface GigabitEthernet0/0
 ip ospf priority 255    ← 최대값

! R3는 DROther로 고정
interface GigabitEthernet0/0
 ip ospf priority 0      ← 선출 불참

! 검증
show ip ospf interface GigabitEthernet0/0
  → State DR, Priority 255
show ip ospf neighbor
  → State FULL/DR, FULL/BDR, 2-WAY/DROTHER
```

> **실무 팁:**  
> - DR/BDR 선출이 필요 없는 링크는 `ip ospf network point-to-point`로 변경  
> - DR 실패 시 BDR이 즉시 승격, 새 BDR만 선출  
> - DR이 바뀌면 모든 neighbor가 새 DR과 Full 재형성

### 2.5 Router-ID 결정 순서

```
1. router-id 명시 설정 (router ospf 하위)
2. 가장 높은 Loopback IP
3. 가장 높은 Physical Interface IP

! 명시적 설정 (강력 권장)
router ospf 1
 router-id 1.1.1.1

! ⚠️ RID 변경 후 반드시 프로세스 리셋
clear ip ospf process
```

---

## 3. LSA Types 완전 정복

### 3.1 LSA Types 전체 요약

```
┌──────┬──────────────┬──────────────┬───────────────────────────────────┐
│ Type │ Name         │ 생성자        │ Flooding Scope                    │
├──────┼──────────────┼──────────────┼───────────────────────────────────┤
│  1   │ Router LSA   │ 모든 라우터   │ Area 내부                          │
│  2   │ Network LSA  │ DR           │ Area 내부                          │
│  3   │ Summary LSA  │ ABR          │ Area 간 (Inter-Area)               │
│  4   │ ASBR Summary │ ABR          │ Area 간 (ASBR 위치 정보)            │
│  5   │ External LSA │ ASBR         │ 전체 OSPF 도메인 (Stub 제외)        │
│  7   │ NSSA External│ ASBR (NSSA)  │ NSSA Area 내부 → ABR에서 5로 변환   │
└──────┴──────────────┴──────────────┴───────────────────────────────────┘
```

### 3.2 LSA Type 1: Router LSA

모든 OSPF 라우터가 생성. 자신의 인터페이스와 연결 상태를 기술.

```
! 확인 명령어
show ip ospf database router

! 출력 예시
       OSPF Router with ID (1.1.1.1) (Process ID 1)
       
       Router Link States (Area 0)
       
 LS age: 352
 Link State ID: 1.1.1.1          ← 생성 라우터의 RID
 Advertising Router: 1.1.1.1
 Number of Links: 3

   Link connected to: a Stub Network
    (Link ID) Network/subnet number: 1.1.1.1
    (Link Data) Network Mask: 255.255.255.255
    TOS 0 Metrics: 1

   Link connected to: another Router (point-to-point)
    (Link ID) Neighboring Router ID: 2.2.2.2
    (Link Data) Router Interface address: 10.1.12.1
    TOS 0 Metrics: 1

   Link connected to: a Transit Network
    (Link ID) Designated Router address: 10.1.12.2
    (Link Data) Router Interface address: 10.1.12.1
    TOS 0 Metrics: 1
```

**Link Type별 의미:**

| Link Type | 설명 | Link ID | Link Data |
|-----------|------|---------|-----------|
| Point-to-Point | P2P 연결 | Neighbor RID | Local Interface IP |
| Transit Network | Broadcast/NBMA (DR 있음) | DR's Interface IP | Local Interface IP |
| Stub Network | Loopback 또는 끝단 | Network Number | Subnet Mask |
| Virtual Link | Virtual Link 연결 | Neighbor RID | Local Interface IP |

### 3.3 LSA Type 2: Network LSA

DR만 생성. Multi-access 네트워크의 모든 라우터를 리스트.

```
show ip ospf database network

  LS age: 1582
  Link State ID: 10.1.12.2          ← DR의 Interface IP
  Advertising Router: 2.2.2.2       ← DR의 RID
  Network Mask: /30
    Attached Router: 2.2.2.2         ← DR 자신
    Attached Router: 1.1.1.1         ← 참여 라우터
    Attached Router: 4.4.4.4         ← 참여 라우터
```

> **P2P로 변경하면 Type 2 LSA가 사라짐** → LSDB 크기 감소, SPF 계산 효율 향상

### 3.4 LSA Type 3: Summary LSA (Inter-Area)

ABR이 생성. 다른 Area의 prefix를 요약해서 전달.

```
show ip ospf database summary

  LS age: 203
  Link State ID: 192.168.10.0       ← 목적지 Network
  Advertising Router: 2.2.2.2       ← ABR의 RID
  Network Mask: /24
  TOS: 0  Metric: 20                ← ABR까지의 Cost + 원래 Cost
```

**Type 3 LSA 동작 핵심:**
```
┌─ Area 1 ─┐   ┌─ Area 0 ─┐   ┌─ Area 2 ─┐
│           │   │           │   │           │
│  R1 ──── ABR1 ──── R3 ──── ABR2 ──── R5  │
│  10.1.0/24│   │           │   │ 10.2.0/24 │
└───────────┘   └───────────┘   └───────────┘

1. R1이 Type 1 LSA로 10.1.0.0/24 광고 (Area 1 내)
2. ABR1이 Area 0으로 Type 3 LSA 생성 (10.1.0.0/24)
3. ABR2가 Area 0에서 Type 3 수신 → Area 2로 새 Type 3 생성
4. ⚠️ ABR은 Type 3를 재생성 (re-originate), 단순 전달(flood) 아님!
   → 이것이 Distance Vector처럼 동작하는 부분
```

### 3.5 LSA Type 4 & 5: External Routes

**Type 5 (AS External LSA):** ASBR이 생성, 외부 경로 (redistribute)

```
show ip ospf database external

  LS age: 1423
  Link State ID: 172.16.0.0         ← External Network
  Advertising Router: 4.4.4.4       ← ASBR의 RID
  Network Mask: /16
  Metric Type: 2 (Larger than any link state path)
  TOS: 0  Metric: 20
  Forward Address: 0.0.0.0
```

**Type 4 (ASBR Summary LSA):** ABR이 생성, ASBR의 위치(도달 방법) 전달

```
show ip ospf database asbr-summary

  LS age: 892
  Link State ID: 4.4.4.4            ← ASBR의 RID
  Advertising Router: 2.2.2.2       ← ABR의 RID
  TOS: 0  Metric: 10                ← ABR에서 ASBR까지의 Cost
```

**E1 vs E2 Metric 차이:**

```
E1: Cost = External Metric + Internal Cost to ASBR
    → ASBR까지의 거리가 경로 선택에 영향
    → 내부적으로 optimal path 보장

E2: Cost = External Metric Only (기본값!)
    → ASBR까지의 거리 무시
    → 같은 E2 Metric이면 Internal Cost로 Tie-break

! Redistribute 시 Metric Type 설정
router ospf 1
 redistribute bgp 65001 subnets metric-type 1   ← E1으로 변경
 redistribute bgp 65001 subnets                  ← 기본 E2
```

### 3.6 LSA Type 7: NSSA External

```
! NSSA Area에서 ASBR이 Type 7 생성
! ABR이 Area 0으로 전달 시 Type 5로 변환

show ip ospf database nssa-external

  LS age: 445
  Link State ID: 10.99.0.0
  Advertising Router: 5.5.5.5       ← NSSA 내 ASBR
  Network Mask: /24
  Metric Type: 2
  TOS: 0  Metric: 20
  Forward Address: 5.5.5.5          ← ⚠️ Type 7의 특징: FA 설정됨
```

### 3.7 LSA 흐름 종합 다이어그램

```
┌── Stub Area ──┐    ┌──── Area 0 ────┐    ┌── NSSA ──────┐
│                │    │                 │    │               │
│  Type 1,2      │    │  Type 1,2,3    │    │  Type 1,2,7   │
│  Type 3 (제한) │    │  Type 4,5      │    │  Type 3 (제한) │
│  NO Type 4,5   │    │                 │    │  NO Type 4,5  │
│                │    │                 │    │               │
│  Default Route │◄───│  ABR            │───►│  ABR          │
│  (Type 3)      │    │  Type 7→5 변환  │◄───│  Type 7 생성  │
└────────────────┘    └─────────────────┘    └───────────────┘
```

---

## 4. Area Design & Stub Variations

### 4.1 Area Types 비교

| Area Type | Type 3 | Type 4 | Type 5 | Type 7 | Default Route |
|-----------|--------|--------|--------|--------|---------------|
| Normal | ✅ | ✅ | ✅ | ❌ | 선택 |
| Stub | ✅ | ❌ | ❌ | ❌ | 자동 (Type 3) |
| Totally Stubby | ❌ (기본만) | ❌ | ❌ | ❌ | 자동 (Type 3) |
| NSSA | ✅ | ❌ | ❌ | ✅ | 수동 |
| Totally NSSA | ❌ (기본만) | ❌ | ❌ | ✅ | 자동 (Type 3) |

### 4.2 Stub Area 설정

**토폴로지:**
```
           Area 0                Area 10 (Stub)
ISP ─── ASBR ─── ABR ─────────── R5
                   │               │
            External Routes    이 Area에는
            (Type 5)           외부 경로 불필요
```

**ABR 설정:**
```
router ospf 1
 area 10 stub
 ! Type 4,5 차단, Default Route (Type 3, cost 1) 자동 주입
```

**R5 설정:**
```
router ospf 1
 area 10 stub
 ! ⚠️ Area 내 모든 라우터에서 설정 필수 (Hello의 Stub Flag)
```

**검증:**
```
show ip ospf | include Area|Stub
  Area 10: Number of interfaces: 2, Stub area
  
show ip ospf database summary 0.0.0.0
  LS age: 52
  Link State ID: 0.0.0.0            ← Default Route
  Advertising Router: 2.2.2.2       ← ABR
  Network Mask: /0
  TOS: 0 Metric: 1                  ← Default cost
```

### 4.3 Totally Stubby Area

```
! ABR에서만 추가 설정
router ospf 1
 area 10 stub no-summary
 ! Type 3도 차단 (Default Route Type 3만 남음)

! R5 (Non-ABR)는 동일
router ospf 1
 area 10 stub
```

> **효과:** LSDB 크기 최소화. Area 내부 라우터는 Default Route 하나만 보유.

### 4.4 NSSA (Not-So-Stubby Area)

**시나리오:** 외부 경로 차단하되, 이 Area 내에서 Redistribution은 허용해야 할 때.

```
         Area 0              Area 20 (NSSA)
ASBR1 ─── ABR ─────────── R6 ─── Static/RIP
(BGP)                     (ASBR)
                          redistribute
```

**ABR 설정:**
```
router ospf 1
 area 20 nssa
```

**R6 (NSSA 내 ASBR) 설정:**
```
router ospf 1
 area 20 nssa
 redistribute static subnets    ← Type 7 LSA로 광고됨
```

**검증:**
```
! NSSA 내부에서
show ip ospf database nssa-external
  → Type 7 LSA 확인

! Area 0에서
show ip ospf database external
  → ABR이 Type 7 → Type 5로 변환한 결과
```

### 4.5 Totally NSSA

```
! ABR에서만
router ospf 1
 area 20 nssa no-summary
 ! Type 3 차단 + Default Route 자동 주입 + Type 7 허용

! Non-ABR
router ospf 1
 area 20 nssa
```

### 4.6 NSSA Default Route 옵션

```
! NSSA에는 기본적으로 Default Route가 자동 주입되지 않음!
! 수동으로 추가해야 함:

! 방법 1: ABR에서 Type 7 Default 주입
router ospf 1
 area 20 nssa default-information-originate

! 방법 2: Totally NSSA (자동 Type 3 Default)
router ospf 1
 area 20 nssa no-summary

! 방법 3: ABR에서 always
router ospf 1
 area 20 nssa default-information-originate always
```

### 4.7 대규모 OSPF Area 설계 Best Practice

```
┌──────────────────────────────────────────────────────┐
│  OSPF Area 설계 가이드라인                              │
│                                                      │
│  • Area당 라우터 수: 50~100대 이하 권장                 │
│  • Area 0는 안정적이고 고속 링크로 구성                  │
│  • ABR 수 최소화 (ABR = SPF 재계산 빈번)               │
│  • Stub/Totally Stubby로 LSDB 크기 관리               │
│  • Summarization은 ABR에서 수행                       │
│  • P2P 링크로 DR 선출 제거                             │
│  • 모든 Area는 Area 0에 물리적으로 연결                 │
│    (불가능 시 Virtual Link — 임시 해결책으로만)          │
└──────────────────────────────────────────────────────┘
```

---

## 5. OSPF Network Types

### 5.1 Network Types 비교

| Network Type | Hello/Dead | DR/BDR | Next-Hop | Neighbor 발견 |
|-------------|-----------|--------|----------|--------------|
| Broadcast | 10/40 | Yes | DR's IP | Auto (Multicast) |
| Point-to-Point | 10/40 | No | Peer's IP | Auto (Multicast) |
| NBMA | 30/120 | Yes | DR's IP | Manual |
| Point-to-Multipoint | 30/120 | No | Peer's IP | Auto (Multicast) |
| P2MP Non-Broadcast | 30/120 | No | Peer's IP | Manual |
| Loopback | N/A | No | N/A | /32 host route |

### 5.2 Point-to-Point 설정 (가장 권장)

```
! Ethernet 링크지만 P2P로 변경 (2대만 연결된 경우)
interface GigabitEthernet0/0
 ip ospf network point-to-point

! 장점:
! 1. DR/BDR 선출 없음 → 빠른 adjacency 형성
! 2. Type 2 LSA 제거 → LSDB 감소
! 3. Subnet 정확하게 광고 (/30, /31 등)
```

### 5.3 NBMA 환경 (Frame Relay / DMVPN)

**시나리오: Hub-and-Spoke DMVPN**

```
            Hub (DR 강제)
           /    |    \
        Spoke1  Spoke2  Spoke3
```

**방법 1: NBMA (Hub = DR 강제)**
```
! Hub
interface Tunnel0
 ip ospf network non-broadcast
 ip ospf priority 255

router ospf 1
 neighbor 10.0.0.2    ← Spoke1 (수동 neighbor)
 neighbor 10.0.0.3    ← Spoke2
 neighbor 10.0.0.4    ← Spoke3

! Spoke
interface Tunnel0
 ip ospf network non-broadcast
 ip ospf priority 0   ← DROther 강제
```

**방법 2: Point-to-Multipoint (권장)**
```
! Hub & 모든 Spoke
interface Tunnel0
 ip ospf network point-to-multipoint

! 장점:
! - DR 선출 없음
! - Neighbor 자동 발견 (Multicast)
! - 각 neighbor에 대해 /32 host route 생성
! - Spoke 간 통신은 Hub 경유 (실제 DMVPN 동작과 일치)
```

### 5.4 Loopback Interface 주의

```
! Loopback은 기본적으로 /32로 광고됨 (Network Type: Loopback)
interface Loopback0
 ip address 10.1.1.1 255.255.255.0
 ! OSPF는 이것을 10.1.1.1/32로 광고함!

! /24로 광고하려면:
interface Loopback0
 ip ospf network point-to-point
 ! → 10.1.1.0/24로 정확하게 광고
```

---

## 6. Route Summarization & Filtering

### 6.1 Inter-Area Summarization (ABR)

```
         Area 1                    Area 0
  10.1.1.0/24  ─┐
  10.1.2.0/24  ─┤── ABR ──────────────
  10.1.3.0/24  ─┤     Summary: 10.1.0.0/22
  10.1.4.0/24  ─┘
```

```
! ABR 설정
router ospf 1
 area 1 range 10.1.0.0 255.255.252.0
 ! /22로 4개 /24를 요약
 ! 개별 Type 3 LSA 대신 요약된 1개 Type 3 생성

! 요약 경로에 특정 cost 부여
 area 1 range 10.1.0.0 255.255.252.0 cost 100

! 요약이 아닌 차단 (해당 prefix 숨기기)
 area 1 range 10.1.99.0 255.255.255.0 not-advertise
```

**검증:**
```
show ip ospf database summary

! Area 0에서 보이는 것:
  10.1.0.0/22  ← 요약된 1개
! Area 1에서 보이는 것:
  10.1.1.0/24, 10.1.2.0/24, 10.1.3.0/24, 10.1.4.0/24  ← 개별 유지
```

### 6.2 External Summarization (ASBR)

```
! ASBR에서 외부 경로 요약
router ospf 1
 summary-address 172.16.0.0 255.255.0.0
 ! Redistribute된 172.16.x.x 경로를 하나의 Type 5로 요약

! Null0 경로 자동 생성됨 (Routing Loop 방지)
show ip route 172.16.0.0
  O  172.16.0.0/16 is directly connected, Null0
```

### 6.3 Inter-Area Route Filtering

```
! ABR에서 특정 Area로 가는 Type 3 LSA 필터링
router ospf 1
 area 1 filter-list prefix BLOCK-ROUTES in   ← Area 1로 들어가는 LSA 차단
 area 1 filter-list prefix BLOCK-ROUTES out  ← Area 1에서 나가는 LSA 차단

ip prefix-list BLOCK-ROUTES seq 5 deny 192.168.99.0/24
ip prefix-list BLOCK-ROUTES seq 10 permit 0.0.0.0/0 le 32
```

> **in vs out:**
> - `in`: 해당 Area로 들어가는 Type 3 차단  
> - `out`: 해당 Area에서 나가는 Type 3 차단

### 6.4 Distribute-List (RIB 필터링)

```
! RIB에 설치되는 것만 필터 (LSDB에는 영향 없음!)
router ospf 1
 distribute-list prefix FILTER-RIB in

ip prefix-list FILTER-RIB seq 5 deny 10.99.0.0/24
ip prefix-list FILTER-RIB seq 10 permit 0.0.0.0/0 le 32

! ⚠️ LSDB에는 LSA가 존재, RIB에만 설치 안 됨
! ⚠️ ABR에서 사용 시 Type 3 생성에는 영향 없어 Black Hole 발생 가능!
```

---

## 7. OSPF Authentication

### 7.1 Authentication Types

```
Type 0: Null (No Authentication) — 기본값
Type 1: Simple Password (Clear Text) — 비권장
Type 2: MD5 Cryptographic — 일반적으로 사용
HMAC-SHA: IOS-XE 16.x+ 지원 — 권장
```

### 7.2 Interface-Level MD5

```
! R1
interface GigabitEthernet0/0
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 OSPF-Secret!2024

! R2 (동일하게)
interface GigabitEthernet0/0
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 OSPF-Secret!2024
```

### 7.3 Area-Level Authentication

```
! Area 단위 일괄 적용
router ospf 1
 area 0 authentication message-digest

! 각 인터페이스에는 key만 설정
interface GigabitEthernet0/0
 ip ospf message-digest-key 1 md5 OSPF-Secret!2024

interface GigabitEthernet0/1
 ip ospf message-digest-key 1 md5 OSPF-Secret!2024
```

### 7.4 Key Chain & Key Rotation

```
! Key Rotation으로 무중단 비밀번호 변경
key chain OSPF-KEYS
 key 1
  key-string OldPassword123
  accept-lifetime 00:00:00 Jan 1 2024 23:59:59 Mar 31 2026
  send-lifetime 00:00:00 Jan 1 2024 23:59:59 Feb 28 2026
 key 2
  key-string NewPassword456
  accept-lifetime 00:00:00 Feb 1 2026 infinite
  send-lifetime 00:00:00 Mar 1 2026 infinite

interface GigabitEthernet0/0
 ip ospf authentication key-chain OSPF-KEYS
```

> **Key Rotation 전략:**  
> 1. 새 key의 accept-lifetime을 먼저 넓게 설정  
> 2. 모든 라우터에 배포  
> 3. send-lifetime을 새 key로 전환  
> → 겹치는 기간 동안 양쪽 key 모두 수신 가능

### 7.5 HMAC-SHA Authentication (IOS-XE)

```
key chain OSPF-SHA
 key 1
  key-string SecureKey!SHA256
  cryptographic-algorithm hmac-sha-256

interface GigabitEthernet0/0
 ip ospf authentication key-chain OSPF-SHA
```

---

## 8. Virtual Link & GRE Workarounds

### 8.1 Virtual Link

Area 0에 직접 연결되지 않은 Area를 연결하기 위한 **임시** 해결책.

```
         Area 0          Area 1           Area 2
      R1 ─────── ABR1 ─────── ABR2 ─────── R4
                  (Transit)    
                  
Area 2는 Area 0에 직접 연결 안 됨 → Virtual Link 필요!
Virtual Link: ABR1 ↔ ABR2 (Area 1을 Transit Area로)
```

**ABR1 설정:**
```
router ospf 1
 area 1 virtual-link 4.4.4.4
 ! area [Transit Area ID] virtual-link [상대 ABR의 Router-ID]
```

**ABR2 설정:**
```
router ospf 1
 area 1 virtual-link 1.1.1.1
```

**검증:**
```
show ip ospf virtual-links

  Virtual Link OSPF_VL0 to router 4.4.4.4 is up
    Transit area 1
    Hello 10, Dead 40, Wait 40, Retransmit 5
    State POINT_TO_POINT
```

### 8.2 Virtual Link + Authentication

```
router ospf 1
 area 1 virtual-link 4.4.4.4 authentication message-digest
 area 1 virtual-link 4.4.4.4 message-digest-key 1 md5 VL-Secret
```

### 8.3 Virtual Link 제약사항

```
❌ Transit Area는 Stub/NSSA가 될 수 없음
❌ 영구 솔루션이 아닌 임시 해결책
❌ 장거리 Virtual Link는 수렴 시간 저하

✅ 대안: GRE Tunnel + OSPF
  - GRE로 Area 0를 물리적으로 확장
  - 더 안정적이고 예측 가능
```

### 8.4 GRE Tunnel로 Area 0 확장

```
! R1 (Area 0)                    R4 (Area 0 연결 필요)
!  Lo0: 1.1.1.1                  Lo0: 4.4.4.4
!  (IGP로 도달 가능)

! R1
interface Tunnel0
 ip address 10.255.0.1 255.255.255.252
 tunnel source Loopback0
 tunnel destination 4.4.4.4
 ip ospf 1 area 0
 ip ospf network point-to-point

! R4
interface Tunnel0
 ip address 10.255.0.2 255.255.255.252
 tunnel source Loopback0
 tunnel destination 1.1.1.1
 ip ospf 1 area 0
 ip ospf network point-to-point

! ⚠️ Recursive Routing 방지: Tunnel destination은 IGP가 아닌 Static으로
ip route 1.1.1.1 255.255.255.255 <next-hop>   ! R4에서
```

---

## 9. OSPFv3 (IPv6)

### 9.1 OSPFv3 vs OSPFv2 차이

| 항목 | OSPFv2 | OSPFv3 |
|------|--------|--------|
| Address Family | IPv4 only | IPv6 (+ IPv4 with AF) |
| Transport | IPv4 | IPv6 Link-Local |
| Multicast | 224.0.0.5/6 | FF02::5/6 |
| Authentication | 자체 MD5 | IPsec (외부) |
| LSA Type 1,2 | Network 정보 포함 | Network 정보 분리 |
| 신규 LSA | - | Type 8 (Link LSA), Type 9 (Intra-Area Prefix) |
| Instance ID | - | 같은 링크에 다수 OSPF 인스턴스 |

### 9.2 OSPFv3 기본 설정 (Traditional)

```
ipv6 unicast-routing

interface GigabitEthernet0/0
 ipv6 address 2001:db8:12::1/64
 ipv6 ospf 1 area 0

interface Loopback0
 ipv6 address 2001:db8::1/128
 ipv6 ospf 1 area 0

ipv6 router ospf 1
 router-id 1.1.1.1    ← 여전히 32-bit (IPv4 형식)
 auto-cost reference-bandwidth 100000
```

### 9.3 OSPFv3 Address-Family Mode (권장)

IPv4와 IPv6를 하나의 OSPFv3 프로세스로 관리:

```
router ospfv3 1
 router-id 1.1.1.1
 auto-cost reference-bandwidth 100000
 !
 address-family ipv4 unicast
  ! IPv4 관련 설정
 !
 address-family ipv6 unicast
  ! IPv6 관련 설정

interface GigabitEthernet0/0
 ip address 10.1.12.1 255.255.255.252
 ipv6 address 2001:db8:12::1/64
 ospfv3 1 ipv4 area 0     ← IPv4
 ospfv3 1 ipv6 area 0     ← IPv6
```

### 9.4 OSPFv3 IPsec Authentication

```
interface GigabitEthernet0/0
 ipv6 ospf authentication ipsec spi 256 sha1 
  1234567890ABCDEF1234567890ABCDEF12345678

! 또는 Area 단위
ipv6 router ospf 1
 area 0 authentication ipsec spi 500 md5
  0123456789ABCDEF0123456789ABCDEF
```

### 9.5 OSPFv3 검증

```
show ospfv3 neighbor
show ospfv3 interface brief
show ospfv3 database
show ospfv3 ipv4 database          ! AF mode
show ospfv3 ipv6 database          ! AF mode
show ospfv3 border-routers
```

---

## 10. OSPF + MPLS / Segment Routing 연동

### 10.1 OSPF as PE-CE Protocol (MPLS L3VPN)

```
! PE Router — VRF context에서 CE와 OSPF
router ospf 100 vrf CUSTOMER-A
 router-id 10.10.10.1
 redistribute bgp 65000 subnets    ← MP-BGP에서 받은 경로를 OSPF로
 network 10.10.1.0 0.0.0.3 area 0

! PE Router — BGP VPNv4에서 OSPF 경로 수신
router bgp 65000
 address-family ipv4 vrf CUSTOMER-A
  redistribute ospf 100 match internal external 1 external 2
```

**OSPF Sham-Link (Backdoor 문제 해결):**

```
         PE1 ════ MPLS Core ════ PE2
          │                       │
         CE1 ─── Backdoor ─── CE2
              (직접 OSPF 연결)

! Backdoor가 Inter-Area (Type 3)보다 선호됨 → MPLS 우회
! 해결: Sham-Link

! PE1
interface Loopback100
 vrf forwarding CUSTOMER-A
 ip address 10.255.1.1 255.255.255.255

router ospf 100 vrf CUSTOMER-A
 area 0 sham-link 10.255.1.1 10.255.2.2 cost 1
 ! Source: PE1 VRF Loopback, Dest: PE2 VRF Loopback
```

### 10.2 OSPF Segment Routing (SR-MPLS)

```
! IOS-XE: OSPF with Segment Routing
router ospf 1
 router-id 1.1.1.1
 segment-routing mpls
 segment-routing prefix-sid-map receive
 segment-routing prefix-sid-map advertise-local

interface Loopback0
 ip address 1.1.1.1 255.255.255.255
 ip ospf 1 area 0
 ip ospf prefix-sid index 1         ← Prefix SID = SRGB Base + Index

! SRGB (Segment Routing Global Block) 설정
segment-routing mpls
 global-block 16000 23999           ← SID Label Range
 ! Prefix SID Label = 16000 + 1 = 16001
```

**IOS-XR:**
```
router ospf 1
 router-id 1.1.1.1
 segment-routing mpls
 segment-routing forwarding mpls
 !
 area 0
  interface Loopback0
   prefix-sid index 1
  !
  interface GigabitEthernet0/0/0/0
   network point-to-point
```

### 10.3 OSPF + TI-LFA (Topology-Independent Loop-Free Alternate)

```
! IOS-XR: 50ms 이하 Convergence
router ospf 1
 area 0
  interface GigabitEthernet0/0/0/0
   fast-reroute per-prefix
   fast-reroute per-prefix ti-lfa enable

! 검증
show ospf 1 fast-reroute
show cef 10.1.1.0/24 detail    ← Backup path 확인
```

---

## 11. OSPF Convergence & Performance Tuning

### 11.1 OSPF Convergence 구성 요소

```
Total Convergence Time = 
  Detection Time + LSA Generation + LSA Flooding + SPF Calculation + RIB/FIB Update

┌─────────────────────────┬────────────┬─────────────────────────┐
│ Component               │ Default    │ Tuned                   │
├─────────────────────────┼────────────┼─────────────────────────┤
│ Failure Detection       │ 40s (Dead) │ <1s (BFD)              │
│ LSA Generation Delay    │ 5s         │ 0ms initial             │
│ LSA Flooding            │ ~instant   │ ~instant                │
│ SPF Delay               │ 5s         │ 50ms initial            │
│ RIB/FIB Update          │ ~instant   │ ~instant                │
├─────────────────────────┼────────────┼─────────────────────────┤
│ Total (worst case)      │ ~50s       │ <1s                     │
└─────────────────────────┴────────────┴─────────────────────────┘
```

### 11.2 BFD 연동

```
! Interface-Level BFD
interface GigabitEthernet0/0
 bfd interval 100 min_rx 100 multiplier 3
 ! → 100ms × 3 = 300ms 이내 장애 감지

! OSPF에서 BFD 활성화
router ospf 1
 bfd all-interfaces
 ! 또는 인터페이스별:
 ! interface GigabitEthernet0/0
 !  ip ospf bfd

! 검증
show bfd neighbors detail
show ip ospf interface | include BFD
```

### 11.3 SPF Throttling (Exponential Backoff)

```
router ospf 1
 timers throttle spf 50 200 5000
 ! spf [initial-delay] [secondary-delay] [max-delay] (ms)
 !
 ! 첫 SPF: 50ms 후 실행
 ! 연속 trigger: 200ms, 400ms, 800ms, ... 최대 5000ms
 ! 안정화 후 다시 50ms로 리셋

! IOS-XR
router ospf 1
 timers throttle spf 50 200 5000
```

### 11.4 LSA Throttling

```
router ospf 1
 timers throttle lsa all 0 1000 5000
 ! lsa [initial-delay] [secondary-delay] [max-delay] (ms)
 !
 ! 첫 LSA: 즉시 생성 (0ms)
 ! 연속 변경: 1000ms, 2000ms, ... 최대 5000ms

! LSA Pacing (Flooding 효율)
 timers pacing lsa-group 30
 ! 30ms 단위로 LSA 그룹핑하여 전송
```

### 11.5 Hello/Dead Timer 조정

```
! Fast Hello (Sub-Second)
interface GigabitEthernet0/0
 ip ospf dead-interval minimal hello-multiplier 4
 ! → Hello: 250ms, Dead: 1s
 ! ⚠️ BFD가 더 효율적 (CPU 부담 적음)

! 일반 Timer 조정
interface GigabitEthernet0/0
 ip ospf hello-interval 2
 ip ospf dead-interval 8
 ! ⚠️ 양쪽 반드시 일치!
```

### 11.6 OSPF Prefix Suppression

불필요한 Transit Link prefix를 LSDB에서 제거:

```
router ospf 1
 prefix-suppression
 ! 모든 Transit link의 prefix가 SPF에서 제외됨
 ! Loopback과 passive interface는 유지

! 특정 인터페이스만 유지
interface GigabitEthernet0/0
 ip ospf prefix-suppression disable
```

> **효과:** LSDB 크기 감소, SPF 계산 시간 단축, FIB 엔트리 감소

### 11.7 Incremental SPF (iSPF)

```
! IOS-XE
router ospf 1
 ispf

! 변경된 부분만 재계산 (Full SPF 대신)
! Type 1/2 LSA 변경 → Full SPF
! Type 3/5/7 LSA 변경 → Partial SPF (더 빠름)
```

### 11.8 OSPF Graceful Restart (NSF)

```
! Cisco NSF (Non-Stop Forwarding)
router ospf 1
 nsf cisco helper                ← Helper mode (neighbor 지원)
 nsf ietf helper                 ← IETF Graceful Restart helper

! NSF-capable Router
router ospf 1
 nsf cisco
 nsf ietf restart-interval 90

! 검증
show ip ospf nsf
show ip ospf neighbor detail | include Graceful
```

---

## 12. OSPF & BGP Redistribution

### 12.1 기본 Redistribution

```
! OSPF → BGP
router bgp 65001
 address-family ipv4 unicast
  redistribute ospf 1 match internal external 1 external 2
  ! match 옵션:
  ! internal       → O, O IA 경로
  ! external 1     → O E1 경로
  ! external 2     → O E2 경로
  ! nssa-external  → O N1, O N2 경로

! BGP → OSPF
router ospf 1
 redistribute bgp 65001 subnets metric 100 metric-type 1
 ! ⚠️ subnets 키워드 필수! 없으면 classful 경로만 재분배
 ! metric: OSPF cost 값
 ! metric-type 1: E1 (내부 cost 포함)
 ! metric-type 2: E2 (기본, 외부 cost만)
```

### 12.2 Mutual Redistribution (양방향) 문제

```
        OSPF          Both          BGP
  R1 ─────── R2 ─────── R3 ─────── R4
              ↕ redistribute        ↕
              R5 ─────── R6 ─────── R7
                 OSPF      redistribute  BGP

! 문제: Routing Loop, Suboptimal Path, 
!       AD 차이로 인한 경로 선호도 역전
```

**해결책: Route Tagging**

```
! R2: BGP → OSPF 재분배 시 Tag 부여
route-map BGP-TO-OSPF permit 10
 set tag 100

router ospf 1
 redistribute bgp 65001 subnets route-map BGP-TO-OSPF

! R5: OSPF → BGP 재분배 시 Tagged 경로 차단
route-map OSPF-TO-BGP deny 10
 match tag 100
route-map OSPF-TO-BGP permit 20

router bgp 65001
 address-family ipv4 unicast
  redistribute ospf 1 route-map OSPF-TO-BGP
```

### 12.3 Redistribution with Prefix Control

```
! 특정 prefix만 재분배
ip prefix-list OSPF-TO-BGP-ALLOW seq 5 permit 10.1.0.0/16 le 24
ip prefix-list OSPF-TO-BGP-ALLOW seq 10 permit 172.16.0.0/12 le 24

route-map OSPF-TO-BGP permit 10
 match ip address prefix-list OSPF-TO-BGP-ALLOW
 set metric 100
 set origin incomplete

router bgp 65001
 address-family ipv4 unicast
  redistribute ospf 1 route-map OSPF-TO-BGP
```

### 12.4 Static/Connected → OSPF

```
router ospf 1
 ! Connected 재분배
 redistribute connected subnets
 
 ! Static 재분배
 redistribute static subnets metric 50 metric-type 1
 
 ! Default Route 주입
 default-information originate                    ← RIB에 default 있을 때만
 default-information originate always             ← 항상 주입
 default-information originate always metric 10 metric-type 1
```

### 12.5 Administrative Distance 조정

```
! OSPF AD 기본값
! Intra-Area (O): 110
! Inter-Area (O IA): 110
! External (O E1/E2): 110

! Redistribution 환경에서 AD 조정
router ospf 1
 distance ospf intra-area 110 inter-area 115 external 170
 ! External AD를 높여서 내부 경로 우선

! 특정 경로의 AD 변경
 distance 90 10.1.1.0 0.0.0.255     ← 특정 소스의 AD 변경
```

---

## 13. Troubleshooting Playbook

### 13.1 Neighbor가 안 올라올 때

```
Step 1: 기본 연결 확인
  ping [neighbor IP]
  show ip ospf interface [interface]
  → Area, Network Type, Timer, Authentication 확인

Step 2: Neighbor 상태 확인
  show ip ospf neighbor
  → 아무것도 안 보이면: Hello 미수신
  → Init: 단방향 통신 (한쪽이 안 보냄/못 받음)
  → ExStart/Exchange: MTU 불일치

Step 3: 상세 점검
  show ip ospf neighbor detail
  → Dead Time countdown 확인
  → Options: Stub flag 확인

Step 4: 패킷 캡처
  debug ip ospf hello
  debug ip ospf adj
  ! ⚠️ production에서 주의!
```

**주요 원인별 해결:**

| 증상 | 원인 | 해결 |
|------|------|------|
| Neighbor 안 보임 | ACL/방화벽 IP 89 차단 | ACL permit ospf |
| Stuck in Init | 단방향: 한쪽 ACL/인터페이스 | Multicast 확인 |
| Stuck in ExStart | MTU 불일치 | ip ospf mtu-ignore 또는 MTU 통일 |
| Stuck in Exchange | DBD 패킷 손실 | MTU, 인터페이스 에러 확인 |
| Down/Init 반복 | Hello/Dead timer 불일치 | Timer 통일 |
| Down/Init 반복 | Area ID 불일치 | Area 설정 확인 |
| Down/Init 반복 | Authentication 불일치 | Key/Type 확인 |
| Down/Init 반복 | Subnet mask 불일치 | IP/Mask 확인 |

### 13.2 ExStart/Exchange Stuck (MTU 문제)

```
! 가장 흔한 원인: MTU 불일치
show ip ospf interface GigabitEthernet0/0
  → MTU mismatch detection is enabled
  → Interface MTU: 1500

! 해결 방법 1: MTU 통일 (권장)
interface GigabitEthernet0/0
 mtu 1500

! 해결 방법 2: MTU 검사 비활성화 (임시)
interface GigabitEthernet0/0
 ip ospf mtu-ignore

! GRE/IPsec 환경에서 특히 주의
! GRE: 1476 (24B overhead)
! GRE+IPsec: ~1400 (변동)
interface Tunnel0
 ip mtu 1400
 ip tcp adjust-mss 1360
```

### 13.3 경로가 OSPF Table에 없을 때

```
Step 1: LSDB 확인
  show ip ospf database
  → LSA가 존재하는지 확인

Step 2: 해당 prefix의 LSA 상세
  show ip ospf database router [advertising-router]
  show ip ospf database summary [network]
  show ip ospf database external [network]

Step 3: SPF 결과 확인
  show ip ospf rib [network]
  → SPF가 선택한 경로 확인

Step 4: RIB 설치 확인
  show ip route ospf
  show ip route [network]
  → 다른 프로토콜이 더 낮은 AD로 선점했을 수 있음

Step 5: Distribute-list 확인
  show ip protocols | section ospf
  → Incoming filter list 확인
```

### 13.4 Sub-Optimal Routing

```
! 비대칭 경로 확인
traceroute [destination]

! Cost 확인
show ip ospf interface brief
show ip ospf border-routers

! Cost 수동 조정
interface GigabitEthernet0/0
 ip ospf cost 10    ← 수동 설정 (auto-cost 무시)

! Reference BW 불일치 확인
show ip ospf | include Reference
  → 모든 라우터에서 동일해야 함!
```

### 13.5 OSPF Flapping

```
! 지속적인 neighbor Up/Down
show logging | include OSPF
  → %OSPF-5-ADJCHG: 메시지 빈도 확인

! 원인 분석
show ip ospf statistics      ← SPF 실행 빈도
show ip ospf database | include Age
  → LSA Age가 비정상적으로 자주 리셋되는 LSA 찾기

! MaxAge LSA 확인 (라우터 이탈 신호)
show ip ospf database | include MaxAge

! 인터페이스 에러 확인
show interface [interface] | include error|CRC|drops

! 일반적인 Flapping 원인:
! 1. 물리적 링크 불안정 (SFP, 케이블)
! 2. CPU 과부하 → Hello 처리 지연
! 3. MTU 간헐적 문제
! 4. Duplicate Router-ID
```

### 13.6 Duplicate Router-ID 문제

```
! 증상: 경로가 나타났다 사라짐, LSA 충돌
show ip ospf database router
  → 같은 Link State ID가 다른 내용으로 반복 갱신

! 확인
show ip ospf | include Router ID
  → 모든 라우터에서 실행하여 중복 확인

! 해결
router ospf 1
 router-id X.X.X.X    ← Unique하게 수동 설정
! 후 clear ip ospf process
```

### 13.7 실전 Debug 명령어

```
! ⚠️ Production에서 필터링과 함께 사용!

! Hello 패킷 확인
debug ip ospf hello
debug ip ospf hello interface GigabitEthernet0/0   ← 인터페이스 제한

! Adjacency 형성 과정
debug ip ospf adj

! LSA 관련
debug ip ospf lsa-generation
debug ip ospf flooding

! SPF 실행
debug ip ospf spf

! IOS-XR
debug ospf 1 hello
debug ospf 1 adj
debug ospf 1 spf

! Debug 끄기
undebug all
```

---

## Appendix A: OSPF Quick Reference Commands

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                   상태 확인
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
show ip ospf                           # OSPF 프로세스 정보
show ip ospf neighbor                  # Neighbor 요약
show ip ospf neighbor detail           # Neighbor 상세
show ip ospf interface                 # 인터페이스 상세
show ip ospf interface brief           # 인터페이스 요약
show ip ospf database                  # LSDB 전체 요약
show ip ospf database router           # Type 1 LSA
show ip ospf database network          # Type 2 LSA
show ip ospf database summary          # Type 3 LSA
show ip ospf database asbr-summary     # Type 4 LSA
show ip ospf database external         # Type 5 LSA
show ip ospf database nssa-external    # Type 7 LSA
show ip ospf border-routers            # ABR/ASBR 정보
show ip ospf virtual-links             # Virtual Link 상태
show ip ospf rib                       # OSPF RIB
show ip ospf statistics                # SPF 실행 통계
show ip protocols                      # 프로토콜 요약

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                   운영 명령
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
clear ip ospf process                  # 프로세스 리셋 (RID 변경 시)
clear ip ospf counters                 # 카운터 리셋
clear ip ospf force-spf                # SPF 강제 실행

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                   IOS-XR 차이점
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
show ospf neighbor
show ospf interface brief
show ospf database
show ospf database router self-originate
show ospf border-routers
show ospf vrf CUSTOMER-A neighbor
clear ospf 1 process
```

## Appendix B: OSPF LSA Cheat Sheet

```
┌─────────────────────────────────────────────────────────────────┐
│                    LSA 생성자 & Scope 요약                       │
│                                                                 │
│  Type 1 (Router)                                                │
│  ├── 생성: 모든 라우터                                           │
│  ├── Scope: Area 내부                                           │
│  ├── 내용: 자신의 링크와 neighbor 정보                            │
│  └── SPF Tree의 기본 재료                                        │
│                                                                 │
│  Type 2 (Network)                                               │
│  ├── 생성: DR                                                   │
│  ├── Scope: Area 내부                                           │
│  ├── 내용: Multi-access 네트워크의 참여 라우터 목록                │
│  └── P2P로 변경하면 제거됨                                       │
│                                                                 │
│  Type 3 (Summary)                                               │
│  ├── 생성: ABR                                                  │
│  ├── Scope: Area 간                                             │
│  ├── 내용: 다른 Area의 prefix 정보                               │
│  └── ABR이 재생성 (Distance Vector 처럼 동작)                    │
│                                                                 │
│  Type 4 (ASBR Summary)                                          │
│  ├── 생성: ABR                                                  │
│  ├── Scope: Area 간                                             │
│  ├── 내용: ASBR의 위치 (도달 방법)                               │
│  └── 다른 Area의 라우터가 ASBR을 찾을 수 있게                    │
│                                                                 │
│  Type 5 (AS External)                                           │
│  ├── 생성: ASBR                                                 │
│  ├── Scope: 전체 OSPF (Stub/NSSA 제외)                          │
│  ├── 내용: 외부 재분배 경로                                      │
│  └── E1 (cost 누적) vs E2 (cost 고정, 기본)                     │
│                                                                 │
│  Type 7 (NSSA External)                                         │
│  ├── 생성: NSSA 내 ASBR                                         │
│  ├── Scope: NSSA 내부                                           │
│  ├── 내용: NSSA에서의 외부 재분배 경로                            │
│  └── ABR에서 Type 5로 변환되어 Area 0으로 전파                   │
└─────────────────────────────────────────────────────────────────┘
```

## Appendix C: OSPF Design Decision Matrix

```
┌──────────────────────────────────┬─────────────────────────────────┐
│  시나리오                         │  권장 설계                       │
├──────────────────────────────────┼─────────────────────────────────┤
│  Campus Access Layer             │  Totally Stubby + P2P Links     │
│  Campus Distribution             │  Area 0 + Summarization         │
│  Branch Office (Small)           │  Stub Area + Default Route      │
│  Branch with Local Internet      │  NSSA                           │
│  Data Center Underlay            │  P2P Only + BFD + SR            │
│  MPLS PE-CE                      │  VRF OSPF + Sham-Link (필요 시) │
│  ISP Backbone                    │  IS-IS 권장 (OSPF 시 Area 0)   │
│  Large Enterprise Core           │  Area 0 + RR (if BGP overlay)  │
│  Multi-VRF Environment           │  OSPFv3 AF Mode                 │
│  High Availability               │  BFD + SPF Tuning + NSF         │
│  Segment Routing Underlay        │  OSPF SR + TI-LFA               │
│  DMVPN Hub-Spoke                 │  Point-to-Multipoint             │
└──────────────────────────────────┴─────────────────────────────────┘
```

---

> **문서 버전:** v1.0  
> **최종 수정:** 2026-03-25  
> **참고:** Cisco IOS-XE 17.x, IOS-XR 7.x, NX-OS 10.x 기반
