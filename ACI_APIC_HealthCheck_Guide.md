# Cisco ACI APIC Health Check Guide

> **APIC Version: 4.2.(3i)** | GUI 확인 방법 기준  
> 확인 형식: `Dashboard > Menu Path > Feature`  
> CLI 명령어가 필요한 항목은 별도 표기

---

## 1. System Overview

| 점검 항목 | APIC 확인 방법 (GUI Path) | 비고 |
|-----------|--------------------------|------|
| IP Address | `System > Controllers > [APIC Node] > General` | APIC OOB/InB IP |
| Hostname | `System > Controllers > [APIC Node] > General` | |
| APIC Version | `System > Controllers > [APIC Node] > General` | Running / Target 버전 확인 |
| | `System > Firmware > Infrastructure > Firmware` | |
| Uptime | `System > Controllers > [APIC Node] > General` | |
| System Status | `Dashboard > System Health Score` | Health Score 확인 |
| | `System > Faults > History` | |

---

## 2. Hardware / Resource

### 2-1. Inventory

| 점검 항목 | APIC 확인 방법 (GUI Path) | 비고 |
|-----------|--------------------------|------|
| Inventory | `Fabric > Inventory > Pod > [Node]` | 등록된 Leaf/Spine/APIC 목록 |
| | `Fabric > Inventory > Fabric Membership` | |

### 2-2. CPU

| 점검 항목 | APIC 확인 방법 (GUI Path) | 비고 |
|-----------|--------------------------|------|
| CPU 사용률 | `System > Controllers > [APIC Node] > General > CPU/Memory` | 임계치 초과 여부 |
| | `Operations > Capacity Dashboard` | |

### 2-3. Memory

| 점검 항목 | APIC 확인 방법 (GUI Path) | 비고 |
|-----------|--------------------------|------|
| Memory 사용률 | `System > Controllers > [APIC Node] > General > CPU/Memory` | 임계치 초과 여부 |
| | `Operations > Capacity Dashboard` | |

### 2-4. Power / Fan / Temperature

| 점검 항목 | APIC 확인 방법 (GUI Path) | 비고 |
|-----------|--------------------------|------|
| Power Supply | `Fabric > Inventory > Pod > [Node] > Chassis > Power Supplies` | 정상/이상 상태 |
| Fan Tray | `Fabric > Inventory > Pod > [Node] > Chassis > Fan Trays` | RPM / 상태 확인 |
| Temperature | `Fabric > Inventory > Pod > [Node] > Chassis > Sensors` | 온도 임계치 확인 |

---

## 3. Topology

| 점검 항목 | APIC 확인 방법 (GUI Path) | 비고 |
|-----------|--------------------------|------|
| Fabric Topology | `Fabric > Inventory > Topology` | 물리 토폴로지 뷰 |
| | `Fabric > Inventory > Pod > Topology` | |

---

## 4. Storage Status

| 점검 항목 | APIC 확인 방법 (GUI Path) | 비고 |
|-----------|--------------------------|------|
| Storage Status | `System > Controllers > [APIC Node] > General` | `/data` 파티션 사용률 확인 |
| | **CLI:** `df -h` (APIC SSH) | |

---

## 5. Timezone / NTP

| 점검 항목 | APIC 확인 방법 (GUI Path) | 비고 |
|-----------|--------------------------|------|
| Timezone | `System > System Settings > Date and Time > Time Zone` | |
| NTP Status | `System > System Settings > Date and Time > NTP` | NTP Sync 상태 확인 |
| | `Fabric > Fabric Policies > Pod Policies > Date and Time` | |

---

## 6. Fault / Events

| 점검 항목 | APIC 확인 방법 (GUI Path) | 비고 |
|-----------|--------------------------|------|
| Active Faults | `System > Faults > Faults (Active)` | Critical / Major / Minor / Warning |
| | `Dashboard > Faults 위젯` | |
| Events | `System > Faults > Events` | 최근 이벤트 내역 |
| | `System > History > Event Records` | |

---

## 7. System History

| 점검 항목 | APIC 확인 방법 (GUI Path) | 비고 |
|-----------|--------------------------|------|
| Audit Log | `System > History > Audit Log` | 변경 이력 추적 |
| Health History | `System > Faults > History` | 과거 장애 이력 |

---

## 8. VRF Status

| 점검 항목 | APIC 확인 방법 (GUI Path) | 비고 |
|-----------|--------------------------|------|
| VRF 목록/상태 | `Tenants > [Tenant] > Networking > VRFs` | VRF Scope, Enforcement 확인 |
| VRF Health | `Tenants > [Tenant] > Networking > VRFs > [VRF] > Health` | Health Score |

---

## 9. BD (Bridge Domain) Status

| 점검 항목 | APIC 확인 방법 (GUI Path) | 비고 |
|-----------|--------------------------|------|
| BD 목록/상태 | `Tenants > [Tenant] > Networking > Bridge Domains` | 서브넷, L2/L3 Unknown Unicast 설정 |
| BD Health | `Tenants > [Tenant] > Networking > Bridge Domains > [BD] > Health` | Health Score |
| BD-EPG 매핑 | `Tenants > [Tenant] > Networking > Bridge Domains > [BD] > Associated EPGs` | 매핑 누락 확인 |

---

## 10. EPG Status

| 점검 항목 | APIC 확인 방법 (GUI Path) | 비고 |
|-----------|--------------------------|------|
| EPG 목록/상태 | `Tenants > [Tenant] > Application Profiles > [AP] > Application EPGs` | Static/Dynamic Binding 확인 |
| EPG Health | `Tenants > [Tenant] > Application Profiles > [AP] > [EPG] > Health` | Health Score |
| EPG Endpoint 수 | `Tenants > [Tenant] > Application Profiles > [AP] > [EPG] > Operational > Client End Points` | 학습된 EP 수 확인 |

---

## 11. L3Out Status

| 점검 항목 | APIC 확인 방법 (GUI Path) | 비고 |
|-----------|--------------------------|------|
| L3Out 목록/상태 | `Tenants > [Tenant] > Networking > L3Outs` | 프로토콜 타입 확인 (OSPF/BGP/EIGRP/Static) |
| L3Out Health | `Tenants > [Tenant] > Networking > L3Outs > [L3Out] > Health` | Health Score |
| L3Out Node/Interface | `Tenants > [Tenant] > Networking > L3Outs > [L3Out] > Logical Node Profiles > Logical Interface Profiles` | 인터페이스 상태 |
| External EPG | `Tenants > [Tenant] > Networking > L3Outs > [L3Out] > External EPGs` | Subnet Scope 확인 |

---

## 12. Fabric Connectivity

| 점검 항목 | APIC 확인 방법 (GUI Path) | 비고 |
|-----------|--------------------------|------|
| Fabric Membership | `Fabric > Inventory > Fabric Membership` | 모든 노드 Active 확인 |
| ISIS Adjacency | `Fabric > Inventory > Pod > [Node] > Protocols > IS-IS > Adjacency` | Full 상태 확인 |
| COOP Status | `Fabric > Inventory > Pod > [Spine Node] > Protocols > COOP` | Oracle 상태 |
| VTEP Peers | `Fabric > Inventory > Pod > [Node] > Protocols > VXLAN > Peers` | Peer 연결 상태 |

---

## 13. Cluster 상태 (APIC Cluster)

| 점검 항목 | APIC 확인 방법 (GUI Path) | 비고 |
|-----------|--------------------------|------|
| Cluster Health | `System > Controllers > APIC Cluster > Cluster Health` | Fully Fit 확인 |
| Cluster Nodes | `System > Controllers > APIC Cluster > Cluster Members` | 모든 APIC Active/Healthy |
| Cluster Convergence | `System > Controllers > APIC Cluster > Convergence Status` | 수렴 완료 확인 |

---

## 14. DB Operation State

| 점검 항목 | APIC 확인 방법 (GUI Path) | 비고 |
|-----------|--------------------------|------|
| DB Status | **CLI:** `acidiag dbcheck` | Shard 상태 확인 |
| | `System > Controllers > [APIC] > Processes > Database` | |
| DB Replication | **CLI:** `acidiag avread` | APIC 간 DB 동기화 |
| | `System > Controllers > APIC Cluster` | |

---

## 15. Interface 상태

| 점검 항목 | APIC 확인 방법 (GUI Path) | 비고 |
|-----------|--------------------------|------|
| Physical Interfaces | `Fabric > Inventory > Pod > [Node] > Interfaces > Physical Interfaces` | Up/Down, Speed, Error 확인 |
| Port Channel | `Fabric > Inventory > Pod > [Node] > Interfaces > Port Channels` | vPC/PC 상태 |
| Interface Errors | `Fabric > Inventory > Pod > [Node] > Interfaces > [Intf] > Stats > Counters` | CRC, Drops, Input/Output Errors |
| Optics (DOM) | `Fabric > Inventory > Pod > [Node] > Interfaces > [Intf] > Transceiver` | Tx/Rx Power, Temperature |

---

## 16. Config Backup

| 점검 항목 | APIC 확인 방법 (GUI Path) | 비고 |
|-----------|--------------------------|------|
| Backup 설정 | `Admin > Import/Export > Export Policies > Configuration Export` | 자동 백업 스케줄 확인 |
| Backup 이력 | `Admin > Import/Export > Export Policies > Configuration Export > [Policy] > History` | 마지막 백업 성공 여부 |
| Remote Location | `Admin > Import/Export > Remote Locations` | 백업 저장소 연결 확인 |
| Snapshot | `Admin > Import/Export > Export Policies > Configuration Export (Snapshot)` | Rollback용 스냅샷 |

---

## 참고: 주요 CLI 명령어 (APIC SSH)

| 명령어 | 설명 |
|--------|------|
| `acidiag avread` | APIC Cluster 상태 및 DB Replica 확인 |
| `acidiag dbcheck` | DB Shard Health 확인 |
| `acidiag fnvread` | Fabric Node 등록 상태 확인 |
| `show firmware upgrade status` | 펌웨어 업그레이드 상태 확인 |
| `show controller` | APIC Controller 요약 정보 |
| `df -h` | 디스크/스토리지 사용률 확인 |
| `cat /aci/system/chassis` | 하드웨어 Inventory 확인 |
