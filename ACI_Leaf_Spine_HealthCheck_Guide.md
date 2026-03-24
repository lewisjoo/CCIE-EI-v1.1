# Cisco ACI Leaf / Spine Health Check Guide

> **APIC Version: 4.2.(3i)** 기준  
> Leaf/Spine 스위치는 APIC GUI 또는 스위치 직접 SSH 접속을 통해 점검  
> 확인 형식: `APIC GUI Path` + `Switch CLI`

---

## 1. Inventory

| 점검 항목 | APIC GUI Path | Switch CLI | 비고 |
|-----------|--------------|------------|------|
| Fabric Membership | `Fabric > Inventory > Fabric Membership` | — | 전체 노드 등록 상태 (Active/Inactive) |
| Node 상세 정보 | `Fabric > Inventory > Pod > [Node] > General` | `show version` | Serial, Model, Role, Node ID |
| Module / Linecard | `Fabric > Inventory > Pod > [Node] > Chassis > Modules` | `show module` | Linecard / Supervisor 상태 |
| Firmware Version | `Fabric > Inventory > Pod > [Node] > General` | `show version` | Running / Target 버전 |
| | `System > Firmware > Infrastructure > Nodes` | | 전체 노드 펌웨어 일괄 확인 |

---

## 2. CPU

| 점검 항목 | APIC GUI Path | Switch CLI | 비고 |
|-----------|--------------|------------|------|
| CPU 사용률 | `Fabric > Inventory > Pod > [Node] > General > CPU/Memory` | `show processes cpu sort` | 전체 CPU % 및 Top Process |
| CPU History | `Fabric > Inventory > Pod > [Node] > General > CPU/Memory > History` | `show processes cpu history` | 시간대별 추이 확인 |
| 개별 Process CPU | — | `show system internal processes cpu-stats` | 특정 프로세스 과점유 확인 |

> **임계치 기준**: CPU 지속 80% 이상 시 성능 영향 가능. `supervisor` 관련 프로세스 과점유 주의

---

## 3. Memory

| 점검 항목 | APIC GUI Path | Switch CLI | 비고 |
|-----------|--------------|------------|------|
| Memory 사용률 | `Fabric > Inventory > Pod > [Node] > General > CPU/Memory` | `show system resources` | Total / Used / Free |
| Process별 Memory | — | `show processes memory sort` | Memory 과점유 프로세스 확인 |
| Memory History | `Fabric > Inventory > Pod > [Node] > General > CPU/Memory > History` | — | 시간대별 추이 |

> **임계치 기준**: Memory 90% 이상 지속 시 주의. `show system internal processes memory-stats` 로 상세 확인

---

## 4. Power / Fan / Temperature

| 점검 항목 | APIC GUI Path | Switch CLI | 비고 |
|-----------|--------------|------------|------|
| Power Supply | `Fabric > Inventory > Pod > [Node] > Chassis > Power Supplies` | `show environment power` | OK / Shutdown / Absent |
| Power Redundancy | — | `show environment power detail` | PS Redundancy Mode (Combined/Redundant) |
| Fan Tray | `Fabric > Inventory > Pod > [Node] > Chassis > Fan Trays` | `show environment fan` | RPM / Direction / 상태 |
| Temperature | `Fabric > Inventory > Pod > [Node] > Chassis > Sensors` | `show environment temperature` | 센서별 현재 온도 / 임계치 |

> **주의**: Fan Direction(Front-to-Back / Back-to-Front)이 DC 환경 Airflow와 일치하는지 확인

---

## 5. Topology

| 점검 항목 | APIC GUI Path | Switch CLI | 비고 |
|-----------|--------------|------------|------|
| Fabric Topology | `Fabric > Inventory > Topology` | — | 물리 토폴로지 뷰 (APIC GUI 전용) |
| Pod Topology | `Fabric > Inventory > Pod > Topology` | — | Pod 단위 상세 뷰 |
| LLDP Neighbors | `Fabric > Inventory > Pod > [Node] > Interfaces > LLDP Adjacency` | `show lldp neighbors` | 물리적 연결 확인 |
| CDP Neighbors | — | `show cdp neighbors` | CDP 활성 시 |

---

## 6. Storage Status

| 점검 항목 | APIC GUI Path | Switch CLI | 비고 |
|-----------|--------------|------------|------|
| Bootflash 사용률 | — | `dir bootflash:` | 잔여 용량 확인 |
| Logflash 사용률 | — | `dir logflash:` | 로그 파일 누적 확인 |
| 디스크 전체 | — | `show system internal flash` | 전체 파티션 상태 |

> **주의**: Bootflash 90% 이상 시 Firmware Upgrade 실패 가능. 불필요 파일 정리 필요

---

## 7. Timezone / NTP

| 점검 항목 | APIC GUI Path | Switch CLI | 비고 |
|-----------|--------------|------------|------|
| NTP Policy (Fabric 레벨) | `Fabric > Fabric Policies > Pod Policies > Date and Time` | — | Fabric 전체 NTP 정책 |
| NTP Status (노드별) | `Fabric > Inventory > Pod > [Node] > Protocols > NTP` | `show ntp peer-status` | Sync 상태 (`*` = synced) |
| Clock 확인 | — | `show clock` | 현재 시각 / Timezone |
| NTP 상세 | — | `show ntp statistics peer ipaddress [IP]` | Offset / Jitter 확인 |

> **정상 기준**: `show ntp peer-status`에서 `*` 표시된 서버가 있어야 정상 동기화

---

## 8. Fault / Events

| 점검 항목 | APIC GUI Path | Switch CLI | 비고 |
|-----------|--------------|------------|------|
| 노드별 Active Faults | `Fabric > Inventory > Pod > [Node] > Faults` | — | 해당 노드의 Fault만 필터링 |
| 전체 Active Faults | `System > Faults > Faults (Active)` | — | Severity 필터: Critical > Major > Minor |
| Fault Records | `System > Faults > Records` | — | Fault 발생/해소 이력 |
| Events | `Fabric > Inventory > Pod > [Node] > Events` | `show logging last 100` | 최근 이벤트 / Syslog |
| System Log | — | `show logging logfile` | 전체 로그 파일 확인 |

> **점검 포인트**: Critical / Major Fault가 0인지 우선 확인. 반복 발생하는 Fault는 Root Cause 분석 필요

---

## 9. System History

| 점검 항목 | APIC GUI Path | Switch CLI | 비고 |
|-----------|--------------|------------|------|
| Audit Log | `System > History > Audit Log` | — | 설정 변경 이력 (누가, 언제, 무엇을) |
| Health History | `Fabric > Inventory > Pod > [Node] > Health` | — | 노드별 Health Score 변화 추이 |
| Event History | `Fabric > Inventory > Pod > [Node] > History > Events` | — | 과거 이벤트 이력 |
| Reboot History | — | `show system reset-reason` | 마지막 리부팅 원인 |
| | — | `show reload cause` | 동일 (플랫폼에 따라) |

---

## 10. Fabric Connectivity

| 점검 항목 | APIC GUI Path | Switch CLI | 비고 |
|-----------|--------------|------------|------|
| IS-IS Adjacency | `Fabric > Inventory > Pod > [Node] > Protocols > IS-IS > Adjacency` | `show isis adjacency` | **Full** 상태 확인 |
| IS-IS Route | — | `show isis route` | Underlay 라우팅 테이블 |
| COOP Status | `Fabric > Inventory > Pod > [Spine] > Protocols > COOP` | `show coop internal info repo ep` | Spine에서만 동작 (Oracle) |
| VTEP Peers | `Fabric > Inventory > Pod > [Node] > Protocols > VXLAN > Peers` | `show nve peers` | VXLAN 터널 상태 |
| VTEP Interface | — | `show nve interface nve1` | NVE 인터페이스 Up 확인 |
| Fabric Link 상태 | `Fabric > Inventory > Pod > [Node] > Interfaces > Fabric Interfaces` | `show interface fabric [x/y]` | Leaf↔Spine 링크 상태 |

> **핵심 체크**: IS-IS = Full, VTEP Peer = Up, COOP Oracle = Active. 하나라도 이상 시 데이터 플레인 장애 가능

---

## 11. Interface 상태

| 점검 항목 | APIC GUI Path | Switch CLI | 비고 |
|-----------|--------------|------------|------|
| Physical Interfaces | `Fabric > Inventory > Pod > [Node] > Interfaces > Physical Interfaces` | `show interface brief` | Up/Down, Speed, Duplex |
| Interface Detail | `Fabric > Inventory > Pod > [Node] > Interfaces > [Intf]` | `show interface [Intf]` | Input/Output Rate, MTU |
| Interface Errors | `Fabric > Inventory > Pod > [Node] > Interfaces > [Intf] > Stats > Counters` | `show interface [Intf] counters errors` | CRC, Drops, Runts, Giants |
| Optics (DOM) | `Fabric > Inventory > Pod > [Node] > Interfaces > [Intf] > Transceiver` | `show interface [Intf] transceiver details` | Tx/Rx Power (dBm), Temp |
| Interface Policy | `Fabric > Access Policies > Interfaces > Leaf Interfaces > Profiles` | — | 적용된 정책 그룹 확인 |
| SFP Inventory | `Fabric > Inventory > Pod > [Node] > Interfaces > [Intf] > Transceiver` | `show interface [Intf] transceiver` | SFP Model, Serial, Type |

> **DOM 기준**: Rx Power가 -30dBm 이하면 사실상 No Light. Cisco SFP 권장 범위를 벗어나면 Fault 발생

---

## 12. vPC / Port Channel

| 점검 항목 | APIC GUI Path | Switch CLI | 비고 |
|-----------|--------------|------------|------|
| vPC Domain | `Fabric > Access Policies > Switch Policies > vPC Domains` | `show vpc` | vPC Domain ID, Role (Primary/Secondary) |
| vPC Peer Status | — | `show vpc` | Peer Status: **peer-ok** 확인 |
| vPC Keepalive | — | `show vpc peer-keepalive` | Keepalive Status: **alive** |
| vPC Consistency | — | `show vpc consistency-parameters global` | Type-1 Mismatch 없는지 확인 |
| | — | `show vpc consistency-parameters interface port-channel [N]` | Per-interface consistency |
| Port Channel 상태 | `Fabric > Inventory > Pod > [Node] > Interfaces > Port Channels` | `show port-channel summary` | Member 포트 Up/Down 확인 |
| LACP Status | — | `show lacp neighbor` | LACP Neighbor 정상 인식 확인 |
| | — | `show lacp counters` | LACP PDU 송수신 확인 |
| vPC Orphan Port | — | `show vpc orphan-ports` | vPC에 속하지 않은 단일 포트 확인 |

> **vPC 점검 핵심**  
> - `show vpc` 에서 **peer-ok**, **peer-alive**, **consistent** 3가지 모두 정상이어야 함  
> - Type-1 Inconsistency 발생 시 vPC 자체가 Suspend될 수 있으므로 즉시 확인 필요

---

## 13. Config Backup

| 점검 항목 | APIC GUI Path | Switch CLI | 비고 |
|-----------|--------------|------------|------|
| Config Export Policy | `Admin > Import/Export > Export Policies > Configuration Export` | — | 자동 백업 스케줄 확인 |
| Backup History | `Admin > Import/Export > Export Policies > [Policy] > History` | — | 마지막 백업 성공/실패 |
| Remote Location | `Admin > Import/Export > Remote Locations` | — | SCP/SFTP/FTP 저장소 연결 |
| Snapshot | `Admin > Import/Export > Export Policies > Configuration Export (Snapshot)` | — | Rollback용 스냅샷 |
| Switch Running Config | — | `show running-config` | 스위치 레벨 Running Config (참고용) |
| Tech-Support | — | `show tech-support` | 장애 시 수집용 전체 로그 |
| | — | `trigger techsupport` (APIC CLI) | APIC에서 원격 수집 가능 |

> **참고**: ACI에서는 모든 Config가 APIC에 중앙 관리되므로, **APIC Config Backup이 핵심**.  
> Leaf/Spine의 `show running-config`은 APIC이 Push한 Rendered Config이며 직접 수정 불가 (Intent-Based)

---

## 참고: 자주 쓰는 Switch CLI 요약

| 카테고리 | 명령어 | 설명 |
|----------|--------|------|
| **General** | `show version` | OS 버전, Uptime, Model |
| | `show system resources` | CPU / Memory 종합 |
| | `show system reset-reason` | 마지막 리부트 원인 |
| **Hardware** | `show module` | Linecard / Supervisor |
| | `show environment` | Power / Fan / Temp 종합 |
| | `show diagnostic result module all` | 하드웨어 진단 결과 |
| **Interface** | `show interface brief` | 전체 인터페이스 요약 |
| | `show interface counters errors` | 에러 카운터 일괄 |
| | `show interface transceiver details` | 전체 SFP DOM 값 |
| **Fabric** | `show isis adjacency` | IS-IS 인접 상태 |
| | `show nve peers` | VXLAN VTEP Peer |
| | `show lldp neighbors` | 물리 연결 확인 |
| **vPC** | `show vpc` | vPC 전체 상태 요약 |
| | `show vpc consistency-parameters global` | Global Consistency |
| | `show port-channel summary` | PC/vPC Member 상태 |
| **NTP** | `show ntp peer-status` | NTP 동기화 상태 |
| **Log** | `show logging last 50` | 최근 Syslog 50줄 |
