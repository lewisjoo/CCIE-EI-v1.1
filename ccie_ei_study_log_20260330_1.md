# CCIE SP SD-WAN Lab 학습 정리

> 작성일: 2026-03-30  
> 범위: Section 2 — SD-WAN Configuration Tasks  
> 환경: WOLF-LAB (CML/EVE-NG), vManage 20.9.x, C8000v cEdge

---

## 2.1 SD-WAN DC Underlay Routing

### 문제 요약

- Data Center, Branch #1, Branch #2 간 **Global SP #2 (AS 10001)**를 통한 underlay 연결 확보
- DC SD-WAN 라우터는 vManage managed mode 유지, **동일 Device Template 유지**
- DC SD-WAN 라우터에 **static route 추가 금지**
- SW211에 `101.22.0.0/30` OSPF route 필요

### 제약조건

| 항목 | 내용 |
|------|------|
| DC cEdge 템플릿 | 변경 불가 (same device templates must remain attached) |
| Static route | DC SD-WAN 라우터에 추가 금지 |
| 새 템플릿 | 이름에 CCIE 포함 필수 |

### 솔루션: R22 (Non-SD-WAN Router)에서 BGP ↔ OSPF Mutual Redistribution

DC cEdge 템플릿을 건드리지 않고, SP측 라우터(R22)에서 양방향 재분배로 해결.

#### R22 설정

```
! Connected 중 특정 인터페이스만 OSPF로 허용
route-map c2o permit 10
 match interface GigabitEthernet1
exit

! OSPF: BGP/Connected → OSPF 재분배
router ospf 1
 redistribute bgp 65002 subnets metric-type 1
 redistribute connected subnets metric-type 1 route-map c2o
exit

! BGP: DC 방향 peering + OSPF → BGP 재분배
router bgp 65002
 bgp router-id 10.2.255.22
 neighbor 101.22.0.1 remote-as 10001
 neighbor 10.2.255.21 remote-as 65002
 neighbor 10.2.255.21 update-source Loopback0
 neighbor 10.2.255.21 next-hop-self
 redistribute ospf 1
exit
```

#### 동작 흐름

```
Branch → SP #2 (BGP) → R22 → OSPF redistribute → DC 내부 (SW211)
DC 내부 → OSPF → R22 → BGP redistribute → SP #2 → Branch
```

#### 주요 포인트

- **metric-type 1 (E1)** 사용 → hop마다 metric 증가, 경로 선택에 유리
- **route-map으로 connected redistribute 제한** → 불필요한 서브넷 OSPF 유출 방지
- R22는 vManage managed가 아니므로 route-map/ACL 이름에 CCIE 필요 없음 (CCIE 규칙은 vManage 템플릿에만 적용)

#### 검증 결과 (SW211)

```
SW211# show ip route 101.22.0.0
O E1  101.22.0.0/30 [110/40] via 10.2.22.1, Ethernet0/1
                    [110/40] via 10.2.20.1, Ethernet0/0    ✅

O E1  101.40.0.0/30 (Branch #1 WAN)  ✅
O E1  101.51.0.0/30 (Branch #2 WAN)  ✅
O E1  101.52.0.0/30 (Branch #2 WAN)  ✅
```

### 추가 이슈: OMP → OSPF Redistribute (Service VPN)

#### 문제

- SW211에서 `10.4.0.0/16` (Branch #1 서비스 대역) 미표시
- cEdge21 VPN 999 (Service VPN)에서 OMP로 학습은 됨: `m 10.4.0.0/16`
- 그러나 OSPF로 재분배되지 않아 SW211에 전파 안 됨

#### 에러 사례

```
Failed to update configuration -
vpn-instance{0}/router/ospf/redistribute{omp} :
Configuring redistribute omp not allowed in VPN 0
```

**원인:** VPN 0 (Transport)과 VPN 999 (Service)가 **같은 OSPF Feature Template을 공유**하고 있어서, redistribute OMP 추가 시 VPN 0에도 적용 → VPN 0에서는 OMP redistribute 금지

#### 해결: OSPF Feature Template 분리

| Template | 적용 VPN | redistribute OMP |
|----------|----------|------------------|
| `C4C-OSPF-cEdge_Template` (기존) | VPN 0 (Transport) | ❌ 없음 |
| `CCIE-OSPF-VPN999` (새로 생성) | VPN 999 (Service) | ✅ 포함 |

**절차:**
1. 기존 `C4C-OSPF-cEdge_Template` 복사하여 `CCIE-OSPF-VPN999` 생성
2. Redistribute 섹션에서 **OMP** 추가
3. Device Template에서 VPN 999의 OSPF → `CCIE-OSPF-VPN999` 연결
4. VPN 0의 OSPF → 기존 `C4C-OSPF-cEdge_Template` 유지

**핵심 학습:** SD-WAN에서 VPN 0은 underlay transport 전용이므로 OMP redistribute 금지. Service VPN에서만 OMP ↔ OSPF 재분배 가능.

---

## 2.2 Branch #1 Device Template

### 문제 요약

- cEdge40에 새 Device Template `BR1-CCIEDGE40` 적용 (vManage)
- 기존 설정 유지하면서 여러 요구사항 충족

### 요구사항

| # | 요구사항 | 솔루션 |
|---|----------|--------|
| 1 | system-ip, site-id, hostname 유지 | 현재 running-config 값을 템플릿에 반영 (Device Specific variable) |
| 2 | vManage 통신 유지 | VPN 0 설정을 정확히 미러링 (tunnel interface, color, vBond) |
| 3 | 10.4.0.0/16 summary만 DC core에 광고 | OMP aggregate (Branch 측에서 설정) |
| 4 | DC에서 summarization 하지 말 것 | Branch(cEdge40)에서만 처리 |
| 5 | cEdge40 ↔ service-side switch OSPF 교환 | Service VPN OSPF 활성화 |
| 6 | Serial console 우선 | `platform console serial` |
| 7 | 새 템플릿에 CCIE 포함 | 명명 규칙 준수 |

### OMP Route Summarization (Branch 측)

vManage OMP Feature Template에서:

```
sdwan
 omp
  address-family ipv4
   advertise aggregate 10.4.0.0/16
```

또는 vManage GUI → OMP Template → IPv4 Advertise → Aggregate: `10.4.0.0/16`

### Serial Console 설정

#### 방법 1: System Feature Template → Advanced → Console → serial

일부 vManage 버전/디바이스 모델에서는 Advanced 섹션에 Console 필드가 없을 수 있음.

#### 방법 2: CLI Add-on Template (권장)

```
platform
 console serial
```

1. Feature Templates → Add Template → CLI Add-on Template
2. Name: `CCIE-Console-Serial`
3. CLI Configuration에 위 명령 입력
4. Device Template에서 Additional CLI Templates에 연결

#### 주의: 가상 라우터 에러

```
Failed to update configuration - platform
syntax error: incomplete path
Error: on line 1: platform
```

**CSR1000v / C8000v 등 가상 라우터에서는 `platform console serial` 미지원.** 물리적 cEdge(ISR/ASR)에서만 유효. 랩 환경에서는 이 설정 스킵.

### Template Scope 개념

| Scope | 설명 | 사용 시기 |
|-------|------|-----------|
| Default | vManage 기본값 | 건드리지 않을 때 |
| Global | 모든 디바이스에 동일 값 | Console = serial 등 |
| Device Specific (Variable) | 디바이스마다 다른 값 | system-ip, hostname 등 |

### 작업 순서

1. cEdge40 `show sdwan running-config` 백업
2. Feature Templates 생성 (모두 CCIE 포함):
   - `CCIE-System`: hostname, system-ip, site-id, console serial
   - `CCIE-VPN0-Transport`: WAN interface, tunnel, color
   - `CCIE-VPN1-Service`: service interface, OSPF
   - `CCIE-OMP`: advertise aggregate 10.4.0.0/16
3. Device Template `BR1-CCIEDGE40` 생성 → Feature Template 연결
4. Attach → Device-specific 값 입력 → Preview → Deploy
5. 검증

### 검증

```
show sdwan control connections                    → vManage reachable
show ip route (DC core switch)                    → 10.4.0.0/16 summary만 존재
show ip ospf neighbor (cEdge40)                   → service-side switch adjacency
show platform | include console                   → serial 확인
```

---

## 2.3 SD-WAN Hub'n'Spoke

### 문제 요약

- Branch #1 VLAN 3999 ↔ Branch #2 VLAN 3999 통신을 **DC 경유만 허용** (Hub-and-Spoke)
- cEdge22 경로 우선
- Branch #1 ↔ Branch #2 간 direct data plane tunnel 금지
- vSmart centralized policy, 다른 vSmart와 sync

### 요구사항 분석

| 요구사항 | 구현 방법 |
|----------|-----------|
| DC 경유만 허용 | Hub-and-Spoke topology (DC = Hub) |
| cEdge22 우선 | TLOC preference 설정 |
| Spoke간 tunnel 금지 | vSmart control policy로 route 차단 |
| vSmart sync | Centralized Policy → 자동 sync |
| VLAN 3999 + DC LAN만 통신 | 해당 VPN에만 정책 적용 |

### 솔루션: vSmart Centralized Control Policy

#### 사전 조건: vSmart를 vManage Managed Mode로 전환

현재 상태 확인:

```
vsmart# show system status
vManaged:                false
Configuration template:  None
Policy template:         None
```

**vSmart가 CLI mode** → Centralized Policy 활성화를 위해 vManage managed로 전환 필요

**전환 절차:**
1. `show running-config`로 현재 설정 백업
2. Feature Templates 생성:
   - `CCIE-vSmart-System`: system-ip, site-id, hostname
   - `CCIE-vSmart-VPN0`: transport 인터페이스
3. Device Template 생성: `CCIE-vSmart-Device`
4. Attach → 기존 설정과 동일하게 Device-specific 값 입력

#### Policy 구성 (vManage GUI)

**1. Policy Lists 생성** (Configuration → Policies → Custom Options → Lists)

- Site List `CCIE-HUB-SITES` → DC site-id
- Site List `CCIE-SPOKE-SITES` → Branch #1, #2 site-id
- VPN List `CCIE-VPN-3999` → 해당 VPN 번호
- TLOC List `CCIE-CEDGE22-TLOC` → cEdge22 system-ip, color, encap

**2. Hub-and-Spoke Topology** (Policies → Custom Options → Topology → Add)

- Type: Hub-and-Spoke
- Name: `CCIE-HUB-SPOKE`
- Hub: `CCIE-HUB-SITES`
- Spoke: `CCIE-SPOKE-SITES`
- VPN: `CCIE-VPN-3999`

vManage가 자동으로 Spoke간 route 차단 control policy 생성.

**3. cEdge22 Preferred Path**

Hub TLOC preference로 cEdge22를 우선 설정:

```
lists
 tloc-list CCIE-CEDGE22-TLOC
  tloc <cEdge22-system-ip> color <color> encap ipsec preference 200
```

**4. Activate**

Policies → Centralized Policy → Activate → vSmart에 자동 push

#### 동작 흐름

```
Branch #1 (VLAN 3999)
    ↓ OMP (spoke → hub only)
DC cEdge22 (preferred, preference 200)
    ↓ OMP
Branch #2 (VLAN 3999)
```

- Spoke → Spoke 직접 경로 **reject** → data plane tunnel 미형성
- 모든 VLAN 3999 트래픽이 DC Hub 경유
- cEdge22가 preferred path

#### 검증

```
show sdwan omp routes vpn <3999-vpn>    → Spoke에서 다른 Spoke 경로 미표시
show sdwan bfd sessions                  → Branch #1 ↔ Branch #2 direct tunnel 없음
traceroute (Branch #1 → Branch #2)       → DC 경유 확인, cEdge22 통과
```

---

## 토폴로지 참조

| 사이트 | AS | 대역 | VPN | 역할 |
|--------|-----|------|-----|------|
| Datacenter | 65002 | 10.2.0.0/16 | 999 (Service) | Hub, OSPF Area 0 |
| HeadQuarters | 65001 | 10.1.0.0/16 | - | OSPF Area 0, EIGRP IPv6 |
| Branch #1 | 65004 | 10.4.0.0/16 | - | Spoke, cEdge40 |
| Branch #2 | 65005 | 10.5.0.0/16 | - | Spoke, cEdge51/52 |
| Branch #3 | 65006 | 10.6.0.0/16 | - | Non-SD-WAN |
| Internet SP #2 | 10001 | 101.x.x.x | - | SD-WAN underlay transport |
| Global SP #1 | 10000 | 100.0.0.0/8 | - | BGP, OSPF Area 0 |

### 주요 장비별 연결

```
R22 (AS 65002, DC)
├── Gi0/1: 101.22.0.0/30 → SP #2 (BGP, AS 10001)
├── Gi0/2: 10.2.99.0/30  → R21
├── Gi0/3: 10.2.12.0/30  → cEdge 방향
├── Gi0/4: 10.2.11.0/30  → cEdge 방향
└── Lo0:   10.2.255.22/32

cEdge21 (DC)
├── Gi2:  10.2.201.0/29  → SW201 방향
├── Gi3:  10.2.202.0/29  → SW202 방향
└── VPN 999 (Service): OSPF + OMP

SW211 (DC Core)
├── E0/0: 10.2.20.0/30
├── E0/1: 10.2.22.0/30
├── E0/2: 10.2.119.0/30
└── Lo0:  10.2.255.211/32
```

---

## 핵심 학습 포인트 정리

### 1. SD-WAN VPN 구조와 OSPF Redistribute 제약

- **VPN 0 (Transport):** underlay 전용, OMP redistribute **금지**
- **VPN 512 (Management):** OOB 관리
- **Service VPN (1~511, 513~65527):** OMP ↔ OSPF redistribute **허용**
- 같은 OSPF Template을 여러 VPN에서 공유하면 VPN 0 에러 발생 → **Template 분리 필수**

### 2. vManage Template 제약조건 해석

- "same device templates must remain attached" → **Device Template은 유지**, Feature Template 내용 수정/교체는 가능
- "do not configure static routes on DC" → 동적 라우팅으로 해결
- Template이 아닌 CLI 설정(route-map, ACL 등)에는 CCIE 명명 규칙 불적용

### 3. Centralized Policy vs vSmart Template

- Centralized Policy는 **vManage → Policies**에서 생성, vSmart 템플릿과 별개
- vSmart가 **CLI mode**이면 Centralized Policy 활성화 불가 → vManage managed로 전환 필요
- Policy 활성화 시 모든 vSmart에 자동 sync

### 4. 가상 라우터 제한사항

- `platform console serial`: 가상 라우터(CSR1000v, C8000v)에서 미지원
- 실제 CCIE 랩 시험은 물리 장비 사용 → 정상 동작
- 랩 연습 시 해당 에러는 스킵하고 진행

### 5. OMP Route Summarization

- Branch 측에서 `advertise aggregate`로 설정
- DC에서 summarization 설정하지 않음
- summary 경로가 OMP → DC cEdge Service VPN OSPF → DC core switch로 전파
