- Swift - ObjectStorage
        
        ---
        
        ### Controller Node 구성
        
        - 서비스 자격 증명 생성
            
            ```bash
            openstack user create --domain default --password-prompt swift
            
            #User Password: 비밀번호 입력
            #Repeat User Password:
            
            # admin사용자 에게 역할을 추가 swift.
            openstack role add --project service --user swift admin
            
            # 서비스 엔터티 생성
            openstack service create --name swift --description "OpenStack Object Storage" object-store
              
            # 블록 스토리지 서비스 API 엔드포인트 생성 -> controller는 호스트네임으로 수정
            openstack endpoint create --region RegionOne object-store public http://controller:8070/v1/AUTH_%\(project_id\)s 
            openstack endpoint create --region RegionOne object-store internal http://controller:8070/v1/AUTH_%\(project_id\)s
            openstack endpoint create --region RegionOne object-store admin http://controller:8070/v1
            ```
            
        - 패키지 설치
            
            ```bash
            sudo apt-get install swift swift-proxy python3-swiftclient \
              python3-keystoneclient python3-keystonemiddleware \
              memcached
            ```
            
        - 디렉토리 생성 및 구성
            
            ```bash
            mkdir /etc/swift
            
            # Object Storage 소스 저장소에서 프록시 서비스 구성 파일 가져오기
            curl -o /etc/swift/proxy-server.conf https://opendev.org/openstack/swift/raw/branch/master/etc/proxy-server.conf-sample
            
            # 설정 파일 수정
            vi /etc/swift/proxy-server.conf
            
            [DEFAULT]
            bind_port = 8070
            swift_dir = /etc/swift
            user = swift
            log_dir = /var/log/proxy
            
            [pipeline:main]
            pipeline = catch_errors gatekeeper healthcheck proxy-logging cache container_sync bulk ratelimit authtoken keystoneauth container-quotas account-quotas slo dlo versioned_writes proxy-logging proxy-server
            
            [app:proxy-server]
            use = egg:swift#proxy
            account_autocreate = True
            
            [filter:keystoneauth]
            use = egg:swift#keystoneauth
            operator_roles = admin,user
            
            [filter:authtoken]
            paste.filter_factory = keystonemiddleware.auth_token:filter_factory
            www_authenticate_uri = http://mncsvrt06:5000
            auth_url = http://mncsvrt06:5000
            memcached_servers = mncsvrt06:11211
            auth_type = password
            project_domain_id = default
            user_domain_id = default
            project_name = service
            username = swift
            password = mnc1!
            delay_auth_decision = True
            
            [filter:cache]
            use = egg:swift#memcache
            memcache_servers = mncsvrt06:11211
            ```
            
        
        ---
        
        ### Storage Node 및 backup 서비스 구성(현재는 별도의 Strorage Node를 구성하지 않고 Compute Node에 구성함)
        
        - 패키지 설치
            
            ```bash
            sudo apt-get install xfsprogs rsync swift swift-account swift-container swift-object
            ```
            
        - 파티션 구성
            
            ```bash
            # XFS로 포맷
            sudo mkfs.xfs /dev/sdb
            sudo mkfs.xfs /dev/sdc
            
            # 디렉토리 생성
            sudo mkdir -p /srv/node/sdb
            sudo mkdir -p /srv/node/sdc
            
            # UUID 지정
            lsblk -f
            
            sudo vi /etc/fstab
            UUID="<UUID-from-output-above>" /srv/node/sdb xfs noatime 0 2
            UUID="<UUID-from-output-above>" /srv/node/sdc xfs noatime 0 2
            
            # 장치 마운트
            sudo mount /srv/node/sdb
            sudo mount /srv/node/sdc
            ```
            
        - 설정 파일 수정
            
            ```bash
            sudo vi /etc/rsyncd.conf
            
            uid = swift
            gid = swift
            log file = /var/log/rsyncd.log
            pid file = /var/run/rsyncd.pid
            address = 192.168.73.171 # swift를 구성할 서버의 아이피
            
            [account]
            max connections = 2
            path = /srv/node/
            read only = False
            lock file = /var/lock/account.lock
            
            [container]
            max connections = 2
            path = /srv/node/
            read only = False
            lock file = /var/lock/container.lock
            
            [object]
            max connections = 2
            path = /srv/node/
            read only = False
            lock file = /var/lock/object.lock
            ```
            
            `sudo vi /etc/default/rsync`
            
            ```bash
            RSYNC_ENABLE=true
            ```
            
        - 서비스 재 시작
            
            ```bash
            service rsync start
            ```
            
        - Object Storage 리포지토리에서 서비스 구성 파일 가져오기
            
            ```bash
            sudo curl -o /etc/swift/account-server.conf https://opendev.org/openstack/swift/raw/branch/master/etc/account-server.conf-sample
            sudo curl -o /etc/swift/container-server.conf https://opendev.org/openstack/swift/raw/branch/master/etc/container-server.conf-sample
            sudo curl -o /etc/swift/object-server.conf https://opendev.org/openstack/swift/raw/branch/master/etc/object-server.conf-sample
            ```
            
        - 설정 파일 수정
            
            ```bash
            sudo vi /etc/swift/account-server.conf
            [DEFAULT]
            bind_ip = 192.168.73.171
            bind_port = 6202
            user = swift
            swift_dir = /etc/swift
            devices = /srv/node
            mount_check = true
            
            [pipeline:main]
            pipeline = healthcheck recon account-server
            
            [filter:recon]
            use = egg:swift#recon
            recon_cache_path = /var/cache/swift
            
            sudo vi /etc/swift/container-server.conf 
            [DEFAULT]
            bind_ip = 192.168.73.171
            bind_port = 6201
            user = swift
            swift_dir = /etc/swift
            devices = /srv/node
            mount_check = true
            
            [pipeline:main]
            pipeline = healthcheck recon container-server
            
            [filter:recon]
            use = egg:swift#recon
            recon_cache_path = /var/cache/swift
            
            sudo vi /etc/swift/object-server.conf 
            [DEFAULT]
            bind_ip = 192.168.73.171
            bind_port = 6200
            user = swift
            swift_dir = /etc/swift
            devices = /srv/node
            mount_check = true
            
            [pipeline:main]
            pipeline = healthcheck recon object-server
            
            [filter:recon]
            use = egg:swift#recon
            recon_cache_path = /var/cache/swift
            recon_lock_path = /var/lock
            ```
            
        - 디렉토리 생성 및 권한 설정
            
            ```bash
            sudo chown -R swift:swift /srv/node
            
            sudo mkdir -p /var/cache/swift
            sudo chown -R root:swift /var/cache/swift
            sudo chmod -R 775 /var/cache/swift
            ```
            
        
        ---
        
        ### Object Storage 계정, 컨테이너, 객체 링 생성
        
        - 명령어 구조
            
            ```bash
            swift-ring-builder <builder 파일> create <part-power> <replica> <min-part-hours>
            ```
            
            | 인자 | 의미 | 설명 |
            | --- | --- | --- |
            | 10 | part-power | - Swift ring의 파티션 수를 결정 - `2^10 = 1024`개의 파티션 생성됨 - 클러스터 크기에 따라 보통 10~14 사용 - 작을수록 rebalance 속도 빠름 (테스트 환경용) |
            | 2 | replica | - 각 파티션을 몇 개의 디바이스에 복제할지 설정 - 장비(파티션)가 2대이면 2 사용 (device 수 이상 안됨) - 운영 환경에선 3 이상 사용 권장 |
            | 1 | min-part-hours | - 파티션이 다른 디바이스로 재배치될 수 있는 **최소 시간 간격(시간)** - ring 변경 시 데이터 이동을 최소화하기 위한 제어 수단 - 1이면 1시간 안에는 동일 파티션이 다시 이동되지 않음 |
            
            > `part-power`가 크면 더 많은 파티션을 생성 → 더 정교한 데이터 분산 가능하지만 **ring 파일 커짐**
            > 
            > 
            > `replica`는 반드시 `device 수 ≤ replica 수` 조건을 만족해야 함 (안 지키면 `rebalance` 실패)
            > 
            > `min-part-hours`는 일반적으로 운영환경에서는 1~24 정도를 사용
            > 
        - 디렉토리 접근
            
            ```bash
            cd /etc/swift
            ```
            
        - account.builder 생성
            
            ```bash
            swift-ring-builder account.builder create 10 2 1
            
            swift-ring-builder account.builder add \
              --region 1 --zone 1 --ip 192.168.73.171 --port 6202 --device sdb2 --weight 100
            
            swift-ring-builder account.builder add \
              --region 1 --zone 2 --ip 192.168.73.173 --port 6202 --device sdb2 --weight 100
              
            # 내용 확인
            swift-ring-builder account.builder
            #account.builder, build version 3, id b51912b84102407eb532c4e1ba8377e6
            #1024 partitions, 2.000000 replicas, 1 regions, 2 zones, 2 devices, 0.00 balance, 0.00 dispersion
            #The minimum number of hours before a partition can be reassigned is 1 (0:00:00 remaining)
            #The overload factor is 0.00% (0.000000)
            #Ring file account.ring.gz is up-to-date
            #Devices:   id region zone     ip address:port replication ip:port  name weight partitions balance flags meta
            #            0      1    1 192.168.73.171:6202 192.168.73.171:6202  sdb2 100.00       1024    0.00       
            #            1      1    2 192.168.73.173:6202 192.168.73.173:6202  sdb2 100.00       1024    0.00  
            
            # 배포 확인
            swift-ring-builder account.builder rebalance 
            ```
            
        - container.builder 생성
            
            ```bash
            swift-ring-builder container.builder create 10 2 1
            
            swift-ring-builder container.builder add \
              --region 1 --zone 1 --ip 192.168.73.171 --port 6201 --device sdb2 --weight 100
            
            swift-ring-builder container.builder add \
              --region 1 --zone 2 --ip 192.168.73.173 --port 6201 --device sdb2 --weight 100
              
            # 내용 확인
            swift-ring-builder container.builder
            #container.builder, build version 3, id b51912b84102407eb532c4e1ba8377e6
            #1024 partitions, 2.000000 replicas, 1 regions, 2 zones, 2 devices, 0.00 balance, 0.00 dispersion
            #The minimum number of hours before a partition can be reassigned is 1 (0:00:00 remaining)
            #The overload factor is 0.00% (0.000000)
            #Ring file account.ring.gz is up-to-date
            #Devices:   id region zone     ip address:port replication ip:port  name weight partitions balance flags meta
            #            0      1    1 192.168.73.171:6202 192.168.73.171:6202  sdb2 100.00       1024    0.00       
            #            1      1    2 192.168.73.173:6202 192.168.73.173:6202  sdb2 100.00       1024    0.00  
            
            # 배포 확인
            swift-ring-builder container.builder rebalance 
            ```
            
        - object.builder 생성
            
            ```bash
            swift-ring-builder object.builder create 10 2 1
            
            swift-ring-builder object.builder add \
              --region 1 --zone 1 --ip 192.168.73.171 --port 6200 --device sdb2 --weight 100
            
            swift-ring-builder object.builder add \
              --region 1 --zone 2 --ip 192.168.73.173 --port 6200 --device sdb2 --weight 100
              
            # 내용 확인
            swift-ring-builder object.builder
            #object.builder, build version 3, id b51912b84102407eb532c4e1ba8377e6
            #1024 partitions, 2.000000 replicas, 1 regions, 2 zones, 2 devices, 0.00 balance, 0.00 dispersion
            #The minimum number of hours before a partition can be reassigned is 1 (0:00:00 remaining)
            #The overload factor is 0.00% (0.000000)
            #Ring file account.ring.gz is up-to-date
            #Devices:   id region zone     ip address:port replication ip:port  name weight partitions balance flags meta
            #            0      1    1 192.168.73.171:6202 192.168.73.171:6202  sdb2 100.00       1024    0.00       
            #            1      1    2 192.168.73.173:6202 192.168.73.173:6202  sdb2 100.00       1024    0.00  
            
            # 배포 확인
            swift-ring-builder object.builder rebalance 
            ```
            
        
        ---
        
        ### 설치 확인
        
        - Object Storage 소스 저장소에서 파일 가져오기
            
            ```bash
            sudo curl -o /etc/swift/swift.conf \
              https://opendev.org/openstack/swift/raw/branch/master/etc/swift.conf-sample
            ```
            
        - 설정 파일 편집
            
            ```bash
            sudo vi /etc/swift/swift.conf
            # HASH_PATH_SUFFIX를 고유한 아무 값으로 변경
            [swift-hash]
            swift_hash_path_suffix = HASH_PATH_SUFFIX
            swift_hash_path_prefix = HASH_PATH_PREFIX
            
            [storage-policy:0]
            name = Policy-0
            default = yes
            aliases = yellow, orange
            ```
            
        - 디렉토리 권한 부여
            
            ```bash
            sudo chown -R root:swift /etc/swift
            ```
            
        - `internal-client.conf 파일 생성`
            
            ```bash
            vi internal-client.conf
            
            [DEFAULT]
            swift_dir = /etc/swift
            user = swift
            
            [pipeline:main]
            pipeline = catch_errors proxy-logging cache proxy-server
            
            [app:proxy-server]
            use = egg:swift#proxy
            account_autocreate = true
            
            [filter:cache]
            use = egg:swift#memcache
            
            [filter:proxy-logging]
            use = egg:swift#proxy_logging
            
            [filter:catch_errors]
            use = egg:swift#catch_errors
            ```
            
        - 서비스 재 시작
            
            ```bash
            service memcached restart
            service swift-proxy restart
            
            sudo systemctl restart swift-*
            ```
            
        
        ---
        
        ### 백업 스토리지 URL 확인
        
        ```bash
        openstack catalog show object-store
        +-----------+----------------------------------------------------------------------------+
        | Field     | Value                                                                      |
        +-----------+----------------------------------------------------------------------------+
        | endpoints | RegionOne                                                                  |
        |           |   admin: http://mncsvrt06:8070/v1/AUTH_6f4e2d25bd5648b48689f0bc1f0e9f62    |
        |           | RegionOne                                                                  |
        |           |   public: http://mncsvrt06:8070/v1/AUTH_6f4e2d25bd5648b48689f0bc1f0e9f62   |
        |           | RegionOne                                                                  |
        |           |   internal: http://mncsvrt06:8070/v1/AUTH_6f4e2d25bd5648b48689f0bc1f0e9f62 |
        |           |                                                                            |
        | id        | 2bdfd4b823e34fccb4c8ccb8889422f0                                           |
        | name      | swift                                                                      |
        | type      | object-store                                                               |
        +-----------+----------------------------------------------------------------------------+
        ```
        
    - Instance GPU 할당 VFIO-PCI GPU Passthrough 설정 및 GPU 드라이버 설치
        
        > NOTE
> VFIO-PCI를 이용한 GPU Passthrough 설정을 통해 Compute Node의 GPU 장치는 VFIO 드라이버로 바인딩되어 일반 커널 드라이버(nvidia 등)에서 분리된다. 이로 인해 Compute Node에서는 해당 GPU를 직접 사용할 수 없으며, 지정된 OpenStack 인스턴스에만 전용으로 할당되어 사용된다.

        
        ### 1. GDM (그래픽 데스크탑) 중지
        
        VFIO 바인딩을 위해 GPU를 점유 중인 Xorg 종료
        
        ```bash
        sudo systemctl stop gdm
        ```
        
        > Ubuntu 환경에 따라 gdm3일 수도 있음:
        > 
        
        ```bash
        sudo systemctl stop gdm3
        ```
        
        GPU 사용 여부 확인:
        
        ```bash
        nvidia-smi
        # "No supported GPUs found"는 정상
        ```
        
        ---
        
        ### 2. GPU 전체 PCI 주소 확인
        
        ```bash
        lspci -nnD | grep -i nvidia
        ```
        
        예시 결과:
        
        ```
        0000:18:00.0 3D controller [0302]: NVIDIA Corporation TU104GL [Tesla T4] [10de:1eb8] ...
        0000:5e:00.0 3D controller [0302]: NVIDIA Corporation TU104GL [Tesla T4] [10de:1eb8] ...
        ...
        ```
        
        ---
        
        ### 3. VFIO 바인딩 스크립트 작성
        
        ```bash
        sudo vi /usr/local/bin/vfio-bind.sh
        ```
        
        ```bash
        #!/bin/bash
        
        modprobe vfio
        modprobe vfio_pci
        modprobe vfio_iommu_type1
        
        echo "10de 1eb8" > /sys/bus/pci/drivers/vfio-pci/new_id 2>/dev/null
        
        GPUS=("0000:18:00.0" "0000:5e:00.0" "0000:86:00.0" "0000:af:00.0")
        
        for GPU in "${GPUS[@]}"; do
            echo "$GPU" > /sys/bus/pci/drivers/nvidia/unbind 2>/dev/null
            echo "$GPU" > /sys/bus/pci/drivers/vfio-pci/bind 2>/dev/null
        done
        
        {
            echo "VFIO 바인딩 상태 ($(date)):"
            for GPU in "${GPUS[@]}"; do
                lspci -nnk -s "$GPU"
            done
        } > /var/log/vfio-bind.log
        ```
        
        실행 권한 부여:
        
        ```bash
        sudo chmod +x /usr/local/bin/vfio-bind.sh
        ```
        
        ---
        
        ### 4. systemd 서비스 등록
        
        ```bash
        sudo vi /etc/systemd/system/vfio-bind.service
        ```
        
        ```bash
        [Unit]
        Description=NVIDIA GPU VFIO-PCI 바인딩
        After=syslog.target
        
        [Service]
        Type=oneshot
        ExecStart=/usr/local/bin/vfio-bind.sh
        RemainAfterExit=yes
        
        [Install]
        WantedBy=multi-user.target
        ```
        
        활성화 및 즉시 실행:
        
        ```bash
        sudo systemctl daemon-reexec
        sudo systemctl daemon-reload
        sudo systemctl enable vfio-bind.service
        sudo systemctl start vfio-bind.service
        ```
        
        상태 확인:
        
        ```bash
        sudo systemctl status vfio-bind.service
        cat /var/log/vfio-bind.log
        lspci -nnk -d 10de:1eb8
        # Kernel driver in use: vfio-pci 나오면 성공
        ```
        
        ---
        
        ### VFIO 바인딩이 실패할 때 대처 방법
        
        `nouveau` 완전 블랙리스트 및 커널 파라미터 설정 적용
        
        1. **modprobe 블랙리스트 설정**
        
        ```bash
        sudo vi /etc/modprobe.d/blacklist-nouveau.conf
        # 내용 추가
        blacklist nouveau
        options nouveau modeset=0
        ```
        
        ### 2. **initramfs 재생성**
        
        ```bash
        sudo update-initramfs -u
        ```
        
        3. **GRUB 커널 파라미터 설정 (중요)**
        
        ```bash
        sudo vi /etc/default/grub
        
        # 아래 내용 수정
        GRUB_CMDLINE_LINUX_DEFAULT="quiet splash \
        intel_iommu=on iommu=pt \
        vfio-pci.ids=10de:1eb8 \
        modprobe.blacklist=nouveau,nvidia,nvidiafb \
        nouveau.modeset=0"
        
        # 저장
        sudo update-grub
        ```
        
        1.  **vfio 모듈 옵션 및 soft-dependency 선언**
        
        ```bash
        sudo tee /etc/modprobe.d/vfio.conf <<'EOF'
        # Tesla T4 (10de:1eb8) 을 vfio-pci로 바인딩
        options vfio-pci ids=10de:1eb8
        
        # nouveau · nvidia 모듈보다 vfio-pci가 먼저 로드되도록
        softdep nouveau pre: vfio-pci
        softdep nvidia pre: vfio-pci
        EOF
        ```
        
        1. **initramfs 안에 vfio 모듈 강제 포함**
        
        ```bash
        sudo tee -a /etc/initramfs-tools/modules <<'EOF'
        # VFIO core
        vfio
        vfio_iommu_type1
        vfio_virqfd
        # 실제 PCI 바인딩
        vfio_pci ids=10de:1eb8
        EOF
        ```
        
        1. GRUB 재생성 → initramfs 재생성 → 재부팅
        
        ```bash
        sudo update-grub
        sudo update-initramfs -u -k all   # 모든 커널 버전에 적용
        sudo reboot
        ```
        
        재부팅 후 다시 확인
        
        ```bash
        lspci -nnk -d 10de:1eb8
        
        # Kernel driver in use: vfio-pci
        18:00.0 3D controller [0302]: NVIDIA Corporation TU104GL [Tesla T4] [10de:1eb8] (rev a1)
                Subsystem: NVIDIA Corporation TU104GL [Tesla T4] [10de:12a2]
                Kernel driver in use: vfio-pci
                Kernel modules: nvidiafb, nouveau
        ```
        
        ---
        
        ## 설정파일 수정
        
        Compute Node
        
        ```bash
        # /etc/nova/nova.conf
        [pci]
        device_spec = [{"vendor_id": "10de", "product_id": "1eb8"}]
        alias = {"vendor_id": "10de", "product_id": "1eb8", "device_type": "type-PF", "name": "t4"}
        
        [filter_scheduler]
        enabled_filters = ComputeFilter, ComputeCapabilitiesFilter, ImagePropertiesFilter, ServerGroupAntiAffinityFilter, ServerGroupAffinityFilter, PciPassthroughFilter, AggregateInstanceExtraSpecsFilter
        ```
        
        - `sudo systemctl restart nova-compute`
        
        Controller Node
        
        ```bash
        # /etc/nova/nova.conf
        [pci]
        alias = {"vendor_id": "10de", "product_id": "1eb8", "device_type": "type-PF", "name": "t4"}
        ```
        
        - `sudo systemctl restart nova-api nova-scheduler nova-conductor`
        
        > NOTE
> 설정파일의 device_type은 환경에 따라 다르기 때문에 /var/log/nova-compute.log 의 로그를 보고 변경필요
>
> pci_stats=[
> PciDevicePool(count=1, ..., tags={address='0000:18:00.0', **dev_type='type-PF'**, ...}),...]

        
        ---
        
        ## GPU Flavor 생성
        
        ```bash
        # flavor 생성
        openstack flavor create t4.gpu --ram 65536 --disk 100 --vcpus 16
        # 2개의 t4 gpu 할당
        openstack flavor set t4.gpu --property "pci_passthrough:alias"="t4:2"
        ```
        
        ---
        
        ## VFIO 바인딩 복구 (nvidia 드라이버 복원)
        
        ### 1. 복구 스크립트 작성
        
        ```bash
        sudo vi /usr/local/bin/vfio-unbind.sh
        ```
        
        ```bash
        #!/bin/bash
        
        modprobe nvidia
        modprobe nvidia_drm
        modprobe nvidia_uvm
        
        GPUS=("0000:18:00.0" "0000:5e:00.0" "0000:86:00.0" "0000:af:00.0")
        
        for GPU in "${GPUS[@]}"; do
            echo "$GPU" > /sys/bus/pci/drivers/vfio-pci/unbind 2>/dev/null
            echo "$GPU" > /sys/bus/pci/drivers/nvidia/bind 2>/dev/null
        done
        
        {
            echo "NVIDIA 드라이버 복구 상태 ($(date)):"
            for GPU in "${GPUS[@]}"; do
                lspci -nnk -s "$GPU"
            done
        } > /var/log/vfio-unbind.log
        ```
        
        실행 권한 부여:
        
        ```bash
        sudo chmod +x /usr/local/bin/vfio-unbind.sh
        ```
        
        ### 2. 복구 실행
        
        ```bash
        sudo /usr/local/bin/vfio-unbind.sh
        ```
        
        확인:
        
        ```bash
        lspci -nnk -d 10de:1eb8
        nvidia-smi
        ```
        
        ### 3. GDM 다시 시작 (GUI 환경 복원)
        
        ```bash
        sudo systemctl start gdm
        # 또는
        sudo systemctl start gdm3
        ```
        
    - Keystone MFA(TOTP) 활성화 절차
        
        **전제 조건**
        
        - Horizon에서 **관리자 계정**으로 MFA를 적용할 사용자 생성
        - 해당 사용자에게 **프로젝트 + Role** 권한 부여
        
        `sudo vi /etc/keystone/keystone.conf`
        
        ```bash
        [auth]
        methods = password,token,totp,application_credential
        ```
        
        **사용자에 MFA 활성화**
        
        ```bash
        # MFA 기능 활성화
        openstack user set <USER_ID_or_NAME> --enable-multi-factor-auth
        ```
        
        **시크릿 키 생성**
        
        ```bash
        # 16자 시크릿(임의) → Base32 변환 후 패딩 제거
        head -c 20 /dev/urandom | base32 | tr -d "="
        ```
        
        **Keystone Credential에 등록**
        
        ```bash
        openstack credential create --type totp <user name> <시크릿 Key>
        ```
        
        **OTP 등록 (Google Authenticator)**
        
        ```bash
        # QR 코드 URI 생성
        export QR_URI="otpauth://totp/<USERNAME>@OpenStack?secret=<시크릿 Key>&issuer=<표시 이름>"
        ```
        
        **패키지 설치**
        
        ```bash
        sudo apt install qrencode
        ```
        
        **QR확인**
        
        ```bash
        # QR 코드 터미널 출력
        echo -n "$QR_URI" | qrencode -t ANSIUTF8
        ```
        
        **MFA 규칙 최종 적용**
        
        웹(Horizon)은 password+totp로, CLI/자동화는 application_credential로 통과시키기 위해 **두 규칙을 모두 추가**합니다. (**둘 중 하나만 만족해도 인증 성공**)
        
        ```bash
        USER="<user name>"
        USER_ID=$(openstack user show "$USER" -f value -c id)
        
        openstack user set \
          --enable-multi-factor-auth \
          --multi-factor-auth-rule password,totp \
          --multi-factor-auth-rule application_credential \
          "$USER_ID"
        ```
        
        ### 적용 확인
        
        ```bash
        openstack user show "$USER" -f json | jq '.options'
        # "multi_factor_auth_enabled": true
        # "multi_factor_auth_rules": [["password","totp"],["application_credential"]]
        ```
        
        **Horizon Web 코드 수정**
        
        1. `sudo vi /var/www/horizon/openstack_dashboard/urls.py`
        
        ```bash
        from django.urls import include, path
        
        urlpatterns = [
            re_path(r'^$', views.splash, name='splash'),
            re_path(r'^api/', include(rest.urls)),
            re_path(r'^header/', views.ExtensibleHeaderView.as_view()),
            re_path(r'', horizon.base._wrapped_include(horizon.urls)),
            path('auth/', include('openstack_auth.urls')),
        ]
        ```
        
        1. `sudo vi /var/www/horizon/openstack_dashboard/local/local_settings.py` 
        
        ```bash
        OPENSTACK_KEYSTONE_MFA_TOTP_ENABLED = True
        OPENSTACK_KEYSTONE_MFA_SUPPORT = True
        OPENSTACK_KEYSTONE_MFA_ENABLED = True
        
        AUTHENTICATION_PLUGINS = [
            'openstack_auth.plugin.password.PasswordPlugin',
            'openstack_auth.plugin.totp.TotpPlugin',
            'openstack_auth.plugin.token.TokenPlugin',
        ]
        # SSO 비활성화(테스트용)
        WEBSSO_ENABLED = False
        
        # HTTP 환경이라면 아래 두 개 False
        SESSION_COOKIE_SECURE = False
        CSRF_COOKIE_SECURE = False
        ```
        
        1. 재시작
        
        ```bash
        sudo systemctl restart apache2
        ```
        
        ### **유저 생성시 사용자 MFA 활성 자동화**
        
        **`sudo vi  /var/www/horizon/openstack_dashboard/local/local_settings.py`**
        
        ```bash
        ENABLE_DEFAULT_MFA = True
        DEFAULT_MFA_RULES = [
            ["password", "totp"],
            ["application_credential"]
        ]
        ```
        
        **`sudo vi /var/www/horizon/openstack_dashboard/dashboards/identity/users/forms.py`**
        
        ```bash
        new_user = \
            api.keystone.user_create(request,
                                     name=data['name'],
                                     email=data['email'],
                                     description=desc or None,
                                     password=data['password'],
                                     project=data['project'] or None,
                                     enabled=data['enabled'],
                                     domain=domain.id,
                                     **kwargs)
        messages.success(request,
                         _('User "%s" was successfully created.')
                         % data['name'])
        
        # ----- 여기부터 추가 -----
        try:
            if getattr(settings, "ENABLE_DEFAULT_MFA", False):
                rules = getattr(settings, "DEFAULT_MFA_RULES", [["password", "totp"], ["application_credential"]])
                kc = api.keystone.keystoneclient(request)
                kc.users.update(
                    new_user.id,
                    options={
                        "multi_factor_auth_enabled": True,
                        "multi_factor_auth_rules": rules,
                    },
                )
        except Exception as exc:
            LOG.exception("Auto-MFA setup failed for user %s: %s", new_user.id, exc)
        # ----- 여기까지 추가 -----
        ```
        
        서비스 재시작
        
        ```bash
        sudo systemctl restart apache2 || sudo systemctl restart httpd
        ```
        
        ### TOTP 추가 후 새로운 CLI 로그인 방식
        
        totp 포함된 계정 생성
        
        ```bash
        export OS_AUTH_URL=http://mncsvrt06:5000/v3
        export OS_AUTH_TYPE=v3applicationcredential
        export OS_APPLICATION_CREDENTIAL_ID=<CRED_ID>
        export OS_APPLICATION_CREDENTIAL_SECRET=<CRED_ID_SECRET>
        ```
        
        ### **Horizon TOTP QR 페이지 생성**
        
        1. **최초 로그인 시**: TOTP Credential이 없으면 QR 등록(Enroll) 페이지로 이동
        2. **이미 등록된 경우**: QR 등록 건너뛰고 바로 TOTP 코드 입력 페이지로 이동
        
        패키지 설치
        
        ```bash
        sudo apt-get install -y python3-qrcode python3-pil || sudo pip3 install qrcode[pil]
        ```
        
        **설정 추가 (admin 측에서 credential을 생성하기 위한 권한)**
        
        1.  Keystone에서 **애플리케이션 크레덴셜** 생성(관리자 계정으로)
        
        ```bash
        openstack application credential create horizon_mfa_enroll \
          --role admin --unrestricted \
          -f json | jq -r '.id + " " + .secret'
          
        # 출력된 <APP_CRED_ID> <APP_CRED_SECRET> 을 메모
        ```
        
        1.  Horizon 설정
        
        **`sudo vi /var/www/horizon/openstack_dashboard/local/local_settings.py`**
        
        ```bash
        # --- Self-enroll (first login QR) ---
        MFA_SELF_ENROLL_ENABLED = True
        MFA_SELF_ENROLL_ISSUER  = "GenCloud"      # OTP 앱에 표시될 발급자
        MFA_SELF_ENROLL_DIGITS  = 6
        MFA_SELF_ENROLL_PERIOD  = 30
        
        # Keystone 연결 (이미 OPENSTACK_KEYSTONE_URL은 설정되어 있어야 함)
        # Application Credential for server-side Keystone calls
        MFA_KS_APP_CRED_ID     = "APP_CRED_ID"
        MFA_KS_APP_CRED_SECRET = "APP_CRED_SECRET"
        MFA_KS_APP_DOMAIN      = "Default"  # admin의 도메인명
        ```
        
        1. **시크릿 생성·QR 이미지 만들기·Keystone 호출**
        
        **`sudo vi /var/www/horizon/openstack_auth/plugin/self_enroll.py`**
        
        ```bash
        import base64, secrets
        from io import BytesIO
        
        import qrcode
        import requests
        from django.conf import settings
        def _resolve_domain_id(domain_hint: str, token: str) -> str:
            """
            domain_hint가 'default'나 실제 UUID든 둘 다 처리.
            """
            if not domain_hint:
                return "default"
            # UUID 같이 보이면 그대로 사용
            if len(domain_hint) in (32, 36) and '-' in domain_hint or len(domain_hint) == 32:
                return domain_hint
            base = settings.OPENSTACK_KEYSTONE_URL.rstrip('/')
            r = requests.get(f"{base}/domains?name={domain_hint}",
                             headers={'X-Auth-Token': token}, timeout=10)
            r.raise_for_status()
            domains = r.json().get("domains", [])
            if domains:
                return domains[0]["id"]
            # 못 찾으면 Keystone 기본 도메인
            return "default"
        
        def _b32_secret(n_bytes=20):
            raw = secrets.token_bytes(n_bytes)
            b32 = base64.b32encode(raw).decode('ascii').rstrip('=')
            return b32
        
        def _qr_png_b64(uri: str) -> str:
            img = qrcode.make(uri)
            buf = BytesIO()
            img.save(buf, format='PNG')
            return base64.b64encode(buf.getvalue()).decode('ascii')
        
        def _system_token():
            url = settings.OPENSTACK_KEYSTONE_URL.rstrip('/') + '/auth/tokens'
            payload = {
                "auth": {
                    "identity": {
                        "methods": ["application_credential"],
                        "application_credential": {
                            "id": settings.MFA_KS_APP_CRED_ID,
                            "secret": settings.MFA_KS_APP_CRED_SECRET
                        }
                    }
                    # scope 제거: 앱 크레덴셜이 가진 스코프가 자동 적용됨
                }
            }
            r = requests.post(url, json=payload,
                              headers={'Content-Type': 'application/json'}, timeout=10)
            r.raise_for_status()
            return r.headers['X-Subject-Token']
        
        def _get_user_id_by_name(user_name: str, domain_hint: str, token: str) -> str:
            base = settings.OPENSTACK_KEYSTONE_URL.rstrip('/')
            # 먼저 domain_hint를 id로 확정
            dom_id = _resolve_domain_id(domain_hint, token)
            # 우선: 도메인 ID + 사용자 이름
            r = requests.get(f"{base}/users?name={user_name}&domain_id={dom_id}",
                             headers={'X-Auth-Token': token}, timeout=10)
            r.raise_for_status()
            users = r.json().get("users", [])
            if users:
                return users[0]["id"]
            # fallback: 도메인 필터 없이 이름만
            r = requests.get(f"{base}/users?name={user_name}",
                             headers={'X-Auth-Token': token}, timeout=10)
            r.raise_for_status()
            users = r.json().get("users", [])
            return users[0]["id"] if users else None
        
        def ensure_user_totp_and_get_qr(user_name:str, domain_id:str):
            """
            TOTP credential이 없으면 생성하고 QR data-uri(및 otpauth URI)를 리턴.
            이미 있으면 None 리턴 → 모달 표시 안 함.
            """
            if not getattr(settings, 'MFA_SELF_ENROLL_ENABLED', False):
                return None
        
            token = _system_token()
        
            # 1) username → user_id
            user_id = _get_user_id_by_name(user_name, domain_id or "default", token)
            if not user_id:
                return None
        
            # 2) credential 존재 여부 확인
            base = settings.OPENSTACK_KEYSTONE_URL.rstrip('/')
            r = requests.get(f"{base}/credentials?type=totp&user_id={user_id}",
                             headers={'X-Auth-Token': token}, timeout=10)
            r.raise_for_status()
            if r.json().get('credentials'):
                return None  # 이미 등록되어 있으면 끝
        
            # 3) 시크릿 생성 + credential 생성
            secret = _b32_secret()
            create = {"credential": {"blob": secret, "type": "totp", "user_id": user_id}}
            r = requests.post(f"{base}/credentials", json=create,
                              headers={'X-Auth-Token': token, 'Content-Type':'application/json'},
                              timeout=10)
            r.raise_for_status()
        
            # 4) otpauth URI & QR data-uri 생성
            issuer = getattr(settings, 'MFA_SELF_ENROLL_ISSUER', 'GenCloud')
            digits = getattr(settings, 'MFA_SELF_ENROLL_DIGITS', 6)
            period = getattr(settings, 'MFA_SELF_ENROLL_PERIOD', 30)
            uri = f"otpauth://totp/{user_name}@{issuer}?secret={secret}&issuer={issuer}&digits={digits}&period={period}"
            png_b64 = _qr_png_b64(uri)
            return f"data:image/png;base64,{png_b64}", uri
        
        def has_user_totp(user_name: str, domain_id: str) -> bool:
            """해당 사용자가 이미 TOTP credential을 갖고 있는지 여부만 확인."""
            token = _system_token()
            user_id = _get_user_id_by_name(user_name, domain_id or "default", token)
            if not user_id:
                return False
            base = settings.OPENSTACK_KEYSTONE_URL.rstrip('/')
            r = requests.get(
                f"{base}/credentials?type=totp&user_id={user_id}",
                headers={'X-Auth-Token': token},
                timeout=10,
            )
            r.raise_for_status()
            return bool(r.json().get('credentials'))
        ```
        
        **로그인 분기**
        
        **`sudo vi /var/www/horizon/openstack_auth/views.py`**
        
        ```bash
        # Licensed under the Apache License, Version 2.0 (the "License");
        # you may not use this file except in compliance with the License.
        # You may obtain a copy of the License at
        #
        #    http://www.apache.org/licenses/LICENSE-2.0
        #
        # Unless required by applicable law or agreed to in writing, software
        # distributed under the License is distributed on an "AS IS" BASIS,
        # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or
        # implied.
        # See the License for the specific language governing permissions and
        # limitations under the License.
        import datetime
        import functools
        import logging
        
        from openstack_auth.plugin.self_enroll import (
            ensure_user_totp_and_get_qr,
            has_user_totp,
        )
        
        from django.conf import settings
        from django.contrib import auth
        from django.contrib.auth.decorators import login_required
        from django.contrib.auth import views as django_auth_views
        from django.contrib import messages
        from django import http as django_http
        from django.middleware import csrf
        from django import shortcuts
        from django.urls import reverse
        from django.utils import http
        from django.utils.translation import gettext_lazy as _
        from django.views.decorators.cache import never_cache
        from django.views.decorators.csrf import csrf_exempt
        from django.views.decorators.csrf import csrf_protect
        from django.views.decorators.debug import sensitive_post_parameters
        from django.views.generic import edit as edit_views
        from keystoneauth1 import exceptions as keystone_exceptions
        
        from openstack_auth import exceptions
        from openstack_auth import forms
        from openstack_auth import plugin
        from openstack_auth import user as auth_user
        from openstack_auth import utils
        
        LOG = logging.getLogger(__name__)
        
        CSRF_REASONS = [
            csrf.REASON_BAD_ORIGIN,
            csrf.REASON_NO_REFERER,
            csrf.REASON_BAD_REFERER,
            csrf.REASON_NO_CSRF_COOKIE,
            csrf.REASON_CSRF_TOKEN_MISSING,
            csrf.REASON_MALFORMED_REFERER,
            csrf.REASON_INSECURE_REFERER,
        ]
        
        def get_csrf_reason(reason):
            if not reason:
                return
        
            if reason not in CSRF_REASONS:
                reason = ""
            else:
                reason += " "
            reason += str(_("Cookies may be turned off. "
                            "Make sure cookies are enabled and try again."))
            return reason
        
        def set_logout_reason(res, msg):
            msg = msg.encode('unicode_escape').decode('ascii')
            res.set_cookie('logout_reason', msg, max_age=10)
        
        def is_ajax(request):
            # See horizon.utils.http.is_ajax() for more detail.
            # NOTE: openstack_auth does not import modules from horizon to avoid
            # import loops, so we copy the logic from horizon.utils.http.
            return request.headers.get('x-requested-with') == 'XMLHttpRequest'
        
        # TODO(stephenfin): Migrate to CBV
        @sensitive_post_parameters()
        @csrf_protect
        @never_cache
        def login(request):
            """Logs a user in using the :class:`~openstack_auth.forms.Login` form."""
        
            # If the user enabled websso and the default redirect
            # redirect to the default websso url
            if (request.method == 'GET' and settings.WEBSSO_ENABLED and
                    settings.WEBSSO_DEFAULT_REDIRECT):
                protocol = settings.WEBSSO_DEFAULT_REDIRECT_PROTOCOL
                region = settings.WEBSSO_DEFAULT_REDIRECT_REGION
                origin = utils.build_absolute_uri(request, '/auth/websso/')
                url = ('%s/auth/OS-FEDERATION/websso/%s?origin=%s' %
                       (region, protocol, origin))
                return shortcuts.redirect(url)
        
            # If the user enabled websso and selects default protocol
            # from the dropdown, We need to redirect user to the websso url
            if request.method == 'POST':
                auth_type = request.POST.get('auth_type', 'credentials')
                request.session['auth_type'] = auth_type
                if settings.WEBSSO_ENABLED and auth_type != 'credentials':
                    region_id = request.POST.get('region')
                    auth_url = getattr(settings, 'WEBSSO_KEYSTONE_URL', None)
                    if auth_url is None:
                        auth_url = forms.get_region_endpoint(region_id)
                    url = utils.get_websso_url(request, auth_url, auth_type)
                    return shortcuts.redirect(url)
        
            if not is_ajax(request):
                # If the user is already authenticated, redirect them to the
                # dashboard straight away, unless the 'next' parameter is set as it
                # usually indicates requesting access to a page that requires different
                # permissions.
                if (request.user.is_authenticated and
                        auth.REDIRECT_FIELD_NAME not in request.GET and
                        auth.REDIRECT_FIELD_NAME not in request.POST):
                    return shortcuts.redirect(settings.LOGIN_REDIRECT_URL)
        
            # Get our initial region for the form.
            initial = {}
            current_region = request.session.get('region_endpoint', None)
            requested_region = request.GET.get('region', None)
            regions = dict(settings.AVAILABLE_REGIONS)
            if requested_region in regions and requested_region != current_region:
                initial.update({'region': requested_region})
        
            if request.method == "POST":
                form = functools.partial(forms.Login)
            else:
                form = functools.partial(forms.Login, initial=initial)
        
            choices = settings.WEBSSO_CHOICES
            reason = get_csrf_reason(request.GET.get('csrf_failure'))
            logout_reason = request.COOKIES.get(
                'logout_reason', '').encode('ascii').decode('unicode_escape')
            logout_status = request.COOKIES.get('logout_status')
            extra_context = {
                'redirect_field_name': auth.REDIRECT_FIELD_NAME,
                'csrf_failure': reason,
                'show_sso_opts': settings.WEBSSO_ENABLED and len(choices) > 1,
                'classes': {
                    'value': '',
                    'single_value': '',
                    'label': '',
                },
                'logout_reason': logout_reason,
                'logout_status': logout_status,
            }
        
            if is_ajax(request):
                template_name = 'auth/_login.html'
                extra_context['hide'] = True
            else:
                template_name = 'auth/login.html'
        
            try:
                res = django_auth_views.LoginView.as_view(
                    template_name=template_name,
                    redirect_field_name=auth.REDIRECT_FIELD_NAME,
                    form_class=form,
                    extra_context=extra_context,
                    redirect_authenticated_user=False)(request)
            
            except exceptions.KeystoneTOTPRequired as exc:
                username = request.POST.get('username')
                domain   = request.POST.get('domain')
                # 이미 세션에 receipt, domain 저장
                request.session['receipt'] = exc.receipt
                request.session['domain']  = domain
        
                # self-enroll이 켜져 있고, 사용자에게 아직 TOTP가 없으면 enroll로,
                # 있으면 바로 TOTP 입력 페이지로.
                try:
                    if getattr(settings, "MFA_SELF_ENROLL_ENABLED", False) and not has_user_totp(username, domain):
                        res = django_http.HttpResponseRedirect(
                            reverse('totp-enroll', args=[username])
                        )
                    else:
                        res = django_http.HttpResponseRedirect(
                            reverse('totp', args=[username])
                        )
                except Exception:
                    # 체크 실패 시 안전하게 TOTP 입력으로 보냄
                    res = django_http.HttpResponseRedirect(
                        reverse('totp', args=[username])
                    )
        
            except exceptions.KeystonePassExpiredException as exc:
                res = django_http.HttpResponseRedirect(
                    reverse('password', args=[exc.user_id]))
                msg = _("Your password has expired. Please set a new password.")
                set_logout_reason(res, msg)
        
            # Save the region in the cookie, this is used as the default
            # selected region next time the Login form loads.
            if request.method == "POST":
                utils.set_response_cookie(res, 'login_region',
                                          request.POST.get('region', ''))
                utils.set_response_cookie(res, 'login_domain',
                                          request.POST.get('domain', ''))
        
            # Set the session data here because django's session key rotation
            # will erase it if we set it earlier.
            if request.user.is_authenticated:
                auth_user.set_session_from_user(request, request.user)
                regions = dict(forms.get_region_choices())
                region = request.user.endpoint
                login_region = request.POST.get('region')
                region_name = regions.get(login_region)
                request.session['region_endpoint'] = region
                request.session['region_name'] = region_name
                # Check for a services_region cookie. Fall back to the login_region.
                services_region = request.COOKIES.get('services_region', region_name)
                if services_region in request.user.available_services_regions:
                    request.session['services_region'] = services_region
                expiration_time = request.user.time_until_expiration()
                threshold_days = settings.PASSWORD_EXPIRES_WARNING_THRESHOLD_DAYS
                if (expiration_time is not None and
                        expiration_time.days <= threshold_days and
                        expiration_time > datetime.timedelta(0)):
                    expiration_time = str(expiration_time).rsplit(':', 1)[0]
                    msg = (_('Please consider changing your password, it will expire'
                             ' in %s minutes') %
                           expiration_time).replace(':', ' Hours and ')
                    messages.warning(request, msg)
            return res
        
        # TODO(stephenfin): Migrate to CBV
        @sensitive_post_parameters()
        @csrf_exempt
        @never_cache
        def websso(request):
            """Logs a user in using a token from Keystone's POST."""
            if settings.WEBSSO_USE_HTTP_REFERER:
                referer = request.META.get('HTTP_REFERER',
                                           settings.OPENSTACK_KEYSTONE_URL)
                auth_url = utils.clean_up_auth_url(referer)
            else:
                auth_url = settings.OPENSTACK_KEYSTONE_URL
            token = request.POST.get('token')
            try:
                request.user = auth.authenticate(request, auth_url=auth_url,
                                                 token=token)
            except exceptions.KeystoneAuthException as exc:
                if settings.WEBSSO_DEFAULT_REDIRECT:
                    res = django_http.HttpResponseRedirect(settings.LOGIN_ERROR)
                else:
                    msg = 'Login failed: %s' % exc
                    res = django_http.HttpResponseRedirect(settings.LOGIN_URL)
                    set_logout_reason(res, msg)
                return res
        
            auth_user.set_session_from_user(request, request.user)
            auth.login(request, request.user)
            if request.session.test_cookie_worked():
                request.session.delete_test_cookie()
            return django_http.HttpResponseRedirect(settings.LOGIN_REDIRECT_URL)
        
        # TODO(stephenfin): Migrate to CBV
        def logout(request, login_url=None, **kwargs):
            """Logs out the user if he is logged in. Then redirects to the log-in page.
        
            :param login_url:
                Once logged out, defines the URL where to redirect after login
        
            :param kwargs:
                see django.contrib.auth.views.logout_then_login extra parameters.
        
            """
            msg = 'Logging out user "%(username)s".' % \
                {'username': request.user.username}
            LOG.info(msg)
        
            """ Securely logs a user out. """
            if (settings.WEBSSO_ENABLED and settings.WEBSSO_DEFAULT_REDIRECT and
                    settings.WEBSSO_DEFAULT_REDIRECT_LOGOUT):
                auth_user.unset_session_user_variables(request)
                return django_http.HttpResponseRedirect(
                    settings.WEBSSO_DEFAULT_REDIRECT_LOGOUT)
        
            return django_auth_views.logout_then_login(request,
                                                       login_url=login_url,
                                                       **kwargs)
        
        # TODO(stephenfin): Migrate to CBV
        @login_required
        def switch(request, tenant_id, redirect_field_name=auth.REDIRECT_FIELD_NAME):
            """Switches an authenticated user from one project to another."""
            LOG.debug('Switching to tenant %s for user "%s".',
                      tenant_id, request.user.username)
        
            endpoint, __ = utils.fix_auth_url_version_prefix(request.user.endpoint)
            client_ip = utils.get_client_ip(request)
            session = utils.get_session(original_ip=client_ip)
            # Keystone can be configured to prevent exchanging a scoped token for
            # another token. Always use the unscoped token for requesting a
            # scoped token.
            unscoped_token = request.user.unscoped_token
            auth = utils.get_token_auth_plugin(auth_url=endpoint,
                                               token=unscoped_token,
                                               project_id=tenant_id)
        
            try:
                auth_ref = auth.get_access(session)
                msg = 'Project switch successful for user "%(username)s".' % \
                    {'username': request.user.username}
                LOG.info(msg)
            except keystone_exceptions.ClientException:
                msg = (
                    _('Project switch failed for user "%(username)s".') %
                    {'username': request.user.username})
                messages.error(request, msg)
                auth_ref = None
                LOG.exception('An error occurred while switching sessions.')
        
            # Ensure the user-originating redirection url is safe.
            # Taken from django.contrib.auth.views.login()
            redirect_to = request.GET.get(redirect_field_name, '')
            if (not http.url_has_allowed_host_and_scheme(
                    url=redirect_to,
                    allowed_hosts=[request.get_host()])):
                redirect_to = settings.LOGIN_REDIRECT_URL
        
            if auth_ref:
                user = auth_user.create_user_from_token(
                    request,
                    auth_user.Token(auth_ref, unscoped_token=unscoped_token),
                    endpoint)
                auth_user.set_session_from_user(request, user)
                message = (
                    _('Switch to project "%(project_name)s" successful.') %
                    {'project_name': request.user.project_name})
                messages.success(request, message)
            response = shortcuts.redirect(redirect_to)
            utils.set_response_cookie(response, 'recent_project',
                                      request.user.project_id)
            return response
        
        # TODO(stephenfin): Migrate to CBV
        @login_required
        def switch_region(request, region_name,
                          redirect_field_name=auth.REDIRECT_FIELD_NAME):
            """Switches the user's region for all services except Identity service.
        
            The region will be switched if the given region is one of the regions
            available for the scoped project. Otherwise the region is not switched.
            """
            if region_name in request.user.available_services_regions:
                request.session['services_region'] = region_name
                LOG.debug('Switching services region to %s for user "%s".',
                          region_name, request.user.username)
        
            redirect_to = request.GET.get(redirect_field_name, '')
            if (not http.url_has_allowed_host_and_scheme(
                    url=redirect_to,
                    allowed_hosts=[request.get_host()])):
                redirect_to = settings.LOGIN_REDIRECT_URL
        
            response = shortcuts.redirect(redirect_to)
            utils.set_response_cookie(response, 'services_region',
                                      request.session['services_region'])
            return response
        
        # TODO(stephenfin): Migrate to CBV
        @login_required
        def switch_keystone_provider(request, keystone_provider=None,
                                     redirect_field_name=auth.REDIRECT_FIELD_NAME):
            """Switches the user's keystone provider using K2K Federation
        
            If keystone_provider is given then we switch the user to
            the keystone provider using K2K federation. Otherwise if keystone_provider
            is None then we switch the user back to the Identity Provider Keystone
            which a non federated token auth will be used.
            """
            base_token = request.session.get('k2k_base_unscoped_token', None)
            k2k_auth_url = request.session.get('k2k_auth_url', None)
            keystone_providers = request.session.get('keystone_providers', None)
            recent_project = request.COOKIES.get('recent_project')
        
            if not base_token or not k2k_auth_url:
                msg = _('K2K Federation not setup for this session')
                raise exceptions.KeystoneAuthException(msg)
        
            redirect_to = request.GET.get(redirect_field_name, '')
            if (not http.url_has_allowed_host_and_scheme(
                    url=redirect_to,
                    allowed_hosts=[request.get_host()])):
                redirect_to = settings.LOGIN_REDIRECT_URL
        
            unscoped_auth_ref = None
            keystone_idp_id = settings.KEYSTONE_PROVIDER_IDP_ID
        
            if keystone_provider == keystone_idp_id:
                current_plugin = plugin.TokenPlugin()
                unscoped_auth = current_plugin.get_plugin(auth_url=k2k_auth_url,
                                                          token=base_token)
            else:
                # Switch to service provider using K2K federation
                plugins = [plugin.TokenPlugin()]
                current_plugin = plugin.K2KAuthPlugin()
        
                unscoped_auth = current_plugin.get_plugin(
                    auth_url=k2k_auth_url, service_provider=keystone_provider,
                    plugins=plugins, token=base_token, recent_project=recent_project)
        
            try:
                # Switch to identity provider using token auth
                unscoped_auth_ref = current_plugin.get_access_info(unscoped_auth)
            except exceptions.KeystoneAuthException as exc:
                msg = 'Switching to Keystone Provider %s has failed. %s' \
                      % (keystone_provider, exc)
                messages.error(request, msg)
        
            if unscoped_auth_ref:
                try:
                    request.user = auth.authenticate(
                        request, auth_url=unscoped_auth.auth_url,
                        token=unscoped_auth_ref.auth_token)
                except exceptions.KeystoneAuthException as exc:
                    msg = 'Keystone provider switch failed: %s' % exc
                    res = django_http.HttpResponseRedirect(settings.LOGIN_URL)
                    set_logout_reason(res, msg)
                    return res
                auth.login(request, request.user)
                auth_user.set_session_from_user(request, request.user)
                request.session['keystone_provider_id'] = keystone_provider
                request.session['keystone_providers'] = keystone_providers
                request.session['k2k_base_unscoped_token'] = base_token
                request.session['k2k_auth_url'] = k2k_auth_url
                message = (
                    _('Switch to Keystone Provider "%(keystone_provider)s" '
                      'successful.') % {'keystone_provider': keystone_provider})
                messages.success(request, message)
        
            response = shortcuts.redirect(redirect_to)
            return response
        
        # TODO(stephenfin): Migrate to CBV
        @login_required
        def switch_system_scope(request, redirect_field_name=auth.REDIRECT_FIELD_NAME):
            """Switches an authenticated user from one system to another."""
            LOG.debug('Switching to system scope for user "%s".', request.user.username)
        
            endpoint, __ = utils.fix_auth_url_version_prefix(request.user.endpoint)
            client_ip = utils.get_client_ip(request)
            session = utils.get_session(original_ip=client_ip)
            # Keystone can be configured to prevent exchanging a scoped token for
            # another token. Always use the unscoped token for requesting a
            # scoped token.
            unscoped_token = request.user.unscoped_token
            auth = utils.get_token_auth_plugin(auth_url=endpoint,
                                               token=unscoped_token,
                                               system_scope='all')
        
            try:
                auth_ref = auth.get_access(session)
            except keystone_exceptions.ClientException:
                msg = (
                    _('System switch failed for user "%(username)s".') %
                    {'username': request.user.username})
                messages.error(request, msg)
                auth_ref = None
                LOG.exception('An error occurred while switching sessions.')
            else:
                msg = 'System switch successful for user "%(username)s".' % \
                    {'username': request.user.username}
                LOG.info(msg)
        
            # Ensure the user-originating redirection url is safe.
            # Taken from django.contrib.auth.views.login()
            redirect_to = request.GET.get(redirect_field_name, '')
            if (not http.url_has_allowed_host_and_scheme(
                    url=redirect_to,
                    allowed_hosts=[request.get_host()])):
                redirect_to = settings.LOGIN_REDIRECT_URL
        
            if auth_ref:
                user = auth_user.create_user_from_token(
                    request,
                    auth_user.Token(auth_ref, unscoped_token=unscoped_token),
                    endpoint)
                auth_user.set_session_from_user(request, user)
                message = _('Switch to system scope successful.')
                messages.success(request, message)
            response = shortcuts.redirect(redirect_to)
            return response
        
        class PasswordView(edit_views.FormView):
            """Changes user's password when it's expired or otherwise inaccessible."""
            template_name = 'auth/password.html'
            form_class = forms.Password
            success_url = settings.LOGIN_URL
        
            def get_initial(self):
                return {
                    'user_id': self.kwargs['user_id'],
                    'region': self.request.COOKIES.get('login_region'),
                }
        
            def form_valid(self, form):
                # We have no session here, so regular messages don't work.
                msg = _('Password changed. Please log in to continue.')
                res = django_http.HttpResponseRedirect(self.success_url)
                set_logout_reason(res, msg)
                return res
        
        class TotpView(edit_views.FormView):
            """Logs a user in using a TOTP authentification"""
            template_name = 'auth/totp.html'
            form_class = forms.TimeBasedOneTimePassword
            success_url = settings.LOGIN_REDIRECT_URL
            fail_url = "/login/"
        
            def get_initial(self):
                return {
                    'request': self.request,
                    'username': self.kwargs['user_name'],
                    'receipt': self.request.session.get('receipt'),
                    'region': self.request.COOKIES.get('login_region'),
                    'domain': self.request.session.get('domain'),
                }
            
            def form_valid(self, form):
                auth.login(self.request, form.user_cache)
                res = django_http.HttpResponseRedirect(self.success_url)
                request = self.request
                if self.request.user.is_authenticated:
                    del request.session['receipt']
                    auth_user.set_session_from_user(request, request.user)
                    regions = dict(forms.get_region_choices())
                    region = request.user.endpoint
                    login_region = request.POST.get('region')
                    region_name = regions.get(login_region)
                    request.session['region_endpoint'] = region
                    request.session['region_name'] = region_name
                return res
        
            def form_invalid(self, form):
                if 'KeystoneNoBackendException' in str(form.errors):
                    return django_http.HttpResponseRedirect(self.fail_url)
                return super().form_invalid(form)
        
        @never_cache
        def totp_enroll(request, user_name):
            """최초 1회 QR을 작은 사이즈로 보여주고, 3분 카운트다운 후 TOTP 입력 화면으로 보냄."""
            domain_id = request.session.get('domain')
            qr_data_uri = None
            otpauth_uri = None
        
            try:
                out = ensure_user_totp_and_get_qr(user_name=user_name, domain_id=domain_id)
                if out:
                    qr_data_uri, otpauth_uri = out
            except Exception as e:
                LOG.exception("MFA enroll page error for %s: %s", user_name, e)
        
            ctx = {
                'username': user_name,
                'qr_data_uri': qr_data_uri,
                'otpauth_uri': otpauth_uri,
                'countdown_seconds': 180,   # 3분
            }
            return shortcuts.render(request, 'auth/totp_enroll.html', ctx)
        ```
        
        **URL 라우팅 추가**
        
        **`sudo vi /var/www/horizon/openstack_dashboard/urls.py`** 
        
        ```bash
        from openstack_auth import views as auth_views
        
        urlpatterns = [
            # ... 기존 항목들 ...
            path('auth/totp/enroll/<str:user_name>/', auth_views.totp_enroll, name='totp-enroll'),
        ]
        ```
        
        **Enroll 템플릿(QR 화면)**
        
        폴더 생성
        
        ```bash
        sudo mkdir -p /var/www/horizon/openstack_auth/templates/auth/
        sudo chown -R www-data:www-data /var/www/horizon/openstack_auth/templates
        ```
        
        **`sudo vi /var/www/horizon/openstack_auth/templates/auth/totp_enroll.html`**
        
        ```bash
        {% extends "auth/login.html" %}
        {% load i18n %}
        
        {% block content %}
        <div class="container">
          <div class="row">
            <div class="col-sm-6 col-sm-offset-3">
              <div class="panel panel-default" style="margin-top:12px">
                <div class="panel-heading">
                  <strong>TOTP {% trans "등록 (최초 1회)" %}</strong>
                </div>
                <div class="panel-body" style="text-align:center">
                  {% if qr_data_uri %}
                    <p style="margin-bottom:8px">
                      {% trans "아래 QR을 인증 앱(예: Google Authenticator)으로 스캔하세요." %}
                    </p>
                    <img src="{{ qr_data_uri }}" alt="TOTP QR"
                         style="max-width:160px;height:auto;display:block;margin:0 auto 8px auto;">
                    {% if otpauth_uri %}
                      <div style="font-size:12px;color:#666;word-break:break-all;margin-bottom:8px">
                        {{ otpauth_uri }}
                      </div>
                    {% endif %}
                    <div style="margin-top:6px">
                      <span id="mfa-timer" style="font-weight:bold">03:00</span>
                      <span style="color:#888"> / {% trans "3분 후 자동 진행" %}</span>
                    </div>
                    <div style="margin-top:12px">
                      <a id="mfa-confirm" class="btn btn-primary">{% trans "확인" %}</a>
                    </div>
                  {% else %}
                    <p style="color:#c00">
                      {% trans "QR 생성에 실패했습니다. 잠시 후 다시 시도하세요." %}
                    </p>
                    <div style="margin-top:12px">
                      <a href="{% url 'totp' username %}" class="btn btn-default">{% trans "TOTP 입력으로 이동" %}</a>
                    </div>
                  {% endif %}
                </div>
              </div>
        
              <!-- 아래는 기본 로그인 조각을 안 보이게 하고 싶을 때(선택) -->
              <style>
                /* 원하면 기본 로그인 폼을 가릴 수 있음 */
                form[action$="/auth/totp/{{ username }}/"] { display:none; }
              </style>
            </div>
          </div>
        </div>
        
        <script>
        (function () {
          var left = {{countdown_seconds|default:180}};
          var timerEl = document.getElementById('mfa-timer');
          function pad(n){ return (n<10?'0':'')+n; }
          function goNext(){ window.location.href = "{% url 'totp' username %}"; }
          var confirmBtn = document.getElementById('mfa-confirm');
          if (confirmBtn) confirmBtn.addEventListener('click', goNext);
          function tick(){
            if (timerEl){
              var m = Math.floor(left/60), s = left%60;
              timerEl.textContent = pad(m)+":"+pad(s);
            }
            if (left <= 0) { goNext(); return; }
            left -= 1; setTimeout(tick, 1000);
          }
          tick();
        })();
        </script>
        {% endblock %}
        ```
        
    - OpenStack 모니터링 및 알람 설정
        
        ## Controller Node 구성
        
        **Prometheus 설치**
        
        ```bash
        sudo useradd --system --no-create-home --shell /bin/false prometheus
        
        sudo apt-get update
        sudo apt-get install -y prometheus prometheus-node-exporter prometheus-node-exporter-collectors
        
        sudo mkdir /etc/prometheus /var/lib/prometheus
        sudo chown prometheus:prometheus /etc/prometheus /var/lib/prometheus
        ```
        
        **Prometheus 설정 파일 수정**
        
        ```bash
        # Sample config for Prometheus.
        
        global:
          scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
          evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
          # scrape_timeout is set to the global default (10s).
        
          # Attach these labels to any time series or alerts when communicating with
          # external systems (federation, remote storage, Alertmanager).
          external_labels:
              monitor: 'example'
        
        # Alertmanager configuration
        alerting:
          alertmanagers:
          - static_configs:
            - targets: ['localhost:9093']
        
        # Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
        rule_files:
          - "/etc/prometheus/rules/openstack.rules.yml"
          # - "first_rules.yml"
          # - "second_rules.yml"
        
        # A scrape configuration containing exactly one endpoint to scrape:
        # Here it's Prometheus itself.
        scrape_configs:
          # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
          - job_name: 'prometheus'
        
            # Override the global default and scrape targets from this job every 5 seconds.
            scrape_interval: 5s
            scrape_timeout: 5s
        
            # metrics_path defaults to '/metrics'
            # scheme defaults to 'http'.
        
            static_configs:
              - targets: ['192.168.73.171:9090']
        
          - job_name: node
            # If prometheus-node-exporter is installed, grab stats about the local
            # machine by default.
            static_configs:
              - targets: ['192.168.73.171:9100']
         
          # openstack monitoring 추가25.04.11
          - job_name: 'node_exporter'
            static_configs:
              - targets: ['192.168.73.171:9100']
                labels:
                  node: 'controller'
                  environment: 'openstack'
              - targets: ['192.168.73.172:9100']
                labels:
                  node: 'compute'
                  environment: 'openstack'
              - targets: ['192.168.73.173:9100']
                labels:
                  node: 'compute'
                  environment: 'openstack'
              - targets: ['192.168.73.168:9100']
                labels:
                  node: 'compute'
                  environment: 'openstack'
              - targets: ['192.168.73.169:9100']
                labels:
                  node: 'compute'
                  environment: 'openstack'
              - targets: ['192.168.73.177:9100']
                labels:
                  node: 'compute'
                  nvironment: 'openstack'
              - targets: ['192.168.73.162:9100']
                labels:
                  node: 'compute'
                  environment: 'openstack'
        
          - job_name: 'openstack_exporter'
            scrape_interval: 60s
            scrape_timeout: 50s
        
            static_configs:
              - targets: ['192.168.73.171:9180']
        ```
        
        **/etc/prometheus/rules/openstack.rules.yml**
        
        ```bash
        groups:
          - name: openstack-and-node-alerts
            rules:
        
            # --- CPU 사용률 65% 초과 ---
            - alert: HighCPUUsage
              expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100) > 65
              for: 10s
              labels:
                alertname: HighCPUUsage
                severity: warning
              annotations:
                summary: "서버 {{ $labels.instance }} CPU 사용률 65% 초과"
                description: "CPU 사용률이 {{ $value }}%로 높습니다."
        
            # --- 메모리 사용률 65% 초과 ---
            - alert: HighMemoryUsage
              expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 65
              for: 10s
              labels:
                alertname: HighMemoryUsage
                severity: warning
              annotations:
                summary: "서버 {{ $labels.instance }} 메모리 사용률 65% 초과"
                description: "메모리 사용률이 {{ $value }}%로 높습니다."
        
            # --- 디스크 사용률 80% 초과 ---
            - alert: HighDiskUsage
              expr: (node_filesystem_size_bytes{fstype!~"tmpfs|overlay"} - node_filesystem_free_bytes{fstype!~"tmpfs|overlay"})
                     / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"} * 100 > 65
              for: 2m
              labels:
                alertname: HighDiskUsage
                severity: warning
              annotations:
                summary: "서버 {{ $labels.instance }} 디스크 사용률 65% 초과"
                description: "디스크 공간 사용률이 {{ $value }}%로 높습니다."
        
            # --- OpenStack vCPU 리소스 사용률 65% 초과 ---
            - alert: NovaVCPUExhausted
              expr: openstack_nova_limits_vcpus_used / openstack_nova_limits_vcpus_max > 0.65
              for: 3m
              labels:
                alertname: NovaVCPUExhausted
                severity: warning
              annotations:
                summary: "Project {{ $labels.tenant }} vCPU 사용률 65% 초과"
                description: "vCPU 사용률={{ $value | printf \"%.2f\" }} (project_id={{ $labels.project_id }})"
        
            # --- OpenStack 메모리(RAM) 사용률 65% 초과 ---
            - alert: NovaMemoryExhausted
              expr: openstack_nova_limits_memory_used / openstack_nova_limits_memory_max > 0.65
              for: 2m
              labels:
                alertname: NovaMemoryExhausted
                severity: warning
              annotations:
                summary: "Project {{ $labels.tenant }} RAM 사용률 65% 초과"
                description: "사용된 RAM 비율이 {{ $value | printf \"%.2f\" }}로 높습니다."
        
            # --- Cinder 용량 부족 (30GB 미만) ---
            - alert: CinderVolumeLow
              expr: openstack_cinder_pool_capacity_free_gb < 30
              for: 2m
              labels:
                alertname: CinderVolumeLow
                severity: warning
              annotations:
                summary: "Cinder 남은 용량 30GB 미만"
                description: "Cinder 볼륨 풀 여유 공간이 {{ $value }}GB로 낮습니다."
        
            # --- Cinder 용량 부족 (20GB 미만) ---
            - alert: CinderVolumeLow
              expr: openstack_cinder_pool_capacity_free_gb < 20
              for: 2m
              labels:
                alertname: CinderVolumeLow
                severity: critical
              annotations:
                summary: "Cinder 남은 용량 20GB 미만"
                description: "Cinder 볼륨 풀 여유 공간이 {{ $value }}GB로 낮습니다."
        
            - alert: GlanceDown
              expr: openstack_glance_up == 0
              for: 1m
              labels: {severity: critical}
              annotations:
                summary: "Glance 다운"
                description: "이미지 서비스 응답 없음"
        
            - alert: KeystoneDown
              expr: openstack_identity_up == 0
              for: 1m
              labels: {severity: critical}
              annotations:
                summary: "Keystone 다운"
                description: "인증 API 비정상"
        
            - alert: NeutronAgentDown
              expr: openstack_neutron_agent_state == 0
              for: 1m
              labels: {severity: critical}
              annotations:
                summary: 'Neutron Agent {{ $labels.service }} @ {{ or $labels.host $labels.hostname $labels.service_host $labels.instance }} 다운'
                #summary: "Neutron Agent {{ $labels.hostname }} 다운"
                description: "{{ $labels.service }} 에이전트 비정상"
            
            - alert: NovaServiceDown
              expr: openstack_nova_agent_state{adminState="enabled"} == 0
              for: 2m
              labels: {severity: critical}
              annotations:
                summary: 'Nova {{ $labels.service }} @ {{ or $labels.host $labels.hostname $labels.service_host $labels.instance }} 다운'
                #summary: "Nova {{ $labels.service }} @ {{ $labels.hostname }} 다운"
                description: "zone={{ $labels.zone }} 상태 비정상"
            
            - alert: CinderServiceDown
              expr: openstack_cinder_agent_state{adminState="enabled"} == 0
              for: 2m
              labels: {severity: critical}
              annotations:
                summary: 'Cinder {{ $labels.service }} @ {{ or $labels.host $labels.hostname $labels.service_host $labels.instance }} 다운'
                #summary: "Cinder {{ $labels.service }} @ {{ $labels.hostname }} 다운"
                description: "zone={{ $labels.zone }} 상태 비정상"
                  
            - alert: OctaviaDown
              expr: openstack_octavia_up == 0
              for: 1m
              labels: {severity: critical}
              annotations:
                summary: "Octavia 다운"
                description: "로드밸런서 API 응답 없음"
        
            - alert: SwiftDown
              expr: openstack_swift_up == 0
              for: 1m
              labels: {severity: critical}
              annotations:
                summary: "Swift 다운"
                description: "오브젝트 스토리지(프록시) API 응답 없음"
        
            - alert: HeatDown
              expr: openstack_heat_up == 0
              for: 1m
              labels: {severity: critical}
              annotations:
                summary: "Heat 다운"
                description: "스택(Orchestration) API 응답 없음"
        
            - alert: GnocchiDown
              expr: openstack_gnocchi_up == 0
              for: 1m
              labels: {severity: critical}
              annotations:
                summary: "Gnocchi 다운"
                description: "Telemetry 저장소(Gnocchi) API 응답 없음"
        
            - alert: PlacementDown
              expr: openstack_placement_up == 0
              for: 1m
              labels: {severity: critical}
              annotations:
                summary: "Placement 다운"
                description: "리소스 배치(Placement) API 응답 없음"
        
            - alert: CloudKittyDown
              expr: openstack_cloudkitty_up == 0
              for: 1m
              labels: {severity: critical}
              annotations:
                summary: "CloudKitty 다운"
                description: "과금(CloudKitty) API 응답 없음"
        ```
        
        **Prometheus 설정 검증 및 재시작**
        
        ```bash
        promtool check config /etc/prometheus/prometheus.yml
        sudo systemctl restart prometheus
        sudo systemctl enable prometheus
        sudo systemctl status prometheus
        ```
        
        **Grafana 설치**
        
        https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/
        
        **OpenStack Exporter 설치**
        
        - **GoLang 설치**
            
            ```bash
            sudo add-apt-repository ppa:longsleep/golang-backports
            sudo apt-get update
            sudo apt-get install golang-go
            ```
            
        - **OpenStack Exporter 다운로드 및 빌드**
            
            ```bash
            git clone https://github.com/openstack-exporter/openstack-exporter.git
            cd openstack-exporter
            go build
            sudo cp openstack-exporter /usr/local/bin/
            ```
            
        - **`clouds.yaml` 설정 파일 이동 및 수정**
            
            ```bash
            sudo mkdir -p /etc/openstack
            sudo mv /etc/openstack-exporter/clouds.yaml /etc/openstack/clouds.yaml
            sudo chown prometheus:prometheus /etc/openstack/clouds.yaml
            sudo chmod 640 /etc/openstack/clouds.yaml
            
            sudo vi /etc/openstack/clouds.yaml
            
            clouds:
              default:
                auth_type: v3applicationcredential
                auth:
                  auth_url: http://192.168.73.171:5000/v3
                  application_credential_id: "d146309ca0e340db96db51af298e657e"
                  application_credential_secret: "9PgejiiEZDTc4pqjXQKl4zl7gCLGSdest3idiD38pW45di-IRtriVIlOhI1Ml_xtbNKKwWXug5DAze29zPMElg"
                region_name: RegionOne
                interface: internal
                verify: fals
            ```
            
        - **OpenStack Exporter 서비스 설정**
            
            ```bash
            sudo vi /etc/systemd/system/openstack-exporter.service
            
            [Unit]
            Description=OpenStack Prometheus Exporter
            After=network.target
            
            [Service]
            User=prometheus
            ExecStart=/usr/local/bin/openstack-exporter \
              --os-client-config=/etc/openstack/clouds.yaml \
              --web.listen-address=0.0.0.0:9180 \
              --endpoint-type=internal \
              default
            Restart=always
            RestartSec=5
            
            [Install]
            WantedBy=multi-user.target
            ```
            
        - **OpenStack Exporter 서비스 적용**
            
            ```bash
            sudo systemctl daemon-reload
            sudo systemctl start openstack-exporter
            sudo systemctl enable openstack-exporter
            sudo systemctl status openstack-exporter
            ```
            
        
        ## **Compute node 구성**
        
        **Node Exporter 설치**
        
        ```bash
        sudo apt-get install -y prometheus-node-exporter
        ```
        
        **Node Exporter 서비스 시작 및 활성화**
        
        ```bash
        sudo systemctl start prometheus-node-exporter
        sudo systemctl enable prometheus-node-exporter
        sudo systemctl status prometheus-node-exporter
        ```
        
        **대시보드 및 전체 설정 아래 링크 참고**
        
        https://velog.io/@nhj7804/%EC%8B%A4%EC%A0%9C-%EB%AC%BC%EB%A6%AC-%EC%84%9C%EB%B2%84%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%9C-OpenStack-%EA%B5%AC%EC%B6%9516-Grafana-Prometheus%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%9C-%EB%AA%A8%EB%8B%88%ED%84%B0%EB%A7%81-%EA%B5%AC%EC%B6%95
        
        ## Prometheus Slack Aletring 설정
        
        https://dewble.tistory.com/entry/prometheus-alertmanager-config-slack
        
        위 링크 참조 후 **Prometheus-stack Helm chart의 alertmanager에 Slack 정보 추가하기 부터 아래 내용 복붙**
        
        `sudo vi /etc/alertmanager/alertmanager.yml` 
        
        ```bash
        global:
          resolve_timeout: 1m
        
        route:
          receiver: openstack-ops-alarm
          group_by: ['alertname','instance']
          group_wait: 30s
          group_interval: 5m
          repeat_interval: 3h
        
        receivers:
        - name: openstack-ops-alarm
          # 방법 A: 전역 slack_api_url을 쓰지 않는 경우
          slack_configs:
          - api_url: '<SLACK_WEBHOOK_URL>'
            send_resolved: true
            title: '{{ .CommonLabels.alertname }}{{ if .CommonLabels.tenant }} [{{ .CommonLabels.tenant }}]{{ end }} ({{ .Status | toUpper }})'
            text: >-
              {{ range .Alerts -}}
              *요약*: {{ .Annotations.summary }}
              *라벨*: alertname={{ .Labels.alertname }}, severity={{ .Labels.severity }}
              {{- if .Labels.instance }}, instance={{ .Labels.instance }}{{ end }}
              {{- if .Labels.tenant }}, tenant={{ .Labels.tenant }}{{ end }}
              {{- if .Labels.hostname }}, hostname={{ .Labels.hostname }}{{ end }}
              {{- if .Labels.service }}, service={{ .Labels.service }}{{ end }}
              {{- end }}
        ```
        
    
    ---
    
    # OpenStack 명령어
    
    - Keystone 인증
        
        ```bash
        # 인증키 다운로드
        1. 호라이즌 로그인
        2. 우측 상단 계정 클릭
        3. OpenStack RC 파일 다운
        4. sftp로 전송
        
        # keystone 인증 key 경로 이동
        cd keystone/
        
        source admin-openrc.sh
        # 비밀번호 입력
        ```
        
    - flavor 생성 명령어 - GPU 할당
        
        ```bash
        # 2 vCPU, 4GB RAM
        openstack flavor create  [flavor.name]--vcpus 2 --ram 4096 --disk 20
        
        # 4 vCPU, 8GB RAM
        openstack flavor create [flavor.name] --vcpus 4 --ram 8192 --disk 20
        
        # 8 vCPU, 16GB RAM
        openstack flavor create [flavor.name] --vcpus 8 --ram 16384 --disk 20
        
        # 16 vCPU, 32GB RAM
        openstack flavor create [flavor.name] --vcpus 16 --ram 32768 --disk 20
        
        # 32 vCPU, 64GB RAM
        openstack flavor create [flavor.name] --vcpus 32 --ram 65536 --disk 20
        
        # 64 vCPU, 128GB RAM
        openstack flavor create [flavor.name] --vcpus 64 --ram 131072 --disk 20
        
        # GPU 할당 - t4는 nova.conf의 [pci] alias에 정의한 GPU PCI alias 이름이며, 1은 할당할 GPU 개수입니다.
        openstack flavor set t4.xlarg --property "pci_passthrough:alias"="t4:1"
        ```
        
    - flavor change
        
        ```bash
        # flavor 목록 확인
        openstack flavor list
        
        # 인스턴스 목록 확인
        openstack server list
        
        # flavor resize
        nova resize --poll [instance-id] [flavor-id]
        
        # 변경 적용
        openstack server resize confirm [instance-id]
        ```
        
        문제 상황
        
        - `openstack server resize` 또는 Live Migration을 수행해도 인스턴스가 다른 Compute 노드로 이동하지 않음
        - 인스턴스가 실행 중인 서버의 `/var/log/nova/nova-compute.log`에 다음과 같은 오류가 발생:
        
        ```bash
        Command: ssh -o BatchMode=yes <target_compute_node> mkdir -p /var/lib/nova/instances/<UUID>
        Stdout: 'This account is currently not available.'
        
        # nova 계정의 SSH 연결이 정상적으로 이루어지지 않아 마이그레이션에 실패
        ```
        
        - 기본 해결 방법
            
            nova 계정에 SSH 키를 생성한다 (예: 8번 서버에서)
            
            ```bash
            sudo -u nova mkdir -p /var/lib/nova/.ssh
            sudo -u nova chmod 700 /var/lib/nova/.ssh
            sudo -u nova ssh-keygen -t rsa -N "" -f /var/lib/nova/.ssh/id_rsa
            ```
            
            공개키를 대상 서버로 복사한다
            
            ```bash
            scp /var/lib/nova/.ssh/id_rsa.pub mnc_admin@192.168.73.172:/tmp/id_rsa_nova_8.pub
            ```
            
            대상 서버(예: 172번)에서 공개키를 authorized_keys에 추가한다
            
            ```bash
            cat /tmp/id_rsa_nova_8.pub | sudo tee -a /var/lib/nova/.ssh/authorized_keys
            sudo chown nova:nova /var/lib/nova/.ssh/authorized_keys
            sudo chmod 600 /var/lib/nova/.ssh/authorized_keys
            ```
            
            nova 계정의 로그인 쉘을 활성화한다 (대상 서버에서)
            
            ```bash
            sudo getent passwd nova
            sudo usermod -s /bin/bash nova
            ```
            
            위 작업 후에도 Flavor 변경이 실패할 경우
            
            nova 계정의 `known_hosts` 파일이 존재하는지 확인한다
            
            ```bash
            ls /var/lib/nova/.ssh/known_hosts
            ```
            
            존재하지 않는 경우, 대상 노드의 host key를 자동 등록한다
            
            ```bash
            sudo -u nova ssh -o StrictHostKeyChecking=no nova@192.168.73.173 hostname
            ```
            
            만약 `known_hosts`가 존재하지만 host key 충돌로 인해 문제가 발생한다면, 기존 정보를 제거하고 재등록한다
            
            ```bash
            sudo -u nova ssh-keygen -R 192.168.73.173
            sudo -u nova ssh -o StrictHostKeyChecking=no nova@192.168.73.173 hostname
            ```
            
    - IP 할당 고정 방지 (인스턴스 생성 시 고정한 IP로 생성되지 않도록 하기)
        
        ```bash
        # 네트워크 목록 확인
        openstack network list
        
        # IP 고정 포트 생성 (VIP)
        openstack port create --network internal --fixed-ip subnet=3b9b4378-eebb-484c-b5e8-5ad79cb3f6e8,ip-address=172.16.0.182 reserved-port
        ```
        
    - image 포맷 변경
        
        ```bash
        dd if=your_file.iso of=your_file.img bs=4M status=progress
        ```
        
    - image create
        
        ```bash
        glance image-create \
          --name "CentOS_8" \
          --file CentOS-8-ec2-8.4.2105-20210603.0.x86_64.qcow2 \
          --disk-format qcow2 \
          --container-format bare \
          --visibility public
        ```
        
    - Instance 계정 생성(구성)
        
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
          
          
        #cloud-config
        hostname: mkp-01
        fqdn: mkp-01
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
        
    - 리소스 사용량(usage) 및 할당량(quota)
        
        ```bash
        openstack hypervisor stats show
        
        # admin 프로젝트에 사용량 할당
        openstack quota set \
          --cores 54 \
          --ram 257188 \
          admin
          
        # 늘어난 할당량 확인
        openstack quota show admin
        ```
        
    - Cinder 물리서버 사용량 확인 명령어
        
        ```bash
        sudo lvs -a -o +devices
        ```
        
    - OpenStack 환경에서 VIP 설정
        
        VIP를 사용할 인스턴스들의 포트에 `allowed-address-pairs`로 수동 등록 필요
        `allowed-address-pairs`는 포트가 **추가적인 IP를 사용할 수 있도록 허용**하는 기능
        어떠한 인스턴스에서도 사용하지 않는 IP를 할당해야함
        
        ```bash
        # 포트 목록 확인
        openstack port list
        # 특정 포트에 VIP 주소 허용 (Allowed Address Pair 설정)
        openstack port set --allowed-address ip-address=172.16.0.181 <포트ID>
        
        # 예시
        #+--------------------------------------+------+-------------------+-------------------------------------------------------------------------------+--------+
        #| ID                                   | Name | MAC Address       | Fixed IP Addresses                                                            | Status |
        #+--------------------------------------+------+-------------------+-------------------------------------------------------------------------------+--------+
        #| e38f3189-5f04-4e6a-885f-59f99f924606 |      | fa:16:3e:0b:1e:a4 | ip_address='172.16.0.100', subnet_id='3b9b4378-eebb-484c-b5e8-5ad79cb3f6e8'   | ACTIVE |
        ```
        
        VIP 포트 할당 확인
        
        ```bash
        openstack port show <포트ID>
        
        # allowed_address_pairs   | ip_address='172.16.0.181', mac_address='fa:16:3e:44:59:e7' 할당한 vip포트가 보이면 할당된것임
        ```
        
    - 멀티 볼륨 설정(하나의 볼륨 여러 인스턴스에 할당)
        
        볼륨 타입 생성 및 속성 설정
        
        ```bash
        # 타입 생성
        openstack volume type create ceph
        
        # 타입이 어떤 백엔드로 갈지 매핑
        openstack volume type set ceph --property volume_backend_name='ceph'
        
        # 동시 연결 허용 플래그
        openstack volume type set ceph --property multiattach="<is> True"
        
        # 확인
        openstack volume type show ceph -f yaml
        ```
        
        볼륨 생성(타입 지정)
        
        ```bash
        # 타입을 지정해서 생성
        openstack volume create --size 20 --type ceph vol-share
        ```
        
        볼륨 다중 연결(CLI 에서만 가능)
        
        ```bash
        # 첫 번째 인스턴스에 연결
        openstack server add volume test vol-share
        
        # 두 번째 인스턴스에 연결
        openstack server add volume mul-vol-test vol-share
        ```
        
    - Nova 자동화(shell 파일 등록 - Controller Node)
        
        **디렉터리 구조**
        
        ```bash
        nova_flavor_set/
        ├─ hosts.ini           # 인벤토리
        ├─ group_vars/
        │   ├─ mnc_admin.yml   # vault 암호화 파일
        │   └─ kube.yml        # vault 암호화 파일
        └─ nova_ssh_setup.yml  # 플레이북
        ```
        
        패키지 설치
        
        ```bash
        sudo apt install -y ansible sshpass
        ```
        
        **Ansible 인벤토리 구성 (hosts.ini)**
        
        ```bash
        [mnc_admin]
        192.168.73.171 ansible_user=mnc_admin
        192.168.73.172 ansible_user=mnc_admin
        192.168.73.173 ansible_user=mnc_admin
        
        [kube]
        192.168.73.168 ansible_user=kube
        192.168.73.169 ansible_user=kube
        192.168.73.177 ansible_user=kube
        
        [computes:children]
        mnc_admin
        kube
        ```
        
         **비밀번호 저장 (Vault 암호화)**
        
        ```bash
        ansible-vault create group_vars/mnc_admin.yml
        # → 다음 두 줄 입력
        ansible_ssh_pass: "비밀번호"
        ansible_become_pass: "비밀번호"
        
        ansible-vault create group_vars/kube.yml
        # 내용
        ansible_ssh_pass: "비밀번호"
        ansible_become_pass: "비밀번호"
        ```
        
        암호화
        
        ```bash
        ansible-vault encrypt group_vars/mnc_admin.yml
        ansible-vault encrypt group_vars/kube.yml
        ```
        
        **최종 플레이북 (nova_ssh_setup.yml)**
        
        ```bash
        # nova_ssh_setup.yml (필터·커스텀 플러그인 완전 제거)
        ---
        - name: Setup nova SSH across all compute nodes
          hosts: computes
          become: true
        
          vars:
            peer_nodes: "{{ ansible_play_hosts_all }}"   # ← 필터 無
        
          tasks:
          - name: Ensure /var/lib/nova/.ssh exists
            file:
              path: /var/lib/nova/.ssh
              state: directory
              owner: nova
              group: nova
              mode: '0700'
        
          - name: Create nova keypair if missing
            become_user: nova
            openssh_keypair:
              path: /var/lib/nova/.ssh/id_rsa
              type: rsa
              size: 2048
              state: present
              force: false
        
          - name: Read nova public key
            become_user: nova
            command: cat /var/lib/nova/.ssh/id_rsa.pub
            register: nova_pub_raw
            changed_when: false
        
          - name: Install nova pubkey on peers
            delegate_to: "{{ item }}"
            become: true
            authorized_key:
              user: nova
              key: "{{ nova_pub_raw.stdout }}"
              state: present         # 중복이면 변화 없음
              manage_dir: false
            loop: "{{ peer_nodes }}"
            when: inventory_hostname != item
        
          - name: Fix permissions on authorized_keys
            file:
              path: /var/lib/nova/.ssh/authorized_keys
              owner: nova
              group: nova
              mode: '0600'
        
          - name: Ensure nova shell is /bin/bash
            user:
              name: nova
              shell: /bin/bash
        
          - name: Register peer hostkeys in known_hosts
            become_user: nova
            known_hosts:
              path: /var/lib/nova/.ssh/known_hosts
              name: "{{ item }}"
              key: "{{ lookup('pipe', 'ssh-keyscan -H ' ~ item) }}"
              hash_host: true
              state: present
            loop: "{{ peer_nodes }}"
            when: inventory_hostname != item
        ```
        
        실행 명령어
        
        ```bash
        ansible-playbook -i hosts.ini nova_ssh_setup.yml --ask-vault-pass -vvv
        ```
        

---
