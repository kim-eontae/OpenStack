- Neutron - Networking service
        - **역할**
            
            가상 네트워크 및 IP 주소 관리
            
        - **설명**
            
            VM 간 네트워크 연결, 라우팅, floating IP, 보안 그룹 등을 관리
            
            → **가상 네트워크를 생성하고 VM에 연결하는 역할**
            
            → **Controller Node**와 **Compute Node** 모두 구성해야 함
            
        - Controller Node
            - Create DataBase
                
                ```bash
                mysql -u root -p
                
                CREATE DATABASE neutron;
                
                GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'NEUTRON_DBPASS';
                GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'NEUTRON_DBPASS';
                ```
                
                > `NEUTRON_DBPASS`는 원하는 비밀번호로 변경
                > 
            - Configure User and Endpoints
            (아래 진행되는 명령어들은 일반 사용자로 진행)
                
                ```bash
                # 관리자 자격 증명
                . admin-openrc
                
                # nova user 생성
                openstack user create --domain default --password-prompt neutron
                **# 비밀번호 입력**
                
                # 사용자 역할 추가
                openstack role add --project service --user neutron admin
                
                # 서비스 엔터티 생성
                openstack service create --name neutron --description "OpenStack Networking" network
                
                # Compute API endpoint 생성
                openstack endpoint create --region RegionOne network public http://controller:9696
                openstack endpoint create --region RegionOne network internal http://controller:9696
                openstack endpoint create --region RegionOne network admin http://controller:9696
                # controller 는 호스트명으로 변경
                ```
                
            - **Configure networking options
            Self-service-networks로 설정**
                - **Install the components**
                    
                    ```bash
                    sudo apt install neutron-server neutron-plugin-ml2 \
                      neutron-openvswitch-agent neutron-l3-agent neutron-dhcp-agent \
                      neutron-metadata-agent
                    ```
                    
                - **Configure the server component**
                    
                    ```bash
                    sudo vi /etc/neutron/neutron.conf
                    
                    # ────────────────────────────────
                    # [database] 설정
                    # NEUTRON_DBPASS: neutron DB 생성시 지정한 비밀번호
                    # controller: 호스트명(IP 가능)
                    [database]
                    connection = mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron
                    
                    # ────────────────────────────────
                    # [DEFAULT] 설정
                    [DEFAULT]
                    core_plugin = ml2
                    service_plugins = router
                    transport_url = rabbit://openstack:RABBIT_PASS@controller  # RABBIT_PASS: rabbitmq 비밀번호
                    auth_strategy = keystone
                    notify_nova_on_port_status_changes = true
                    notify_nova_on_port_data_changes = true
                    debug = true
                    
                    # ────────────────────────────────
                    # [keystone_authtoken] 설정
                    # controller: 호스트명으로 변경 가능 (또는 IP)
                    [keystone_authtoken]
                    www_authenticate_uri = http://controller:5000
                    auth_url = http://controller:5000
                    memcached_servers = controller:11211
                    auth_type = password
                    project_domain_name = Default
                    user_domain_name = Default
                    project_name = service
                    username = neutron
                    password = NEUTRON_PASS  # neutron 유저 생성 시 입력한 비밀번호
                    
                    # ────────────────────────────────
                    # [nova] 설정
                    # Nova 서비스와의 연동
                    [nova]
                    auth_url = http://controller:5000
                    auth_type = password
                    project_domain_name = Default
                    user_domain_name = Default
                    region_name = RegionOne
                    project_name = service
                    username = nova
                    password = NOVA_PASS  # nova 유저 생성 시 입력한 비밀번호
                    
                    # ────────────────────────────────
                    # [oslo_concurrency] 설정
                    [oslo_concurrency]
                    lock_path = /var/lib/neutron/tmp
                    
                    ```
                    
                - **Configure the Modular Layer 2 (ML2) plug-in**
                    
                    ```bash
                    vi /etc/neutron/plugins/ml2/ml2_conf.ini 
                    
                    [ml2]
                    type_drivers = flat,vlan,vxlan
                    tenant_network_types = vxlan
                    mechanism_drivers = openvswitch,l2population
                    extension_drivers = port_security
                    
                    [ml2_type_flat]
                    flat_networks = provider
                    
                    [ml2_type_vxlan]
                    vni_ranges = 1:1000
                    ```
                    
                - **Configure the Open vSwitch agent**
                    
                    ```bash
                    vi /etc/neutron/plugins/ml2/openvswitch_agent.ini
                    
                    [ovs]
                    bridge_mappings = provider:br-provider
                    tunnel_bridge = br-tun
                    local_ip = 192.168.73.171
                    
                    [agent]
                    tunnel_types = vxlan
                    l2_population = true
                    
                    [securitygroup]
                    enable_security_group = true
                    firewall_driver = openvswitch
                    ```
                    
                - 네트워크 브리지 필터 지원 확인
                    
                    ```bash
                    sysctl net.bridge.bridge-nf-call-iptables
                    sysctl net.bridge.bridge-nf-call-ip6tables
                    
                    # 출력값이 1 → 지원됨
                    ```
                    
                - **Configure the layer-3 agent**
                    
                    ```bash
                    vi /etc/neutron/l3_agent.ini
                    
                    [DEFAULT]
                    interface_driver = openvswitch
                    external_network_bridge = br-provider
                    agent_mode = legacy
                    ```
                    
                - **Configure the DHCP agent**
                    
                    ```bash
                    vi /etc/neutron/dhcp_agent.ini
                    
                    [DEFAULT]
                    interface_driver = openvswitch
                    dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
                    enable_isolated_metadata = true
                    ```
                    
                - **Configure the metadata agent**
                    
                    ```bash
                    vi /etc/neutron/metadata_agent.ini
                    
                    [DEFAULT]
                    # controller 호스트명 변경 
                    # METADATA_SECRET 하고 싶은 비밀번호 지정
                    nova_metadata_host = controller
                    metadata_proxy_shared_secret = METADATA_SECRET
                    ```
                    
                - **Configure the Compute service to use the Networking service**
                    
                    ```bash
                    vi /etc/nova/nova.conf
                    
                    # ────────────────────────────────
                    # [neutron] 설정
                    # controller → IP 사용 권장 (호스트명 대신)
                    [neutron]
                    auth_url = http://controller_node_ip:5000
                    auth_type = password
                    project_domain_name = Default
                    user_domain_name = Default
                    region_name = RegionOne
                    project_name = service
                    username = neutron
                    password = NEUTRON_PASS  # neutron 유저 생성 시 입력한 비밀번호
                    service_metadata_proxy = true
                    metadata_proxy_shared_secret = METADATA_SECRET  # METADATA_SECRET 비밀번호 입력
                    ```
                    
                - DB 동기화
                    
                    ```bash
                    sudo -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
                      --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
                    ```
                    
                - **OpenVSwitch 설정**
                    
                    ```bash
                    # br-provider라는 이름의 가상 스위치 생성
                    ovs-vsctl add-br br-provider
                    
                    # ────────────────────────────────
                    # 물리 네트워크 이름 확인
                    ip a
                    # 사용 중인 IP를 가진 네트워크 장치 이름 확인
                    
                    # ────────────────────────────────
                    # 원래 네트워크 설정 백업
                    cp /etc/netplan/01-network-manager-all.yaml /etc/netplan/01-network-manager-all.yaml.v0
                    
                    # ────────────────────────────────
                    # 네트워크 설정 수정
                    vi /etc/netplan/01-network-manager-all.yaml
                    
                    # 기본 주석 처리된 내용 (예시)
                    # Let NetworkManager manage all devices on this system
                    #network:
                    #  version: 2
                    #  renderer: NetworkManager
                    
                    # 아래처럼 수정
                    network:
                      version: 2
                      ethernets:
                        eno3np0:
                          dhcp4: false
                      bridges:
                        br-provider:
                          interfaces:
                            - eno3np0
                          addresses:
                            - 192.168.73.171/24  # Controller Node IP 입력
                          routes:
                            - to: default
                              via: 192.168.73.161  # ip route로 확인한 라우터 IP
                          nameservers:
                            addresses:
                              - 192.168.73.161
                          openvswitch: {}
                          dhcp4: false
                    
                    # ────────────────────────────────
                    # 적용 (인터넷 연결 일시 중단됨 주의!)
                    netplan apply
                    
                    # ────────────────────────────────
                    # 아래 명령어는 netplan에서 이미 처리했기 때문에 필요 없음
                    # ovs-vsctl add-port br-provider eno3np0
                    # netplan apply
                    ```
                    
                - **OpenVSwitch 설정 확인**
                    
                    ```bash
                    ovs-vsctl show
                    
                    Bridge br-provider
                        ...
                        Port eno3np0
                            Interface eno3np0
                                
                    br-provider 브릿지에 eno3np0 인터페이스가 Port로 연결된 것을 확인하면 설정이 정상적으로 완료된 것임.
                    ```
                    
                - 서비스 재 시작
                    
                    ```bash
                    sudo service nova-api restart
                    sudo service neutron-server restart
                    sudo service neutron-openvswitch-agent restart
                    sudo service neutron-dhcp-agent restart
                    sudo service neutron-metadata-agent restart
                    sudo service neutron-l3-agent restart
                    ```
                    
        - Compute Node
            - Compute Node - Self-service Networks 설정
                - **Install the components**
                    
                    ```bash
                    sudo apt install -y neutron-openvswitch-agent
                    ```
                    
                - **Configure the common component**
                    
                    ```bash
                    vi /etc/neutron/neutron.conf
                    
                    [DEFAULT]
                    # RABBIT_PASS → rabbitmq 비밀번호 입력
                    transport_url = rabbit://openstack:RABBIT_PASS@controller
                    
                    [oslo_concurrency]
                    lock_path = /var/lib/neutron/tmp
                    ```
                    
                - **Configure the Open vSwitch agent**
                    
                    ```bash
                    vi /etc/neutron/plugins/ml2/openvswitch_agent.ini
                    
                    [ovs]
                    bridge_mappings = provider:br-provider
                    tunnel_bridge = br-tun
                    local_ip = 192.168.73.172  # 본인 Compute Node IP
                    
                    [agent]
                    tunnel_types = vxlan
                    l2_population = true
                    
                    [securitygroup]
                    enable_security_group = true
                    firewall_driver = openvswitch
                    # firewall_driver = iptables_hybrid
                    ```
                    
                - 네트워크 브리지 필터 지원 확인
                    
                    ```bash
                    sysctl net.bridge.bridge-nf-call-iptables
                    sysctl net.bridge.bridge-nf-call-ip6tables
                    
                    # 출력값이 1 → 지원됨
                    ```
                    
                - **Configure the Compute service to use the Networking service**
                    
                    ```bash
                    vi /etc/nova/nova.conf
                    
                    [neutron]
                    auth_url = http://controller:5000
                    auth_type = password
                    project_domain_name = Default
                    user_domain_name = Default
                    region_name = RegionOne
                    project_name = service
                    username = neutron
                    password = NEUTRON_PASS  # neutron 유저 생성 시 입력한 비밀번호
                    ```
                    
                - **OpenVSwitch 설정**
                    
                    ```bash
                    # br-provider라는 이름의 가상 스위치 생성
                    ovs-vsctl add-br br-provider
                    
                    # ────────────────────────────────
                    # 물리 네트워크 이름 확인
                    ip a
                    # 사용 중인 IP를 가진 네트워크 장치 이름 확인
                    
                    # ────────────────────────────────
                    # 원래 네트워크 설정 백업
                    cp /etc/netplan/01-network-manager-all.yaml /etc/netplan/01-network-manager-all.yaml.v0
                    
                    # ────────────────────────────────
                    # 네트워크 설정 수정
                    vi /etc/netplan/01-network-manager-all.yaml
                    
                    # 기본 주석 처리된 내용 (예시)
                    # Let NetworkManager manage all devices on this system
                    #network:
                    #  version: 2
                    #  renderer: NetworkManager
                    
                    # 아래처럼 수정
                    network:
                      version: 2
                      ethernets:
                        eno3np0:
                          dhcp4: false
                      bridges:
                        br-provider:
                          interfaces:
                            - eno3np0
                          addresses:
                            - 192.168.73.171/24  # Controller Node IP 입력
                          routes:
                            - to: default
                              via: 192.168.73.161  # ip route로 확인한 라우터 IP
                          nameservers:
                            addresses:
                              - 192.168.73.161
                          openvswitch: {}
                          dhcp4: false
                    
                    # ────────────────────────────────
                    # 적용 (인터넷 연결 일시 중단됨 주의!)
                    netplan apply
                    
                    # ────────────────────────────────
                    # 아래 명령어는 netplan에서 이미 처리했기 때문에 필요 없음
                    # ovs-vsctl add-port br-provider eno3np0
                    # netplan apply
                    ```
                    
                - **OpenVSwitch 설정 확인**
                    
                    ```bash
                    ovs-vsctl show
                    
                    Bridge br-provider
                        ...
                        Port eno3np0
                            Interface eno3np0
                                
                    	br-provider 브릿지에 eno3np0 인터페이스가 Port로 연결된 것을 확인하면 설정이 정상적으로 완료된 것임.
                    ```
                    
                - 서비스 재 시작
                    
                    ```bash
                    sudo systemctl restart nova-compute neutron-openvswitch-agent
                    ```
                    
                - 검증
                    
                    ```bash
                    openstack network agent list
                    
                    +--------------------------------------+--------------------+-----------+-------------------+-------+-------+---------------------------+
                    | ID                                   | Agent Type         | Host      | Availability Zone | Alive | State | Binary                    |
                    +--------------------------------------+--------------------+-----------+-------------------+-------+-------+---------------------------+
                    | 700047ce-e4ad-43cf-a72f-2112de3c9d2a | Open vSwitch agent | mncsvrt06 | None              | :-)   | UP    | neutron-openvswitch-agent |
                    | 9762e49e-73e8-4267-8ea8-8077059e9a1d | DHCP agent         | mncsvrt06 | nova              | :-)   | UP    | neutron-dhcp-agent        |
                    | 976efbe3-1e65-407b-b475-b37663978c21 | Metadata agent     | mncsvrt06 | None              | :-)   | UP    | neutron-metadata-agent    |
                    | f5a6f1d7-35e4-4bb3-b578-f7d607309af8 | L3 agent           | mncsvrt06 | nova              | :-)   | UP    | neutron-l3-agent          |
                    | f7ed0a2d-a5b0-4b95-bc7a-3b17f9f9a035 | Open vSwitch agent | mncsvrt07 | None              | :-)   | UP    | neutron-openvswitch-agent |
                    +--------------------------------------+--------------------+-----------+-------------------+-------+-------+---------------------------+
                    # 위 결과와 비슷하게 나오면 성공
                    ```
                    
        - Open vSwitch vs Linux Bridge (비교 정리)
            
            ### 기본 개념
            
            | 항목 | Open vSwitch (OVS) | Linux Bridge |
            | --- | --- | --- |
            | 사용 목적 | 고급 가상 네트워크 구성 | 단순한 브리지 구성 |
            | 사용 환경 | 대규모 클라우드, SDN | 소규모 가상화, 단일 서버 |
            | 통신 방식 | 커널 모듈 + 사용자 공간 | 커널 내장 브리지 모듈 |
            
            ---
            
            ### 기능 비교
            
            | 기능 | OVS | Linux Bridge |
            | --- | --- | --- |
            | VXLAN 지원 | ✅ | ⚠️ 제한적 |
            | GRE 터널 | ✅ | ❌ |
            | VLAN | ✅ | ✅ |
            | OpenFlow 지원 | ✅ | ❌ |
            | 보안 그룹 (SG) 연동 | ✅ (ML2 plugin) | ✅ (ML2 plugin) |
            | 성능 최적화 (DPDK 등) | ✅ 고급 최적화 가능 | ❌ 제한적 |
            | 관리 도구 | `ovs-vsctl`, `ovs-ofctl` | `brctl`, `ip` |
            | 복잡도 | 중~고 | 낮음 |
            
            ---
            
            ### 사용 추천
            
            | 환경 | 추천 기술 |
            | --- | --- |
            | OpenStack, Kubernetes, 멀티 노드 | **Open vSwitch** |
            | 단일 서버, 간단한 테스트 | **Linux Bridge** |
        
        ---
