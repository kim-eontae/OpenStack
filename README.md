# OpenStack Caracal (2024.1) 설치 가이드

이 저장소는 **OpenStack 2024.1 Caracal** 설치/운영 내용을 **GitHub README + docs 구조**로 재정리한 문서입니다.

- 대상 OS: **Ubuntu 기반**
- 작업 권한 원칙: **패키지/시스템 작업은 root**, **유저/룰/서비스/엔드포인트 생성은 일반 사용자**
- 주의: endpoint 및 conf URL을 hostname만으로 통일했을 때 API 간 통신 문제가 발생할 수 있어, 환경에 맞게 **hostname/IP를 적절히 혼용**해야 합니다.

> 배포 자동화(Kolla-ansible, OpenStack Helm) 문서는 사용자 요청에 따라 **본 저장소에서 제외**했습니다.

## 문서 목차

### 0) 구성/환경/네트워크
- [OpenStack Component 구성 목록](docs/00-overview/01-components.md)
- [기본 구성(버전/서버/스토리지)](docs/00-overview/02-environment.md)
- [네트워크(Provider / Self-Service)](docs/00-overview/03-network.md)

### 1) 사전 준비(Controller/Compute 공통)
- [권한/전제(주의사항)](docs/01-prerequisites/04-user-policy.md)
- [Host networking 설정](docs/01-prerequisites/01-host-networking.md)
- [NTP(Chrony) 설정](docs/01-prerequisites/02-time-sync.md)
- [OpenStack 패키지/리포 설치](docs/01-prerequisites/03-repos-packages.md)

### 2) Controller 전용 구성요소
- [SQL database (MariaDB)](docs/02-controller-only/01-sql-database.md)
- [Message queue (RabbitMQ)](docs/02-controller-only/02-message-queue.md)
- [Memcached](docs/02-controller-only/03-memcached.md)

### 3) OpenStack 서비스 설치
- [Keystone](docs/03-openstack-services/01-keystone.md)
- [Glance](docs/03-openstack-services/02-glance.md)
- [Placement](docs/03-openstack-services/03-placement.md)
- [Nova](docs/03-openstack-services/04-nova.md)
- [Neutron](docs/03-openstack-services/05-neutron.md)
- [Horizon](docs/03-openstack-services/06-horizon.md)
- [Cinder (LVM)](docs/03-openstack-services/07-cinder-lvm.md)
- [Cinder (NFS)](docs/03-openstack-services/08-cinder-nfs.md)
- [Cinder (Ceph)](docs/03-openstack-services/09-cinder-ceph.md)
- [Swift](docs/03-openstack-services/10-swift.md)
- [Octavia(LB)](docs/03-openstack-services/11-octavia.md)
- [Heat](docs/03-openstack-services/12-heat.md)
- [Billing (Ceilometer, Cloudkitty)](docs/03-openstack-services/13-billing.md)
- [Ironic](docs/03-openstack-services/14-ironic.md)
- [Magnum](docs/03-openstack-services/15-magnum.md)

### 4) GPU Passthrough
- [VFIO-PCI GPU Passthrough](docs/04-gpu/01-vfio-passthrough.md)
- [GPU 드라이버 설치](docs/04-gpu/02-gpu-driver.md)

### 5) 운영/보안
- [Keystone MFA(TOTP)](docs/05-ops-security/01-keystone-mfa-totp.md)
- [모니터링/알람](docs/05-ops-security/02-monitoring-alerting.md)
- [VIP 설정](docs/05-ops-security/03-vip.md)
- [Nova 자동화(shell 파일 등록)](docs/05-ops-security/04-nova-automation.md)
- [인스턴스 백신/보안(ClamAV 등)](docs/05-ops-security/05-instance-security.md)
- [OPNsense / OpenVAS](docs/05-ops-security/06-opnsense-openvas.md)

### 6) 명령어 모음
- [OpenStack 명령어](docs/06-cli-recipes/00-index.md)

### 7) 에러 대응
- [에러 및 해결 방법 모음](docs/07-troubleshooting/01-errors.md)
