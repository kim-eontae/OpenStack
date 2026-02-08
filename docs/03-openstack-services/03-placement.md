- Placement - Placement service
        - **역할**
            
            가상 리소스(가상 CPU, RAM, 디스크 등)의 사용 가능 여부와 할당 상태를 추적
            
        - **설명**
            
            인스턴스를 어디에 생성할지 결정하기 위해 **노드 자원 상태를 추적하고 보고**하는 서비스
            
            → Nova와 연동되어 자원 스케줄링에 사용됨
            
        - Create DataBase
            
            ```bash
            mysql
            
            CREATE DATABASE placement;
            
            GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'PLACEMENT_DBPASS';
            GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'PLACEMENT_DBPASS';
            #PLACEMENT_DBPASS 는 원하걸로 변경해야함
            ```
            
        - Configure User and Endpoints 
        (아래 진행되는 명령어들은 일반 사용자로 진행)
            
            ```bash
            # 관리자 자격 증명 로드
            . admin-openrc
            
            # placement 사용자 생성
            openstack user create --domain default --password-prompt placement
            
            # 역할 추가
            openstack role add --project service --user placement admin
            
            # 서비스 엔터티 생성
            openstack service create --name placement --description "Placement API" placement
            
            # 서비스 API 엔드포인트 생성
            openstack endpoint create --region RegionOne placement public http://controller:8778
            openstack endpoint create --region RegionOne placement internal http://controller:8778
            openstack endpoint create --region RegionOne placement admin http://controller:8778
            # controller 는 호스트명으로 변경
            ```
            
        - Install packages and Edit configure
            
            ```bash
            sudo apt install placement-api
            
            sudo vi /etc/placement/placement.conf
            
            # 주요 설정
            [placement_database]
            connection = mysql+pymysql://placement:PLACEMENT_DBPASS@controller/placement
            
            [api]
            auth_strategy = keystone
            
            [keystone_authtoken]
            auth_url = http://controller:5000/v3
            memcached_servers = controller:11211
            auth_type = password
            project_domain_name = Default
            user_domain_name = Default
            project_name = service
            username = placement
            password = PLACEMENT_PASS
            ```
            
            > ⚠️ `PLACEMENT_PASS`는 placement 사용자 생성시 입력한 비밀번호로 설정
            > 
        - placement 서비스 서비스 DB 동기화
            
            ```bash
            sudo -s /bin/sh -c "placement-manage db sync" placement
            ```
            
        - 서비스 재시작
            
            ```bash
            service apache2 restart
            ```
            
        - 작동 확인
            
            ```bash
            # 관리자 환경 로드
            . admin-openrc
            
            # root로 실행
            placement-status upgrade check
            
            # 플러그인 설치
            pip3 install osc-placement
            
            # 테스트
            openstack --os-placement-api-version 1.2 resource class list --sort-column name
            openstack --os-placement-api-version 1.6 trait list --sort-column name
            ```
            
            > ⚠️ 어떤 값이라도 나오면 설치가 성공적으로 완료된것
            >
