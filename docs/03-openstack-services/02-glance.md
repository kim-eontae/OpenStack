- Glance - Image service
        - **역할**
            
            VM 이미지를 저장하고 제공
            
        - **설명**
            
            인스턴스를 생성할 때 사용하는 **디스크 이미지(예: Ubuntu, CentOS)를 저장, 검색, 배포**하는 역할
            
        - Create DataBase
            
            ```bash
            mysql
            
            CREATE DATABASE glance;
            
            GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'GLANCE_DBPASS';
            GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'GLANCE_DBPASS';
            # GLANCE_DBPASS는 원하는 비밀번호로 변경
            ```
            
        - 관리자 자격 증명 설정
        (아래 진행되는 명령어들은 일반 사용자로 진행)
            
            ```bash
            . admin-openrc
            
            # glance 사용자 생성
            openstack user create --domain default --password-prompt glance
            
            # 사용자에게 admin 역할 추가
            openstack role add --project service --user glance admin
            
            # 서비스 엔터티 생성
            openstack service create --name glance --description "OpenStack Image" image
            
            # 이미지 서비스 API 엔드포인트 생성
            openstack endpoint create --region RegionOne image public http://controller:9292
            openstack endpoint create --region RegionOne image internal http://controller:9292
            openstack endpoint create --region RegionOne image admin http://controller:9292
            # controller 는 호스트명으로 변경
            ```
            
        - Install packages and Edit configure
            
            ```bash
            sudo apt install glance
            
            sudo vi /etc/glance/glance-api.conf
            
            # 주요 설정
            [DEFAULT]
            enabled_backends=fs:file
            transport_url = rabbit://openstack:mnc_rabbit@mncsvrt06
            
            [database]
            connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance
            
            [keystone_authtoken]
            www_authenticate_uri  = http://controller:5000
            auth_url = http://controller:5000
            memcached_servers = controller:11211
            auth_type = password
            project_domain_name = Default
            user_domain_name = Default
            project_name = service
            username = glance
            password = GLANCE_PASS
            
            [paste_deploy]
            flavor = keystone
            
            [glance_store]
            default_backend = fs
            
            [fs]
            filesystem_store_datadir = /var/lib/glance/images/
            
            [oslo_limit]
            auth_url = http://controller:5000
            auth_type = password
            user_domain_id = default
            username = glance
            system_scope = all
            password = GLANCE_PASS
            endpoint_id = 340be3625e9b4239a6415d034e98aace **# endpoint_id는 endpoint create 명령으로 생성된 public endpoint ID로 변경**
            region_name = RegionOne
            ```
            
            > ⚠️ `GLANCE_PASS`는 glance 사용자 생성시 입력한 비밀번호로 설정
            > 
        - 이미지 서비스 DB 동기화
            
            ```bash
            sudo -s /bin/sh -c "glance-manage db_sync" glance
            ```
            
        - 서비스 재시작
            
            ```bash
            service glance-api restart
            ```
            
        - 작동 확인
            
            ```bash
            # 관리자 환경 로드
            . admin-openrc
            
            # 테스트 이미지 다운로드
            sudo wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
            
            # glance 이미지 업로드
            glance image-create --name "cirros" \
              --file cirros-0.4.0-x86_64-disk.img \
              --disk-format qcow2 --container-format bare \
              --visibility=public
            
            # 업로드된 이미지 확인
            glance image-list
            ```
            
            출력 예시
            
            ```bash
            +--------------------------------------+--------+--------+
            | ID                                   | Name   | Status |
            +--------------------------------------+--------+--------+
            | 38047887-61a7-41ea-9b49-27987d5e8bb9 | cirros | active |
            +--------------------------------------+--------+--------+
            ```
