- Nova - Compute service
        - **역할**
            
            가상 머신 인스턴스의 생성, 삭제, 관리
            
        - **설명**
            
            OpenStack의 **가상 서버를 실제로 생성/관리**하는 컴포넌트
            
            하이퍼바이저(KVM 등)를 통해 인스턴스를 실행
            **Controller Node**와 **Compute Node**에 둘 다 구성해야 함
            
        - Controller Node
            - Create DataBase
                
                ```bash
                mysql
                
                CREATE DATABASE nova_api;
                CREATE DATABASE nova;
                CREATE DATABASE nova_cell0;
                
                GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
                GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';
                
                GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
                GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';
                
                GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
                GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';
                
                # NOVA_DBPASS는 원하는 값으로 변경
                ```
                
            - Configure User and Endpoints 
            (아래 진행되는 명령어들은 일반 사용자로 진행)
                
                ```bash
                # 관리자 자격 증명
                . admin-openrc
                
                # nova user 생성
                openstack user create --domain default --password-prompt nova
                **# 비밀번호 입력**
                
                # 사용자 역할 추가
                openstack role add --project service --user nova admin
                
                # 서비스 엔터티 생성
                openstack service create --name nova --description "OpenStack Compute" compute
                
                # Compute API endpoint 생성
                openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1
                openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1
                openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1
                # controller 는 호스트명으로 변경
                ```
                
            - **Install and configure components (root)**
                
                ```bash
                sudo apt install nova-api nova-conductor nova-novncproxy nova-scheduler
                
                # 설정파일 편집
                sudo vi /etc/nova/nova.conf
                
                # ────────────────────────────────
                # [api_database] 설정
                # NOVA_DBPASS: nova DB 생성 시 지정한 비밀번호
                # controller는 실제 호스트명 or IP로 교체
                [api_database]
                connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova_api
                
                # [database] 설정
                [database]
                connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova
                
                # ────────────────────────────────
                # [DEFAULT] 설정
                # controller → 호스트명으로 변경 가능 (혹은 IP) (인증 문제 방지)
                [DEFAULT]
                log_dir = /var/log/nova
                lock_path = /var/lock/nova
                state_path = /var/lib/nova
                transport_url = rabbit://openstack:RABBIT_PASS@controller:5672/
                my_ip = 192.168.73.171  # Controller Node IP 입력
                allow_resize_to_same_host = true
                
                # ────────────────────────────────
                # [api] 설정
                [api]
                auth_strategy = keystone
                
                # ────────────────────────────────
                # [keystone_authtoken] 설정
                # controller → 호스트명으로 변경 가능 (혹은 IP)
                [keystone_authtoken]
                www_authenticate_uri = http://mncsvrt06:5000/
                auth_url = http://mncsvrt06:5000/
                memcached_servers = 192.168.73.171:11211
                auth_type = password
                project_domain_name = Default
                user_domain_name = Default
                project_name = service
                username = nova
                password = NOVA_PASS  # nova 유저 생성 시 입력한 비밀번호
                
                # ────────────────────────────────
                # [service_user] 설정
                [service_user]
                send_service_user_token = true
                auth_url = http://192.168.73.171:5000/
                auth_strategy = keystone
                auth_type = password
                project_domain_name = Default
                project_name = service
                user_domain_name = Default
                username = nova
                password = NOVA_PASS  # nova 유저 생성 시 입력한 비밀번호
                
                # ────────────────────────────────
                # [vnc] 설정
                [vnc]
                enabled = true
                server_listen = $my_ip
                server_proxyclient_address = $my_ip
                novncproxy_base_url = http://192.168.73.171:6080/vnc_auto.html  # Controller Node IP
                
                # ────────────────────────────────
                # [oslo_concurrency] 설정
                [oslo_concurrency]
                lock_path = /var/lib/nova/tmp
                
                # ────────────────────────────────
                # [placement] 설정
                [placement]
                region_name = RegionOne
                project_domain_name = Default
                project_name = service
                auth_type = password
                user_domain_name = Default
                auth_url = http://192.168.73.171:5000/v3
                username = placement
                password = PLACEMENT_PASS  # placement 유저 생성 시 입력한 비밀번호
                
                # ────────────────────────────────
                # [scheduler] 설정
                [scheduler]
                discover_hosts_in_cells_interval = 300
                ```
                
            - 데이터베이스 동기화 및 cell 생성
                
                ```bash
                sudo -s /bin/sh -c "nova-manage api_db sync" nova
                sudo -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
                sudo -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
                sudo -s /bin/sh -c "nova-manage db sync" nova
                sudo -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
                ```
                
            - 시스템 재 시작
                
                ```bash
                systemctl restart nova-api nova-scheduler nova-conductor nova-novncproxy
                ```
                
        - Compute Node
            - **Install and configure components (root)**
                
                ```bash
                sudo apt install -y nova-compute
                
                # 설정파일 편집
                sudo vi /etc/nova/nova.conf
                
                # ────────────────────────────────
                # [DEFAULT] 설정
                # controller → 호스트명으로 변경 가능 (혹은 IP)
                [DEFAULT]
                log_dir = /var/log/nova
                lock_path = /var/lock/nova
                state_path = /var/lib/nova
                # RABBIT_PASS: rabbitmq 비밀번호 입력
                transport_url = rabbit://openstack:RABBIT_PASS@mncsvrt06
                my_ip = 192.168.73.172  # Compute Node IP 입력
                allow_resize_to_same_host = true
                
                # ────────────────────────────────
                # [api] 설정
                [api]
                auth_strategy = keystone
                
                # ────────────────────────────────
                # [keystone_authtoken] 설정
                # controller → 호스트명으로 변경 가능 (혹은 IP)
                [keystone_authtoken]
                www_authenticate_uri = http://mncsvrt06:5000/
                auth_url = http://mncsvrt06:5000/
                memcached_servers = 192.168.73.171:11211
                auth_type = password
                project_domain_name = Default
                user_domain_name = Default
                project_name = service
                username = nova
                password = NOVA_PASS  # nova 유저 생성 시 입력한 비밀번호
                
                # ────────────────────────────────
                # [service_user] 설정
                # controller → 호스트명으로 변경 가능 (혹은 IP)
                [service_user]
                send_service_user_token = true
                auth_url = http://mncsvrt06:5000/
                auth_strategy = keystone
                auth_type = password
                project_domain_name = Default
                project_name = service
                user_domain_name = Default
                username = nova
                password = NOVA_PASS  # nova 유저 생성 시 입력한 비밀번호
                
                # ────────────────────────────────
                # [vnc] 설정
                [vnc]
                enabled = true
                server_listen = 0.0.0.0
                server_proxyclient_address = $my_ip
                novncproxy_base_url = http://192.168.73.171:6080/vnc_auto.html  # Controller Node IP
                
                # ────────────────────────────────
                # [oslo_concurrency] 설정
                [oslo_concurrency]
                lock_path = /var/lib/nova/tmp
                
                # ────────────────────────────────
                # [placement] 설정
                # controller → 호스트명으로 변경 가능 (혹은 IP)
                [placement]
                region_name = RegionOne
                project_domain_name = Default
                project_name = service
                auth_type = password
                user_domain_name = Default
                auth_url = http://mncsvrt06:5000/v3
                username = placement
                password = PLACEMENT_PASS  # placement 유저 생성 시 입력한 비밀번호
                ```
                
            - 시스템 재 시작
                
                ```bash
                service nova-compute restart
                ```
                
        - 검증 (Controller Node에서 실행)
            
            ```bash
            # 관리자 자격 증명
            . admin-openrc
            
            openstack compute service list --service nova-compute
            +----+-------+--------------+------+-------+---------+----------------------------+
            | ID | Host  | Binary       | Zone | State | Status  | Updated At                 |
            +----+-------+--------------+------+-------+---------+----------------------------+
            | 1  | node1 | nova-compute | nova | up    | enabled | 2017-04-14T15:30:44.000000 |
            +----+-------+--------------+------+-------+---------+----------------------------+
            
            openstack compute service list
            +----+--------------------+------------+----------+---------+-------+----------------------------+
            | Id | Binary             | Host       | Zone     | Status  | State | Updated At                 |
            +----+--------------------+------------+----------+---------+-------+----------------------------+
            |  1 | nova-scheduler     | controller | internal | enabled | up    | 2016-02-09T23:11:15.000000 |
            |  2 | nova-conductor     | controller | internal | enabled | up    | 2016-02-09T23:11:16.000000 |
            |  3 | nova-compute       | compute1   | nova     | enabled | up    | 2016-02-09T23:11:20.000000 |
            +----+--------------------+------------+----------+---------+-------+----------------------------+
            # 위 처럼 나오면 됨
            ```
            
        - `nova-compute` 서비스 등록 (cell 매핑)
            
            > NOTE
> Controller Node의 nova.conf 의 아래 설정을 하고 난뒤 300초 후에도 cell이 매핑되지 않으면 아래 내용 진행
>
> **[scheduler]
> discover_hosts_in_cells_interval = 300**

            
            현재 등록된 셀 호스트 목록 확인
            
            ```bash
            sudo nova-manage cell_v2 list_hosts
            
            +-----------+--------------------------------------+-----------+
            | Cell Name |              Cell UUID               |  Hostname |
            +-----------+--------------------------------------+-----------+
            |   cell1   | f09f96aa-1e1e-4a35-9fd1-1698ff5c3386 | mncsvrt06 |
            ```
            
            `nova-compute` 노드를 셀에 매핑
            
            ```bash
            # nova가 설치된 compute 노드를 자동으로 등록함.
            nova-manage cell_v2 discover_hosts --verbose
            
            # 매핑확인
            sudo nova-manage cell_v2 list_hosts
            +-----------+--------------------------------------+-----------+
            | Cell Name |              Cell UUID               |  Hostname |
            +-----------+--------------------------------------+-----------+
            |   cell1   | f09f96aa-1e1e-4a35-9fd1-1698ff5c3386 | mncsvrt06 |
            |   cell1   | f09f96aa-1e1e-4a35-9fd1-1698ff5c3386 | mncsvrt07 |
            |   cell1   | f09f96aa-1e1e-4a35-9fd1-1698ff5c3386 | mncsvrt08 |
            +-----------+--------------------------------------+-----------+
            ```
