# OpenStack 에러 및 해결 방법 모음

- Horizon 로그인 시 "인증 도중 오류가 발생했습니다. 잠시 후 다시 시도하세요." 라는 메시지 발생.
    
    `keystone.log`에서 다음과 같은 오류 확인됨:
    
    ```bash
    oslo_config.cfg.ArgsAlreadyParsedError: arguments already parsed: cannot register CLI option
    ```
    
    **원인**
    
    - `/usr/bin/keystone-wsgi-public` 파일 내 `argparse`로 CLI 인자를 파싱하는 코드가 존재.
    - Apache WSGI 환경에서는 CLI 인자를 사용할 수 없어 `oslo_config`에서 에러 발생.
    
    **해결 방법**
    
    ```bash
    cp /usr/bin/keystone-wsgi-public /usr/bin/keystone-wsgi-public.back
    vi /usr/bin/keystone-wsgi-public
    ```
    
    내용 전체 삭제 후 아래 코드 삽입:
    
    ```python
    
    from keystone.server.wsgi import initialize_public_application
    
    application = initialize_public_application()
    ```
    
    실행 권한 부여 및 Apache 재시작:
    
    ```bash
    sudo chmod +x /usr/bin/keystone-wsgi-public
    sudo systemctl restart apache2
    ```
    

---

- `nova.conf`에서 다음과 같은 deprecated 경고 발생:
    
    ```bash
    [WARNING] Option "auth_strategy" from group "api" is deprecated for removal
    ```
    
    **조치**
    
    - 버전 변경으로 인해 `[glance]`의 `api_servers` 설정 불필요해져 주석 처리하여 해결.

---

- OpenStack 인증 문제
    
    **문제**
    
    nova 인증 정보 확인 `openstack token issue` 명령 실행 시 401 Unauthorized 오류 발생.
    
    ```bash
    openstack --os-username nova \
      --os-password mnc1! \
      --os-project-name service \
      --os-auth-url http://mncsvrt06:5000/v3 \
      --os-user-domain-name Default \
      --os-project-domain-name Default \
      token issue
    ```
    
    **원인**
    
    - `admin-openrc` 파일 내 `OS_PROJECT_ID` 환경변수가 `project_name`과 충돌.
    
    **해결 방법**
    
    - `admin-openrc` 파일 내 `OS_PROJECT_ID` 항목 주석 처리.

---

- Keystone 인증 오류 (403 Forbidden)
    
    **문제**
    
    다음과 같은 Keystone 인증 오류 발생:
    
    ```bash
    Failed to discover available identity versions when contacting http://mncsvrt06/identity
    keystoneauth1.exceptions.http.Forbidden: Forbidden (HTTP 403)
    ```
    
    **원인**
    
    - URL 설정 오류 (`:5000/v3` 포트 누락).
    
    **조치 방법**
    
    잘못된 설정 경로 확인:
    
    ```bash
    sudo grep -r "http://mncsvrt06/identity" /etc/*
    ```
    
    수정 위치:
    
    ```
    # /etc/nova/nova.conf
    [service_user]
    auth_url = http://mncsvrt06:5000/v3
    ```
    

---

- Keystone 인증 오류 (401 Unauthorized)
    
    **문제**
    
    다음 오류 발생:
    
    ```bash
    keystoneauth1.exceptions.http.Unauthorized: The request you have made requires authentication. (HTTP 401)
    ```
    
    **원인**
    
    - `/etc/nova/nova.conf` 내 `[neutron]` 섹션의 `password` 누락 또는 오타.
    
    **해결 방법**
    
    - `[neutron]`에 올바른 비밀번호 입력되어 있는지 확인.

---

- Horizon 관리자 메뉴 표시되지 않음
    
    **문제**
    
    관리자 계정으로 로그인했으나 Horizon에서 "관리" 탭이 보이지 않음.
    
    **해결 방법**
    
    - **인증 > 프로젝트**로 이동.
    - 관리자 계정 옆에 있는 **"활성 프로젝트로 설정"** 클릭.
- Instance 생성 시 <class 'keystoneauth1.exceptions.discovery.DiscoveryFailure'> (HTTP 500) 에러
    - `nova.conf` 파일의 `[keystone]`, `[neutron]`, `[service_user]` 섹션에서 설정 수정
    - 기존에는 `mncsvrt06` 호스트 이름을 사용했으나, **IP 주소로 변경하여 적용**
    - 일반적으로는 호스트 이름을 사용해도 무방하지만, 어떤 문제인지 파악되지 않음
- Horizon Compute Instance 생성 후 status ERROR 및 콘솔 접속시 접근할 수 없다고 나옴
    
    `openstack compute service list` 입력 후 
    
    1. compute node(mncsvrt07)이 nova-compute로 등록되어 있는지 확인
    
    ```bash
      +--------------------------------------+----------------+-----------+----------+---------+-------+----------------------------+
    | ID                                   | Binary         | Host      | Zone     | Status  | State | Updated At                 |
    +--------------------------------------+----------------+-----------+----------+---------+-------+----------------------------+
    | 8a4173e5-dd07-41b3-841f-c417c777589e | nova-scheduler | mncsvrt06 | internal | enabled | up    | 2025-04-17T07:56:35.000000 |
    | 3b8d7c5e-5ed1-4f66-bade-954cdcf5476d | nova-conductor | mncsvrt06 | internal | enabled | up    | 2025-04-17T07:56:35.000000 |
    | 65432083-054e-4049-a78b-286146dcfa9b | nova-compute   | mncsvrt06 | nova     | enabled | up    | 2025-04-17T07:56:38.000000 |
    | 05870cef-4b24-4d36-ae34-9087def422ce | nova-compute   | mncsvrt07 | nova     | enabled | up    | 2025-04-17T07:56:36.000000 |
    +--------------------------------------+----------------+-----------+----------+---------+-------+----------------------------+
    ```
    
    1. compute node 접속 후 
    sudo virsh list --all 입력 → 아무것도 나오지 않음
    
    > NOTE
> 문제점
>
> service에는 등록이 되어있지만 Cell에는 매핑이 되어있지 않음

    
    1. Controller Node 에서 아래 명령어 입력
    
    ```bash
    sudo nova-manage cell_v2 list_hosts --verbose
    # 이 명령은 nova-compute가 설치된 모든 호스트를 찾아서 셀에 자동 등록합니다.
    
    # 아래 처럼 Creating 으로 먼가 만들어지면 매핑성공
    Found 2 cell mappings.
    Skipping cell0 since it does not contain hosts.
    Getting computes from cell 'cell1': f09f96aa-1e1e-4a35-9fd1-1698ff5c3386
    Checking host mapping for compute host 'mncsvrt06': a801bcbd-34a4-4466-800e-20729b47bdfa
    Creating host mapping for compute host 'mncsvrt06': a801bcbd-34a4-4466-800e-20729b47bdfa
    Checking host mapping for compute host 'mncsvrt07': db7c615d-6189-4e43-8787-e6c4c36d04f7
    Creating host mapping for compute host 'mncsvrt07': db7c615d-6189-4e43-8787-e6c4c36d04f7
    Found 2 unmapped computes in cell: f09f96aa-1e1e-4a35-9fd1-1698ff5c3386
    ```
    
    1. 기존 인스턴스 삭제
    
    ```bash
    # 서버 아이디 확인
    openstack server list
    
    # 인스턴스 삭제
    openstack server delete <id>
    ```
    
    1. 인스턴스 재 생성
    
    ![image.png](assets/images/OpenStack_image_eb4350ed.png)
    
    ERROR가 표시 되지 않으면 성공
    
- OpenStack 인스턴스에 **Floating IP(192.168.73.123)** 할당했지만, 외부에서 **ping/SSH 접속이 되지 않음**
    
    ## Controller Node 설정
    
    1. `l3_agent.ini` 설정 보완
    
    ```bash
    [DEFAULT]
    interface_driver = openvswitch
    # 아래 두 줄 추가
    external_network_bridge = br-provider
    agent_mode = legacy
    ```
    
    1. 서비스 재시작
    
    ```bash
    sudo systemctl restart neutron-l3-agent
    ```
    
    → 이후 `ip netns list`에서 `qrouter-xxxx` 네임스페이스 생성 확인
    
    ## Compute Node 설정
    
    1. `ovs-vsctl show` 로 네트워크 구성 확인
    error: could not add network device vxlan-c0a849ab to ofproto (Address already in use) 가 존재하면 
    `ovs-vsctl del-port` <Bridge Name> <Port Name> 으로 제거
    ex) ovs-vsctl del-port br-tun vxlan-c0a849ab
    2. systemctl restart openvswitch-switch
    3. restart 후에도 error가 남아있을 경우 서버 reboot
    
    ## 인스턴스 재 생성
    
    1. 인스턴스 생성 구성 탭 → 사용자 정의 스크립트
    
    Ubuntu 로그인 스크립트 입력 → ssh key 없이 콘솔에 로그인 가능
    
    ```bash
    #cloud-config
    hostname: hostname # 호스트 네임 지정
    fqdn: hostname 
    
    disable_root: false
    ssh_pwauth: true
    chpasswd:
      list: |
        ubuntu:ubuntu123
      expire: false
    ```
    
    Centos 로그인 스크립트 입력 → ssh key 없이 콘솔에 로그인 가능
    
    ```bash
    #cloud-config
    hostname: hostname # 호스트 네임 지정
    fqdn: hostname 
    
    disable_root: false
    ssh_pwauth: true
    chpasswd:
      list: |
        centos:centos123
      expire: false
    ```
    
    네트워크 설정(나중에 neutron 구성으로 옮길거임)
    
    Centos9 은 기본 설정 계정이 cloud-user 임 →인스턴스 생성 후 로그로 확인
    
    ```bash
    #cloud-config
    hostname: hostname # 호스트 네임 지정
    fqdn: hostname 
    
    disable_root: false
    ssh_pwauth: true
    chpasswd:
      list: |
        cloud-user:mnc1!
      expire: false
    ```
    
    사용자 계정 생성 방법
    
    ```bash
    #cloud-config
    hostname: master-k8s-01 # 호스트네임 지정
    fqdn: master-k8s-01 # 위와 동일하게 변경
    
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
    ```
    
- Prometheus Grafana 연동
    - `openstack-exporter`가 OpenStack 인증 정보를 제대로 인식하지 못해 Prometheus 메트릭 수집 실패
    - 해결 방법: `vi /etc/systemd/system/openstack-exporter.service` 파일에 OpenStack 인증 정보(Environment)를 직접 명시하고, `systemctl`을 통해 적용
    
    ```bash
    # 기존 버전
    [Unit]
    Description=OpenStack Exporter
    After=network.target
    
    [Service]
    Type=simple
    User=prometheus
    ExecStart=/usr/local/bin/openstack-exporter --os-client-config=/etc/openstack/clouds.yaml default --web.listen-address=:9180
    Restart=always
    
    [Install]
    WantedBy=multi-user.target
    
    # 수정 버전
    [Unit]
    Description=OpenStack Prometheus Exporter
    After=network.target
    
    [Service]
    User=prometheus
    Environment="OS_AUTH_URL=http://192.168.73.171:5000/v3"
    Environment="OS_USERNAME=admin"
    Environment="OS_PASSWORD=mnc1!"
    Environment="OS_PROJECT_NAME=admin"
    Environment="OS_USER_DOMAIN_NAME=Default"
    Environment="OS_PROJECT_DOMAIN_NAME=Default"
    Environment="OS_IDENTITY_API_VERSION=3"
    ExecStart=/usr/local/bin/openstack-exporter --web.listen-address=:9180
    Restart=always
    
    [Install]
    WantedBy=multi-user.target
    ```
    
- 인스턴스 접속 후 ping 8.8.8.8 안되는 문제
    - 인스턴스 접속 후 외부로 ping이 안되는 문제가 발생함
    - 함명호 상무님께 연락드려서 사용 가능한 IP 대역 확인 후
        
        ```bash
        for i in {161..190}; do ping -c 1 -W 1 192.168.73.$i >/dev/null 2>&1; [ $? -ne 0 ] && echo "192.168.73.$i"; done
        ```
        
        명령어로 사용가능한 아이피 확인
        
    - 외부 네트워크 생성시 서브넷 세부 정보 → Pools 할당에 위에서 확인한 IP 대역 입력
    - 인스턴스 재 생성 후 **Floating IP** 생성 → 인스턴스 접속한 뒤 ping 8.8.8.8 테스트
- 인스턴스 생성 후 설정한 아이디 비밀번호 ssh key로 접속이 되지 않음
    - /var/log/neutron/neutron-dhcp-agent.log
    디버깅 후 해결
    - 지금 같은 경우는 trove 구성에서 환경이 꼬여서 문제가 발생했었음
- 인스턴스에 할당된 볼륨이 해제되지 않고 409 에러가 발생
    - 2023년 5월 10일 이후 모든 OpenStack 릴리스의 경우, Nova가 Cinder로 서비스 토큰을 전송하고 Cinder가 해당 토큰을 수신하도록 구성 필요
        
        ```bash
        # cinder.conf 에 아래 항목 추가
        [keystone_authtoken]
        service_token_roles = service
        service_token_roles_required = true
        
        [service_user]
        send_service_user_token = true
        auth_url = http://mncsvrt06:5000/
        auth_strategy = keystone
        auth_type = password
        project_domain_name = Default
        project_name = service
        user_domain_name = Default
        username = cinder
        password = mnc1!
        ```
        
    - 위 항목을 추가해도 Nova 사용자에게 `service` 역할이 없으면 서비스 토큰에 `service` 역할이 들어가지 않아 Keystone이 이를 거부 → Cinder가 Unauthorized 오류를 반환한다.
        
        ```bash
        # nova 역할 확인
        openstack role assignment list --user nova --project service --names
        
        +-------+--------------+-------+-----------------+--------+--------+-----------+
        | Role  | User         | Group | Project         | Domain | System | Inherited |
        +-------+--------------+-------+-----------------+--------+--------+-----------+
        | admin | nova@Default |       | service@Default |        |        | False     |
        +-------+--------------+-------+-----------------+--------+--------+-----------
        
        # admin만 추가되어 있어 문제가 발생
        
        # service 역할 할당
        openstack role add --project service --user nova service 
        openstack role add --user nova --project service service
        # 확인
        openstack role assignment list --user nova --project service --names
        +---------+--------------+-------+-----------------+--------+--------+-----------+
        | Role    | User         | Group | Project         | Domain | System | Inherited |
        +---------+--------------+-------+-----------------+--------+--------+-----------+
        | admin   | nova@Default |       | service@Default |        |        | False     |
        | service | nova@Default |       | service@Default |        |        | False     |
        +---------+--------------+-------+-----------------+--------+--------+-----------+
        
        openstack role assignment list --user cinder --project service --names
        +---------+----------------+-------+-----------------+--------+--------+-----------+
        | Role    | User           | Group | Project         | Domain | System | Inherited |
        +---------+----------------+-------+-----------------+--------+--------+-----------+
        | admin   | cinder@Default |       | service@Default |        |        | False     |
        | service | cinder@Default |       | service@Default |        |        | False     |
        +---------+----------------+-------+-----------------+--------+--------+-----------+
        ```
        
- 관리의 네트워크 탭 선택시 에러 발생
    
    ```bash
    # ERROR
    neutron.api.v2.resource AttributeError: 'NoneType' object has no attribute 'registered_rules'
    ```
    
    해결방법
    
    ```bash
    sudo vi /etc/neutron/neutron.cong
    # 아래 내용 추가
    enforce_new_defaults = False
    
    # 재시작
    sudo systemctl restart neutron-server.service 
    ```
