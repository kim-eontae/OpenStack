# OpenStack Container Orchestration(magnum) 구축

---

---

# **Magnum 설치 및 구성**

데이터베이스 생성

```bash
mysql

CREATE DATABASE magnum;

GRANT ALL PRIVILEGES ON magnum.* TO 'magnum'@'localhost' IDENTIFIED BY 'mnc1!';
GRANT ALL PRIVILEGES ON magnum.* TO 'magnum'@'%' IDENTIFIED BY 'mnc1!'
```

사용자 및 서비스 엔드포인트 등록 

```bash
# 관리자 자격 증명
. admin-openrc

# heat 사용자 생성 및 역할 부여
openstack user create --domain default --password-prompt magnum
openstack role add --project service --user magnum admin

# 서비스 등록
openstack service create --name magnum --description "OpenStack Container Infrastructure Management Service" container-infra

# 엔드포인트 등록 (controller는 호스트명/IP로 대체)
openstack endpoint create --region RegionOne container-infra public http://mncsvrt02:9511/v1
openstack endpoint create --region RegionOne container-infra internal http://mncsvrt02:9511/v1
openstack endpoint create --region RegionOne container-infra admin http://mncsvrt02:9511/v1

# 도메인 생성
openstack domain create --description "Owns users and projects created by magnum" magnum

# magnum 도메인 및 magnum_domain_admin 사용자 도메인 관리자 생성
openstack user create --domain magnum --password-prompt magnum_domain_admin

# magnum 사용자 역할 생성
openstack role add --domain magnum --user-domain magnum --user magnum_domain_admin admin
```

패키지 설치

```bash
sudo DEBIAN_FRONTEND=noninteractive apt-get install magnum-api magnum-conductor python3-magnumclient
```

**`sudo vi /etc/magnum/magnum.conf`**

```bash
[DEFAULT]
transport_url = rabbit://openstack:mnc_rabbit@mncsvrt02

[api]
host = 192.168.73.167

[certificates]
cert_manager_type = x509keypair

[cinder_client]
region_name = RegionOne

[database]
connection = mysql+pymysql://magnum:mnc1!@mncsvrt02/magnum

[keystone_auth]
auth_type=password
auth_url=http://mncsvrt02:5000/v3
username=magnum
password=mnc1!
project_name=service
user_domain_name=Default
project_domain_name=Default
interface=public
region_name=RegionOne

[keystone_authtoken]
memcached_servers = 192.168.73.167:11211
auth_version = v3
www_authenticate_uri = http://mncsvrt02:5000/v3
project_domain_id = default
project_name = service
user_domain_id = default
password = mnc1!
username = magnum
auth_url = http://mncsvrt02:5000
auth_type = password
admin_user = magnum
admin_password = mnc1!
admin_tenant_name = service

[trust]
trustee_domain_name = magnum
trustee_domain_admin_name = magnum_domain_admin
trustee_domain_admin_password = mnc1!
trustee_keystone_interface = public

[oslo_messaging_notifications]
driver = messaging

[drivers]
keystone_auth_default_policy = /etc/magnum/keystone_auth_default_policy.json
```

기본 정책 파일 생성

```bash
sudo tee /etc/magnum/keystone_auth_default_policy.json >/dev/null <<'JSON'
{
  "rules": [
    {
      "apiGroups": [""],
      "resources": ["*"],
      "verbs": ["*"],
      "match": {"type": "role", "values": ["admin"]}
    },
    {
      "apiGroups": [""],
      "resources": ["*"],
      "verbs": ["get","list","watch","create","update","patch","delete"],
      "match": {"type": "role", "values": ["member"]}
    },
    {
      "apiGroups": [""],
      "resources": ["*"],
      "verbs": ["get","list","watch"],
      "match": {"type": "role", "values": ["reader"]}
    }
  ]
}
JSON
```

데이터베이스 초기화 및 서비스 재시작

```bash
sudo -s /bin/sh -c "magnum-db-manage upgrade" magnum

sudo service magnum-api restart
sudo service magnum-conductor restart
```

---

# Magnum Cluster API

https://cluster-api.sigs.k8s.io/user/quick-start

**Management Cluster 준비 (Kind)**

```bash
vi kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  apiServerAddress: "192.168.73.167"
  apiServerPort: 6443
  
kind create cluster --config=kind-config.yaml
# 버전 지정해서 설치
kind create cluster --config kind-config.yaml --image kindest/node:v1.30.13
```

**clusterctl 설치**

```bash
curl -L https://github.com/kubernetes-sigs/cluster-api/releases/download/v1.10.5/clusterctl-linux-amd64 -o clusterctl
sudo install -o root -g root -m 0755 clusterctl /usr/local/bin/clusterctl
```

**OpenStack Resource Controller(ORC) 설치**

```bash
kubectl apply -f https://github.com/k-orc/openstack-resource-controller/releases/latest/download/install.yaml
```

**관리 클러스터에 CAPI + CAPO 초기화**

```bash
clusterctl init --infrastructure openstack
```

**Magnum이 사용할 kubeconfig 준비**

```bash
sudo cp ~/.kube/config /etc/magnum/kubeconfig
sudo chown magnum:magnum /etc/magnum/kubeconfig
sudo chmod 600 /etc/magnum/kubeconfig
```

**Magnum CAPI Helm 드라이버 설치 + 설정**

```bash
git clone https://opendev.org/openstack/magnum-capi-helm.gita
cd magnum-capi-helm

sudo python3 -m pip install --no-deps . --target /usr/lib/python3/dist-packages
```

**Addons Provider/CRDs 설치 – Helm**

https://github.com/azimuth-cloud/capi-helm-charts?tab=readme-ov-file

```bash
# 1) root(또는 sudo)로 capi-addons repo 추가/업데이트
sudo helm repo add capi-addons https://azimuth-cloud.github.io/cluster-api-addon-provider
sudo helm repo update

# 2) addon provider(HelmRelease/Manifests CRD 제공) 설치
sudo helm upgrade --install cluster-api-addon-provider capi-addons/cluster-api-addon-provider \
  -n capi-addon-system --create-namespace \
  --kubeconfig /etc/magnum/kubeconfig

# 3) CRD 설치 확인 (둘 다 보여야 정상)
sudo kubectl --kubeconfig /etc/magnum/kubeconfig get crd | grep addons.stackhpc.com

# (선택) 클러스터 공용 애드온 묶음 설치
sudo helm repo add azimuth-cloud https://azimuth-cloud.github.io/capi-helm-charts
sudo helm repo update
sudo helm upgrade --install cluster-addons azimuth-cloud/cluster-addons \
  --namespace cluster-addons --create-namespace \
  --kubeconfig /etc/magnum/kubeconfig
```

---

# **Kind ↔ OpenStack 연동 네트워크 설정**

## **배경**

- **문제** : Kind(Kubernetes in Docker) 컨트롤러가 OpenStack API 서버(Keystone, Neutron 등)에 접근해야 하지만,
    
    Docker 브리지 네트워크(172.18.0.0/16)와 OpenStack 관리망(192.168.73.0/24) 사이 라우팅/방화벽이 막혀있음.
    
- **해결** : iptables NAT 및 포워딩 규칙을 추가하여 Kind Pod → OpenStack API 통신을 허용함.

## **네트워크 구성 개요**

- **Kind 브리지 네트워크** : 172.18.0.0/16
- **OpenStack API 네트워크** : 192.168.73.0/24
- **허용해야 할 API 포트** : 5000 (Keystone), 9696 (Neutron), 9511, 9311, 8774 (Nova), 9292 (Glance), 8778, 8004 (Heat), 9876, 8776 (Cinder), 8070 …

## 네트워크 설정 추가

```bash
DOCKER_BRIDGE="172.18.0.0/16"   # kind 네트워크
OS_API_CIDR="192.168.73.167/24"   # OpenStack API 망
OS_API_PORTS="5000,9696,9511,9311,8774,9292,8778,8004,9876,8776,8070"

# 커널 포워딩
sudo sysctl -w net.ipv4.ip_forward=1

# NAT (SNAT, Kind → OpenStack API)
sudo iptables -t nat -C POSTROUTING -s "$DOCKER_BRIDGE" -d "$OS_API_CIDR" -j MASQUERADE 2>/dev/null \
  || sudo iptables -t nat -A POSTROUTING -s "$DOCKER_BRIDGE" -d "$OS_API_CIDR" -j MASQUERADE

# FORWARD 허용 (정방향)
sudo iptables -C FORWARD -s "$DOCKER_BRIDGE" -d "$OS_API_CIDR" -j ACCEPT 2>/dev/null \
  || sudo iptables -A FORWARD -s "$DOCKER_BRIDGE" -d "$OS_API_CIDR" -j ACCEPT

# FORWARD 허용 (역방향 응답)
sudo iptables -C FORWARD -s "$OS_API_CIDR" -d "$DOCKER_BRIDGE" -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT 2>/dev/null \
  || sudo iptables -A FORWARD -s "$OS_API_CIDR" -d "$DOCKER_BRIDGE" -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# OpenStack API 포트 호스트 인바운드 허용 (Kind 컨테이너 → 호스트:포트)
sudo iptables -C INPUT -s "$DOCKER_BRIDGE" -p tcp -m multiport --dports $OS_API_PORTS -j ACCEPT 2>/dev/null \
  || sudo iptables -A INPUT -s "$DOCKER_BRIDGE" -p tcp -m multiport --dports $OS_API_PORTS -j ACCEPT
```

네트워크 설정 규칙 확인

```bash
# NAT 규칙 확인
sudo iptables -t nat -L POSTROUTING -n -v | grep 172.18

# FORWARD 규칙 확인
sudo iptables -L FORWARD -n -v | egrep "172.18|192.168.73"

# INPUT 규칙 확인
sudo iptables -L INPUT -n -v | egrep "172.18.*(5000|9696|9511|9311|8774|9292|8778|8004|9876|8776|8070)"
```

**Kind 컨테이너에서 OpenStack API 호출**

```bash
docker exec -it kind-control-plane curl -s -m 5 http://192.168.73.167:5000/v3/ | head
```

## 네트워크 설정 초기화(원복)

```bash
# NAT 룰 삭제
sudo iptables -t nat -D POSTROUTING -s 172.18.0.0/16 -d 192.168.73.167/24 -j MASQUERADE

# FORWARD 룰 삭제
sudo iptables -D FORWARD -s 172.18.0.0/16 -d 192.168.73.167/24 -j ACCEPT
sudo iptables -D FORWARD -s 192.168.73.167/24 -d 172.18.0.0/16 -m state --state ESTABLISHED,RELATED -j ACCEPT

# INPUT 룰 삭제
sudo iptables -D INPUT -s 172.18.0.0/16 -p tcp -m multiport --dports \
5000,9696,9511,9311,8774,9292,8778,8004,9876,8776,8070 -j ACCEPT

# 커널 포워딩 비활성화
sudo sysctl -w net.ipv4.ip_forward=0
```

---

# **Magnum 생성**

https://vexxhost.github.io/magnum-cluster-api/user/getting-started/#creating

키페어 생성

```bash
openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey
```

이미지 다운로드 후 업로드

https://github.com/osism/k8s-capi-images?utm_source=chatgpt.com

```bash
IMG_NAME="ubuntu-2204-kube-v1.32.4"
IMG_FILE="./ubuntu-2204-kube-v1.32.4"

openstack image create "$IMG_NAME" \
  --disk-format qcow2 \
  --container-format bare \
  --file "$IMG_FILE" \
  --property os_distro=ubuntu \
  --property os_admin_user=ubuntu \
  --property hw_disk_bus=virtio \
  --property hw_rng_model=virtio \
  --property kube_version=v1.32.4 \
  --property kube_tag=v1.32.4 \
  --public
```

Kubernetes 클러스터 템플릿 생성 및 클러스터 배포

```bash
openstack coe cluster template create new_driver \
  --coe kubernetes \
  --label octavia_provider=ovn,capi_helm_chart_version=0.15.0 \
  --image $(openstack image show ubuntu-2204-kube-v1.32.4 -c id -f value) \
  --external-network external_network \
  --fixed-network $(openstack subnet show internal_subnet -c network_id -f value) \
  --fixed-subnet  $(openstack subnet show internal_subnet -c id -f value) \
  --master-flavor ds2G20 \
  --flavor ds2G20 \
  --public \
  --master-lb-enabled 
  
openstack coe cluster create devstacktest \
  --cluster-template new_driver \
  --master-count 1 --node-count 2 \
  --keypair mykey
```

---

# **Magnum UI 적용**

```bash
git clone https://github.com/openstack/magnum-ui.git

# 경로 이동
cd magnum-ui
```

패키지 설치

```bash
sudo python3 setup.py install
```

Magnum UI 복사

```bash
sudo cp magnum_ui/enabled/_1370_project_container_infra_panel_group.py \
    /var/www/horizon/openstack_dashboard/local/enabled/

sudo cp magnum_ui/enabled/_1371_project_container_infra_clusters_panel.py \
    /var/www/horizon/openstack_dashboard/local/enabled/

sudo cp magnum_ui/enabled/_1372_project_container_infra_cluster_templates_panel.py \
    /var/www/horizon/openstack_dashboard/local/enabled/
```

정적 파일 수집/압축

```bash
cd /var/www/horizon

./manage.py collectstatic --noinput
./manage.py compress --force
systemctl restart apache2.service
```

---

# 설정 제거

```bash
KCONF="/etc/magnum/kubeconfig"   # 관리 클러스터 kubeconfig
# 만약 현재 셸 기본 kubeconfig가 관리 클러스터면 --kubeconfig 생략 가능

# (1) Helm 애드온 제거
sudo helm --kubeconfig "$KCONF" -n capi-addon-system uninstall cluster-api-addon-provider || true
sudo helm --kubeconfig "$KCONF" -n cluster-addons      uninstall cluster-addons || true

# (2) CRD 제거(Helm이 CRD 안 지운 경우가 많음)
sudo kubectl --kubeconfig "$KCONF" delete crd helmreleases.addons.stackhpc.com 2>/dev/null || true
sudo kubectl --kubeconfig "$KCONF" delete crd manifests.addons.stackhpc.com    2>/dev/null || true

# (3) ORC(OpenStack Resource Controller) 제거
sudo kubectl --kubeconfig "$KCONF" delete -f https://github.com/k-orc/openstack-resource-controller/releases/latest/download/install.yaml || true
# 네임스페이스가 남아있으면 강제 삭제
sudo kubectl --kubeconfig "$KCONF" delete ns openstack-resource-controller-system --ignore-not-found

# (4) CAPI/CAPO 네임스페이스 통째로 제거
for ns in capi-system capi-kubeadm-bootstrap-system capi-kubeadm-control-plane-system capo-system capi-addon-system cluster-addons; do
  kubectl --kubeconfig "$KCONF" delete ns "$ns" --ignore-not-found
done

# (5) 매그넘 전용 네임스페이스(있다면) 제거
sudo kubectl --kubeconfig "$KCONF" delete ns "$NS" --ignore-not-found

# KIND 클러스터 자체 삭제
kind delete cluster || true

#/etc/magnum/kubeconfig 제거
sudo rm -f /etc/magnum/kubeconfig

# KIND 없음
kind get clusters || true

# 관리클러스터 네임스페이스 없음
kubectl --kubeconfig "$KCONF" get ns | egrep 'capi|capo|addon|magnum' || true

# 매그넘 자원 없음
openstack coe cluster list
openstack coe cluster template list

# LB 없음
openstack loadbalancer list

# 이미지/키페어 제거 확인
openstack image list | grep "$OS_IMAGE" || true
openstack keypair list | grep "$OS_KEY" || true
```
