- Cinder - BlockStorage(NFS)
        
        ---
        
        ### Controller Node 구성
        
        DB생성 (controller node)
        
        ```sql
        mysql
        
        CREATE DATABASE cinder;
        
        GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'CINDER_DBPASS';
        GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'CINDER_DBPASS';
        ```
        
        > `CINDER_DBPASS`는 원하는 비밀번호로 변경
        > 
        
        서비스 자격 증명 생성 (controller node)
        
        ```bash
        openstack user create --domain default --password-prompt cinder
        
        #User Password: 비밀번호 입력
        #Repeat User Password:
        
        # admin사용자 에게 역할을 추가 cinder.
        openstack role add --project service --user cinder admin
        
        # 서비스 엔터티 생성
        openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3
          
        # 블록 스토리지 서비스 API 엔드포인트 생성 -> controller는 호스트네임으로 수정
        openstack endpoint create --region RegionOne volumev3 public http://controller:8776/v3/%\(project_id\)s
        openstack endpoint create --region RegionOne volumev3 internal http://controller:8776/v3/%\(project_id\)s
        openstack endpoint create --region RegionOne volumev3 admin http://controller:8776/v3/%\(project_id\)s
        ```
        
        패키지 설치 (controller node)
        
        ```bash
        sudo apt install -y cinder-api cinder-scheduler cinder-backup
        ```
        
        ---
        
        ### Storage Node 및 backup 서비스 구성(현재는 별도의 Strorage Node를 구성하지 않고 Compute Node에 구성함)
        
        패키지 설치
        
        ```bash
        sudo apt install -y nfs-kernel-server thin-provisioning-tools cinder-volume tgt
        ```
        
        ### 1. 디스크 파티션 포맷
        
        ---
        
        ### 기존 LVM 삭제 (존재한다면)
        
        ```bash
        sudo lvremove -f cinder-volumes/cinder-volumes-pool
        sudo vgremove -f cinder-volumes
        sudo pvremove -f /dev/sdb2
        ```
        
        ### 현재 파티션 및 파일시스템 확인
        
        ```bash
        sudo mkfs.xfs /dev/sdb2
        ```
        
        ### 파티션 적용 및 마운트
        
        ```bash
        # 마운트 지점 생성
        sudo mkdir -p /storage
        # 마운트
        sudo mount /dev/sdb2 /storage
        # 부팅 시 자동 마운트 설정
        echo '/dev/sdb2 /storage xfs defaults 0 0' | sudo tee -a /etc/fstab
        ```
        
        ### 백업 Swift_url 확인
        
        ```bash
        # controller node 에서 실행 (swift 구성 되어 있어야함)
        openstack catalog show object-store
        
        # 아래 내용 cinder.conf의 backup_swift_url에 사용 (public url)
        #RegionOne                                                                  |
        #|           |   public: http://mncsvrt06:8070/v1/AUTH_6f4e2d25bd5648b48689f0bc1f0e9f62
        
        ```
        
        ### cinder.conf 설정(controller, compute 동일)
        
        ```bash
        [DEFAULT]
        rootwrap_config = /etc/cinder/rootwrap.conf
        api_paste_confg = /etc/cinder/api-paste.ini
        volume_name_template = volume-%s
        volume_group = cinder-volumes
        verbose = True
        auth_strategy = keystone
        state_path = /var/lib/cinder
        lock_path = /var/lock/cinder
        volumes_dir = /var/lib/cinder/volumes
        enabled_backends = nfs
        transport_url = rabbit://openstack:mnc_rabbit@mncsvrt02
        auth_strategy = keystone
        my_ip = 192.168.73.167
        glance_api_servers = http://mncsvrt02:9292
        backup_driver = cinder.backup.drivers.swift.SwiftBackupDriver
        backup_swift_url = http://mncsvrt02:8070/v1/AUTH_7497da54fa3f40e2a65873559dcc224c
        
        [database]
        connection = mysql+pymysql://cinder:mnc1!@mncsvrt02/cinder
        
        [keystone_authtoken]
        www_authenticate_uri = http://mncsvrt02:5000
        auth_url = http://mncsvrt02:5000
        memcached_servers = mncsvrt02:11211
        auth_type = password
        project_domain_name = default
        user_domain_name = default
        project_name = service
        username = cinder
        password = mnc1!
        service_token_roles = service
        service_token_roles_required = true
        
        [oslo_concurrency]
        lock_path = /var/lib/cinder/tmp
        
        [nfs]
        volume_driver = cinder.volume.drivers.nfs.NfsDriver
        volume_backend_name = nfs
        nfs_shares_config = /etc/cinder/nfs.conf
        nfs_mount_point_base = $state_path/mnt
        nfs_snapshot_support = True
        nas_secure_file_operations = false
        # 단일 서버 운영시 볼륨 스냅샷 에러 발생하면 아래 주석 해제
        #nas_secure_file_permissions = false
        #nfs_mount_options = rw,vers=3,proto=tcp,sec=sys,uid=$(id -u cinder),gid=$(id -g cinder)
        
        [oslo_messaging_notifications]
        driver = messagingv2
        transport_url = rabbit://openstack:mnc_rabbit@mncsvrt02
        topics = notifications
        
        [service_user]
        send_service_user_token = true
        auth_url = http://mncsvrt02:5000/
        auth_strategy = keystone
        auth_type = password
        project_domain_name = Default
        project_name = service
        user_domain_name = Default
        username = cinder
        password = mnc1!
        ```
        
        ### **/etc/exports 설정 (단일 서버)**
        
        ```bash
        /storage 192.168.73.167(rw,sync,no_root_squash)
        ```
        
        ### **/etc/exports 설정 (controller, compute 서버 구분)**
        
        ```bash
        /storage 192.168.73.171(rw,sync,no_root_squash) \
                 192.168.73.172(rw,sync,no_root_squash) \
                 192.168.73.173(rw,sync,no_root_squash)
        ```
        
        ### NFS 서비스 재시작
        
        ```bash
        sudo exportfs -ra
        sudo systemctl restart nfs-server
        ```
        
        ### **/etc/cinder/nfs.conf 생성 (단일 서버)**
        
        ```bash
        192.168.73.167:/storage
        ```
        
        ### **/etc/cinder/nfs.conf 생성 (controller, compute 서버 구분)**
        
        ```bash
        192.168.73.171:/storage
        192.168.73.172:/storage
        192.168.73.173:/storage
        ```
        
        ### 폴더 권한 설정 (폴더 없으면 있으면 무시)
        
        ```bash
        # /storage 디렉토리의 소유자를 cinder로 변경 (1단계)
        sudo chown cinder:cinder /storage
        
        # /storage 하위 전체에 대해 소유자 재귀 변경
        sudo chown -R cinder:cinder /storage
        
        # /var/lib/cinder/mnt 하위 모든 마운트 지점들의 소유자 재귀 변경
        sudo chown -R cinder:cinder /var/lib/cinder/mnt/*
        ```
        
        ### DB 동기화(controller node)
        
        ```bash
        sudo -s /bin/sh -c "cinder-manage db sync" cinder
        ```
        
        ### cinder 서비스 재시작 및 로그 확인
        
        ```bash
        sudo systemctl restart cinder-volume
        sudo tail -f /var/log/cinder/cinder-volume.log
        ```
