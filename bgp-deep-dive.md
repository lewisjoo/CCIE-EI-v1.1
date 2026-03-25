# BGP Deep Dive — 예제 중심 완전 정복

> **대상:** 네트워크 엔지니어 / 아키텍트 / CCIE 준비자  
> **범위:** BGP Fundamentals → Advanced Design → Real-World Troubleshooting  
> **플랫폼:** Cisco IOS-XE / IOS-XR 기반 예제

---

## 목차

1. [BGP 기본 개념 리뷰](#1-bgp-기본-개념-리뷰)
2. [Neighbor Establishment 상세](#2-neighbor-establishment-상세)
3. [Path Attribute & Best Path Selection](#3-path-attribute--best-path-selection)
4. [iBGP 설계: Route Reflector & Confederation](#4-ibgp-설계-route-reflector--confederation)
5. [BGP Filtering 완전 정복](#5-bgp-filtering-완전-정복)
6. [BGP Communities 활용](#6-bgp-communities-활용)
7. [MP-BGP: Address Family 확장](#7-mp-bgp-address-family-확장)
8. [BGP EVPN / VXLAN 연동](#8-bgp-evpn--vxlan-연동)
9. [BGP in SD-WAN & Segment Routing](#9-bgp-in-sd-wan--segment-routing)
10. [BGP Convergence & Performance Tuning](#10-bgp-convergence--performance-tuning)
11. [BGP Security](#11-bgp-security)
12. [Troubleshooting Playbook](#12-troubleshooting-playbook)

---

## 1. BGP 기본 개념 리뷰

### 1.1 BGP의 본질

BGP는 **Path Vector Protocol**이다. Distance Vector도, Link-State도 아닌 독립적 카테고리다.

| 특성 | 설명 |
|------|------|
| Protocol Type | Path Vector |
| Transport | TCP 179 |
| Admin Distance | eBGP: 20, iBGP: 200 |
| Algorithm | Best Path Selection (NOT SPF) |
| Update Type | Incremental (Triggered only) |
| Full Table Size | ~950K+ prefixes (IPv4 DFZ, 2024 기준) |

### 1.2 AS 유형

```
┌─────────────────────────────────────────────────────┐
│  Transit AS (ISP)                                   │
│  - 다른 AS 트래픽을 중계                              │
│  - Full BGP Table 유지                               │
│                                                     │
│  Stub AS (Enterprise)                               │
│  - Single-homed 또는 Multi-homed                     │
│  - 자기 prefix만 광고                                 │
│                                                     │
│  Multi-homed AS                                     │
│  - 2개 이상 ISP 연결                                  │
│  - Inbound/Outbound 트래픽 엔지니어링 필수             │
└─────────────────────────────────────────────────────┘
```

### 1.3 eBGP vs iBGP 핵심 차이

| 구분 | eBGP | iBGP |
|------|------|------|
| AS 관계 | 서로 다른 AS | 같은 AS |
| Next-Hop | 자신으로 변경 | 변경 안 함 (기본) |
| AD | 20 | 200 |
| TTL | 1 (기본) | 255 |
| AS-Path | 자기 AS 추가 | 변경 없음 |
| Loop Prevention | AS-Path 기반 | Split Horizon (RR 제외) |
| Full Mesh | 불필요 | 필요 (또는 RR/Confederation) |

---

## 2. Neighbor Establishment 상세

### 2.1 BGP FSM (Finite State Machine)

```
Idle → Connect → OpenSent → OpenConfirm → Established
  │        │          │           │
  │        ▼          ▼           ▼
  └── Active ←── (TCP fail) ← (Open mismatch)
```

**각 상태별 의미:**

| State | 설명 | Stuck 원인 |
|-------|------|-----------|
| Idle | 초기 상태 | admin down, 잘못된 neighbor 설정 |
| Connect | TCP SYN 전송 중 | ACL 차단, 라우팅 불가 |
| Active | TCP 재시도 | **가장 흔한 문제!** TCP 179 도달 불가 |
| OpenSent | OPEN 메시지 전송 완료 | AS 번호 불일치, capability 불일치 |
| OpenConfirm | OPEN 교환 완료, Keepalive 대기 | Hold timer 불일치 |
| Established | 정상 | - |

### 2.2 기본 eBGP Peering

**토폴로지:**
```
R1 (AS 65001) ------- R2 (AS 65002)
10.1.1.1/30          10.1.1.2/30
```

**R1 설정:**
```
router bgp 65001
 bgp router-id 1.1.1.1
 bgp log-neighbor-changes
 !
 neighbor 10.1.1.2 remote-as 65002
 neighbor 10.1.1.2 description eBGP-to-R2
 !
 address-family ipv4 unicast
  network 192.168.1.0 mask 255.255.255.0
  neighbor 10.1.1.2 activate
 exit-address-family
```

**R2 설정:**
```
router bgp 65002
 bgp router-id 2.2.2.2
 bgp log-neighbor-changes
 !
 neighbor 10.1.1.1 remote-as 65001
 neighbor 10.1.1.1 description eBGP-to-R1
 !
 address-family ipv4 unicast
  network 172.16.0.0 mask 255.255.0.0
  neighbor 10.1.1.1 activate
 exit-address-family
```

### 2.3 eBGP Multihop (Loopback Peering)

ISP 간 이중화 링크가 있을 때, Loopback으로 peering하면 한 링크 장애 시에도 세션 유지.

```
        link1: 10.1.1.0/30
R1 ════════════════════ R2
   ════════════════════
        link2: 10.1.2.0/30

R1 Lo0: 1.1.1.1          R2 Lo0: 2.2.2.2
```

**R1 설정:**
```
! Static routes for reachability to R2 loopback
ip route 2.2.2.2 255.255.255.255 10.1.1.2
ip route 2.2.2.2 255.255.255.255 10.1.2.2

router bgp 65001
 neighbor 2.2.2.2 remote-as 65002
 neighbor 2.2.2.2 ebgp-multihop 2
 neighbor 2.2.2.2 update-source Loopback0
 !
 address-family ipv4 unicast
  neighbor 2.2.2.2 activate
```

> **핵심 포인트:**  
> `ebgp-multihop`은 TTL을 증가시키고, `update-source`는 TCP 소스를 Loopback으로 변경한다. 반드시 양쪽 다 설정해야 한다.

### 2.4 iBGP Peering (Loopback 기반)

```
     OSPF Area 0 (IGP)
R1 ─────── R2 ─────── R3
Lo0: 1.1.1.1  Lo0: 2.2.2.2  Lo0: 3.3.3.3
       All in AS 65000
```

**R1 설정:**
```
router bgp 65000
 bgp router-id 1.1.1.1
 !
 ! iBGP는 Loopback peering이 표준
 neighbor 2.2.2.2 remote-as 65000
 neighbor 2.2.2.2 update-source Loopback0
 neighbor 3.3.3.3 remote-as 65000
 neighbor 3.3.3.3 update-source Loopback0
 !
 address-family ipv4 unicast
  neighbor 2.2.2.2 activate
  neighbor 2.2.2.2 next-hop-self        ← eBGP에서 받은 경로의 NH 변경
  neighbor 3.3.3.3 activate
  neighbor 3.3.3.3 next-hop-self
```

> **next-hop-self 사용 이유:**  
> eBGP에서 수신한 경로의 Next-Hop은 외부 IP다. iBGP peer는 이 외부 IP에 대한 라우팅이 없으므로, `next-hop-self`로 자신의 Loopback IP로 변경해야 한다.

### 2.5 BGP Authentication (MD5)

```
router bgp 65001
 neighbor 10.1.1.2 password 0 SecureBGP!2024
```

**검증:**
```
show tcp brief | include 179
show ip bgp neighbors 10.1.1.2 | include password
```

> **실무 팁:** MD5는 여전히 가장 널리 쓰이지만, IOS-XR에서는 TCP-AO 지원. Password 변경 시 세션이 리셋되므로 maintenance window에서 수행.

---

## 3. Path Attribute & Best Path Selection

### 3.1 Best Path Selection Algorithm (순서 중요!)

```
┌──────────────────────────────────────────────────────────────┐
│  BGP Best Path Selection (위에서 아래로, 먼저 결정되면 종료)      │
├──────────────────────────────────────────────────────────────┤
│  0. Next-Hop Reachable?         → 아니면 후보에서 제외          │
│  1. Highest Weight              → Cisco 전용, Local only       │
│  2. Highest Local Preference    → iBGP 내 전파, 기본값 100      │
│  3. Locally Originated          → network/redistribute/aggregate│
│  4. Shortest AS-Path            → AS 개수 비교                  │
│  5. Lowest Origin Type          → IGP(i) < EGP(e) < Incomplete(?)│
│  6. Lowest MED                  → 같은 이웃 AS에서만 비교 (기본) │
│  7. eBGP over iBGP              → eBGP 경로 우선               │
│  8. Lowest IGP Metric to NH     → Interior cost                │
│  9. Oldest eBGP Route           → Stability 선호               │
│ 10. Lowest Router-ID            → Tie-breaker                  │
│ 11. Lowest Neighbor IP          → 최종 Tie-breaker             │
└──────────────────────────────────────────────────────────────┘
```

> **암기법:** **"W**e **L**ove **O**ranges **A**nd **O**nions **M**ake **E**very **I**ndian **O**ld **R**eally **N**ice"

### 3.2 Weight 조정 예제

**시나리오:** R1이 ISP-A와 ISP-B에 dual-homed. ISP-A를 Primary로 사용.

```
            ISP-A (AS 100)
           /
R1 (65001)
           \
            ISP-B (AS 200)
```

```
! R1: ISP-A를 preferred path로 설정 (Weight는 local only)
router bgp 65001
 neighbor 10.1.1.2 remote-as 100
 neighbor 10.2.2.2 remote-as 200
 !
 address-family ipv4 unicast
  ! 방법 1: neighbor 단위 Weight
  neighbor 10.1.1.2 weight 200
  neighbor 10.2.2.2 weight 100
  !
  ! 방법 2: route-map으로 세밀한 제어
  neighbor 10.1.1.2 route-map SET-WEIGHT-PRIMARY in
  neighbor 10.2.2.2 route-map SET-WEIGHT-BACKUP in

! Route-map 예제
route-map SET-WEIGHT-PRIMARY permit 10
 set weight 200
!
route-map SET-WEIGHT-BACKUP permit 10
 set weight 100
```

### 3.3 Local Preference 예제

**시나리오:** AS 65000 내부에서 ISP-A를 선호하도록 LP 조정.

```
! Border Router connected to ISP-A
router bgp 65000
 address-family ipv4 unicast
  neighbor 10.1.1.1 route-map ISP-A-IN in

route-map ISP-A-IN permit 10
 set local-preference 200    ← 기본 100보다 높게

! Border Router connected to ISP-B
router bgp 65000
 address-family ipv4 unicast
  neighbor 10.2.2.1 route-map ISP-B-IN in

route-map ISP-B-IN permit 10
 set local-preference 50     ← 기본 100보다 낮게
```

**검증:**
```
show ip bgp 0.0.0.0/0
   Network       Next Hop     Metric  LocPrf  Weight  Path
*> 0.0.0.0/0     10.1.1.1              200      0     100 i    ← Best (LP 200)
*  0.0.0.0/0     10.2.2.1               50      0     200 i
```

### 3.4 AS-Path Prepending 예제

**Inbound 트래픽 엔지니어링:** 상대 AS에서 나의 prefix를 덜 선호하게 만들기.

```
! ISP-B 방향으로 나가는 광고에 AS-Path를 길게
router bgp 65001
 address-family ipv4 unicast
  neighbor 10.2.2.2 route-map PREPEND-TO-ISPB out

route-map PREPEND-TO-ISPB permit 10
 set as-path prepend 65001 65001 65001   ← 자기 AS만 prepend!

! 결과: ISP-B가 보는 AS-Path
! Before: 65001
! After:  65001 65001 65001 65001  (4 hops → 덜 선호)
```

> **실무 주의사항:**
> - 자기 AS만 prepend (다른 AS 번호 넣으면 BGP hijacking)
> - 3회 이상 prepend는 효과가 제한적 (ISP가 LP로 override 가능)
> - ISP가 AS-Path 기반 선택을 하지 않으면 무용지물

### 3.5 MED (Multi-Exit Discriminator) 예제

**시나리오:** AS 65001이 AS 100에게 "이 prefix는 이 링크로 보내달라"고 힌트.

```
        10.1.1.0/30 (Primary)
AS 65001 ═══════════════ AS 100
         ═══════════════
        10.1.2.0/30 (Backup)
```

```
! AS 65001 Border Router
router bgp 65001
 address-family ipv4 unicast
  ! Primary 링크: 낮은 MED
  neighbor 10.1.1.2 route-map MED-PRIMARY out
  ! Backup 링크: 높은 MED
  neighbor 10.1.2.2 route-map MED-BACKUP out

route-map MED-PRIMARY permit 10
 set metric 100

route-map MED-BACKUP permit 10
 set metric 200
```

> **MED 주의사항:**
> - MED는 같은 이웃 AS에서 온 경로끼리만 비교 (기본 동작)
> - `bgp always-compare-med` 명령으로 모든 AS 간 비교 가능
> - MED는 "제안"일 뿐, 상대 AS가 무시 가능

---

## 4. iBGP 설계: Route Reflector & Confederation

### 4.1 iBGP Full Mesh 문제

iBGP peer 수 = n × (n-1) / 2

| 라우터 수 | 필요 세션 |
|-----------|----------|
| 5 | 10 |
| 10 | 45 |
| 50 | 1,225 |
| 100 | 4,950 |

→ **해결책:** Route Reflector 또는 Confederation

### 4.2 Route Reflector 설계

**기본 동작 규칙:**

```
┌──────────────────────────────────────────────────┐
│  Route Reflector가 Best Path를 수신했을 때:         │
│                                                  │
│  Client에서 수신 → Client + Non-Client에게 반사    │
│  Non-Client에서 수신 → Client에게만 반사            │
│  eBGP에서 수신 → Client + Non-Client에게 반사      │
└──────────────────────────────────────────────────┘
```

**Loop Prevention:**
- **Originator-ID:** 최초 광고자의 Router-ID. 자신의 ID가 보이면 DROP.
- **Cluster-List:** RR의 Cluster-ID 목록. 자신의 Cluster-ID가 보이면 DROP.

**토폴로지:**
```
              ┌──── Client: R3
RR (R1) ─────┤
              └──── Client: R4
    │
    │ iBGP (Non-Client)
    │
   R2 (Non-Client)
```

**R1 (Route Reflector) 설정:**
```
router bgp 65000
 bgp router-id 1.1.1.1
 bgp cluster-id 1.1.1.1
 !
 ! Client 설정
 neighbor 3.3.3.3 remote-as 65000
 neighbor 3.3.3.3 update-source Loopback0
 neighbor 4.4.4.4 remote-as 65000
 neighbor 4.4.4.4 update-source Loopback0
 !
 ! Non-Client
 neighbor 2.2.2.2 remote-as 65000
 neighbor 2.2.2.2 update-source Loopback0
 !
 address-family ipv4 unicast
  neighbor 3.3.3.3 activate
  neighbor 3.3.3.3 route-reflector-client    ← Client 지정
  neighbor 4.4.4.4 activate
  neighbor 4.4.4.4 route-reflector-client    ← Client 지정
  neighbor 2.2.2.2 activate
  ! R2는 Non-Client (route-reflector-client 미적용)
```

**R3 (Client) 설정 — Client는 자신이 Client임을 모름:**
```
router bgp 65000
 neighbor 1.1.1.1 remote-as 65000
 neighbor 1.1.1.1 update-source Loopback0
 !
 address-family ipv4 unicast
  neighbor 1.1.1.1 activate
```

### 4.3 Hierarchical Route Reflector

**대규모 네트워크 (ISP/DC):**

```
            ┌─── RR1-L2 (Super RR) ───┐
            │                          │
       RR1-L1 (Regional)        RR2-L1 (Regional)
       /    \                   /    \
     PE1   PE2               PE3   PE4
```

```
! Super RR (RR1-L2) 설정
router bgp 65000
 bgp cluster-id 10.10.10.10
 !
 neighbor 11.11.11.11 remote-as 65000
 neighbor 11.11.11.11 update-source Loopback0
 neighbor 22.22.22.22 remote-as 65000
 neighbor 22.22.22.22 update-source Loopback0
 !
 address-family ipv4 unicast
  neighbor 11.11.11.11 route-reflector-client
  neighbor 22.22.22.22 route-reflector-client
```

> **설계 핵심:**  
> - 각 계층의 Cluster-ID는 반드시 달라야 함  
> - RR는 데이터 플레인에 있지 않아도 됨 (Out-of-band RR 가능)  
> - Redundancy: 각 Cluster에 RR 2대 권장

### 4.4 Confederation 설계

대형 ISP에서 AS를 sub-AS로 분할. 외부에서는 하나의 AS로 보임.

```
┌──────────────────── AS 65000 (Public AS) ────────────────────┐
│                                                              │
│   ┌─ Sub-AS 65534 ─┐     ┌─ Sub-AS 65535 ─┐                │
│   │  R1 ───── R2    │─────│  R3 ───── R4   │                │
│   │  (iBGP)         │eBGP │  (iBGP)        │                │
│   └─────────────────┘     └────────────────┘                │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**R1 설정 (Sub-AS 65534):**
```
router bgp 65534
 bgp confederation identifier 65000    ← 외부 공개 AS
 bgp confederation peers 65535         ← 다른 Sub-AS 목록
 !
 ! Sub-AS 내부: 일반 iBGP
 neighbor 2.2.2.2 remote-as 65534
 neighbor 2.2.2.2 update-source Loopback0
 !
 ! 다른 Sub-AS: Confederation eBGP (Next-Hop 변경, LP 유지)
 neighbor 10.1.1.2 remote-as 65535
```

> **Confederation vs Route Reflector:**
> - RR: 설정 간단, 확장성 좋음, 대부분 선택
> - Confederation: Optimal path selection 보장, 설정 복잡, 대형 ISP에서 사용

---

## 5. BGP Filtering 완전 정복

### 5.1 Prefix-List 필터링

```
! 특정 prefix 허용/차단
ip prefix-list ALLOW-SPECIFIC seq 5 permit 192.168.0.0/16 le 24
ip prefix-list ALLOW-SPECIFIC seq 10 permit 10.0.0.0/8 le 24
ip prefix-list ALLOW-SPECIFIC seq 99 deny 0.0.0.0/0 le 32

! Default route만 허용
ip prefix-list DEFAULT-ONLY seq 5 permit 0.0.0.0/0

! /24보다 작은 prefix 차단 (Bogon 방어)
ip prefix-list NO-SMALL-PREFIXES seq 5 permit 0.0.0.0/0 le 24

! 적용
router bgp 65001
 address-family ipv4 unicast
  neighbor 10.1.1.2 prefix-list ALLOW-SPECIFIC in
```

**Prefix-List 문법 해석:**
```
permit 10.0.0.0/8 le 24
  → 10.0.0.0/8 ~ 10.0.0.0/24 범위의 모든 prefix 허용
  → /8, /9, /10, ... /24 모두 매칭

permit 172.16.0.0/16 ge 20 le 24
  → /20 ~ /24 범위만 매칭

permit 192.168.1.0/24
  → 정확히 /24만 매칭 (ge/le 없으면 exact match)
```

### 5.2 AS-Path ACL 필터링

```
! 특정 AS를 경유하는 경로 차단
ip as-path access-list 10 deny _666_
ip as-path access-list 10 permit .*

! 특정 AS에서 originate된 경로만 허용
ip as-path access-list 20 permit ^100$
! → AS-Path가 정확히 "100"인 것만 (직접 연결된 AS 100이 생성한 경로)

! 자기 AS에서 originate된 경로만 허용 (Customer 필터)
ip as-path access-list 30 permit ^65001$

! 적용
router bgp 65000
 address-family ipv4 unicast
  neighbor 10.1.1.2 filter-list 10 in
```

**자주 쓰는 정규식:**
```
^$          → Locally originated (AS-Path 비어있음)
^100$       → 정확히 AS 100만 통과
^100_       → AS 100에서 시작
_100$       → AS 100에서 originate
_100_       → AS 100을 경유
.*          → 모든 AS-Path (any)
^[0-9]+$    → 정확히 1개 AS만 있는 경로
```

### 5.3 Route-Map 종합 필터링

```
! 복합 조건 필터링
ip prefix-list CUSTOMER-ROUTES seq 5 permit 192.168.0.0/16 le 24
ip community-list standard PREMIUM-CUSTOMER permit 65001:100

route-map COMPLEX-FILTER permit 10
 match ip address prefix-list CUSTOMER-ROUTES
 match community PREMIUM-CUSTOMER
 set local-preference 200
 set community 65000:1000 additive
!
route-map COMPLEX-FILTER permit 20
 match ip address prefix-list CUSTOMER-ROUTES
 set local-preference 100
!
route-map COMPLEX-FILTER deny 30
 ! 나머지 모두 차단 (implicit deny와 동일하지만 명시적으로)

router bgp 65000
 address-family ipv4 unicast
  neighbor 10.1.1.2 route-map COMPLEX-FILTER in
```

### 5.4 Maximum-Prefix 보호

```
router bgp 65001
 address-family ipv4 unicast
  ! 1000개 초과 시 세션 종료, 75%에서 warning
  neighbor 10.1.1.2 maximum-prefix 1000 75 restart 5
  !
  ! warning-only: 세션 종료 안 함
  neighbor 10.2.2.2 maximum-prefix 500 warning-only
```

> **실무:** ISP에서 Customer에게 반드시 설정. BGP leak/hijack 방지의 첫 번째 방어선.

---

## 6. BGP Communities 활용

### 6.1 Community 기본

```
형식: AA:NN (AS번호:임의값)
  65001:100  → AS 65001이 정의한 Community 100

Well-Known Communities:
  no-export        → eBGP peer에게 광고 안 함
  no-advertise     → 어떤 peer에게도 광고 안 함
  local-AS         → Confederation sub-AS 밖으로 광고 안 함
  internet         → 모든 곳에 광고 (기본)
```

### 6.2 Community 설정 & 매칭

```
! Community 설정 (route-map)
route-map SET-COMMUNITY permit 10
 match ip address prefix-list MY-ROUTES
 set community 65001:100 65001:200 additive   ← additive 중요!

! Community 매칭
ip community-list standard GOLD permit 65001:100
ip community-list standard SILVER permit 65001:200
ip community-list expanded PREMIUM permit 65001:1[0-9][0-9]

route-map APPLY-POLICY permit 10
 match community GOLD
 set local-preference 300
!
route-map APPLY-POLICY permit 20
 match community SILVER
 set local-preference 200
!
route-map APPLY-POLICY permit 30
 set local-preference 100

! Community 전송 활성화 (기본적으로 strip됨!)
router bgp 65001
 address-family ipv4 unicast
  neighbor 10.1.1.2 send-community both   ← standard + extended
```

### 6.3 ISP Community 활용 실전

```
┌────────────────────────────────────────────────┐
│  일반적인 ISP Community Convention:              │
│                                                │
│  ISP-AS:100  = Customer Route                  │
│  ISP-AS:200  = Peer Route                      │
│  ISP-AS:300  = Transit Route                   │
│  ISP-AS:666  = Blackhole                       │
│  ISP-AS:1000 = Prepend 1x to All Peers         │
│  ISP-AS:2000 = Prepend 2x to All Peers         │
│  ISP-AS:3000 = Prepend 3x to All Peers         │
│  ISP-AS:9000 = Do Not Announce to Specific Peer│
└────────────────────────────────────────────────┘
```

**Blackhole Routing 예제:**
```
! DDoS 대상 prefix를 ISP에게 Blackhole 요청
router bgp 65001
 address-family ipv4 unicast
  neighbor 10.1.1.2 route-map BLACKHOLE out

ip prefix-list DDOS-TARGET seq 5 permit 192.168.1.100/32

route-map BLACKHOLE permit 10
 match ip address prefix-list DDOS-TARGET
 set community 100:666         ← ISP의 Blackhole community
 set ip next-hop 192.0.2.1    ← RFC 5737 documentation address
!
route-map BLACKHOLE permit 20
 ! 나머지 정상 광고
```

### 6.4 Extended Community (RT/SOO)

```
! Route-Target (MPLS L3VPN)
route-target export 65000:100
route-target import 65000:100

! Site-of-Origin (Loop prevention in multi-homed VPN)
route-map SOO-SITE-A permit 10
 set extcommunity soo 65000:1
```

### 6.5 Large Community (RFC 8092)

4바이트 AS 시대를 위한 확장:
```
형식: Global Admin : Local Data 1 : Local Data 2
      (4 bytes)     (4 bytes)      (4 bytes)

! IOS-XE 설정
route-map SET-LARGE-COMM permit 10
 set large-community 4200000001:100:200 additive
```

---

## 7. MP-BGP: Address Family 확장

### 7.1 MP-BGP 개요

```
┌─────────────────────────────────────────────────────┐
│  AFI (Address Family Identifier)                     │
│  ├── 1: IPv4                                        │
│  │   ├── SAFI 1: Unicast                            │
│  │   ├── SAFI 2: Multicast                          │
│  │   ├── SAFI 4: MPLS Label                         │
│  │   ├── SAFI 128: MPLS VPN                         │
│  │   └── SAFI 133: FlowSpec                         │
│  ├── 2: IPv6                                        │
│  │   ├── SAFI 1: Unicast                            │
│  │   └── SAFI 128: MPLS VPN                         │
│  └── 25: L2VPN                                      │
│      └── SAFI 70: EVPN                              │
└─────────────────────────────────────────────────────┘
```

### 7.2 IPv4 + IPv6 Dual-Stack BGP

```
router bgp 65001
 bgp router-id 1.1.1.1
 !
 neighbor 10.1.1.2 remote-as 65002
 neighbor 2001:db8::2 remote-as 65002
 !
 address-family ipv4 unicast
  neighbor 10.1.1.2 activate
  no neighbor 2001:db8::2 activate    ← IPv6 peer에서 IPv4 비활성화
  network 192.168.1.0 mask 255.255.255.0
 !
 address-family ipv6 unicast
  no neighbor 10.1.1.2 activate       ← IPv4 peer에서 IPv6 비활성화
  neighbor 2001:db8::2 activate
  network 2001:db8:1::/48
```

### 7.3 MPLS L3VPN (VPNv4)

**토폴로지:**
```
CE1 ─── PE1 ═══ P ═══ PE2 ─── CE2
         │    MPLS Core    │
       VRF-A             VRF-A
```

**PE1 설정:**
```
! VRF 정의
vrf definition CUSTOMER-A
 rd 65000:100
 address-family ipv4
  route-target export 65000:100
  route-target import 65000:100

! CE-facing 인터페이스
interface GigabitEthernet0/1
 vrf forwarding CUSTOMER-A
 ip address 10.10.1.1 255.255.255.252

! MP-BGP VPNv4 (PE-PE)
router bgp 65000
 neighbor 2.2.2.2 remote-as 65000
 neighbor 2.2.2.2 update-source Loopback0
 !
 address-family vpnv4 unicast
  neighbor 2.2.2.2 activate
  neighbor 2.2.2.2 send-community extended   ← RT 전달 필수!
 !
 ! CE-PE BGP (VRF context)
 address-family ipv4 vrf CUSTOMER-A
  neighbor 10.10.1.2 remote-as 65501
  neighbor 10.10.1.2 activate
```

**검증:**
```
show bgp vpnv4 unicast all
show bgp vpnv4 unicast vrf CUSTOMER-A
show ip route vrf CUSTOMER-A
show mpls forwarding-table
```

---

## 8. BGP EVPN / VXLAN 연동

### 8.1 EVPN Route Types

```
┌───────┬─────────────────────────────────────────────────┐
│ Type  │ 용도                                             │
├───────┼─────────────────────────────────────────────────┤
│   1   │ Ethernet Auto-Discovery (Multi-homing)           │
│   2   │ MAC/IP Advertisement (MAC+IP 학습/광고)           │
│   3   │ Inclusive Multicast (BUM traffic 처리)            │
│   4   │ Ethernet Segment (ES 선출)                       │
│   5   │ IP Prefix (L3 routing, Inter-subnet)             │
└───────┴─────────────────────────────────────────────────┘
```

### 8.2 VXLAN + BGP EVPN 설정 (Nexus 9K)

**Spine-Leaf 토폴로지:**
```
          Spine1          Spine2
         (BGP RR)        (BGP RR)
          / | \           / | \
         /  |  \         /  |  \
      Leaf1 Leaf2 Leaf3
      VTEP  VTEP  VTEP
```

**Leaf1 (VTEP) 설정 (NX-OS):**
```
! Feature 활성화
feature bgp
feature nv overlay
feature vn-segment-vlan-based
nv overlay evpn

! VXLAN Tunnel
interface nve1
 no shutdown
 source-interface loopback1
 host-reachability protocol bgp
 member vni 10010
  suppress-arp
  ingress-replication protocol bgp
 member vni 10020
  suppress-arp
  ingress-replication protocol bgp
 member vni 50000 associate-vrf

! VLAN to VNI 매핑
vlan 10
 vn-segment 10010
vlan 20
 vn-segment 10020

! L2 VNI 설정
evpn
 vni 10010 l2
  rd auto
  route-target import auto
  route-target export auto
 vni 10020 l2
  rd auto
  route-target import auto
  route-target export auto

! L3 VNI (Inter-VXLAN Routing)
vrf context TENANT-A
 vni 50000
 rd auto
 address-family ipv4 unicast
  route-target import auto
  route-target import auto evpn
  route-target export auto
  route-target export auto evpn

! Anycast Gateway
fabric forwarding anycast-gateway-mac 0000.2222.3333

interface vlan 10
 vrf member TENANT-A
 ip address 10.10.10.1/24
 fabric forwarding mode anycast-gateway

interface vlan 20
 vrf member TENANT-A
 ip address 10.10.20.1/24
 fabric forwarding mode anycast-gateway

! BGP EVPN (to Spine/RR)
router bgp 65000
 neighbor 10.255.0.1 remote-as 65000
  update-source loopback0
  address-family l2vpn evpn
   send-community extended
 neighbor 10.255.0.2 remote-as 65000
  update-source loopback0
  address-family l2vpn evpn
   send-community extended
```

**Spine1 (Route Reflector) 설정:**
```
router bgp 65000
 ! Leaf peers
 neighbor 10.255.1.1 remote-as 65000
  update-source loopback0
  address-family l2vpn evpn
   send-community extended
   route-reflector-client
 neighbor 10.255.1.2 remote-as 65000
  update-source loopback0
  address-family l2vpn evpn
   send-community extended
   route-reflector-client
 neighbor 10.255.1.3 remote-as 65000
  update-source loopback0
  address-family l2vpn evpn
   send-community extended
   route-reflector-client
```

**검증 명령어:**
```
show bgp l2vpn evpn summary
show bgp l2vpn evpn
show bgp l2vpn evpn route-type 2
show nve peers
show nve vni
show l2route evpn mac all
show l2route evpn mac-ip all
```

### 8.3 EVPN Multihoming

```
! Leaf1 & Leaf2: Dual-homed ESI 설정
interface port-channel 10
 description Dual-Homed-Server
 switchport mode trunk
 ethernet-segment
  system-mac 0011.2233.4455       ← 양쪽 Leaf에서 동일!
  df-election mode preference     ← DF 선출 방식

! 결과: Type-1, Type-4 Route 자동 생성
```

---

## 9. BGP in SD-WAN & Segment Routing

### 9.1 Cisco SD-WAN + BGP 연동

```
! vEdge/cEdge에서 Service-Side BGP
router bgp 65001
 address-family ipv4 unicast
  redistribute omp            ← OMP → BGP redistribution
 neighbor 10.10.1.2 remote-as 65500
  address-family ipv4 unicast
   ! Service-Side에서 학습한 BGP를 OMP로 전달

! vSmart Policy로 BGP Route 제어
policy
 control-policy BGP-CONTROL
  sequence 10
   match route
    omp-tag 100
   action accept
    set preference 200
```

### 9.2 BGP-SR (Segment Routing with BGP)

```
! IOS-XR: BGP with Segment Routing MPLS
router bgp 65000
 address-family ipv4 unicast
  network 1.1.1.1/32
  allocate-label all            ← BGP Prefix SID 할당
 !
 neighbor 2.2.2.2
  remote-as 65000
  update-source Loopback0
  address-family ipv4 labeled-unicast    ← Labeled Unicast
   route-reflector-client
```

### 9.3 BGP-LS (Link-State Distribution)

IGP 토폴로지 정보를 BGP를 통해 SDN Controller에게 전달:

```
! IOS-XR
router bgp 65000
 address-family link-state link-state
 !
 neighbor 10.10.10.1                   ← SDN Controller
  remote-as 65000
  address-family link-state link-state
```

---

## 10. BGP Convergence & Performance Tuning

### 10.1 Timer 최적화

```
router bgp 65001
 ! Keepalive / Hold Timer (기본: 60/180)
 ! eBGP 빠른 수렴
 neighbor 10.1.1.2 timers 10 30
 
 ! BFD 연동 (ms 단위 감지)
 neighbor 10.1.1.2 fall-over bfd
 !
 ! BGP Scan Timer (기본 60초) — NH 도달성 확인 주기
 bgp scan-time 15
 !
 ! Advertisement Interval (기본: eBGP 30s, iBGP 0s)
 neighbor 10.1.1.2 advertisement-interval 5
```

### 10.2 BFD (Bidirectional Forwarding Detection) 연동

```
! Interface-Level BFD
interface GigabitEthernet0/0
 bfd interval 300 min_rx 300 multiplier 3
 ! → 300ms × 3 = 900ms 이내 장애 감지

! BGP에서 BFD 활성화
router bgp 65001
 neighbor 10.1.1.2 fall-over bfd

! 검증
show bfd neighbors
show bfd neighbors detail
```

### 10.3 BGP Graceful Restart

```
router bgp 65001
 bgp graceful-restart
 bgp graceful-restart restart-time 120   ← 재시작 대기 시간
 bgp graceful-restart stalepath-time 360 ← Stale 경로 유지 시간

! 검증
show ip bgp neighbors 10.1.1.2 | include Graceful
```

### 10.4 BGP PIC (Prefix Independent Convergence)

장애 시 모든 prefix를 개별 업데이트하지 않고, Next-Hop 단위로 일괄 전환:

```
router bgp 65001
 address-family ipv4 unicast
  bgp additional-paths install
  bgp bestpath prefix-sid-map
 !
 ! PIC Edge
 neighbor 10.1.1.2 additional-paths receive
```

### 10.5 BGP Add-Path

RR이 Best Path 외에 추가 경로도 전파:

```
! Route Reflector 설정
router bgp 65000
 address-family ipv4 unicast
  bgp additional-paths select all    ← 또는 best 2 등
  neighbor 3.3.3.3 advertise additional-paths all
```

---

## 11. BGP Security

### 11.1 RPKI (Resource Public Key Infrastructure)

```
! IOS-XE RPKI Configuration
router bgp 65001
 bgp rpki server tcp 10.10.10.1 port 8282 refresh 600
 !
 address-family ipv4 unicast
  ! RPKI 검증 결과에 따른 정책
  neighbor 10.1.1.2 route-map RPKI-POLICY in

route-map RPKI-POLICY permit 10
 match rpki valid
 set local-preference 200

route-map RPKI-POLICY permit 20
 match rpki not-found
 set local-preference 100

route-map RPKI-POLICY deny 30
 match rpki invalid            ← Invalid prefix 차단!

! 검증
show ip bgp rpki table
show ip bgp rpki server
```

### 11.2 BGP Flowspec (DDoS Mitigation)

```
! IOS-XR Flowspec Configuration
router bgp 65000
 address-family ipv4 flowspec
 !
 neighbor 10.10.10.1
  remote-as 65000
  address-family ipv4 flowspec
   route-policy PASS-ALL in

! Flowspec Rule Injection
flowspec
 address-family ipv4
  flow BLOCK-DDOS
   match destination-address ipv4 192.168.1.100/32
   match protocol udp
   match destination-port 53
   action traffic-rate 0       ← Drop
```

### 11.3 GTSM (Generalized TTL Security Mechanism)

```
! TTL이 254 미만인 BGP 패킷 차단 (1-hop neighbor만 허용)
router bgp 65001
 neighbor 10.1.1.2 ttl-security hops 1
```

### 11.4 BGP 보안 Best Practices 체크리스트

```
✅ MD5 Authentication 또는 TCP-AO
✅ GTSM (ttl-security)
✅ Maximum-Prefix 제한
✅ Prefix-List (Inbound/Outbound)
✅ Bogon Prefix 필터링
✅ RPKI Validation
✅ AS-Path 필터링 (Customer cone)
✅ Community 기반 Blackhole 준비
✅ BFD for 빠른 장애 감지
✅ Logging & Monitoring (bgp log-neighbor-changes)
```

---

## 12. Troubleshooting Playbook

### 12.1 Neighbor가 Established 안 될 때

```
Step 1: TCP 연결 확인
  show tcp brief | include 179
  → 세션이 없으면 L3 reachability 또는 ACL 문제

Step 2: BGP 상태 확인
  show ip bgp summary
  → State가 "Active"면 TCP 연결 실패
  → State가 "Idle (Admin)"이면 neighbor shutdown

Step 3: 상세 neighbor 확인
  show ip bgp neighbors 10.1.1.2
  → "Last Error" 메시지 확인
  → "BGP state = Active" → TCP 연결 불가
  → "Notification received: OPEN Message Error" → 설정 불일치

Step 4: Debug (주의: production에서 신중히)
  debug ip bgp 10.1.1.2 events
  debug ip bgp 10.1.1.2 updates
```

**주요 원인별 해결:**

| 증상 | 원인 | 해결 |
|------|------|------|
| Stuck in Active | TCP 179 도달 불가 | ACL, 라우팅, MTU 확인 |
| OpenSent → Active 반복 | AS 번호 불일치 | remote-as 확인 |
| Notification: Hold Timer Expired | Keepalive 미수신 | 경로, CPU, timer 확인 |
| Notification: Cease | Maximum-prefix 초과 | maximum-prefix 조정 |
| Idle (PfxCt) | Prefix 한도 초과 후 | clear ip bgp X.X.X.X |

### 12.2 경로가 BGP Table에 없을 때

```
Step 1: Neighbor에서 받았는지 확인
  show ip bgp neighbors 10.1.1.2 received-routes
  ! ⚠️ soft-reconfiguration inbound 필요
  
  ! 또는 (Adj-RIB-In)
  show ip bgp neighbors 10.1.1.2 routes

Step 2: 필터링 확인
  show ip bgp neighbors 10.1.1.2 | section route-map|prefix-list|filter-list

Step 3: 해당 prefix 상세 확인
  show ip bgp 192.168.1.0/24
  show ip bgp 192.168.1.0/24 bestpath

Step 4: Network 문의 확인
  show ip bgp | include 192.168.1
  ! 'r' = RIB-failure (같은 경로가 더 낮은 AD로 존재)
```

### 12.3 경로가 RIB에 설치 안 될 때

```
! RIB-Failure 확인
show ip bgp rib-failure

! 일반적인 원인:
! 1. 같은 prefix가 Static/OSPF 등으로 더 낮은 AD로 존재
! 2. Next-Hop 도달 불가
! 3. Maximum-paths 초과

! Next-Hop 확인
show ip bgp 192.168.1.0/24
  → Next Hop: 10.1.1.2 (확인: show ip route 10.1.1.2)
  → "inaccessible"이면 next-hop-self 누락 또는 IGP 문제
```

### 12.4 트래픽이 비대칭일 때

```
! Outbound 경로 확인 (우리가 선택한 경로)
show ip bgp 0.0.0.0/0
  → Local Preference, Weight 확인

! Inbound 경로 확인 (상대가 선택한 경로)
! → 직접 확인 불가! 상대 AS에서 확인 필요
! → Looking Glass 활용 (lg.he.net 등)

! 우리가 제어 가능한 Inbound 영향:
! 1. AS-Path Prepending
! 2. MED 설정
! 3. Community를 통한 ISP 정책 요청
```

### 12.5 BGP 메모리 & CPU 이슈

```
! BGP 메모리 사용량
show ip bgp summary
show ip bgp statistics

! 프로세스 CPU
show processes cpu | include BGP

! BGP Table 크기 최적화
! 1. Soft Reconfiguration 비활성화 (메모리 절약)
no neighbor X.X.X.X soft-reconfiguration inbound
! 대신 Route Refresh capability 사용 (RFC 2918)

! 2. ORF (Outbound Route Filtering)
neighbor X.X.X.X capability orf prefix-list both

! 3. Selective Address-Family Activation
address-family ipv4 unicast
 no neighbor X.X.X.X activate    ← 불필요한 AF 비활성화
```

### 12.6 실전 Debug 명령어 (주의해서 사용)

```
! ⚠️ Production에서는 반드시 특정 neighbor로 제한

! Update 추적
debug ip bgp updates 10.1.1.2 in
debug ip bgp updates 10.1.1.2 out

! Keepalive/Open 메시지
debug ip bgp 10.1.1.2 events

! IOS-XR 스타일
debug bgp update neighbor 10.1.1.2 in
debug bgp update neighbor 10.1.1.2 out

! Conditional Debug (IOS-XE)
debug ip bgp updates 10.1.1.2 in prefix-filter
ip prefix-list prefix-filter seq 5 permit 192.168.1.0/24

! Debug 끄기
undebug all
```

---

## Appendix A: BGP Quick Reference Commands

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                   상태 확인
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
show ip bgp summary                    # Neighbor 상태 요약
show ip bgp                            # BGP Table 전체
show ip bgp 10.0.0.0/24               # 특정 prefix 상세
show ip bgp neighbors X.X.X.X         # Neighbor 상세
show ip bgp community 65001:100       # Community 기반 검색
show ip bgp regexp _65001$            # AS-Path 정규식 검색
show ip bgp rib-failure                # RIB 설치 실패 경로
show ip bgp dampening dampened-paths   # Dampening 적용 경로

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                   운영 명령
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
clear ip bgp * soft                    # 전체 Soft Reset
clear ip bgp X.X.X.X soft in          # 특정 peer Inbound Soft Reset
clear ip bgp X.X.X.X soft out         # 특정 peer Outbound Soft Reset
clear ip bgp X.X.X.X                  # Hard Reset (세션 끊김!)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                   IOS-XR 차이점
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
show bgp ipv4 unicast summary
show bgp ipv4 unicast 10.0.0.0/24
show bgp l2vpn evpn summary
show bgp vrf CUSTOMER-A ipv4 unicast
clear bgp ipv4 unicast X.X.X.X soft
```

## Appendix B: BGP Design Decision Matrix

```
┌──────────────────────────────────────────────────────────────┐
│  시나리오                    │  권장 설계                      │
├──────────────────────────────┼────────────────────────────────┤
│  Enterprise Single-homed    │  Default route from ISP        │
│  Enterprise Dual-homed      │  LP + AS-Prepend               │
│  Enterprise Multi-ISP       │  Full table + LP + Community   │
│  ISP Backbone               │  RR + Confederation (대형)     │
│  Data Center Fabric         │  eBGP Spine-Leaf + EVPN       │
│  MPLS Core                  │  iBGP RR + VPNv4/v6           │
│  SD-WAN Overlay             │  OMP + BGP Service-Side       │
│  DCI (Data Center Interconn)│  EVPN Type-5 + BGP-EVPN      │
└──────────────────────────────┴────────────────────────────────┘
```

---

> **문서 버전:** v1.0  
> **최종 수정:** 2026-03-25  
> **참고:** Cisco IOS-XE 17.x, IOS-XR 7.x, NX-OS 10.x 기반
