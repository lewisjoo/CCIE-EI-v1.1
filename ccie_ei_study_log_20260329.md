# CCIE EI v1.1 LAB 학습 정리 — 2026.03.29

---

## 1. 학습 플랜 수립

### 30일 컷 준비 계획
- 환경: EVE-NG (WOLF-LAB 토폴로지)
- 일일 투자: 10.5–11시간 (강화 루틴)
- 약점 도메인: SD-WAN / SD-Access
- Week 1: Network Infrastructure (30%)
- Week 2: SD-Access / SD-WAN (25%)
- Week 3: Transport + Security + Automation (45%)
- Week 4: Full-Lab 모의시험 3회 + 마무리

### 강화 일일 루틴
- 06:30 기상 → 23:30 취침 (수면 7시간)
- 6개 세션: 오전 이론(3.5h) → 오전 LAB(1.5h) → 오후 LAB(3h) → LAB 연장(1.5h) → 야간 약점 보강(2h) → 마무리 정리(1.25h)

---

## 2. IP CEF (Cisco Express Forwarding)

### 핵심 개념
- FIB (Forwarding Information Base): RIB의 미러, longest prefix match
- Adjacency Table: next-hop의 L2 rewrite 정보 (MAC, VLAN 등)
- 한 번의 lookup으로 포워딩 완료 → CPU 부하 없음

### Switching 방식 비교
- Process Switching: 매 패킷 CPU 처리 (가장 느림)
- Fast Switching: 첫 패킷만 CPU, 이후 cache (flow-driven)
- CEF: FIB/Adjacency 사전 구축 (topology-driven, 가장 빠름)

### CEF 모드
- Central CEF: RP에서 포워딩 (단일 CPU)
- dCEF: 라인카드에 FIB 복사, 독립 포워딩 (분산 아키텍처)

---

## 3. Section 1.9 — Redistribution (Variant)

### 문제 핵심
- HQ CE가 SP로부터 학습한 경로를 OSPF로 HQ 내부에 전파

### 조건 및 솔루션

**R11:**
```
ip prefix-list FLT deny 101.22.0.0/30
ip prefix-list FLT permit 0.0.0.0/0 le 32

router ospf 1
 redistribute bgp 65001 subnets metric-type 1
 distribute-list prefix FLT out bgp 65001
```

**R12 (BGP 없음 — connected만):**
```
route-map c2o permit 10
 match interface GigabitEthernet0/0

router ospf 1
 redistribute connected subnets metric-type 1 route-map c2o
```

### 핵심 학습 포인트

| 조건 | 해결 |
|------|------|
| BGP 내부 전파 금지 | OSPF redistribute 사용 |
| 내부 bandwidth 반영 | metric-type 1 (E1) |
| Classful 광고 금지 | subnets 키워드 |
| 101.22.0.0/30 차단 | prefix-list + distribute-list (route-map 금지) |
| R12 ISP connected 광고 | redistribute connected + route-map |

### R12 Connected 광고의 의미
- BGP가 없으므로 경로 학습이 아닌 **링크 도달성(reachability) 광고**
- Connected 광고만으로는 외부 트래픽이 흐르지 않음 (default route 필요)
- 문제에서 요구하지 않으면 default route 설정 안 함

### E1 vs E2 비교
- E2: metric = external metric 고정 (internal cost 무시)
- E1: metric = external + internal OSPF cost (bandwidth 반영)

---

## 4. DMVPN (Dynamic Multipoint VPN)

### Phase 비교

| 항목 | Phase 1 | Phase 2 | Phase 3 |
|------|---------|---------|---------|
| Spoke-to-Spoke 직접 터널 | ❌ | ✅ | ✅ |
| Hub에서 Summary 가능 | ✅ | ❌ | ✅ |
| NHRP Redirect (Hub) | ❌ | ❌ | ✅ |
| NHRP Shortcut (Spoke) | ❌ | ❌ | ✅ |

### Phase 3 핵심 커맨드
- Hub: `ip nhrp redirect`
- Spoke: `ip nhrp shortcut`

### 중요 교훈
- 문제에서 Phase 3를 명시하지 않으면 **임의로 Phase 3 설정하지 않음**
- 기존 설정의 Phase를 유지

---

## 5. Section 1.11 — DMVPN (Branch 3/4)

### 구조
- R24: Hub / R61, R70: Spoke / R62: 제외

### IPsec 설정 검수 (R24 기존 → 변경)

| 항목 | 기존 | 변경 | 이유 |
|------|------|------|------|
| PSK | cisco | CCIEInfr4 | 문제 요구 |
| hash | 미설정 (기본 SHA) | md5 | 문제 요구 |
| mode | tunnel | **유지** | 솔루션에서 변경 요구 없음 |

### R24 Tunnel0 변경사항

| 항목 | 변경 |
|------|------|
| tunnel source | Gi0/4 → GigabitEthernet1 |
| tunnel protection | ipsec profile prof 추가 |
| 나머지 | 전부 유지 (ip mtu 1440 포함) |

### DMVPN IPsec mode tunnel vs transport
- transport가 best practice이지만, 기존 설정이 tunnel이고 문제에서 변경 요구 없으면 **tunnel 유지**
- "preconfigured MTU"와의 정합성 유지

### R61 (Spoke) 변경사항
- tunnel source: Loopback0 → 물리 인터페이스 (조건: loopback 금지)
- NHRP map NBMA 주소: R24 Gi1 IP로 변경
- tunnel vrf ISP 추가 (FVRF)
- ip mtu 1440 추가 (MTU 통일)
- ip nhrp redirect 삭제 (Hub 전용 커맨드)

### FVRF (Front-door VRF) 패턴
- WAN 인터페이스 → VRF (ISP 연결용)
- Tunnel 인터페이스 → Global (내부 라우팅)
- `tunnel vrf <VRF-name>` → 터널의 transport가 VRF를 통해 나감

### `no ip nhrp map multicast dynamic` 처리
- Spoke에서는 Hub 방향 정적 매핑(`ip nhrp map multicast <Hub-IP>`)이 있으면 EIGRP 동작에 영향 없음
- 솔루션에서 빼라고 안 했으면 **빼지 않아도 됨**

---

## 6. Section 1.6 — Intra-Site Routing (BR3)

### EIGRP Named Mode 설정

```
key chain CCIE_MD5
 key 1
  key-string CCIEInfr4

router eigrp ccie
 address-family ipv4 unicast autonomous-system 65006
  af-interface <LAN-facing-intf>
   authentication mode md5
   authentication key-chain CCIE_MD5
  network 10.0.0.0
```

### 핵심 포인트
- `network 10.0.0.0` = `network 10.0.0.0 0.255.255.255` (classful 자동 적용)
- "single command"로 모든 LAN 서브넷 광고

---

## 7. BGP activate 명령어

### 필요 여부

| 상황 | activate 필요 |
|------|--------------|
| IPv4 unicast (기본 상태) | ❌ 자동 |
| `no bgp default ipv4-unicast` 설정됨 | ✅ 필요 |
| VPNv4 / VPNv6 | ✅ 필요 |
| IPv6 unicast | ✅ 필요 |

---

## 8. Section 1.7 — MPLS Underlay (Variant)

### 솔루션 요약

**LDP 설정 (전 장비):**
```
mpls ldp router-id Loopback0 force
```

**OSPF (전 장비):**
```
router ospf 1
 mpls ldp autoconfig
 prefix-suppression

interface <core-facing-intf>
 ip ospf network point-to-point
```

**LDP 인증 — R1, R2 (Global):**
```
mpls ldp password required
mpls ldp password option 1 for LDP_PEERS CCIEInfr4
```

**LDP 인증 — R3, R4, R5, R6 (Per-peer):**
```
mpls ldp neighbor <peer-Lo0-IP> password CCIEInfr4
```

### OSPF Type-2 LSA 제거
- `ip ospf network point-to-point` → DR/BDR 선출 없음 → Type-2 LSA 제거

### OSPF prefix-suppression
- Transit 링크의 stub network(서브넷 정보)를 LSA에서 제거
- Loopback은 stub link이므로 **prefix-suppression 영향 받지 않음**
- `ip ospf prefix-suppression disable` 불필요

### prefix-suppression 적용 이유
1. **보안**: SP 코어 인프라 IP 노출 방지
2. **LSDB/라우팅 테이블 축소**: SPF 계산 속도 향상
3. **안정성**: 링크 flap 시 라우팅 테이블 변경 최소화
4. **SP 코어에서 transit 서브넷 불필요**: 모든 통신이 Loopback 기반

---

## 9. Section 1.8 — MPLS Overlay (Variant)

### 구조
- R3, R4, R5, R6: PE (iBGP VPNv4 full-mesh, RR 금지)
- R1, R2: P (BGP 없음, label switching만)

### 핵심 설정
- VRF FABD2, RT 10000:1
- R3, R6: 5개 active BGP peer
- R3: `maximum-prefix 100000 90 restart 5` (90% warning, 100K shutdown, 5분 재수립)
- 전체 BGP: `password CCIEInfr4`
- VPNv4: `send-community extended` 필수

---

## 10. BGP next-hop-self

### 필요 조건
- eBGP에서 학습한 경로를 iBGP로 전달할 때 필요

### R21에 next-hop-self가 없는 이유
- R21이 BGP 경로를 **OSPF로 재배포** (`redistribute bgp 65002 subnets metric-type 1`)
- R22는 BGP가 아닌 OSPF E1으로 경로를 학습
- OSPF 재배포 시 next-hop이 R21 자신의 IP가 되므로 next-hop 도달성 문제 없음

---

## 11. Section 2.1 — SD-WAN DC Underlay Routing

### 핵심 조건
- DC SD-WAN 라우터: vManage managed 유지, device template 유지
- 새 template 이름에 "CCIE" 포함
- 추가 static route 금지
- OSPF로 underlay 연결성 확보

### cEdge21/22 Template 구조

```
VPN 0 (Transport): Gi2 (default), Gi3 (private1)
VPN 999 (Management): Gi2.3999, Gi3.3999
System: system-ip, site-id 65002
```

### SD-WAN Color
- default: 인터넷/공용 WAN
- private1: 사설 MPLS/전용선
- 같은 color끼리만 터널 형성

### VPN 0에서 redistribute OMP 금지
- VPN 0 = underlay 전용, OMP = overlay
- overlay 경로가 underlay에 유출되면 분리 원칙 위반
- Service VPN(VPN 1+)에서만 OMP 재배포 가능

### EXSTART 문제 해결
- 원인: cEdge21(MTU 1496) vs cEdge22(MTU 1500) 불일치
- OSPF DBD 교환 시 MTU 검사 실패 → EXSTART에서 멈춤
- 해결: MTU 통일 (문제에서 underlay 연결성 요구 → MTU 수정은 요구사항에 포함)

### vManage Template 수정 절차
1. Configuration → Templates → Device Templates → Edit
2. 해당 VPN의 OSPF Feature Template 확인/수정
3. 새 template 생성 시 이름에 CCIE 포함
4. Detach 하지 않고 in-place Edit → Update → Push

### Template Locked 에러 해결
- 로그아웃 → 재로그인 (가장 빠른 해결)

---

## 12. CCIE LAB 철칙 (오늘 반복 학습)

### 가장 중요한 원칙

> **"문제가 요구한 것만 한다. 요구하지 않은 것은 절대 하지 않는다."**

### 오늘 적용한 사례

| 상황 | 판단 |
|------|------|
| R12에 BGP 임의 설정 | ❌ 문제가 요구 안 함 |
| R12에 default route 설정 | ❌ 문제가 요구 안 함 |
| DMVPN Phase 3 임의 적용 | ❌ 문제가 Phase 3 언급 안 함 |
| IPsec mode tunnel → transport 변경 | ❌ 솔루션에서 변경 요구 없음 |
| ip nhrp redirect 임의 추가 | ❌ 문제가 요구 안 함 |
| R21 next-hop-self 임의 추가 | ❌ 솔루션에서 요구 안 함 |
| OSPF prefix-suppression disable on Loopback | ❌ 불필요 (Loopback은 영향 안 받음) |
| no ip nhrp map multicast dynamic 제거 | ❌ 솔루션에서 요구 안 함 |

---

> **인과법칙 (因果法則)**: 오늘 하루의 정확한 학습이 합격이라는 결과를 만든다.
