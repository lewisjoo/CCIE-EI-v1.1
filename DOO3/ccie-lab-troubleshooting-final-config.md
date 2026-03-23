# CCIE DOO Lab — 최종 적용 컨피그 정리

> Enterprise Network Troubleshooting & Optimization  
> 작성일: 2026-03-21 | 카테고리: CCIE / BGP / OSPF / SD-WAN

---

## 목차

1. [트러블슈팅 개요](#1-트러블슈팅-개요)
2. [R21 — Datacenter CE](#2-r21--datacenter-ce)
3. [R11 — HQ CE](#3-r11--hq-ce)
4. [R12 — HQ CE](#4-r12--hq-ce)
5. [R61 — Branch#3 CE](#5-r61--branch3-ce)
6. [R62 — Branch#3 CE (이중화)](#6-r62--branch3-ce-이중화)
7. [R70 — Branch#4 CE (VRF WAN)](#7-r70--branch4-ce-vrf-wan)
8. [cEdge21 / cEdge22 — DC SD-WAN Edge](#8-cedge21--cedge22--dc-sd-wan-edge)
9. [트러블슈팅 히스토리 요약](#9-트러블슈팅-히스토리-요약)
10. [핵심 교훈 (Key Takeaways)](#10-핵심-교훈-key-takeaways)

---

## 1. 트러블슈팅 개요

Multi-AS, Multi-Technology 토폴로지에서 발생한 라우팅 문제를 단계적으로 해결한 최종 변경분 정리.

### 토폴로지 구성

```
                    ┌──────────┐
                    │  SP#1    │  AS 10000
                    │ (Global) │
                    └────┬─────┘
           ┌─────────────┼──────────────┐
           │             │              │
      ┌────▼───┐    ┌────▼───┐    ┌─────▼────┐
      │  R11   │    │  R21   │    │  R61/R62 │
      │  HQ CE │    │  DC CE │    │ Branch#3 │
      └────────┘    └────┬───┘    └──────────┘
                         │
                    ┌────▼───────┐
                    │ cEdge21/22 │  SD-WAN
                    └────┬───────┘
                         │ OMP Overlay
              ┌──────────┼──────────┐
              │                     │
         ┌────▼───┐           ┌────▼───┐
         │ SW400  │           │ SW601  │
         │Branch#1│           │Branch#2│
         └────────┘           └────────┘
```

### 변경 대상 장비: 7대

| 장비 | 역할 | 변경 프로토콜 |
|------|------|-------------|
| R21 | Datacenter CE | BGP |
| R11 | HQ CE | OSPF, BGP |
| R12 | HQ CE | OSPF |
| R61 | Branch#3 CE | BGP |
| R62 | Branch#3 CE | BGP |
| R70 | Branch#4 CE | BGP (VRF WAN) |
| cEdge21/22 | DC SD-WAN Edge | OMP |

---

## 2. R21 — Datacenter CE

**변경 목적:** iBGP next-hop-self 추가, SD-WAN Branch 대역을 SP#1에 BGP 광고

### 문제 1: iBGP next-hop 미해석

R22가 eBGP에서 받은 경로를 iBGP로 R21에 전달할 때 `next-hop-self`가 있었지만, 반대 방향(R21 → R22)에는 누락. R21이 eBGP에서 받은 default(next-hop 100.3.21.1)를 R22에 전달하면, R22는 100.3.21.1을 resolve하지 못하거나 엉뚱한 방향으로 포워딩.

### 문제 2: SD-WAN Branch 대역 미광고

R61에서 `ping 10.4.1.2`(Branch#1) 실패. SP#1이 10.4.x.x, 10.5.x.x 대역으로 가는 경로를 모름. R21에 `redistribute ospf → bgp`가 없었기 때문.

### 최종 컨피그

```
! iBGP next-hop-self
router bgp 65002
 neighbor 10.2.255.22 next-hop-self

! SD-WAN Branch 대역을 SP#1에 광고 (route-map 필터)
ip prefix-list SDWAN_BRANCHES permit 10.4.0.0/16 le 32
ip prefix-list SDWAN_BRANCHES permit 10.5.0.0/16 le 32

route-map OSPF_TO_BGP permit 10
 match ip address prefix-list SDWAN_BRANCHES
 set metric 0

router bgp 65002
 redistribute ospf 1 route-map OSPF_TO_BGP
```

> **`set metric 0` 이유:** cEdge가 OSPF E2 metric 16777214(max-metric)로 광고한 경로가 BGP MED로 전파되면, SP#1이 해당 경로를 비선호. metric을 0으로 리셋하여 정상 선호되도록 조정.

---

## 3. R11 — HQ CE

**변경 목적:** OSPF default route 광고, SD-WAN 대역 재분배 루프 방지

### 문제 1: R22 인터넷 불통

`R22# ping 8.8.8.8` → `!H` (Host Unreachable). R22의 downstream 라우터가 default route를 모름. R21이 eBGP에서 받은 default를 OSPF로 광고하지 않고 있었음.

> **OSPF 특수 규칙:** `redistribute bgp`만으로는 0.0.0.0/0이 OSPF에 광고되지 않음. 반드시 `default-information originate` 명시 필요.

### 문제 2: 라우팅 루프

R21이 SD-WAN 대역을 SP#1에 BGP 광고 → SP#1이 R11에도 전달 → R11이 OSPF E1으로 재분배 → R21이 OSPF에서 학습(HQ 방향) → 루프.

```
R21 ──BGP──▶ SP#1 ──BGP──▶ R11 ──OSPF E1──▶ R21  🔄 LOOP
```

### 최종 컨피그

```
! OSPF default route 광고
router ospf 1
 default-information originate metric-type 1

! prefix-list에 SD-WAN 대역 deny 추가 (루프 방지)
ip prefix-list DENY-R22 deny 101.22.0.0/30
ip prefix-list DENY-R22 deny 10.4.0.0/16 le 32
ip prefix-list DENY-R22 deny 10.5.0.0/16 le 32
ip prefix-list DENY-R22 permit 0.0.0.0/0 le 32
```

> **루프 방지 원리:** R11이 SP#1에서 받은 10.4.x.x, 10.5.x.x를 OSPF로 재분배하지 않으므로, R21은 DC cEdge 방향의 OSPF E2 경로만 사용하게 됨.

---

## 4. R12 — HQ CE

**변경 목적:** ISP connected interface를 OSPF에 재분배

### 최종 컨피그

```
route-map CONN permit 10
 match interface GigabitEthernet0/0

router ospf 1
 redistribute connected route-map CONN subnets metric-type 1
```

---

## 5. R61 — Branch#3 CE

**변경 목적:** 내부 대역 /16 summary를 SP#1에 BGP 광고

### 문제

R61의 실제 connected 서브넷은 10.6.10.0/30, 10.6.12.0/30 등 개별 네트워크. BGP `network 10.6.0.0 mask 255.255.0.0`만 넣으면, RIB에 정확히 일치하는 엔트리가 없어서 BGP가 광고하지 않음.

> **해결:** `ip route 10.6.0.0 255.255.0.0 Null0` 스태틱 루트를 추가하여 RIB에 /16 엔트리를 생성 → BGP network 문이 매칭되어 광고.

### 최종 컨피그

```
ip route 10.6.0.0 255.255.0.0 Null0

router bgp 65006
 network 10.6.0.0 mask 255.255.0.0
```

---

## 6. R62 — Branch#3 CE (이중화)

**변경 목적:** R61과 동일한 대역을 이중화 광고. R61 다운 시에도 SP#1이 Branch#3 대역을 유지.

### 최종 컨피그

```
ip route 10.6.0.0 255.255.0.0 Null0

router bgp 65006
 network 10.6.0.0 mask 255.255.0.0
```

---

## 7. R70 — Branch#4 CE (VRF WAN)

**변경 목적:** VRF WAN context에서 내부 대역을 BGP 광고

### 최종 컨피그

```
ip route vrf WAN 10.7.0.0 255.255.0.0 Null0

router bgp 65007
 address-family ipv4 vrf WAN
  network 10.7.0.0 mask 255.255.0.0
```

> **VRF 주의:** static route에도 `vrf WAN`을 명시해야 VRF RIB에 엔트리가 생성됨. `address-family ipv4 vrf WAN` 내에서 network 문 설정.

---

## 8. cEdge21 / cEdge22 — DC SD-WAN Edge

**변경 목적:** OSPF external 경로를 OMP로 전파하여 SD-WAN Branch 간 통신 활성화

### 문제

R21까지 Branch#3, #4 대역이 OSPF E1으로 들어왔고, cEdge까지 OSPF로 도달했지만 OMP로 전파되지 않음. `advertise ospf external` 누락.

### 최종 컨피그

```
sdwan
 omp
  address-family ipv4
   advertise connected
   advertise static
   advertise ospf external    ← 추가
```

> **효과:** OSPF E1으로 학습한 Branch#3(10.6.0.0/16), Branch#4(10.7.0.0/16) 대역이 OMP → vSmart → 모든 SD-WAN Branch로 전파. SW400에서 10.6.0.0/16, 10.7.0.0/16 확인 완료.

---

## 9. 트러블슈팅 히스토리 요약

### 해결 순서

| 순서 | 문제 | 원인 | 해결 장비 |
|------|------|------|----------|
| 1 | R22 인터넷 불통 (`ping 8.8.8.8` 실패) | R21 iBGP next-hop-self 누락 + OSPF default 미광고 | R21, R11 |
| 2 | R12 ISP facing 경로 미광고 | OSPF redistribute connected 누락 | R12 |
| 3 | Branch#3, #4 내부 대역 미광고 | BGP network 문 + Null0 static 누락 | R61, R62, R70 |
| 4 | SW400에서 Branch#3, #4 경로 미학습 | cEdge OMP `advertise ospf external` 누락 | cEdge21, cEdge22 |
| 5 | R61 → 10.4.1.2 라우팅 루프 | R11이 SD-WAN 대역을 OSPF로 재분배하여 루프 | R11 (prefix-list deny) |
| 6 | SP#1이 SD-WAN 대역 비선호 | OSPF max-metric이 BGP MED로 전파 | R21 (set metric 0) |

### 최종 검증 결과

```
R61# traceroute 10.4.1.2
  1  100.5.61.1        ← SP#1
  2  100.0.15.1        ← SP Core (MPLS)
  3  100.3.21.1        ← SP#1 → R21
  4  100.3.21.2        ← R21
  5  10.2.x.x          ← DC 내부
  6  cEdge             ← SD-WAN Overlay
  7  10.4.1.2  ✅      ← SW400 (Branch#1)
```

---

## 10. 핵심 교훈 (Key Takeaways)

### 1. OSPF default route는 특별하다

`redistribute bgp`만으로 0.0.0.0/0이 OSPF에 광고되지 않음. 반드시 `default-information originate` 필요. OSPF의 고유 설계 규칙.

### 2. iBGP next-hop-self 양방향 확인

eBGP에서 받은 경로를 iBGP로 전달할 때, eBGP next-hop이 iBGP 피어에서 resolve 안 되면 블랙홀 발생. **양쪽 모두** next-hop-self 필요 여부 확인.

### 3. SD-WAN OMP advertise 명시 필요

connected, static 외에 OSPF 경로를 OMP로 전파하려면 `advertise ospf external` 명시 필요. 기본적으로 OSPF external은 OMP에 포함되지 않음.

### 4. 재분배 루프 주의

BGP ↔ OSPF 양방향 재분배 시 route-map / prefix-list로 반드시 필터링. 특히 여러 CE가 같은 SP에 연결된 구조에서 루프 발생 가능성 높음.

### 5. MED(metric) 영향도 파악

OSPF max-metric(16777214)이 BGP MED로 전파되면 SP에서 비선호 경로가 됨. route-map에서 `set metric 0`으로 리셋 필요.

### 6. BGP network 문 + Null0 static 패턴

Summary prefix를 BGP로 광고할 때는 RIB에 정확히 일치하는 엔트리 필요. `ip route x.x.x.x mask Null0`으로 RIB 엔트리 생성 → BGP network 문 매칭.

### 7. CCIE DOO 시험 전략

- **우선순위:** 명시된 문제 먼저, Optimize는 시간 여유가 있을 때
- **최소 변경 원칙:** 문제 scope 내에서 필요한 설정만 변경
- **시간 관리:** Branch 간 통신 같은 추가 최적화는 시간 배분 고려

---

## Reference

- CCIE Enterprise Infrastructure v1.1 Lab Blueprint
- Cisco SD-WAN OMP Configuration Guide
- BGP Best Path Selection Algorithm
- OSPF External Route Redistribution Rules

---

*Tags: `CCIE` `DOO` `BGP` `OSPF` `SD-WAN` `OMP` `next-hop-self` `default-information originate` `redistribute` `route-map` `prefix-list` `VRF` `troubleshooting`*
