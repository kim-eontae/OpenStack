- Keystone - Identity service
        - **역할**
            
            인증(로그인) 및 권한(토큰, 역할) 관리
            
        - **설명**
            
            사용자, 프로젝트, 서비스 등의 신원을 확인하고 토큰을 발급하여 OpenStack 전체의 **접근 제어를 담당**하는 핵심 서비스
            
        - Create DataBase
            
            ```bash
            mysql
            
            CREATE DATABASE keystone;
            
            GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';
            
            GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';
            
            # KEYSTONE_DBPASS 는 원하는 값으로 변경
            ```
            
        - Install packages and Edit configure
            
            ```bash
            sudo apt install keystone
            
            # Edit keystone.conf
            sudo vi /etc/keystone/keystone.conf
            
            # [database] 설정
            [database]
            connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone
            
            # [token] 설정
            [token]
            provider = fernet
            ```
            
            > vim에서 원하는 값 찾을 때: **esc → /[database] → 엔터**
            > 
        - 데이터베이스 동기화
            
            ```bash
            sudo -s /bin/sh -c "keystone-manage db_sync" keystone
            ```
            
        - Fernet 키 저장소 초기화
            
            ```bash
            sudo keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
            sudo keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
            ```
            
        - Identity 서비스 부트스트랩
            
            ```bash
            # ADMIN_PASS -> 원하는 값으로 변경
            # controller -> 호스트명으로 변경
            sudo keystone-manage bootstrap --bootstrap-password ADMIN_PASS \
              --bootstrap-admin-url http://controller:5000/v3/ \
              --bootstrap-internal-url http://controller:5000/v3/ \
              --bootstrap-public-url http://controller:5000/v3/ \
              --bootstrap-region-id RegionOne
            ```
            
        - Apache HTTP 서버 구성
            
            ```bash
            sudo vi /etc/apache2/apache2.conf
            
            # 229번 라인으로 이동
            :229 입력
            
            # 설정 변경
            ServerName controller
            ```
            
        - 서비스 재 시작
            
            ```bash
            service apache2 restart
            ```
            
        - 환경 변수 설정 (사용자 계정)
            
            ```bash
            export OS_USERNAME=admin
            export OS_PASSWORD=ADMIN_PASS
            export OS_PROJECT_NAME=admin
            export OS_USER_DOMAIN_NAME=Default
            export OS_PROJECT_DOMAIN_NAME=Default
            export OS_AUTH_URL=http://controller:5000/v3 # controller 변경 필요
            export OS_IDENTITY_API_VERSION=3
            ```
            
        - keystone ****도메인, 프로젝트, 사용자, 역할** 생성
            
            ```bash
            # 도메인 생성
            openstack domain create --description "An Example Domain" example
            
            # service 프로젝트 생성
            openstack project create --domain default --description "Service Project" service
            
            # myproject 생성
            openstack project create --domain default --description "Demo Project" myproject
            
            # 사용자 생성
            openstack user create --domain default --password-prompt myuser
            # **비밀번호 입력 -> 중요함
            
            # 역할 생성
            openstack role create myrole
            
            # 프로젝트와 사용자에게 역할 추가
            openstack role add --project myproject --user myuser myrole**
            ```
            
        - 작동 확인
            
            ```bash
            # OS_AUTH_URL및 OS_PASSWORD 환경 변수 설정 해제
            unset OS_AUTH_URL OS_PASSWORD
            
            # admin인증 토큰 요청
            openstack --os-auth-url http://controller:5000/v3 \
              --os-project-domain-name Default --os-user-domain-name Default \
              --os-project-name admin --os-username admin token issue
              # 비밀번호 입력
            
            # myuser이전에 생성한 사용자로 인증 토큰 요청
            openstack --os-auth-url http://controller:5000/v3 \
              --os-project-domain-name Default --os-user-domain-name Default \
              --os-project-name myproject --os-username myuser token issue
              # 비밀번호 입력
            ```
            
        - 사용자(관리자) 스크립트 생성
            
            ```bash
            vi admin-openrc
            # 아래 내용 복붙 후 수정
            export OS_PROJECT_DOMAIN_NAME=Default
            export OS_USER_DOMAIN_NAME=Default
            export OS_PROJECT_NAME=admin
            export OS_USERNAME=admin
            export OS_PASSWORD=ADMIN_PASS # 비밀번호 변경 해야함(keystone password)
            export OS_AUTH_URL=http://controller:5000/v3
            export OS_IDENTITY_API_VERSION=3
            export OS_IMAGE_API_VERSION=2
            ```
            
        - 일반 사용자 스크립트 생성
            
            ```bash
            vi demp-openrc
            # 아래 내용 복붙 후 수정
            export OS_PROJECT_DOMAIN_NAME=Default
            export OS_USER_DOMAIN_NAME=Default
            export OS_PROJECT_NAME=myproject
            export OS_USERNAME=myuser
            export OS_PASSWORD=DEMO_PASS # 비밀번호 변경 해야함(keystone password)
            export OS_AUTH_URL=http://controller:5000/v3
            export OS_IDENTITY_API_VERSION=3
            export OS_IMAGE_API_VERSION=2
            ```
            
        - 스크립트 사용
            
            ```bash
            . admin-openrc
            
            openstack token issue
            # 아래처럼 나오면 성공
            +------------+-----------------------------------------------------------------+
            | Field      | Value                                                           |
            +------------+-----------------------------------------------------------------+
            | expires    | 2016-02-12T20:44:35.659723Z                                     |
            | id         | gAAAAABWvjYj-Zjfg8WXFaQnUd1DMYTBVrKw4h3fIagi5NoEmh21U72SrRv2trl |
            |            | JWFYhLi2_uPR31Igf6A8mH2Rw9kv_bxNo1jbLNPLGzW_u5FC7InFqx0yYtTwa1e |
            |            | eq2b0f6-18KZyQhs7F3teAta143kJEWuNEYET-y7u29y0be1_64KYkM7E       |
            | project_id | 343d245e850143a096806dfaefa9afdc                                |
            | user_id    | ac3377633149401296f6c0d92d79dc16                                |
            +------------+-----------------------------------------------------------------+
            ```
