# OpenStack Heat(오케스트레이션) 구축

---

---

## Heat 사용 이유

| 이유  | 설명  |
| --- | --- |
|  **Infrastructure as Code (IaC)** | 인프라를 코드로 관리 → 버전 관리, 형상 관리 용이 |
|  **자동화** | VM, 네트워크, 볼륨 등 여러 리소스를 한 번에 생성 가능 |
|  **반복 가능한 배포** | 동일한 템플릿으로 여러 환경에 일관되게 배포 |
|  **수정 및 롤백 가능** | 변경사항을 `stack update`로 적용, 실패 시 rollback 가능 |
|  **Horizon, API, CLI 통합** | GUI, 명령어, API로 통합 관리 가능 |
|  **역할 기반 권한 관리** | Keystone과 연동하여 스택 권한 관리 가능 |

## Heat **설치 및 구성**

데이터베이스 생성

```sql
mysql -u root -p

CREATE DATABASE heat;

GRANT ALL PRIVILEGES ON heat.* TO 'heat'@'localhost' IDENTIFIED BY 'mnc1!';
GRANT ALL PRIVILEGES ON heat.* TO 'heat'@'%' IDENTIFIED BY 'mnc1!';
```

사용자 및 서비스 엔드포인트 등록 

```bash
# 관리자 자격 증명
. admin-openrc

# heat 사용자 생성 및 역할 부여
openstack user create --domain default --password-prompt heat
openstack role add --project service --user heat admin

# 서비스 등록
openstack service create --name heat --description "Orchestration" orchestration
openstack service create --name heat-cfn --description "Orchestration"  cloudformation

# 엔드포인트 등록 (controller는 호스트명/IP로 대체)
openstack endpoint create --region RegionOne orchestration public http://controller:8004/v1/%\(tenant_id\)s
openstack endpoint create --region RegionOne orchestration internal http://controller:8004/v1/%\(tenant_id\)s
openstack endpoint create --region RegionOne orchestration admin http://controller:8004/v1/%\(tenant_id\)s

openstack endpoint create --region RegionOne cloudformation public http://controller:8000/v1
openstack endpoint create --region RegionOne cloudformation internal http://controller:8000/v1
openstack endpoint create --region RegionOne cloudformation admin http://controller:8000/v1

# heat 도메인 및 스택 사용자 도메인 관리자 생성
openstack domain create --description "Stack projects and users" heat
openstack user create --domain heat --password-prompt heat_domain_admin
****openstack role add --domain heat --user-domain heat --user heat_domain_admin admin

# 스택 사용자 역할 생성
openstack role create heat_stack_owner
openstack role create heat_stack_user
```

Heat 패키지 설치 및 설정

```bash
sudo apt-get install -y heat-api heat-api-cfn heat-engine

# 설정파일 편집
sudo vi /etc/heat/heat.conf 

# ────────────────────────────────
# [DEFAULT] 설정
# controller → 호스트명으로 변경 가능 (혹은 IP) (인증 문제 방지)
[DEFAULT]
heat_metadata_server_url = http://mncsvrt06:8000
heat_waitcondition_server_url = http://mncsvrt06:8000/v1/waitcondition
transport_url = rabbit://openstack:mnc_rabbit@mncsvrt06
stack_domain_admin = heat_domain_admin
stack_domain_admin_password = mnc1! 
stack_user_domain_name = heat

# ────────────────────────────────
# [database] 설정
[database]
connection = mysql+pymysql://heat:mnc1!@mncsvrt06/heat

# ────────────────────────────────
# [api] 설정
[api]
auth_strategy = keystone

# ────────────────────────────────
# [keystone_authtoken] 설정
# controller → 호스트명으로 변경 가능 (혹은 IP)
[keystone_authtoken]
www_authenticate_uri = http://mncsvrt06:5000
auth_url = http://mncsvrt06:5000
memcached_servers = mncsvrt06:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = heat
password = mnc1!

# ────────────────────────────────
# [trustee] 설정
[trustee]
auth_type = password
auth_url = http://mncsvrt06:5000
username = heat
password = mnc1!
user_domain_name = Default

# ────────────────────────────────
# [clients_keystone] 설정
[clients_keystone]
auth_url = http://mncsvrt06:5000
```

데이터베이스 초기화 및 서비스 재시작

```bash
sudo -s /bin/sh -c "heat-manage db_sync" heat
```

heat 정책 설정

`sudo vi /etc/heat/policy.yaml`

```bash
# /etc/heat/policy.yaml

# 조회 계열
stacks:index:  "role:reader or role:member or role:admin"
stacks:show:   "role:reader or role:member or role:admin"
events:index:  "role:reader or role:member or role:admin"
events:show:   "role:reader or role:member or role:admin"
resources:index: "role:reader or role:member or role:admin"
resources:show:  "role:reader or role:member or role:admin"
build_info:    "role:reader or role:member or role:admin"
template:validate: "role:reader or role:member or role:admin"

# 변경 계열
stacks:create: "role:member or role:admin"
stacks:update: "role:member or role:admin"
stacks:delete: "role:member or role:admin"
stacks:abandon: "role:member or role:admin"
stacks:adopt:   "role:member or role:admin"

# (스택 유저/SoftwareDeployment 계열을 쓰면 필요)
software_deployments:index: "role:reader or role:member or role:admin"
software_deployments:show:  "role:reader or role:member or role:admin"
software_deployments:create: "role:member or role:admin"
software_deployments:delete: "role:member or role:admin"
```

시스템 재 시작

```bash
sudo systemctl restart heat-api.service heat-api-cfn.service heat-engine.service
```

---

## Heat Template (HOT) 생성

템플릿 작성

```bash
# create folder
mkdir ~/heat-templates
cd ~/heat-templates
```

예시 `basic.yaml`

```bash
heat_template_version: 2021-04-16

description: Simple template to deploy a single compute instance

parameters:
  NetID:
    type: string
    description: Network ID to use for the instance.
  
  VolumeName:
    type: string
    description: Volume name to create.

conditions:
  CreateVolume: { not: { equals: [ { get_param: VolumeName }, "" ] } }

resources:
  server:
    type: OS::Nova::Server
    properties:
      name: test-heat # 인스턴스 이름 지정   
      key_name: ubuntu #  key(Compute -> key pair)
      image: Ubuntu22.04_LTS # 이미지 이름
      flavor: m1.medium-slim # flavor 이름
      networks:
        - network: { get_param: NetID }
      security_groups: # 보안 그룹 지정
        - GenOS
        - default
      user_data_format: RAW
      user_data:
        str_replace:
          template: |
            #cloud-config
            hostname: master-k8s # hostname지정
            fqdn: master-k8s

            disable_root: false
            ssh_pwauth: true

            users:
              - name: kube
                gecos: "kube"
                sudo: ["ALL=(ALL) NOPASSWD:ALL"]
                shell: /bin/bash
                lock_passwd: false

            chpasswd:
              list: |
                kube:mnc1!
              expire: false
          params: {}

  volume: # volume생성
    type: OS::Cinder::Volume
    condition: CreateVolume
    deletion_policy: Retain # volume 제거 안함
    properties:
      size: 10 # size
      name: { get_param: VolumeName }

  volume_attach: # volume 할당
    type: OS::Cinder::VolumeAttachment
    condition: CreateVolume  
    properties:
      instance_uuid: { get_resource: server }
      volume_id: { get_resource: volume }
      mountpoint: /dev/vdb

outputs:
  instance_name:
    description: Name of the instance.
    value: { get_attr: [ server, name ] }

  instance_ip:
    description: IP address of the instance.
    value: { get_attr: [ server, first_address ] }

  volume_name:
    condition: CreateVolume
    description: Name of the volume.
    value: { get_param: VolumeName }
```

스택 배포

```bash
# 사용 가눙한 네트워크 확인
openstack network list

# NET_ID가 네트워크 ID를 반영하도록 환경변수 생성
export NET_ID=$(openstack network list | awk '/ internal / { print $2 }')

# 스택 생성
openstack stack create -t instance.yaml --parameter "NetID=$NET_ID" --parameter "VolumeName=<volume_name>"   stack
```

스택 확인 및 삭제

```bash
# 스택 생성 확인
openstack stack list

# 만들어진 인스턴스 확인
openstack server list

# 스택 삭제 -> 인스턴스도 삭제 됨
openstack stack delete --yes stack
```

스택 배포 - 여러 버전

```bash
# 프로젝트 지정
openstack --os-project-name Infrastructure stack create -t instance.yaml --parameter "NetID=$NET_ID" --parameter "VolumeName=<volume_name>" demo-stack

# 볼륨 생성 안함
openstack stack create -t instance.yaml --parameter "NetID=$NET_ID" --parameter VolumeName='' stack

# mysql
openstack stack create -t mysql_instance.yaml --parameter "NetID=$NET_ID" --parameter "RootPassword=mnc1!" --parameter VolumeName=''   mysql-stack
```

스택 업데이트

```bash
openstack --os-project-name Infrastructure stack update --template hyundai-capital-worker.yaml --parameter "NetID=$NET_ID" --parameter VolumeName='' --wait hyundai-capital-worker
```

> NOTE
> heat로 인스턴스의 flavor를 업데이트하면 인스턴스의 root디스크가 새로 만들어지는 위험이 있음 flavor업데이트는
> # flavor resize
> nova resize --poll [instance-id] [flavor-id]
> # 변경 적용
> openstack server resize --confirm [instance-id]
> 명령어를 사용하는게 안전함


---

## Heat Dashboard 추가

패키지 설치

```bash
sudo pip3 install heat-dashboard
```

설치 파일 위치 확인

```bash
python3 -m pip show heat-dashboard

Name: heat-dashboard
Version: 13.0.0
Summary: Heat Management Dashboard
Home-page: https://docs.openstack.org/heat-dashboard/latest/
Author: OpenStack
Author-email: openstack-discuss@lists.openstack.org
License: 
**Location: /home/mnc_admin/.local/lib/python3.10/site-packages**
Requires: horizon, pbr, python-heatclient, xstatic-angular-uuid, xstatic-angular-vis, xstatic-filesaver, xstatic-js-yaml, xstatic-json2yaml
Required-by: 
```

Horizon dashboard로 파일 옮기기

```bash
sudo cp /home/mnc_admin/.local/lib/python3.10/site-packages/heat_dashboard/enabled/_[1-9]*.py /var/www/horizon/openstack_dashboard/local/enabled/
```

Horizon 파일 수정

```bash
sudo vi /var/www/horizon/openstack_dashboard/local/local_settings.py

# heat
POLICY_FILES = {
    'orchestration': '/usr/local/lib/python3.10/site-packages/heat_dashboard/conf/heat_policy.json',
        }
HEAT_TEMPLATE_GENERATOR_ENABLED_SETTINGS = {
    'enable_user_pass': True
}
```

설정 적용

```bash
python3 ./manage.py compilemessages
python3 /var/www/horizon/manage.py collectstatic --noinput
python3 /var/www/horizon/manage.py compress --force
 
```

`vi /etc/neutron/neutron.conf` 파일 수정

```bash
[DEFAULT]
service_plugins = router,qos
```

`vi /etc/neutron/plugins/ml2/ml2_conf.ini`  파일 수정

```bash
[ml2]
extension_drivers = port_security,qos
```

`vi /etc/neutron/plugins/ml2/openvswitch_agent.ini` 파일 수정

```bash
[agent]
extensions = qos
```

서비스 재 시작

```bash
sudo systemctl restart neutron-server.service neutron-dhcp-agent.service neutron-l3-agent.service neutron-metadata-agent.service neutron-openvswitch-agent.service neutron-ovs-cleanup.service
```

---
