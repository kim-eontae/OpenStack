# OpenStack Octavia(LB) 구축

---

---

## **DB생성**

```sql
mysql

CREATE DATABASE octavia;

GRANT ALL PRIVILEGES ON octavia.* TO 'octavia'@'localhost'IDENTIFIED BY 'mnc1!';
GRANT ALL PRIVILEGES ON octavia.* TO 'octavia'@'%' IDENTIFIED BY 'mnc1!';
```

## **사용자 생성**

```bash
openstack user create --domain default --password-prompt octavia

openstack role add --project service --user octavia admin

openstack service create --name octavia --description "OpenStack Octavia" load-balancer

openstack endpoint create --region RegionOne load-balancer public http://mncsvrt02:9876
openstack endpoint create --region RegionOne load-balancer internal http://mncsvrt02:9876
openstack endpoint create --region RegionOne load-balancer admin http://mncsvrt02:9876
```

## **octavia-openrc  생성**

```bash
cat << EOF >> $HOME/keystone/octavia-openrc
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=service
export OS_USERNAME=octavia
export OS_PASSWORD=mnc1!
export OS_AUTH_URL=http://mncsvrt02:5000
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
export OS_VOLUME_API_VERSION=3
EOF
```

## **octavia-openrc로 로그인**

```bash
source $HOME/keystone/octavia-openrc
```

## **패키지 설치**

```bash
sudo apt install -y octavia-api octavia-health-manager octavia-housekeeping octavia-worker python3-octavia python3-octaviaclient
```

## **인증서 생성**

```bash
git clone https://opendev.org/openstack/octavia.git
cd octavia/bin/
source create_dual_intermediate_CA.sh
sudo mkdir -p /etc/octavia/certs/private
sudo chmod 755 /etc/octavia -R
sudo cp -p etc/octavia/certs/server_ca.cert.pem /etc/octavia/certs
sudo cp -p etc/octavia/certs/server_ca-chain.cert.pem /etc/octavia/certs
sudo cp -p etc/octavia/certs/server_ca.key.pem /etc/octavia/certs/private
sudo cp -p etc/octavia/certs/client_ca.cert.pem /etc/octavia/certs
sudo cp -p etc/octavia/certs/client.cert-and-key.pem /etc/octavia/certs/private

sudo chown -R octavia:octavia /etc/octavia/certs/*
sudo chown -R octavia:octavia /etc/octavia/certs/private/*
sudo chmod 644 /etc/octavia/certs/*.pem
sudo chmod 600 /etc/octavia/certs/private/*.pem
# client CA → ca_01.pem 링크
sudo ln -s /etc/octavia/certs/client_ca.cert.pem /etc/octavia/certs/ca_01.pem
```

## **관리 네트워크(lb-mgmt-net) 및 Health-Manager 포트**

```bash
###############################
# 컨트롤러 노드에서 실행
###############################
export OCTAVIA_MGMT_NET=lb-mgmt-net 
export OCTAVIA_MGMT_SUBNET=lb-mgmt-subnet 
export OCTAVIA_MGMT_CIDR=10.0.0.0/24 
export OCTAVIA_MGMT_POOL_START=10.0.0.3 
export OCTAVIA_MGMT_POOL_END=10.0.0.254 
export OCTAVIA_MGMT_PORT_IP=10.0.0.2 
export OCTAVIA_MGMT_SG=lb-mgmt-sec-grp 
export HOSTNAME=$(hostname)

# 1) 네트워크 생성
openstack network create lb-mgmt-net

# 2) 서브넷이 없으면 생성  (← gateway 없음 + DHCP 사용)
if ! openstack subnet show $OCTAVIA_MGMT_SUBNET >/dev/null 2>&1; then
  # openstackclient 6.x↑ : --gateway none / 5.x↓ : gateway 옵션 생략
  if openstack subnet create -h | grep -q -- '--gateway'; then
      GW_OPT="--gateway none"
  else
      GW_OPT=""
  fi

  openstack subnet create --subnet-range $OCTAVIA_MGMT_CIDR \
    --allocation-pool start=$OCTAVIA_MGMT_POOL_START,end=$OCTAVIA_MGMT_POOL_END \
    --network $OCTAVIA_MGMT_NET --dhcp $GW_OPT \
    $OCTAVIA_MGMT_SUBNET
fi

# 3) 서브넷 ID 변수
SUBNET_ID=$(openstack subnet show $OCTAVIA_MGMT_SUBNET -f value -c id)
echo "SUBNET_ID = $SUBNET_ID"

###############################
# 보안 그룹 + 룰 생성 (안전판)
###############################

export OCTAVIA_MGMT_SG=lb-mgmt-sec-grp

# SG 없으면 생성
openstack security group show $OCTAVIA_MGMT_SG >/dev/null 2>&1 || \
openstack security group create $OCTAVIA_MGMT_SG

# ── Ingress ──────────────────────────────────────
# 포트가 필요한 tcp/udp
for RULE in "9443/tcp" "5555/udp" "67:68/udp"; do
  PROTO=${RULE##*/}            # tcp or udp
  PORT=${RULE%/*}              # 9443, 5555, 67:68
  openstack security group rule create $OCTAVIA_MGMT_SG \
      --ingress --ethertype IPv4 \
      --protocol $PROTO --dst-port $PORT \
      2>/dev/null || true       # 이미 있으면 무시
done

# icmp (포트 없이)
openstack security group rule create $OCTAVIA_MGMT_SG \
    --ingress --ethertype IPv4 --protocol icmp \
    2>/dev/null || true

# ── Egress All ────────────────────────────────────
openstack security group rule create $OCTAVIA_MGMT_SG --egress \
    2>/dev/null || true
    

# Health-Mgr 포트 (allowed-address = CIDR)
MGMT_PORT_ID=$(openstack port create octavia-health-manager-listen-port \
  --network $OCTAVIA_MGMT_NET --host $HOSTNAME \
  --device-owner Octavia:health-mgr \
  --fixed-ip ip-address=$OCTAVIA_MGMT_PORT_IP \
  --security-group $OCTAVIA_MGMT_SG \
  --allowed-address ip-address=$OCTAVIA_MGMT_CIDR \
  -f value -c id)
MGMT_PORT_MAC=$(openstack port show $MGMT_PORT_ID -f value -c mac_address)

echo "HM Port: $MGMT_PORT_ID  MAC: $MGMT_PORT_MAC"
```

## **컨트롤러 veth (o-hm0 / o-bhm0 + external-ids)**

```bash
sudo ip link add o-hm0 type veth peer name o-bhm0
sudo ip link set o-hm0 address $MGMT_PORT_MAC
sudo ip link set o-hm0 up
sudo ip link set o-bhm0 up
sudo ovs-vsctl --may-exist add-port br-int o-bhm0
sudo ovs-vsctl set interface o-bhm0 external-ids:iface-status=active \
  external-ids:attached-mac=$MGMT_PORT_MAC \
  external-ids:iface-id=$MGMT_PORT_ID \
  external-ids:skip_cleanup=true
sudo ip addr add $OCTAVIA_MGMT_PORT_IP/24 dev o-hm0
sudo iptables -C INPUT -i o-hm0 -p udp --dport 5555 -j ACCEPT 2>/dev/null || \
sudo iptables -I INPUT -i o-hm0 -p udp --dport 5555 -j ACCEPT
```

## heartbeat_key 생성(3개중 선택)

**heartbeat_key는 Health-Manager↔amphora 하트비트/제어 채널의 인증 키라 미설정·불일치 시 하트비트를 버려 상태 동기화·failover가 불안정해지므로 설정 필요**

```bash
# 1) 256bit HEX (64자) – 가장 무난
openssl rand -hex 32

# 2) 32바이트 URL-safe Base64 (≈43자)
python3 - <<'PY'
import secrets, base64
print(base64.urlsafe_b64encode(secrets.token_bytes(32)).rstrip(b'=').decode())
PY

# 3) 48바이트 URL-safe Base64 (좀 더 길게)
python3 - <<'PY'
import secrets, base64
print(base64.urlsafe_b64encode(secrets.token_bytes(48)).rstrip(b'=').decode())
PY
```

## **이미지 생성**

```bash
sudo apt install python3-virtualenv

virtualenv octavia_disk_image_create

source octavia_disk_image_create/bin/activate

cd octavia/diskimage-create
pip install -r requirements.txt

sudo apt install qemu-utils git kpartx debootstrap

export AMP_HEALTH_KEY="c1768c02628351914e0e5d9fd880535dd75edf92e448a2c2d4266a55d867b6d9"
sudo ./diskimage-create.sh -t qcow2 -s 3 -d jammy
```

## **이미지 업로드**

```bash
glance image-create   --name "amphora"   --file amphora-x64-haproxy.qcow2  --disk-format qcow2   --container-format bare   --tag amphora   --visibility private
```

## Flavor 등록

```bash
openstack flavor create --id 200 --vcpus 1 --ram 2048 --disk 4 "amphora" --private
```

## Key 등록

```bash
openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey
```

## **/etc/octavia/octavia.conf파일 설정**

```bash
[DEFAULT]
transport_url = rabbit://openstack:mnc_rabbit@mncsvrt02

[database]
connection = mysql+pymysql://octavia:mnc1!@mncsvrt02/octavia

[oslo_messaging]
topic = octavia_prov

[api_settings]
bind_host = 0.0.0.0
bind_port = 9876

[keystone_authtoken]
www_authenticate_uri = http://mncsvrt02:5000
auth_url = http://mncsvrt02:5000
memcached_servers = mncsvrt02:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = octavia
password = mnc1!

[service_auth]
auth_url = http://mncsvrt02:5000
memcached_servers = mncsvrt02:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = octavia
password = mnc1!

[certificates]
server_certs_key_passphrase = insecure-key-do-not-use-this-key
ca_private_key_passphrase = not-secure-passphrase
ca_private_key = /etc/octavia/certs/private/server_ca.key.pem
ca_certificate = /etc/octavia/certs/server_ca.cert.pem

[haproxy_amphora]
server_ca = /etc/octavia/certs/server_ca-chain.cert.pem
client_cert = /etc/octavia/certs/private/client.cert-and-key.pem

[health_manager]
bind_port = 5555
bind_ip = 10.0.0.2
controller_ip_port_list = 10.0.0.2:5555
heartbeat_key = c1768c02628351914e0e5d9fd880535dd75edf92e448a2c2d4266a55d867b6d9

[controller_worker]
amp_image_owner_id = b4908086145c46ef9f7f27a6951c2aa8# openstack image show amphora -f value -c owner
amp_image_tag = amphora
amp_ssh_key_name = mykey
amp_secgroup_list = 1c34db28-5c61-4694-8a80-755d51d0468c# openstack security group list -> lb-mgmt-sec-grp ID 입력
amp_boot_network_list = 735c34e2-c9db-430a-b18a-ba21886db775# openstack network list -> lb-mgmt-net ID 입력
amp_flavor_id = 200
network_driver = allowed_address_pairs_driver
compute_driver = compute_nova_driver
amphora_driver = amphora_haproxy_rest_driver
client_ca = /etc/octavia/certs/client_ca.cert.pem

loadbalancer_topology = ACTIVE_STANDBY

[nova]
enable_anti_affinity = True
```

## DB 동기화

```bash
sudo octavia-db-manage --config-file /etc/octavia/octavia.conf upgrade head
```

## **서비스 기동**

```bash
sudo systemctl enable --now octavia-worker octavia-health-manager octavia-housekeeping octavia-api
sudo systemctl restart octavia-api.service octavia-health-manager.service octavia-housekeeping.service octavia-worker.service neutron-dhcp-agent neutron-openvswitch-agent
```

## **서브넷 정보 확인**

```bash
openstack subnet list
#+--------------------------------------+-----------------+--------------------------------------+-----------------+
#| ID                                   | Name            | Network                              | Subnet          |
#+--------------------------------------+-----------------+--------------------------------------+-----------------+
#| 8308109c-71f0-4598-9c78-025d1db5f192 | external_subnet | 739cd02c-03ed-4d45-bed4-c299d79cc735 | 192.168.73.0/24 |
#| b2073dcc-7079-4eb5-812e-83e4b31ac03b | lb-mgmt-subnet  | 735c34e2-c9db-430a-b18a-ba21886db775 | 172.31.0.0/24   |
#| cb688c41-322e-466f-8082-bd246d494bd8 | internal_subnet | 2fe092db-ca0e-46e1-9b81-cd43cc06277c | 172.16.0.0/24   |
#+--------------------------------------+-----------------+--------------------------------------+-----------------+
```

## **LB 생성**

```bash
openstack loadbalancer create --name lb0 --vip-subnet-id <서브넷 ID>
```

# **컨트롤러 재부팅 → o-hm0 / o-bhm0 자동 복구**

```bash
HM_PORT_ID=$(openstack port list --device-owner Octavia:health-mgr -f value -c id)
HM_MAC=$(openstack port show $HM_PORT_ID -f value -c mac_address)
echo "ID=$HM_PORT_ID  MAC=$HM_MAC"
```

## **/usr/local/bin/octavia-hm-iface.sh**

```bash
#!/bin/bash
set -e

# 이미 o-hm0가 있고, IPv4 주소도 붙어있다면 종료
if ip addr show dev o-hm0 | grep -q "inet 10.0.0.2"; then
  echo "[INFO] o-hm0 already configured"
  exit 0
fi

# 포트 정보 (MAC은 실제와 일치시켜야 함)
HM_PORT_ID="a4e07a75-97d7-40e1-8750-08a752b462f0"
HM_MAC="fa:16:3e:ac:68:fb"   # 실제 o-hm0 MAC 주소와 일치시킴
LB_MGMT_IP="10.0.0.2/24"

# veth 페어 생성 (이미 있을 수 있으므로 실패 무시)
ip link add o-hm0 type veth peer name o-bhm0 2>/dev/null || true
ip link set o-hm0 address $HM_MAC

# IP 설정 먼저
ip addr flush dev o-hm0 || true
ip addr add $LB_MGMT_IP dev o-hm0

# 링크 활성화
ip link set o-hm0 up
ip link set o-bhm0 up

# OVS 포트 추가 및 설정
ovs-vsctl --may-exist add-port br-int o-bhm0
ovs-vsctl set interface o-bhm0 external-ids:iface-status=active \
  external-ids:attached-mac=$HM_MAC \
  external-ids:iface-id=$HM_PORT_ID \
  external-ids:skip_cleanup=true

# iptables 허용 (중복 방지)
iptables -C INPUT -i o-hm0 -p udp --dport 5555 -j ACCEPT 2>/dev/null || \
iptables -I INPUT -i o-hm0 -p udp --dport 5555 -j ACCEPT

echo "[INFO] o-hm0 configuration complete"
```

## **/etc/systemd/system/octavia-hm-iface.service**

```bash
[Unit]
Description=Octavia Health-Manager veth auto-recreate
After=network.target openvswitch-switch.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/octavia-hm-iface.sh
RemainAfterExit=yes
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

## **시스템 등록**

```bash
sudo systemctl daemon-reload
sudo chmod +x /usr/local/bin/octavia-hm-iface.sh
sudo systemctl enable --now octavia-hm-iface
sudo systemctl restart octavia-hm-iface
```

---

# **octavia-dashboard**

```bash
git clone https://github.com/openstack/octavia-dashboard.git
```

## Package the octavia_dashboard by running:

```bash
cd octavia-dashboard/

python setup.py sdist
sudo pip3 install octavia-dashboard
```

## Copy `_1482_project_load_balancer_panel.py`

```bash
sudo cp -a octavia_dashboard/enabled/_1482_*.py /var/www/horizon/openstack_dashboard/local/enabled
```

## 적용 및 재시작

```bash
./manage.py collectstatic --noinput
./manage.py compress --force
sudo service apache2 restart
```

---

# LB 생성 및 외부 아이피 할당

Horizon을 기준으로 설명

```bash
Horizon 접속 -> 프로젝트 -> 네트워크 -> Load Balancers -> Create Load Balancers
1. Load Balancers Details 설정
   이름 : 원하는 이름 입력
   Subnet : Internal(내부 네트워크) 선택
2. Listener Details 설정
   이름 : 원하는 이름 입력
   프로토콜 : TCP
   Port : 포워딩할 포트 입력 (ex. 32031)
3. Pool Details 살장
   이름 : 원하는 이름 입력
   Algorithm : 원하는거 선택 (Default : ROUND_ROBIN)
4. Pool Member 설정
   원하는 풀(인스턴스) 추가 버튼 클릭 -> port에 22 입력
5. Monitor Details
   이름 : 원하는 이름 입력
   유형 : TCP
6. 생성
```

생성한 로드밸런서에 리스터 및 풀 추가

```bash
생성된 로드벨런서 이름 클릭
Listeners -> Create Listener
위 2~5번 동일하게 수행
```

생성한 로드밸런서에 외부 아이피 할당

```bash
Horizon 접속 -> 프로젝트 -> 네트워크 -> Floating IP
1. Floating IP 생성(이미 있으면 2부터 진행)
2. 연결 버튼 클릭 -> 포트 선택(생성한 로드밸런서의 IP 선택) 후 연결 클릭
```

[각 노드 root 통신 설정](OpenStack%20Octavia(LB)%20%EA%B5%AC%EC%B6%95/%EA%B0%81%20%EB%85%B8%EB%93%9C%20root%20%ED%86%B5%EC%8B%A0%20%EC%84%A4%EC%A0%95%202fd20970c6f78191b87dc832b0811c58.md)

---

# 각 노드 root 통신 설정
