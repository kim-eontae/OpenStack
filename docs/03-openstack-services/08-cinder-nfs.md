- Cinder - BlockStorage(Ceph)
        
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
        
        ### 1. 디스크 파티션 포맷
        
        ---
        
        ### **Cephadm 설치 및 클러스터 부트스트랩**
        
        ```bash
        export CEPH_RELEASE=19.2.3  # ← 활성화된 릴리즈 버전
        curl -sLO https://download.ceph.com/rpm-${CEPH_RELEASE}/el9/noarch/cephadm
        chmod +x cephadm
        sudo ./cephadm add-repo --release squid
        sudo ./cephadm install
        which cephadm
        ```
        
        ### **클러스터 초기 부트스트랩**
        
        ```bash
        sudo cephadm bootstrap \
          --mon-ip 192.168.73.171 \
          --initial-dashboard-user admin \
          --initial-dashboard-password 'mnc1!' \
          --allow-overwrite
        ```
        
        ### 도커 설치(패키지 없을경우)
        
        ```bash
        sudo apt install -y docker.io
        
        # 도커 런타임 변경
        sudo vi /etc/docker/daemon.json
        "default-runtime": "nvidia", -> "default-runtime": "runc"
        ```
        
        ### **Ceph CLI 접속 및 공통 패키지 설치**
        
        ```bash
        sudo cephadm install ceph-common
        sudo cephadm shell -- ceph -s
        ```
        
        ### **클러스터 SSH 키 배포 + Ceph 노드 등록 자동화**
        
        ```bash
        #!/usr/bin/env bash
        set -euo pipefail
        
        # 1) cephadm 키 확보 (없으면 생성)
        if [[ ! -f /etc/ceph/ceph || ! -f /etc/ceph/ceph.pub ]]; then
          echo "[INFO] /etc/ceph/ceph 키가 없어 생성합니다"
          ssh-keygen -t rsa -N "" -f /etc/ceph/ceph
        else
          echo "[INFO] 기존 /etc/ceph/ceph 키를 사용합니다"
        fi
        
        PUB=/etc/ceph/ceph.pub
        
        # 2) 관리 노드에 _admin 라벨(옵션)
        ceph orch host label add mncsvrt06 _admin || true
        
        # 3) 대상 호스트 목록 (호스트명=DNS/FQDN이 ceph가 인식하는 이름과 일치해야 함)
        declare -A HOSTS=(
          [mncsvrt03]=192.168.73.168
          [mncsvrt04]=192.168.73.169
          [mncsvrt07]=192.168.73.172
          [mncsvrt08]=192.168.73.173
          [mncsvrt12]=192.168.73.177
          [mncstorage01]=192.168.73.162
        )
        
        # 4) 각 노드에 루트 키 배포 (idempotent)
        for H in "${!HOSTS[@]}"; do
          IP=${HOSTS[$H]}
          echo "==> push key to $H ($IP)"
        
          # 비밀번호 로그인 임시 허용이 전제(ssh-copy-id 사용) - 실패 시 수동으로 열어주세요
          ssh-copy-id -i "$PUB" "root@$IP" || {
            echo "[WARN] ssh-copy-id 실패(비번 로그인 불가?). 수동으로 열어주세요: $H"
            continue
          }
        
          # 5) Ceph에 호스트 등록 (존재하면 무시)
          if ! ceph orch host ls --format json | jq -r '.[].hostname' | grep -qx "$H"; then
            ceph orch host add "$H" "$IP"
          else
            # 주소가 다르면 set-addr
            CUR=$(ceph orch host ls --format json | jq -r ".[] | select(.hostname==\"$H\") | .addr")
            [[ "$CUR" != "$IP" ]] && ceph orch host set-addr "$H" "$IP" || true
          fi
        done
        
        # 6) 접속/상태 확인
        echo
        ceph orch host ls
        echo
        echo "[INFO] SSH reachability quick-test"
        for H in "${!HOSTS[@]}"; do
          echo -n "$H: "
          ssh -o BatchMode=yes -i /etc/ceph/ceph "root@${HOSTS[$H]}" "echo ok" || echo "fail"
        done
        ```
        
        ### **MON 추가 (3개로, 평기수 권장)**
        
        ```bash
        ceph orch apply mon --unmanaged3
        ceph orch daemon add mon mncsvrt03:192.168.73.168
        ceph orch daemon add mon mncsvrt04:192.168.73.169
        
        # 확인
        ceph mon dump
        ```
        
        ### **Ceph OSD 디스크 구성 (NFS -> Ceph 전환)**
        
        ```bash
        sudo fuser -vm /storage
        sudo fuser -km /storage
        
        sudo vi /etc/exports   # → 내용 제거
        sudo exportfs -ra
        sudo systemctl stop nfs-server
        
        sudo umount /storage
        ```
        
        ### **새로운 OSD 디스크 구성**
        
        ```bash
        sudo pvcreate /dev/sdb1
        sudo vgcreate ceph-osd-vg01 /dev/sdb1
        sudo lvcreate -n osd-block-1 -l 100%FREE ceph-osd-vg01
        sudo lvdisplay
        
        # OSD 추가
        ceph orch daemon add osd mncsvrt03:/dev/ceph-osd-vg01/osd-block-1
        ceph orch daemon add osd mncsvrt04:/dev/ceph-osd-vg01/osd-block-1
        ceph orch daemon add osd mncsvrt06:/dev/ceph-osd-vg01/osd-block-1
        ceph orch daemon add osd mncsvrt07:/dev/ceph-osd-vg01/osd-block-1
        ceph orch daemon add osd mncsvrt08:/dev/ceph-osd-vg01/osd-block-1
        ceph orch daemon add osd mncsvrt12:/dev/ceph-osd-vg01/osd-block-1
        ceph orch daemon add osd mncstorage01:/dev/ceph-osd-vg01/osd-block-1
        
        # 상태 확인
        ceph orch ps | grep osd
        ```
        
        ### **OSD 메모리 튜닝(과도한 메모리를 사용중이면 튜닝)**
        
        ```bash
        ceph config set osd.0 osd_memory_target_autotune false
        ceph config set osd.0 osd_memory_target 16G
        ```
        
        ### **Ceph Pool 구성**
        
        ```bash
        ceph osd pool create volumes
        ceph osd pool create backups
        ```
        
        > volumes → Cinder main pool
        > 
        > 
        > backups → Cinder backup 전용 pool
        > 
        
        ### **Ceph 설정 파일 작성 및 키링 발급**
        
        1. **/etc/ceph/ceph.conf**
        
        ```bash
        # 아래 내용 추가
        auth_cluster_required = cephx
        auth_service_required = cephx
        auth_client_required = cephx
        ```
        
        1. **클라이언트 인증 발급**
        
        ```bash
        ceph auth get-or-create client.cinder \
          mon 'profile rbd' \
          osd 'profile rbd pool=volumes, profile rbd pool=vms, profile rbd-read-only pool=images' \
          mgr 'profile rbd pool=volumes, profile rbd pool=vms' | sudo tee /etc/ceph/ceph.client.cinder.keyring
        
        ceph auth get-or-create client.cinder-backup \
          mon 'profile rbd' \
          osd 'profile rbd pool=backups' \
          mgr 'profile rbd pool=backups' | sudo tee /etc/ceph/ceph.client.cinder-backup.keyring
        
        sudo chown cinder:cinder /etc/ceph/ceph.client.cinder*.keyring
        ```
        
        1. 패키지 설치
        
        ```bash
        sudo apt-get install -y python3-rbd ceph-common podman
        sudo apt install libvirt-clients libvirt-daemon-system -y
        ```
        
        1. 파일 공유
        
        ```bash
        for HOST in mncsvrt03 mncsvrt04 mncsvrt07 mncsvrt08 mncsvrt12 mncstorage01; do
          echo ">>> Distributing ceph.conf to $HOST"
        
          # /etc/ceph 디렉터리 생성 (없을 경우)
          ssh $HOST "sudo mkdir -p /etc/ceph"
        
          # ceph.conf 복사
          scp /etc/ceph/ceph.conf $HOST:/tmp/
        
          # 적절한 위치로 이동
          ssh $HOST "sudo mv /tmp/ceph.conf /etc/ceph/ceph.conf"
        done
        ```
        
        ```bash
        for HOST in mncsvrt03 mncsvrt04 mncsvrt07 mncsvrt08 mncsvrt12 mncstorage01; do
          echo ">>> Distributing to $HOST"
        
          # 디렉터리 없을 경우 생성
          ssh $HOST "sudo mkdir -p /etc/ceph"
        
          # 키링 파일 복사
          scp /etc/ceph/ceph.client.cinder.keyring $HOST:/tmp/
          scp /etc/ceph/ceph.client.cinder-backup.keyring $HOST:/tmp/
        
          # 키링 파일 이동 및 권한 설정
          ssh $HOST "sudo mv /tmp/ceph.client.cinder.keyring /etc/ceph/"
          ssh $HOST "sudo mv /tmp/ceph.client.cinder-backup.keyring /etc/ceph/"
          ssh $HOST "sudo chown cinder:cinder /etc/ceph/ceph.client.cinder*.keyring"
        
          # client.cinder.key 추가 (공식 문서 방식)
          ceph auth get-key client.cinder | ssh $HOST "sudo tee /etc/ceph/client.cinder.key"
        done
        ```
        
        1. **libvirt용 Secret 등록 (Nova 연동 시)**
        
        ```bash
        uuidgen
        # 위에서 나온 UUID 아래 입력
        UUID=bbe63137-8e96-4580-9e06-69867db1a782
        KEY=$(ceph auth get-key client.cinder | tr -d '\n')
        HOSTS="mncsvrt07 mncsvrt08 mncsvrt09 mncsvrt10 mncsvrt11 mncsvrt12"
        
        for H in $HOSTS; do
          echo ">>> $H"
          # 없으면 define
          ssh "$H" "LC_ALL=C sudo virsh -c qemu:///system secret-dumpxml $UUID >/dev/null 2>&1 || \
            sudo virsh -c qemu:///system secret-define /dev/stdin" <<EOF
        <secret ephemeral='no' private='no'>
          <uuid>$UUID</uuid>
          <usage type='ceph'><name>client.cinder secret</name></usage>
        </secret>
        EOF
          # 값 주입 + 검증
          ssh "$H" "LC_ALL=C sudo virsh -c qemu:///system secret-set-value --secret $UUID --base64 '$KEY' && \
                    LC_ALL=C sudo virsh -c qemu:///system secret-get-value $UUID >/dev/null && echo '$H: OK'"
        done
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
        enabled_backends = ceph
        transport_url = rabbit://openstack:mnc_rabbit@mncsvrt02
        auth_strategy = keystone
        my_ip = 192.168.73.167
        glance_api_servers = http://mncsvrt02:9292
        backup_driver = cinder.backup.drivers.ceph.CephBackupDriver
        backup_ceph_conf = /etc/ceph/ceph.conf
        backup_ceph_user = cinder-backup
        backup_ceph_chunk_size = 134217728
        backup_ceph_pool = backups
        backup_ceph_stripe_unit = 0
        backup_ceph_stripe_count = 0
        restore_discard_excess_bytes = true
        
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
        
        [ceph]
        volume_driver = cinder.volume.drivers.rbd.RBDDriver
        volume_backend_name = ceph
        rbd_pool = volumes
        rbd_ceph_conf = /etc/ceph/ceph.conf
        rbd_flatten_volume_from_snapshot = false
        rbd_max_clone_depth = 5
        rbd_store_chunk_size = 4
        rados_connect_timeout = -1
        rbd_user = cinder
        rbd_secret_uuid = ced0419f-b6dc-4094-a9c6-c31e2439dc8f
        **# 다중 연결 설정**
        rbd_default_features = 3
        rbd_exclusive_cinder_pool = true
        report_discard_supported = true
        
        [libvirt]
        rbd_user = cinder
        rbd_secret_uuid = ced0419f-b6dc-4094-a9c6-c31e2439dc8f
        
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
        
        ### **기존 NFS Backend 제거**
        
        ```bash
        sudo cinder-manage service list
        sudo cinder-manage service remove cinder-volume mncsvrt02@nfs
        ```
        
        ### **Ceph Pool 복제 수 조정 (개발환경에서만)**
        
        ```bash
        ceph config set mon mon_allow_pool_size_one true
        ceph osd pool set backups size 1 --yes-i-really-mean-it
        ```
        
        ### **Grafana 포트 변경 및 Ceph 연동**
        
        ```bash
        sudo vi /etc/grafana/grafana.ini
        [server]
        http_port = 3001
        sudo systemctl restart grafana-server
        ```
        
        ### **Ceph Grafana 대시보드 활성화**
        
        ```bash
        ceph orch apply grafana
        ```
        
        ### Ceph node-export port 변경
        
        ```bash
        cat <<EOF > node-exporter-9101.yaml
        service_type: node-exporter
        service_name: node-exporter
        spec:
          port: 9101
        EOF
        
        ceph orch apply -i node-exporter-9101.yaml
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
        
        ### 각 노드 Root 통신 해제
        
        ```bash
        vi /etc/ssh/sshd_config
        PermitRootLogin yes
        systemctl restart sshd
        ```
        
        Ceph 웹 페이지 에러 발생시 조치 사항
        
        ```bash
        # Dashboard 모듈 비활성화
        ceph mgr module disable dashboard
        
        # 설정 제거
        ceph config rm mgr mgr/dashboard/server_addr
        ceph config rm mgr mgr/dashboard/server_port
        
        # (중요) 상태 초기화 위해 mgr 데몬 재시작
        ceph mgr fail
        # 재시작
        ceph orch restart mgr
        
        ceph config set mgr mgr/dashboard/server_addr 0.0.0.0
        ceph config set mgr mgr/dashboard/server_port 8445
        ceph mgr module enable dashboard
        ceph mgr services
        curl -k https://<mgr-ip>:8445
        ```
        
        ## ceph 클러스터 경고 해결
        
        1. CephadmDaemonFailed 
        
        ```bash
        # 상태 확인
        ceph health detail
        
        # 에러난 3개만 재시작 → 실패 시 재배포
        for h in mncsvrt03 mncsvrt04 mncsvrt12; do
          echo "== $h =="
          ceph orch daemon restart  ceph-exporter.$h || \
          ceph orch daemon redeploy ceph-exporter.$h
        done
        # 확인
        ceph orch ps --daemon_type=ceph-exporter --refresh
        ```
        
        아키텍처
        
        ```mermaid
        flowchart TD
        
            %% 0) 볼륨
            VOL["1TB Volume"]
        
            %% 1) 오브젝트 집합 (라벨을 따옴표로 감싸기)
            subgraph OBJS["Objects (4MB 단위로 분할)"]
                O1["Object A"]
                O2["Object B"]
                O3["Object C"]
                O4["Object D"]
                O5["Object E"]
                O6["Object ..."]
            end
        
            VOL --> O1
            VOL --> O2
            VOL --> O3
            VOL --> O4
            VOL --> O5
            VOL --> O6
        
            %% 2) 각 오브젝트의 레플리카 노드 (3중화)
            O1 --> O1R1["Replica A1"]
            O1 --> O1R2["Replica A2"]
            O1 --> O1R3["Replica A3"]
        
            O2 --> O2R1["Replica B1"]
            O2 --> O2R2["Replica B2"]
            O2 --> O2R3["Replica B3"]
        
            O3 --> O3R1["Replica C1"]
            O3 --> O3R2["Replica C2"]
            O3 --> O3R3["Replica C3"]
        
            O4 --> O4R1["Replica D1"]
            O4 --> O4R2["Replica D2"]
            O4 --> O4R3["Replica D3"]
        
            O5 --> O5R1["Replica E1"]
            O5 --> O5R2["Replica E2"]
            O5 --> O5R3["Replica E3"]
        
            %% 3) 7개 OSD (클러스터) - 라벨 따옴표 필수
            subgraph OSDs["Ceph Cluster: 7 OSDs"]
                osd1["OSD1"]
                osd2["OSD2"]
                osd3["OSD3"]
                osd4["OSD4"]
                osd5["OSD5"]
                osd6["OSD6"]
                osd7["OSD7"]
            end
        
            %% 4) CRUSH에 따른 분산 배치 예시 (레플리카 -> OSD)
            O1R1 --> osd1
            O1R2 --> osd2
            O1R3 --> osd3
        
            O2R1 --> osd4
            O2R2 --> osd5
            O2R3 --> osd7
        
            O3R1 --> osd2
            O3R2 --> osd6
            O3R3 --> osd3
        
            O4R1 --> osd7
            O4R2 --> osd1
            O4R3 --> osd4
        
            O5R1 --> osd3
            O5R2 --> osd5
            O5R3 --> osd6
        ```
