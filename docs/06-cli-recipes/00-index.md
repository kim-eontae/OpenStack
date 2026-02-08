# OpenStack 명령어

OpenStack 명령어
    
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

#
