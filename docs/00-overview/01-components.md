# OpenStack

## OpenStack Component 구성 목록

| Component**Name** | **역할** | **설명** | **구축 여부** | **에로 사항** |
| --- | --- | --- | --- | --- |
| **Keystone** | 인증 서비스 | OpenStack 서비스 및 사용자 인증/권한 관리 담당 | O |  |
| **Glance** | 이미지 서비스 | VM 인스턴스 생성에 사용하는 디스크 이미지 저장 및 관리 | O |  |
| **Placement** | 자원 예약 | 가용 리소스(CPU, RAM 등)의 추적 및 스케줄링을 위한 자원 예약 관리 | O |  |
| **Nova** | 컴퓨트 서비스 | 가상 머신 인스턴스의 생성, 삭제, 관리 담당 | O |  |
| **Neutron** | 네트워크 서비스 | L2/L3 네트워크, Floating IP, 보안 그룹 등 네트워크 리소스 제공 | O |  |
| **Horizon** | 대시보드 | 웹 기반 OpenStack 관리 UI 제공 | O |  |
| **Cinder** | 블록 스토리지 | 인스턴스에 연결 가능한 영구 블록 스토리지 볼륨 제공 | O |  |
| **Swift** | 객체 스토리지 | 확장 가능한 객체 스토리지 서비스 (이미지, 백업, 데이터 저장용 등) | O | 사용 중 아님 |
| **Heat** | 오케스트레이션 | 스택(template 기반 리소스 배치) 관리, IaC 제공 | O |  |
| **Trove** | DB 서비스 | DBaaS(Database as a Service) 기능 제공 (MySQL, PostgreSQL 등 관리) | O | 사용 중 아님 |
| **Ironic** | 베어메탈 프로비저닝 | 물리 서버(베어메탈) 인스턴스를 관리하는 서비스 | X | Controller Node로 관리하는 서버에 두 개의 렌선이 필요하여 중단 (렌선 추가 할당시 구축 진행 예정) |
| **Octavia** | 로드밸런서 서비스 | L4/L7 로드밸런서를 제공하는 LBaaS(Load Balancer as a Service) 기능 | O |  |
| **Magnum** | 컨테이너 오케스트레이션 | Kubernetes, Swarm, Mesos 등의 클러스터 배포 및 관리 | X | Octavia 구축 완료로 개발서버 구축 진행 |

## OpenStack Component 외 주요 구성 목록

| **Name** | **역할** | **설명** | 관리 |
| --- | --- | --- | --- |
| **OPNsense** | 방화벽 | 네트워크 트래픽 제어, NAT, 포트포워딩, IDS/IPS(Suricata), VPN 등을 지원하는 오픈소스 방화벽 플랫폼 | VM 인스턴스로 관리 |
| **OpenVAS** | 네트워크 취약점 분석 | 오픈소스 기반의 취약점 스캐너로, 네트워크 서비스 및 시스템에 대한 보안 취약점 진단을 수행 | VM 인스턴스로 관리 |
