# Cisco Catalyst 9800 WLC Deep Dive — 튜닝 가이드

> **IOS-XE 17.x 기반 | C9800-40/80/CL/L/EWC | 튜닝 & 최적화 중심**
> 작성일: 2026-03-26

---

## 1. 아키텍처 & 구성 모델 개요

### 1.1 C9800 Configuration Model (Tags & Profiles)

C9800은 AireOS와 완전히 다른 모듈형 구성 모델을 사용한다. 핵심은 **Tags & Profiles** 구조이다.

```
AP
├── Policy Tag ─────── WLAN Profile ↔ Policy Profile (1:1 매핑)
├── Site Tag ────────── AP Join Profile + Flex Profile
└── RF Tag ─────────── 2.4GHz RF Profile + 5GHz RF Profile + 6GHz RF Profile
```

| 구성 요소 | 역할 | 튜닝 포인트 |
|-----------|------|------------|
| **WLAN Profile** | SSID 이름, 보안(WPA2/3, 802.1X, PSK), 11r/k/v | 보안, 로밍 |
| **Policy Profile** | VLAN, AAA Override, QoS, ACL, Central/Local Switching | 트래픽 제어 |
| **RF Profile** | Data Rate, TPC, DCA, RxSOP, Band Select, CHD | **RF 튜닝의 핵심** |
| **AP Join Profile** | CAPWAP, DTLS, SSH, LED, Syslog, CDP/LLDP | 관리 |
| **Flex Profile** | FlexConnect 전용 VLAN, ACL, Local Auth | 분산 배포 |
| **Site Tag** | AP 그룹핑, 로밍 도메인, WNCd 프로세스 분배 | 스케일 |

### 1.2 튜닝 설계 원칙

> **사이트별 고유 RF Tag → 사이트별 고유 RF Profile**

RF Profile을 사이트 간 공유하면, 한 사이트의 변경이 다른 사이트에 영향을 미친다. 반드시 사이트별로 분리한다.

```
RF Tag: Seoul-HQ-RF-Tag
  ├── 2.4GHz Profile: Seoul-HQ-24G
  ├── 5GHz Profile: Seoul-HQ-5G
  └── 6GHz Profile: Seoul-HQ-6G

RF Tag: Busan-BR-RF-Tag
  ├── 2.4GHz Profile: Busan-BR-24G
  ├── 5GHz Profile: Busan-BR-5G
  └── 6GHz Profile: Busan-BR-6G
```

---

## 2. RRM (Radio Resource Management) 튜닝

### 2.1 RF Group 설정

```
경로: Configuration > Wireless > RF Group
CLI:  wireless rf-network <rf-group-name>
```

| 항목 | 설명 | 튜닝 |
|------|------|------|
| RF Group Name | RF 그룹 식별자. 동일 이름의 AP만 neighbor 관계 형성 | 모든 WLC에 동일하게 설정 |
| Grouping Mode | Auto / Static | Auto 권장 (자동 Leader 선출) |
| RF Group Leader | 가장 높은 capacity WLC가 자동 선출 | 9800-80 > 9800-40 > 9800-L |

> **주의**: RF Group Name ≠ Mobility Group Name. 서로 다른 RF Group Name을 가진 AP는 서로를 Rogue로 인식한다.

**RF Group AP Maximums:**

| Controller | Auto Mode (연결 AP 수 기준) | Static Mode (라이선스 기준) |
|-----------|---------------------------|--------------------------|
| C9800-L | 250 | 250 |
| C9800-40 | 2,000 | 6,000 |
| C9800-80 | 6,000 | 6,000 |
| C9800-CL | 모델 종속 | 모델 종속 |

### 2.2 DCA (Dynamic Channel Assignment) 튜닝

```
경로: Configuration > Radio Configurations > RRM > DCA
CLI:  ap dot11 {24ghz | 5ghz | 6ghz} rrm channel dca ...
```

| 파라미터 | 기본값 | 튜닝 권장 | 설명 |
|----------|--------|-----------|------|
| **Channel Assignment** | Auto (Global) | Auto 유지 | 수동 채널 고정은 특수 환경에서만 |
| **DCA Sensitivity** | Medium (10dB) | Medium 유지 | High=5dB(잦은 변경), Low=20dB(둔감) |
| **DCA Interval** | 10분 | 10분 유지 | Startup Mode: 100분(10회×10분) |
| **Channel Width (5GHz)** | 20MHz | `best` 또는 `40MHz` | HD 환경=20MHz, 일반=40MHz, 저밀도=80MHz |
| **Channel Width (6GHz)** | 20MHz | `best` | DBS(Dynamic Bandwidth Selection) 활용 |
| **Avoid Foreign AP** | Enabled | 환경에 따라 | 다중 테넌트 빌딩에서는 Disable 고려 |
| **Avoid AP Load** | Disabled | Disabled 유지 | 부하 기반 채널 변경은 불안정 유발 |
| **DFS (5GHz)** | Enabled | Enabled | 레이더 탐지 채널 활용 (5.3/5.6GHz) |

**DCA Startup Mode 강제 실행:**

```
ap dot11 5ghz rrm dca restart
! 100분간 High Sensitivity로 10회 DCA 실행 → 최적 채널 플랜 수렴
```

**5GHz 채널 선택 (한국 기준):**

```
ap dot11 5ghz rrm channel dca chan-list 36,40,44,48,52,56,60,64,100,104,108,112,116,120,124,128,132,136,140,144,149,153,157,161
! DFS 채널 포함 → 채널 풀 극대화
```

**DBS (Dynamic Bandwidth Selection):**

```
ap dot11 5ghz rrm channel dca chan-width best
! RRM이 채널 상태에 따라 20/40/80/160MHz 자동 선택
```

### 2.3 TPC (Transmit Power Control) 튜닝

```
경로: Configuration > Radio Configurations > RRM > TPC
CLI:  ap dot11 {24ghz | 5ghz} rrm txpower ...
```

| 파라미터 | 기본값 | 튜닝 권장 | 설명 |
|----------|--------|-----------|------|
| **TPC Algorithm** | TPCv1 | **TPCv2** (5GHz) | TPCv2는 클라이언트 밀도 고려, 더 정밀 |
| **Power Assignment** | Auto (Global) | Auto 유지 | 수동 고정은 특수 환경에서만 |
| **Max Power** | 30 dBm | 환경별 조정 | 실내: 14~17dBm, 고천장: 20dBm |
| **Min Power** | -10 dBm | 환경별 조정 | 너무 낮으면 Coverage Hole 발생 |
| **Power Threshold** | -70 dBm (v1) / -65 dBm (v2) | 환경별 조정 | neighbor AP RSSI 기준 |

> **TPCv1 vs TPCv2**: TPCv1은 neighbor AP 간 RSSI만 고려. TPCv2는 추가로 클라이언트 분포와 채널 상태를 반영하여 더 정밀한 파워 조절.

**실예제 — 고밀도 오피스:**

```
! 5GHz TPC 튜닝
ap dot11 5ghz rrm txpower max 14
ap dot11 5ghz rrm txpower min 5
ap dot11 5ghz rrm txpower threshold -65

! 2.4GHz TPC 튜닝 (커버리지 최소화)
ap dot11 24ghz rrm txpower max 8
ap dot11 24ghz rrm txpower min 2
ap dot11 24ghz rrm txpower threshold -65
```

### 2.4 Coverage Hole Detection (CHD)

```
경로: Configuration > Radio Configurations > RRM > General
```

| 파라미터 | 기본값 | 설명 |
|----------|--------|------|
| Data RSSI Threshold | -80 dBm | 클라이언트 RSSI가 이 값 이하일 때 Coverage Hole 감지 |
| Voice RSSI Threshold | -80 dBm | Voice 클라이언트 기준 |
| Min Failed Client Count | 3 | 이 수 이상의 클라이언트가 threshold 이하일 때 알람 |
| Coverage Exception Level | 25% | 실패 패킷 비율 threshold |

> **주의**: CHD가 TPC max power를 초과하여 파워를 올리는 것은 TPC Max 설정으로 방지된다.

### 2.5 ED-RRM (Event Driven RRM)

```
ap dot11 5ghz rrm channel cleanair-event
ap dot11 5ghz rrm channel cleanair-event rogue-contribution dutycycle 80
```

CleanAir가 심각한 간섭(Jammer, DECT Phone 등)을 감지하면, 다음 DCA 사이클을 기다리지 않고 즉시 채널을 변경한다.

| 설정 | 권장 |
|------|------|
| CleanAir | **Enabled** (항상) |
| ED-RRM | **Enabled** |
| Rogue Contribution | **Enabled** (단, 다중 테넌트 100% 오버랩 건물에서는 Disable) |

### 2.6 FRA (Flexible Radio Assignment)

```
경로: RF Profile > FRA
CLI:  ap dot11 5ghz rrm fra
```

XOR 라디오(2800/3800/9120/9130/9136 등)의 2.4GHz 라디오를 자동으로 5GHz 또는 Monitor 역할로 전환한다.

| 파라미터 | 권장 |
|----------|------|
| FRA | **Enabled** (XOR AP 사용 시) |
| FRA Interval | ≥ DCA Interval |

> FRA 결과를 확인 후 적용하려면: `ap fra revert all` (모든 XOR 라디오를 legacy dual-band로 복원)

---

## 3. RF Profile 튜닝 (Data Rate & Radio)

### 3.1 Data Rate 최적화

```
경로: Configuration > Tags & Profiles > RF > RF Profile > [Profile Name]
```

**2.4GHz Data Rate 튜닝 (가장 중요한 튜닝 항목 중 하나):**

| Rate | 기본값 | 고밀도 권장 | 이유 |
|------|--------|------------|------|
| 1 Mbps | Mandatory | **Disabled** | 저속 레이트 사용 AP → 더 넓은 셀 → CCI 증가 |
| 2 Mbps | Mandatory | **Disabled** | 동일 |
| 5.5 Mbps | Mandatory | **Disabled** | 동일 |
| 6 Mbps | Supported | **Disabled** | 동일 |
| 9 Mbps | Supported | **Disabled** | 동일 |
| 11 Mbps | Mandatory | **Disabled** | 동일 |
| 12 Mbps | Supported | **Mandatory** ← 최소 기본 레이트 | Cell 크기 축소, CCI 감소 |
| 18 Mbps | Supported | Supported | — |
| 24 Mbps | Supported | Supported | — |
| 36 Mbps | Supported | Supported | — |
| 48 Mbps | Supported | Supported | — |
| 54 Mbps | Supported | Supported | — |

> **핵심**: 2.4GHz에서 1, 2, 5.5, 6, 9, 11Mbps를 Disable하고 12Mbps를 Mandatory로 설정하면, AP의 실효 커버리지 셀이 줄어들어 Co-Channel Interference가 감소한다.

```cli
! 2.4GHz Data Rate CLI
ap dot11 24ghz rate RATE_1M disable
ap dot11 24ghz rate RATE_2M disable
ap dot11 24ghz rate RATE_5_5M disable
ap dot11 24ghz rate RATE_6M disable
ap dot11 24ghz rate RATE_9M disable
ap dot11 24ghz rate RATE_11M disable
ap dot11 24ghz rate RATE_12M mandatory
ap dot11 24ghz rate RATE_24M mandatory
```

**5GHz Data Rate 튜닝:**

| Rate | 기본값 | 고밀도 권장 |
|------|--------|------------|
| 6 Mbps | Mandatory | **Disabled** |
| 9 Mbps | Supported | **Disabled** |
| 12 Mbps | Mandatory | **Disabled** 또는 Mandatory |
| 18 Mbps | Supported | Supported |
| 24 Mbps | Mandatory | **Mandatory** ← 최소 기본 레이트 |
| 36 Mbps | Supported | Supported |
| 48 Mbps | Supported | Supported |
| 54 Mbps | Supported | Supported |

### 3.2 Rx-SOP (Receiver Start of Packet) 튜닝

```
경로: RF Profile > Rx-SOP Threshold
CLI:  ap dot11 {24ghz|5ghz} rrm rx-sop threshold {auto|high|medium|low|custom}
```

Rx-SOP는 AP가 프레임을 수신하기 시작하는 최소 신호 강도를 결정한다.

| Level | 2.4GHz Threshold | 5GHz Threshold | 용도 |
|-------|-----------------|----------------|------|
| Auto | -95 dBm | -95 dBm | 기본 |
| High | -79 dBm | -76 dBm | 초고밀도 (스타디움) |
| Medium | -82 dBm | -78 dBm | 고밀도 (컨벤션) |
| Low | -85 dBm | -80 dBm | 중밀도 |
| Custom | 사용자 정의 | 사용자 정의 | — |

> **주의**: Rx-SOP를 너무 높이면 원거리 클라이언트의 프레임을 무시하여 연결 장애 발생 가능. 반드시 site survey 후 적용.

### 3.3 Band Select (Dual-Band → 5GHz 유도)

```
경로: RF Profile > Band Select
CLI:  ap dot11 24ghz rrm bss-selection ...
```

2.4GHz probe response를 지연시켜 dual-band 클라이언트를 5GHz로 유도한다.

| 파라미터 | 기본값 | 권장 |
|----------|--------|------|
| Probe Cycle Count | 2 | 2 |
| Probe Cycle Threshold (ms) | 200 | 200 |
| Age Out Suppression (sec) | 20 | 20 |
| Age Out Dual Band (sec) | 60 | 60 |
| Client RSSI Threshold (dBm) | -80 | -80 |

> **대안**: 802.11v BSS Transition Management를 활성화하면 보다 표준 기반의 band steering이 가능.

### 3.4 802.11ax (Wi-Fi 6/6E) 튜닝

| 파라미터 | 경로 | 권장 |
|----------|------|------|
| OFDMA Downlink | RF Profile > 802.11ax | **Enabled** |
| OFDMA Uplink | RF Profile > 802.11ax | **Enabled** |
| MU-MIMO | RF Profile > 802.11ax | **Enabled** |
| BSS Color | RF Profile > 802.11ax | **Enabled** (CCI 완화) |
| Target Wake Time (TWT) | RF Profile > 802.11ax | Enabled (IoT 배터리 절약) |
| Spatial Reuse (OBSS PD) | RF Profile > 802.11ax | **Enabled** (고밀도 환경) |

---

## 4. WLAN & Security 튜닝

### 4.1 SSID 수 최적화

> **권장: 라디오당 최대 4개 SSID**

각 SSID는 별도의 Beacon과 Probe Response를 생성한다. SSID가 많을수록 RF overhead 증가.

```
! SSID 수 확인
show wlan summary
```

### 4.2 보안 프로토콜 선택

| 환경 | 권장 보안 |
|------|-----------|
| Enterprise (최신 디바이스) | WPA3-Enterprise (192-bit, SAE) |
| Enterprise (레거시 혼재) | WPA2/WPA3 Mixed Mode + 802.1X |
| Guest | WPA2-PSK 또는 Open + CWA (Central Web Auth) |
| IoT | WPA2-PSK 또는 MAB (MAC Auth Bypass) |

**WPA2/WPA3 Mixed Mode CLI:**

```
wlan Corp-WLAN 1 Corp-SSID
  security wpa psk set-key type clear <passphrase>
  security wpa wpa2
  security wpa wpa2 ciphers aes
  security wpa wpa3
  no shutdown
```

### 4.3 Fast Roaming 튜닝 (802.11r / 11k / 11v)

```
경로: Configuration > Tags & Profiles > WLANs > [WLAN] > Security > Fast Transition
```

| 프로토콜 | 기본값 | 권장 | 역할 |
|----------|--------|------|------|
| **802.11r (FT)** | Enabled | **Mixed Mode** (FT + non-FT 공존) | Fast BSS Transition. 로밍 시 사전 PMK 교환 |
| **FT Over-the-DS** | Enabled | **Disabled** (FlexConnect) / Enabled (Local) | DS를 통한 FT. FlexConnect에서는 OTA 권장 |
| **802.11k** | Enabled | **Enabled** | Neighbor List 제공 → 스캔 최적화 |
| **802.11v** | Enabled | **Enabled** | BSS Transition Mgmt, Max Idle Period (배터리 절약) |
| **OKC (PMKID Caching)** | Enabled | **Enabled** | non-11r 클라이언트용 fast roam fallback |
| **CCKM** | Disabled | 환경에 따라 | Cisco 전용. Voice 환경에서 사용 |

**802.11r Mixed Mode (권장):**

```
wlan Corp-WLAN 1 Corp-SSID
  security ft
  security ft over-the-ds       ! Local Mode
  ! 또는
  ! no security ft over-the-ds   ! FlexConnect Mode (OTA only)
  security pmf optional          ! PMF와 FT 호환
  no shutdown
```

> **주의**: 802.11r만 활성화하면 non-FT 레거시 클라이언트가 접속 불가. 반드시 Mixed Mode 또는 Adaptive 사용.

### 4.4 AAA 튜닝

```
경로: Configuration > Security > AAA
```

**RADIUS 서버 Dead Criteria:**

```
radius-server dead-criteria time 5 tries 3
radius-server deadtime 5
! 5초 내 3회 응답 없으면 Dead 처리, 5분간 Dead 유지
```

**AAA Override (Policy Profile):**

```
wireless profile policy Corp-Policy
  aaa-override
  ! RADIUS에서 VLAN, ACL, QoS를 동적으로 할당
```

---

## 5. Policy Profile 튜닝

### 5.1 핵심 설정

```
경로: Configuration > Tags & Profiles > Policy > [Profile Name]
```

| 파라미터 | 경로 | 권장 |
|----------|------|------|
| Central Switching | Policy Profile > Access Policies | Local Mode: Enabled / Flex: Disabled |
| Central Association | Policy Profile > Access Policies | Enabled (항상) |
| Central DHCP | Policy Profile > Access Policies | 환경에 따라 |
| AAA Override | Policy Profile > Access Policies | **Enabled** (ISE 연동 시 필수) |
| VLAN | Policy Profile > Access Policies | 클라이언트 VLAN 지정 |
| Session Timeout | Policy Profile > Advanced | 1800~28800초 (환경별) |
| Idle Timeout | Policy Profile > Advanced | 300초 (기본) |
| AAA Policy | Policy Profile > Mobility | 환경에 따라 |

### 5.2 QoS 튜닝

```
경로: Policy Profile > QoS & AVC
```

| 프로파일 | DSCP/UP | 용도 |
|----------|---------|------|
| Platinum | EF (46) / UP 6 | Voice (VoIP) |
| Gold | AF41 (34) / UP 5 | Video |
| Silver | AF21 (18) / UP 0 | Best Effort (기본) |
| Bronze | CS1 (8) / UP 1 | Background |

**AVC (Application Visibility and Control):**

```
wireless profile policy Corp-Policy
  service-policy input AVC-Policy
  service-policy output AVC-Policy
```

### 5.3 Multicast 최적화

| 파라미터 | 권장 | 설명 |
|----------|------|------|
| Multicast Mode | **Multicast-Multicast** | 유니캐스트 변환 대비 효율적 |
| IGMP Snooping | **Enabled** | 불필요한 멀티캐스트 플러딩 방지 |
| mDNS Gateway | **Enabled** | Apple Bonjour, Chromecast 등 서비스 디스커버리 |

---

## 6. Site Tag & AP 스케일 튜닝

### 6.1 Site Tag 설계

하나의 Site Tag 내의 AP는 동일 WNCd 프로세스에서 관리된다.

| 모델 | WNCd 프로세스 수 | Site Tag당 최대 AP |
|------|----------------|--------------------|
| C9800-L | 1 | 250 |
| C9800-40 | 4 | 500 (4×125 or 2×250 등) |
| C9800-80 | 8 | 750 |
| C9800-CL | 인스턴스별 | 가변 |

> **핵심**: 로밍 도메인 내 AP는 동일 Site Tag에 배치하되, Site Tag당 AP 수가 한계를 초과하지 않도록 분산.

> **17.7.1 이후**: 서로 다른 Site Tag 간에도 802.11k/v가 동작하므로 Site Tag 분할이 더 유연.

### 6.2 FlexConnect Site Tag 설계

FlexConnect에서는 Site Tag = Fast Roaming Domain이다. Key caching과 key distribution이 동일 Flex Site Tag 내에서만 동작.

```
wireless tag site Branch-Seoul
  ap-profile Branch-AP-Join
  flex-profile Branch-Flex
  ! 동일 리모트 사이트의 AP만 이 태그에 할당
```

---

## 7. AP Join & CAPWAP 튜닝

### 7.1 DTLS (Data Encryption)

```
경로: Configuration > Tags & Profiles > AP Join > [Profile]
```

| 파라미터 | 기본값 | 튜닝 |
|----------|--------|------|
| CAPWAP Data Encryption | Disabled | 필요 시만 Enable (CPU 오버헤드 ~20%) |
| DTLS Version | TLS 1.2 | TLS 1.2 유지 |

> **참고**: Data DTLS를 활성화하면 AP↔WLC 간 모든 데이터 트래픽이 암호화된다. 보안 컴플라이언스 요구가 없으면 비활성화가 성능에 유리.

### 7.2 AP 운영 모드

| 모드 | 용도 |
|------|------|
| **Local** | 중앙 집중형. 데이터가 CAPWAP 터널을 통해 WLC로 전송 |
| **FlexConnect** | 분산형. 데이터는 AP에서 로컬 스위칭, 관리는 중앙 |
| **Monitor** | 전용 모니터. CleanAir, wIPS, Rogue Detection |
| **SE-Connect** | Spectrum Expert 연결 전용 |

### 7.3 Install Mode vs Bundle Mode

| 모드 | 권장 | 설명 |
|------|------|------|
| **Install Mode** | **권장** | 패키지 분리 부팅, ISSU(In-Service Software Upgrade) 지원 |
| Bundle Mode | 비권장 | 단일 바이너리 부팅. ISSU 미지원, 롤백 제한 |

```
! Install Mode로 업그레이드
install add file bootflash:C9800-universalk9.17.xx.xx.SPA.bin
install activate
install commit
```

---

## 8. High Availability (HA) 튜닝

### 8.1 SSO (Stateful Switchover)

```
경로: Administration > Device > Redundancy
```

| HA 모드 | 설명 |
|---------|------|
| SSO | Active/Standby. 클라이언트 세션 유지. 권장 |
| N+1 | AP Fallback 기반. Mobility 설정으로 구현 |

### 8.2 AP Fallback

```
wireless profile policy Corp-Policy
  ap fallback
  ! AP가 Primary WLC 복구 시 자동 복귀
```

### 8.3 Mobility (Inter-Controller Roaming)

```
경로: Configuration > Wireless > Mobility > Peer
```

| 항목 | 설명 |
|------|------|
| Mobility Group Name | 동일 그룹 내 WLC 간 seamless roaming |
| Mobility Tunnel | CAPWAP Mobility Tunnel로 클라이언트 컨텍스트 전달 |
| Guest Anchor | Guest SSID → Anchor WLC로 터널링 (DMZ 격리) |

---

## 9. 6GHz (Wi-Fi 6E) 튜닝

### 9.1 6GHz 핵심 차이점

| 항목 | 2.4/5GHz | 6GHz |
|------|----------|------|
| AP Discovery | Passive/Active Scan | **OOB Discovery (RNR)** + Broadcast Probe |
| Preferred Scanning Channel (PSC) | N/A | 20MHz PSC 채널만 기본 스캔 |
| Security | WPA2/WPA3 | **WPA3 필수** (SAE) |
| BSS Color | Optional | **필수** |
| PMF | Optional | **필수** |

### 9.2 6GHz 튜닝 포인트

```
경로: RF Profile > 6GHz
```

| 파라미터 | 권장 |
|----------|------|
| Discovery Frames | **Broadcast Probe Response** (기본) |
| FILS Discovery | Enabled (빠른 AP 발견) |
| PSC Channel Enforcement | 환경에 따라 |
| Min Data Rate | 24 Mbps 이상 |
| Channel Width | Best (DBS) |
| BSS Color | Enabled (필수) |
| UPR (Unsolicited Probe Response) | Interval 20ms (기본) |

---

## 10. CleanAir & Interference 관리

```
경로: Configuration > Radio Configurations > CleanAir
```

| 파라미터 | 권장 |
|----------|------|
| CleanAir | **Enabled** (2.4GHz & 5GHz) |
| CleanAir on Persistent Devices | Enabled |
| Interference Reporting | Enabled |
| Security Alert (Jammer) | **Enabled** |
| Security Alert (Generic DECT) | Enabled |
| ED-RRM | **Enabled** |

**간섭원 유형:**

| 간섭 소스 | 주파수 | 영향 |
|-----------|--------|------|
| Microwave Oven | 2.4GHz | High |
| Bluetooth | 2.4GHz | Medium |
| DECT Phone | 1.9GHz (일부 2.4GHz) | Medium |
| Video Bridge | 2.4/5GHz | High |
| Jammer | All | **Critical** → ED-RRM 즉시 채널 변경 |

---

## 11. 운영 & 모니터링 튜닝

### 11.1 핵심 Show 명령어

```
! RF 상태 전체 확인
show ap dot11 5ghz summary
show ap dot11 24ghz summary

! 특정 AP의 상세 RF 설정
show ap name <AP-NAME> config slot {0|1|2}

! RRM 채널/파워 현황
show ap dot11 5ghz channel
show ap dot11 5ghz txpower

! DCA 상태
show ap dot11 5ghz cleanair device type all

! 클라이언트 RF 정보
show wireless client mac-address <MAC> detail

! RF Profile 확인
show ap rf-profile summary
show ap rf-profile name <profile-name> detail

! Rogue AP 현황
show wireless wps rogue ap summary

! RRM 그룹
show ap dot11 5ghz group
```

### 11.2 클라이언트 연결 품질 지표

| 지표 | 양호 | 경고 | 불량 |
|------|------|------|------|
| RSSI | > -65 dBm | -65 ~ -75 dBm | < -75 dBm |
| SNR | > 25 dB | 15~25 dB | < 15 dB |
| Data Rate | > 54 Mbps | 12~54 Mbps | < 12 Mbps |
| Retry Rate | < 10% | 10~25% | > 25% |

### 11.3 Syslog & SNMP 튜닝

```
! 필수 Syslog Severity
logging buffered 32768 informational
logging trap notifications

! AP SNMP
snmp-server enable traps wireless ap-auth-failure
snmp-server enable traps wireless rogue-ap
```

---

## 12. 실전 튜닝 시나리오

### 시나리오 1: 고밀도 오피스 (AP 80대, 클라이언트 2,000+)

```
[RF Profile — 2.4GHz]
  Data Rate: 12Mbps Mandatory, 1~11Mbps Disable
  TPC Max: 8 dBm / Min: 2 dBm
  Rx-SOP: Medium (-82 dBm)
  DCA: Auto, Sensitivity Medium
  Band Select: Enabled

[RF Profile — 5GHz]
  Data Rate: 24Mbps Mandatory, 6~12Mbps Disable
  TPC Max: 14 dBm / Min: 5 dBm (TPCv2)
  Rx-SOP: Low (-80 dBm)
  DCA: Auto, Chan-Width 40MHz, DFS Enabled
  FRA: Enabled

[WLAN]
  SSID 수: 3개 이하 (Corp, Guest, IoT)
  Security: WPA2/WPA3 Mixed + 802.1X
  802.11r: Mixed Mode
  802.11k: Enabled
  802.11v: Enabled

[Policy Profile]
  QoS: Silver (기본), Platinum (Voice SSID)
  AAA Override: Enabled (ISE 연동)
  Session Timeout: 28800 (8시간)
```

### 시나리오 2: 분산 지점 FlexConnect (Branch 50곳, AP 2~5대/지점)

```
[FlexConnect]
  Local Switching: Enabled
  Local Auth (Fallback): Enabled (WAN 장애 시 로컬 인증)
  VLAN-based Central/Local: 환경별 매핑
  
[RF Profile — 5GHz]
  TPC Max: 17 dBm / Min: 8 dBm
  DCA: Auto, Chan-Width Best
  FRA: Disabled (AP 수 적음, 2.4GHz 커버리지 필요)

[Roaming]
  802.11r: OTA Only (no FT Over-the-DS)
  OKC: Enabled
  Site Tag: 지점별 개별 (Fast Roaming Domain 분리)

[HA]
  Primary/Secondary WLC: AP Fallback Enabled
  RADIUS: Local Auth Fallback (FlexConnect)
```

### 시나리오 3: 대형 병원 (Wi-Fi 6E, C9136 AP, 로밍 크리티컬)

```
[6GHz]
  Security: WPA3-Enterprise (SAE)
  PMF: Required
  BSS Color: Enabled
  FILS Discovery: Enabled
  UPR Interval: 20ms
  DCA: Auto, Chan-Width Best
  Min Data Rate: 24Mbps (HE MCS0 NSS1 이상)

[Roaming — Voice/Telemetry]
  802.11r: Mixed Mode + FT Over-the-DS
  802.11k: Enabled (Neighbor List for guided roam)
  802.11v: Enabled (BSS Transition for load balance)
  CCKM: Enabled (legacy VoWiFi 폰 호환)

[Site Tag 설계]
  병동별 Site Tag (각 200 AP 이하)
  17.7+ → Site Tag 간 11k/v 동작 확인

[QoS]
  Voice SSID: Platinum + Fastlane (Apple)
  Data SSID: Silver
  Biomedical IoT: Bronze (별도 SSID)
```

---

## 13. AI-Enhanced RRM (Catalyst Center 연동)

```
경로: Catalyst Center > Design > Network Settings > Wireless > AI RF Profile
```

Catalyst Center (구 DNAC) + AI Analytics Cloud를 통해 클라우드 기반 RRM을 수행한다.

| 기존 RRM | AI-Enhanced RRM |
|----------|-----------------|
| WLC RF Group Leader가 10분 주기 DCA | Cloud에서 클러스터 기반 최적화 |
| 로컬 데이터만 활용 | 장기간 저장된 RF 텔레메트리 + ML 분석 |
| DCA, TPC | DCA, TPC, FRA, DBS 모두 AI 최적화 |

**활성화 조건:**
- RF Group Mode: **Automatic**
- Channel/Power Assignment: **Global (Automatic)**
- Catalyst Center에서 Building별 AI RF Profile 할당

> **권장**: Building별 개별 AI RF Profile 사용. RF Tag가 여러 Building에 걸치면 Insight 적용이 제한됨.

---

## 14. 트러블슈팅 Quick Reference

| 증상 | 확인 명령어 | 주요 원인 |
|------|-----------|-----------|
| AP 미조인 | `show ap summary`, `debug capwap events` | CAPWAP Discovery 실패, DTLS 인증서, MTU |
| 느린 속도 | `show wireless client mac <MAC> detail` | 저속 Data Rate, 높은 Retry, 낮은 RSSI |
| 로밍 끊김 | `show wireless client mac <MAC> mobility history` | 11r 미지원 클라이언트, OKC 미활성, PMK 미공유 |
| 채널 잦은 변경 | `show ap dot11 5ghz channel` | DCA Sensitivity 과민, Rogue AP 과다, CleanAir 간섭 |
| Coverage Hole | `show ap dot11 5ghz coverage-hole` | TPC Min 과소, AP 간 거리 과다 |
| 인증 실패 | `show wireless client mac <MAC> detail`, `debug aaa all` | RADIUS Timeout, 인증서 만료, VLAN 미설정 |
| WLC CPU High | `show process cpu sorted` | 과다 SSID, DTLS Data 암호화, 대량 로밍 |
| GUI 접속 불가 | `show ip http server status` | SSL 인증서 만료, HTTP/HTTPS 서비스 비활성 |

**Trace 기반 디버그 (Always-On):**

```
! 특정 클라이언트 MAC에 대한 trace
show logging profile wireless filter mac <MAC-ADDR>
show logging profile wireless internal filter mac <MAC-ADDR>

! 실시간 trace
debug wireless mac <MAC-ADDR>
! 완료 후 반드시 해제
no debug wireless mac <MAC-ADDR>
```

---

> **참고 문서:**
> - [C9800 Configuration Best Practices](https://www.cisco.com/c/en/us/td/docs/wireless/controller/9800/technical-reference/c9800-best-practices.html)
> - [C9800 RRM Deployment Guide](https://www.cisco.com/c/en/us/td/docs/wireless/controller/technotes/8-8/b_C9800_rrm_dg.html)
> - [AI-Enhanced RRM Deployment Guide](https://www.cisco.com/c/en/us/td/docs/wireless/controller/9800/technical-reference/ai-enhanced-rrm-dg.html)
> - [802.11r/11k/11v Fast Roams on 9800](https://www.cisco.com/c/en/us/support/docs/wireless/catalyst-9800-series-wireless-controllers/221671-understand-802-11r-11k-11v-fast-roams-on.html)
> - [802.1X Authentication on C9800](https://www.cisco.com/c/en/us/support/docs/wireless/catalyst-9800-series-wireless-controllers/213919-configure-802-1x-authentication-on-catal.html)
