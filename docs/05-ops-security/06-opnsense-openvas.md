# OPNsenese 및 OpenVAS 구축

---

---

# OPNsense - 방화벽

### 1. OPNsense 이미지 다운

```bash
sudo wget https://mirror.ntct.edu.tw/opnsense/releases/24.1/OPNsense-24.1-nano-amd64.img.bz2

# 압축 해제
bzip2 -d OPNsense-24.1-nano-amd64.img.bz2

# 이미지 변환 (raw → qcow2)
qemu-img convert -f raw -O qcow2 OPNsense-24.1-nano-amd64.img OPNsense-24.1-nano-amd64.qcow2
```

### 2. Glance 이미지 등록

```bash
openstack image create \
  --disk-format qcow2 \
  --container-format bare \
  --min-disk 8 \
  --file OPNsense-24.1-nano-amd64.qcow2 \
  "OPNsense_24.1"
```

### 3. 인스턴스 생성 - Horizon 기준

```bash
[Compute] -> [인스턴스] -> [인스턴스 시작] -> [세부 정보]  
  -> 인스턴스 이름 입력  
-> [소스]  
  -> 새로운 볼륨 생성 (아니오)  
  -> 위에서 생성한 이미지 선택  
-> [Flavor]  
  -> m1.small 선택  
-> [네트워크]  
  -> internal 선택  
-> [보안그룹]  
  -> 필요한 보안 그룹 생성 또는 선택 (일단 GenOS로 설정)
```

### 4. 외부 네트워크  IP 할당

> NOTE
>  기존 인스턴스에 할당하던 외부 아이피(Floating IP)와 다르게
>
> Floating IP에 할당되지 않은 외부 아이피를 할당해야 함
>
> → [할당 방법]
>
> 인스턴스 Actions의 **인터페이스 연결** 클릭
>
> → [네트워크] → 외부망 선택
>
> → [고정 IP 주소] → 할당할 아이피 입력 (ex. 192.168.73.183)


### 5. 인스턴스 접속 후 네트워크 설정

```bash
0) login
-> ID : root
-> PW : opnsense

1) Assign interfaces
-> WAN - 외부 아이피 선택 - vtnet1
-> LAN - 내부 아이피 선택 - vtnet0

# WAN 인터페이스 IP 재설정
2) Set interface IP address
# 수동 아이피 할당
Configure IPv4 address WAN interface via DHCP? [Y/n] n
Enter the new IPv4 address for WAN: 192.168.73.183
Subnet bits: 24
Upstream Gateway: 192.168.73.161
Do you want to use the gateway as the IPv4 name server? n
Enter the IPv4 name server or press <ENTER> for none: 8.8.8.8

# IPv6 사용 안 함
Configure IPv6 address WAN interface via DHCP6? n

# 웹 GUI 프로토콜
Do you want to change the web GUI protocol from HTTPS to HTTP? n 

# self-signed certificate
Do you want to generate a new self-signed web GUI certificate? n

# GUI 기본 설정 복구 여부
Restore web GUI access defaults? n
```

### 6. 외부 연결 확인

```bash
ping 8.8.8.8
```

### 7. 비밀번호 변경

```bash
3) Reset the root password
# 비밀번호 변경
```

### 8. 웹 접속 방화벽 설정

```bash
8) Shell
Enter an option: 8

# 방화벽 전체 해제
pfctl -d

# 아래 진행하는 방화벽 설정후 방화벽 활성화 필요
pfctl -e
```

### 9. 웹 접속

```bash
# WAN(외부 네트워크로 접속)
https://192.168.73.183
```

### 10. 외부 방화벽 설정

```bash
- 위치: `[Interfaces] -> [WAN]`

 Block private networks	**체크 해제**
 Block bogon networks **체크 해제**
```

---

### OPNsense Firewall 설정 (GenOS 기준)

```bash
### OPNsense Firewall 설정 (GenOS 기준)

- 위치: `[Firewall] -> [Rules] -> [WAN]` -> `[ADD]`

- 설정 항목:
  - Direction  
    - IN, OUT (방향 지정)
  - Protocol  
    - TCP, UDP 등 선택
  - Destination  
    - WAN address : 외부망에서만 적용  
    - This Firewall : 방화벽 자체에 적용
  - Destination port range  
    - 허용할 포트 또는 포트 범위 지정
  - Description  
    - 규칙 설명 작성

----------------------------
# 생성 예시
IPv4	Protocol	Source Port	Description
IPv4	TCP	*	*	This Firewall	443 (HTTPS)	Allow WebUI (HTTPS)
IPv4	TCP	*	*	This Firewall	22 (SSH)	Allow SSH
IPv4	TCP	*	*	WAN address	30908	k8s_api_front port
IPv4	TCP	*	*	WAN address	32022 - 32028	SSH Proxy Ports
IPv4	TCP	*	*	WAN address	8080	gpu-operator, llmops API, 모니터링, 오케스트레이터 등 내부 API 서비스
IPv4	TCP	*	*	WAN address	3306	llmops-mariadb-service
IPv4	TCP	*	*	WAN address	27017	llmops-mongodb-service
IPv4	TCP	*	*	WAN address	9000 - 9001	llmops-minio-service
```

### OPNsense Firewall Aliases 설정 (GenOS - Istio)

```bash
- 위치: `[Firewall] -> [Alias]`

- 사용 목적
	- 여러 포트를 그룹화하여 규칙에서 이름으로 간편하게 참조하기 위해 사용

- 설정 항목:
  - Name  
    - 설정 이름
  - Type  
    - Port(s)
  - Content
		- 허용할 포트들
	- Description  
    - 규칙 설명 작성

----------------------------
# 생성 예시
	
Name Type Description Content
Istio_ingressgateway	Port(s)	Istio ingressgateway ports	31466 32154 30367 31558 32642 3104...
```

---

### VIP 설정

```bash
[Interfaces] → [Virtual IPs] → [Settings]
→ [Add]
→ Mode: IP Alias
→ Interface: LAN
→ Address: [VIP로 사용할 IP/대역] (예: 172.16.0.182/32)
```

---

### 방화벽 서비스 업데이트 및 플러그인(haproxy) 설치

```bash
# 업데이트
[System] → [Firmware] → [Status]
→ [Check for updates] 클릭
→ [Updates] 항목 확인 후 [Upgrade] 클릭

# 플러그인 설치
[System] → [Firmware] → [Plugins]
→ 'os-haproxy' 검색
→ [Install] 클릭
```

## Haproxy 설정

```bash
# Real Servers 설정
```
[Services] → [HAProxy] → [Settings]
→ [Real Servers] → Add

Name or Prefix       : 이름 입력 (예: master-k8s-01-genos)
Type                 : static
FQDN or IP           : IP 입력 (예: 172.16.0.194)
Port                 : 사용할 포트 입력 (예: 30908)
Description          : 설명 입력 (예: genos-front)
Mode                 : active [default]
```

---

Backend Pools 설정

```
[Services] → [HAProxy] → [Settings]
→ [Virtual Services] → [Backend Pools] → Add

Name                 : 이름 입력 (예: genos-back)
Description          : 설명 입력
Mode                 : TCP / HTTP 중 선택
Balancing Algorithm  : 예: Round Robin
Servers              : Real Servers에서 만든 서버 선택
                      (예: master-k8s-01-genos, master-k8s-02-genos, master-k8s-03-genos)
```

---

Public Services 설정 (프론트엔드)

```
[Services] → [HAProxy] → [Settings]
→ [Virtual Services] → [Public Services] → Add

Name                 : 이름 입력 (예: genos-front)
Description          : 설명 입력
Listen Addresses     : IP:Port 입력 (예: *:30908)
Type                 : HTTP/HTTPS (SSL offloading) [default]
Default Backend Pool : Backend Pools에서 만든 항목 선택 (예: genos-back)
```

---

기본 HAProxy 설정 (Default Parameters)

```
[Services] → [HAProxy] → [Settings]
→ [Settings] → [Default Parameters]

Client Timeout       : 30s (기본값)
Connection Timeout   : 30s (기본값)
Server Timeout       : 30s (기본값)
```
```

### HAProxy 설정 확인

```bash
[Services] -> [HAProxy] -> [Config Export]

#
# Automatically generated configuration.
# Do not edit this file manually.
#

global
    uid                         80
    gid                         80
    chroot                      /var/haproxy
    daemon
    stats                       socket /var/run/haproxy.socket group proxy mode 775 level admin
    nbthread                    1
    hard-stop-after             60s
    no strict-limits
    httpclient.resolvers.prefer   ipv4
    tune.ssl.default-dh-param   2048
    spread-checks               2
    tune.bufsize                16384
    tune.lua.maxmem             0
    log                         /var/run/log local0 info
    lua-prepend-path            /tmp/haproxy/lua/?.lua

defaults
    log     global
    option redispatch -1
    timeout client 1h
    timeout connect 12h
    timeout server 12h
    retries 3
    default-server init-addr libc,last

# autogenerated entries for ACLs

# autogenerated entries for config in backends/frontends

# autogenerated entries for stats

# Frontend: genos-front ()
frontend genos-front
    bind *:30908 name *:30908 
    mode http
    option http-keep-alive
    default_backend genos-back

    # logging options

# Backend: genos-back ()
backend genos-back
    # health checking is DISABLED
    mode http
    balance roundrobin
    # stickiness
    stick-table type ip size 50k expire 30m  
    stick on src
    http-reuse safe
    server master-k8s-01-genos 172.16.0.194:30908 
    server master-k8s-02-genos 172.16.0.96:30908 
    server master-k8s-03-genos 172.16.0.99:30908 

# statistics are DISABLED
```

---

OPNsense - 방화벽 설정 사항

- Firewall → Rules → LAN
    
    
    | IN / OUT | Protocol | Source | port | Destination | Port | Description | State Type |
    | --- | --- | --- | --- | --- | --- | --- | --- |
    | IN | IPv4 * | LAN net | * | * | * | Default allow LAN to any rule | none |
    | IN | IPv6 * | LAN net | * | * | * | Default allow LAN IPv6 to any rule | none |
    
    > State Type이 none으로 설정되어야 내부 통신과 k8s 통신이 block처리 되지 않음
    > 
- Firewall → Rules → WAN
    
    
    | IN / OUT | Protocol | Source | port | Destination | Port | Description | State Type |
    | --- | --- | --- | --- | --- | --- | --- | --- |
    | IN | IPv4 TCP | * | * | This Firewall | 443 (HTTPS) | Allow WebUI (HTTPS) |  |
    | IN | IPv4 TCP | * | * | This Firewall | 22 (SSH) | Allow SSH |  |
    | IN | IPv4 ICMP | * | * | * | * | Allow Ping |  |
    | IN | IPv4 TCP | * | * | WAN address | - 30908 | k8s_api_front port |  |
    | IN | IPv4 TCP | * | * | WAN address | 32022 - 32028 | SSH Proxy Port 1 |  |
    | IN | IPv4 TCP | * | * | WAN address | - 8080 | gpu-operator, llmops 각종 API 서비스, 모니터링, 오케스트레이터 등 내부 API 서비스 |  |
    | IN | IPv4 TCP | * | * | WAN address | - 3306 | llmops-mariadb-service |  |
    | IN | IPv4 TCP | * | * | WAN address | - 27017 | llmops-mongodb-service |  |
    | IN | IPv4 TCP | * | * | WAN address | 9000 - 9001 | llmops-minio-service |  |
    | IN | IPv4 TCP | * | * | WAN address | - 5000 | llmops-registry-service |  |
    | IN | IPv4 TCP | * | * | WAN address | - 6379 | llmops-redis-service |  |
    | IN | IPv4 TCP | * | * | WAN address | 15672 - 15674 | RabbitMQ 관리 및 통신 포트 |  |
    | IN | IPv4 TCP | * | * | WAN address | - 5672 | RabbitMQ 기본 통신 포트 |  |
    | IN | IPv4 TCP | * | * | WAN address | 9090 - 9093 | Prometheus 모니터링 서비스 포트 |  |
    | IN | IPv4 TCP | * | * | WAN address | - 31920 | Redis NodePort 포트 |  |
    | IN | IPv4 TCP | * | * | WAN address | - 30500 | Registry NodePort 포트 |  |
    | IN | IPv4 TCP | * | * | WAN address | - 30281 | PyPI 서버 NodePort 포트 |  |
    | IN | IPv4 TCP | * | * | WAN address | - 80 | HTTP |  |
    | IN | IPv4 TCP | * | * | WAN address | - 3000 | 특정 앱 서비스(Flowise 등) |  |
    | IN | IPv4 TCP | * | * | WAN address | - 16443 | kubernetes-lb |  |
    | IN | IPv4 TCP | * | * | WAN address | Istio_ingressgateway  | Istio ingressgateway |  |
    | IN | IPv4 UDP | * | * | WAN address | - 53 |  |  |
    | IN | IPv4 TCP | * | * | WAN address | - 30999 | minio |  |
    | IN | IPv4 TCP | * | * | WAN address | - 30920 | OpenVAS |  |
    | IN | IPv4 TCP | * | * | WAN address | - 6443 | kube-apiserver |  |
    | IN | IPv4 * | 10.10.10.0/24 | * | WAN address | * | k8s 내부 네트워크 |  |

---

# OepnVAS - 네트워크 취약점 탐지

## 공식문서 참고하여 설치

[https://greenbone.github.io/docs/latest/22.4/source-build/index.html](https://greenbone.github.io/docs/latest/22.4/source-build/index.html)

> NOTE
>  OpenVAS설치시 CentOS9 OS에 구축 시 구축 사항에서 많은 에러가 발생하기에
> Ubuntu 24.04 OS로 설치 함


## OpenVAS 설치 스크립트(Ubuntu 24.04)

`213번` 라인에 비밀번호를 수정하지 않으면 기본 계정은 admin / mnc1!로 생성됨

`install_gvm_full.sh`

```bash
#!/bin/bash
set -e

# Create gvm user and groups
sudo useradd -r -M -U -G sudo -s /usr/sbin/nologin gvm
sudo usermod -aG gvm $USER

# Environment setup
export INSTALL_PREFIX=/usr/local
export PATH=$PATH:$INSTALL_PREFIX/sbin
export SOURCE_DIR=$HOME/source
export BUILD_DIR=$HOME/build
export INSTALL_DIR=$HOME/install

mkdir -p $SOURCE_DIR $BUILD_DIR $INSTALL_DIR

# Install required packages
sudo apt update
sudo apt install --no-install-recommends --assume-yes build-essential curl cmake pkg-config python3 python3-pip gnupg

# Import GPG key
curl -f -L https://www.greenbone.net/GBCommunitySigningKey.asc -o /tmp/GBCommunitySigningKey.asc
gpg --import /tmp/GBCommunitySigningKey.asc
echo "8AE4BE429B60A59B311C2E739823FAA60ED1E580:6:" | gpg --import-ownertrust

# Install dependencies for gvm-libs
export GVM_LIBS_VERSION=22.22.0
sudo apt install -y libcjson-dev libcurl4-gnutls-dev libgcrypt-dev libglib2.0-dev libgnutls28-dev libgpgme-dev libhiredis-dev libnet1-dev libpaho-mqtt-dev libpcap-dev libssh-dev libxml2-dev uuid-dev libldap2-dev libradcli-dev

# Download and build gvm-libs
curl -f -L https://github.com/greenbone/gvm-libs/archive/refs/tags/v$GVM_LIBS_VERSION.tar.gz -o $SOURCE_DIR/gvm-libs-$GVM_LIBS_VERSION.tar.gz
curl -f -L https://github.com/greenbone/gvm-libs/releases/download/v$GVM_LIBS_VERSION/gvm-libs-$GVM_LIBS_VERSION.tar.gz.asc -o $SOURCE_DIR/gvm-libs-$GVM_LIBS_VERSION.tar.gz.asc
gpg --verify $SOURCE_DIR/gvm-libs-$GVM_LIBS_VERSION.tar.gz.asc $SOURCE_DIR/gvm-libs-$GVM_LIBS_VERSION.tar.gz
tar -C $SOURCE_DIR -xvzf $SOURCE_DIR/gvm-libs-$GVM_LIBS_VERSION.tar.gz
mkdir -p $BUILD_DIR/gvm-libs
cmake -S $SOURCE_DIR/gvm-libs-$GVM_LIBS_VERSION -B $BUILD_DIR/gvm-libs -DCMAKE_INSTALL_PREFIX=$INSTALL_PREFIX -DCMAKE_BUILD_TYPE=Release -DSYSCONFDIR=/etc -DLOCALSTATEDIR=/var
cmake --build $BUILD_DIR/gvm-libs -j$(nproc)
mkdir -p $INSTALL_DIR/gvm-libs && cd $BUILD_DIR/gvm-libs
make DESTDIR=$INSTALL_DIR/gvm-libs install
sudo cp -rv $INSTALL_DIR/gvm-libs/* /

# Continue the rest of the script manually from this point (it is very long and best maintained in segments or external file)

# Set GVMD version and install dependencies
export GVMD_VERSION=26.0.0
sudo apt install -y libbsd-dev libcjson-dev libglib2.0-dev libgnutls28-dev libgpgme-dev libical-dev libpq-dev postgresql-server-dev-all rsync xsltproc
sudo apt install -y --no-install-recommends dpkg fakeroot gnupg gnutls-bin gpgsm nsis openssh-client python3 python3-lxml rpm smbclient snmp socat sshpass texlive-fonts-recommended texlive-latex-extra wget xmlstarlet zip

# Download and build GVMD
curl -f -L https://github.com/greenbone/gvmd/archive/refs/tags/v$GVMD_VERSION.tar.gz -o $SOURCE_DIR/gvmd-$GVMD_VERSION.tar.gz
curl -f -L https://github.com/greenbone/gvmd/releases/download/v$GVMD_VERSION/gvmd-$GVMD_VERSION.tar.gz.asc -o $SOURCE_DIR/gvmd-$GVMD_VERSION.tar.gz.asc
gpg --verify $SOURCE_DIR/gvmd-$GVMD_VERSION.tar.gz.asc $SOURCE_DIR/gvmd-$GVMD_VERSION.tar.gz
tar -C $SOURCE_DIR -xvzf $SOURCE_DIR/gvmd-$GVMD_VERSION.tar.gz
mkdir -p $BUILD_DIR/gvmd
cmake -S $SOURCE_DIR/gvmd-$GVMD_VERSION -B $BUILD_DIR/gvmd -DCMAKE_INSTALL_PREFIX=$INSTALL_PREFIX -DCMAKE_BUILD_TYPE=Release -DLOCALSTATEDIR=/var -DSYSCONFDIR=/etc -DGVM_DATA_DIR=/var -DGVM_LOG_DIR=/var/log/gvm -DGVMD_RUN_DIR=/run/gvmd -DOPENVAS_DEFAULT_SOCKET=/run/ospd/ospd-openvas.sock -DGVM_FEED_LOCK_PATH=/var/lib/gvm/feed-update.lock -DLOGROTATE_DIR=/etc/logrotate.d
cmake --build $BUILD_DIR/gvmd -j$(nproc)
mkdir -p $INSTALL_DIR/gvmd && cd $BUILD_DIR/gvmd
make DESTDIR=$INSTALL_DIR/gvmd install
sudo cp -rv $INSTALL_DIR/gvmd/* /

# Continue appending rest of the script as needed...

# Set PG_GVM version and install dependencies
export PG_GVM_VERSION=22.6.9
sudo apt install -y libglib2.0-dev libical-dev postgresql-server-dev-all
curl -f -L https://github.com/greenbone/pg-gvm/archive/refs/tags/v$PG_GVM_VERSION.tar.gz -o $SOURCE_DIR/pg-gvm-$PG_GVM_VERSION.tar.gz
curl -f -L https://github.com/greenbone/pg-gvm/releases/download/v$PG_GVM_VERSION/pg-gvm-$PG_GVM_VERSION.tar.gz.asc -o $SOURCE_DIR/pg-gvm-$PG_GVM_VERSION.tar.gz.asc
tar -C $SOURCE_DIR -xvzf $SOURCE_DIR/pg-gvm-$PG_GVM_VERSION.tar.gz
mkdir -p $BUILD_DIR/pg-gvm
cmake -S $SOURCE_DIR/pg-gvm-$PG_GVM_VERSION -B $BUILD_DIR/pg-gvm -DCMAKE_BUILD_TYPE=Release
cmake --build $BUILD_DIR/pg-gvm -j$(nproc)
mkdir -p $INSTALL_DIR/pg-gvm && cd $BUILD_DIR/pg-gvm
make DESTDIR=$INSTALL_DIR/pg-gvm install
sudo cp -rv $INSTALL_DIR/pg-gvm/* /

# Set GSA version and install
export GSA_VERSION=25.0.0
curl -f -L https://github.com/greenbone/gsa/releases/download/v$GSA_VERSION/gsa-dist-$GSA_VERSION.tar.gz -o $SOURCE_DIR/gsa-$GSA_VERSION.tar.gz
curl -f -L https://github.com/greenbone/gsa/releases/download/v$GSA_VERSION/gsa-dist-$GSA_VERSION.tar.gz.asc -o $SOURCE_DIR/gsa-$GSA_VERSION.tar.gz.asc
gpg --verify $SOURCE_DIR/gsa-$GSA_VERSION.tar.gz.asc $SOURCE_DIR/gsa-$GSA_VERSION.tar.gz
mkdir -p $SOURCE_DIR/gsa-$GSA_VERSION
tar -C $SOURCE_DIR/gsa-$GSA_VERSION -xvzf $SOURCE_DIR/gsa-$GSA_VERSION.tar.gz
sudo mkdir -p $INSTALL_PREFIX/share/gvm/gsad/web/
sudo cp -rv $SOURCE_DIR/gsa-$GSA_VERSION/* $INSTALL_PREFIX/share/gvm/gsad/web/

# Set GSAD version and install
export GSAD_VERSION=24.3.0
sudo apt install -y libbrotli-dev libglib2.0-dev libgnutls28-dev libmicrohttpd-dev libxml2-dev
curl -f -L https://github.com/greenbone/gsad/archive/refs/tags/v$GSAD_VERSION.tar.gz -o $SOURCE_DIR/gsad-$GSAD_VERSION.tar.gz
curl -f -L https://github.com/greenbone/gsad/releases/download/v$GSAD_VERSION/gsad-$GSAD_VERSION.tar.gz.asc -o $SOURCE_DIR/gsad-$GSAD_VERSION.tar.gz.asc
gpg --verify $SOURCE_DIR/gsad-$GSAD_VERSION.tar.gz.asc $SOURCE_DIR/gsad-$GSAD_VERSION.tar.gz
tar -C $SOURCE_DIR -xvzf $SOURCE_DIR/gsad-$GSAD_VERSION.tar.gz
mkdir -p $BUILD_DIR/gsad
cmake -S $SOURCE_DIR/gsad-$GSAD_VERSION -B $BUILD_DIR/gsad -DCMAKE_INSTALL_PREFIX=$INSTALL_PREFIX -DCMAKE_BUILD_TYPE=Release -DSYSCONFDIR=/etc -DLOCALSTATEDIR=/var -DGVMD_RUN_DIR=/run/gvmd -DGSAD_RUN_DIR=/run/gsad -DGVM_LOG_DIR=/var/log/gvm -DLOGROTATE_DIR=/etc/logrotate.d
cmake --build $BUILD_DIR/gsad -j$(nproc)
mkdir -p $INSTALL_DIR/gsad && cd $BUILD_DIR/gsad
make DESTDIR=$INSTALL_DIR/gsad install
sudo cp -rv $INSTALL_DIR/gsad/* /

# Set OPENVAS_SMB version and install
export OPENVAS_SMB_VERSION=22.5.3
sudo apt install -y gcc-mingw-w64 libgnutls28-dev libglib2.0-dev libpopt-dev libunistring-dev heimdal-multidev perl-base
curl -f -L https://github.com/greenbone/openvas-smb/archive/refs/tags/v$OPENVAS_SMB_VERSION.tar.gz -o $SOURCE_DIR/openvas-smb-$OPENVAS_SMB_VERSION.tar.gz
curl -f -L https://github.com/greenbone/openvas-smb/releases/download/v$OPENVAS_SMB_VERSION/openvas-smb-v$OPENVAS_SMB_VERSION.tar.gz.asc -o $SOURCE_DIR/openvas-smb-$OPENVAS_SMB_VERSION.tar.gz.asc
gpg --verify $SOURCE_DIR/openvas-smb-$OPENVAS_SMB_VERSION.tar.gz.asc $SOURCE_DIR/openvas-smb-$OPENVAS_SMB_VERSION.tar.gz
tar -C $SOURCE_DIR -xvzf $SOURCE_DIR/openvas-smb-$OPENVAS_SMB_VERSION.tar.gz
mkdir -p $BUILD_DIR/openvas-smb
cmake -S $SOURCE_DIR/openvas-smb-$OPENVAS_SMB_VERSION -B $BUILD_DIR/openvas-smb -DCMAKE_INSTALL_PREFIX=$INSTALL_PREFIX -DCMAKE_BUILD_TYPE=Release
cmake --build $BUILD_DIR/openvas-smb -j$(nproc)
mkdir -p $INSTALL_DIR/openvas-smb && cd $BUILD_DIR/openvas-smb
make DESTDIR=$INSTALL_DIR/openvas-smb install
sudo cp -rv $INSTALL_DIR/openvas-smb/* /

# Set OPENVAS_SCANNER version and install
export OPENVAS_SCANNER_VERSION=23.20.1
sudo apt install -y bison libglib2.0-dev libgnutls28-dev libgcrypt20-dev libpcap-dev libgpgme-dev libksba-dev rsync nmap libjson-glib-dev libcurl4-gnutls-dev libbsd-dev krb5-multidev python3-impacket libsnmp-dev
curl -f -L https://github.com/greenbone/openvas-scanner/archive/refs/tags/v$OPENVAS_SCANNER_VERSION.tar.gz -o $SOURCE_DIR/openvas-scanner-$OPENVAS_SCANNER_VERSION.tar.gz
curl -f -L https://github.com/greenbone/openvas-scanner/releases/download/v$OPENVAS_SCANNER_VERSION/openvas-scanner-v$OPENVAS_SCANNER_VERSION.tar.gz.asc -o $SOURCE_DIR/openvas-scanner-$OPENVAS_SCANNER_VERSION.tar.gz.asc
gpg --verify $SOURCE_DIR/openvas-scanner-$OPENVAS_SCANNER_VERSION.tar.gz.asc $SOURCE_DIR/openvas-scanner-$OPENVAS_SCANNER_VERSION.tar.gz
tar -C $SOURCE_DIR -xvzf $SOURCE_DIR/openvas-scanner-$OPENVAS_SCANNER_VERSION.tar.gz
mkdir -p $BUILD_DIR/openvas-scanner
cmake -S $SOURCE_DIR/openvas-scanner-$OPENVAS_SCANNER_VERSION -B $BUILD_DIR/openvas-scanner -DCMAKE_INSTALL_PREFIX=$INSTALL_PREFIX -DCMAKE_BUILD_TYPE=Release -DSYSCONFDIR=/etc -DLOCALSTATEDIR=/var -DOPENVAS_FEED_LOCK_PATH=/var/lib/openvas/feed-update.lock -DOPENVAS_RUN_DIR=/run/ospd
cmake --build $BUILD_DIR/openvas-scanner -j$(nproc)
mkdir -p $INSTALL_DIR/openvas-scanner && cd $BUILD_DIR/openvas-scanner
make DESTDIR=$INSTALL_DIR/openvas-scanner install
sudo cp -rv $INSTALL_DIR/openvas-scanner/* /
printf "table_driven_lsc = yes
" | sudo tee /etc/openvas/openvas.conf
printf "openvasd_server = http://127.0.0.1:3000
" | sudo tee -a /etc/openvas/openvas.conf

# Set OSPD_OPENVAS version and install
export OSPD_OPENVAS_VERSION=22.9.0
sudo apt install -y python3 python3-pip python3-setuptools python3-packaging python3-wrapt python3-cffi python3-psutil python3-lxml python3-defusedxml python3-paramiko python3-redis python3-gnupg python3-paho-mqtt
curl -f -L https://github.com/greenbone/ospd-openvas/archive/refs/tags/v$OSPD_OPENVAS_VERSION.tar.gz -o $SOURCE_DIR/ospd-openvas-$OSPD_OPENVAS_VERSION.tar.gz
curl -f -L https://github.com/greenbone/ospd-openvas/releases/download/v$OSPD_OPENVAS_VERSION/ospd-openvas-v$OSPD_OPENVAS_VERSION.tar.gz.asc -o $SOURCE_DIR/ospd-openvas-$OSPD_OPENVAS_VERSION.tar.gz.asc
gpg --verify $SOURCE_DIR/ospd-openvas-$OSPD_OPENVAS_VERSION.tar.gz.asc $SOURCE_DIR/ospd-openvas-$OSPD_OPENVAS_VERSION.tar.gz
tar -C $SOURCE_DIR -xvzf $SOURCE_DIR/ospd-openvas-$OSPD_OPENVAS_VERSION.tar.gz
cd $SOURCE_DIR/ospd-openvas-$OSPD_OPENVAS_VERSION
mkdir -p $INSTALL_DIR/ospd-openvas
python3 -m pip install --root=$INSTALL_DIR/ospd-openvas --no-warn-script-location .
sudo cp -rv $INSTALL_DIR/ospd-openvas/* /

# Set OPENVAS_DAEMON version and install via Rust
export OPENVAS_DAEMON=23.20.0
sudo apt install -y rustup pkg-config libssl-dev
rustup update stable
curl -f -L https://github.com/greenbone/openvas-scanner/archive/refs/tags/v$OPENVAS_DAEMON.tar.gz -o $SOURCE_DIR/openvas-scanner-$OPENVAS_DAEMON.tar.gz
curl -f -L https://github.com/greenbone/openvas-scanner/releases/download/v$OPENVAS_DAEMON/openvas-scanner-v$OPENVAS_DAEMON.tar.gz.asc -o $SOURCE_DIR/openvas-scanner-$OPENVAS_DAEMON.tar.gz.asc
gpg --verify $SOURCE_DIR/openvas-scanner-$OPENVAS_DAEMON.tar.gz.asc $SOURCE_DIR/openvas-scanner-$OPENVAS_DAEMON.tar.gz
tar -C $SOURCE_DIR -xvzf $SOURCE_DIR/openvas-scanner-$OPENVAS_DAEMON.tar.gz
mkdir -p $INSTALL_DIR/openvasd/usr/local/bin
cd $SOURCE_DIR/openvas-scanner-$OPENVAS_DAEMON/rust/src/openvasd
cargo build --release
cd $SOURCE_DIR/openvas-scanner-$OPENVAS_DAEMON/rust/src/scannerctl
cargo build --release
sudo cp -v ../../target/release/openvasd $INSTALL_DIR/openvasd/usr/local/bin/
sudo cp -v ../../target/release/scannerctl $INSTALL_DIR/openvasd/usr/local/bin/
sudo cp -rv $INSTALL_DIR/openvasd/* /

# Install greenbone-feed-sync and gvm-tools
sudo apt install -y python3 python3-pip
mkdir -p $INSTALL_DIR/greenbone-feed-sync
python3 -m pip install --root=$INSTALL_DIR/greenbone-feed-sync --no-warn-script-location greenbone-feed-sync
sudo cp -rv $INSTALL_DIR/greenbone-feed-sync/* /

sudo apt install -y python3-lxml python3-packaging python3-paramiko python3-setuptools python3-venv
mkdir -p $INSTALL_DIR/gvm-tools
python3 -m pip install --root=$INSTALL_DIR/gvm-tools --no-warn-script-location gvm-tools
sudo cp -rv $INSTALL_DIR/gvm-tools/* /

# Setup Redis for openvas
sudo apt install -y redis-server
sudo cp $SOURCE_DIR/openvas-scanner-$OPENVAS_SCANNER_VERSION/config/redis-openvas.conf /etc/redis/
sudo chown redis:redis /etc/redis/redis-openvas.conf
echo "db_address = /run/redis-openvas/redis.sock" | sudo tee -a /etc/openvas/openvas.conf
sudo systemctl enable redis-server@openvas.service
sudo systemctl start redis-server@openvas.service

# Permissions
sudo usermod -aG redis gvm
sudo mkdir -p /var/lib/notus /run/gvmd
sudo chown -R gvm:gvm /var/lib/gvm /var/lib/openvas /var/lib/notus /var/log/gvm /run/gvmd
sudo chmod -R g+srw /var/lib/gvm /var/lib/openvas /var/log/gvm
sudo chown gvm:gvm /usr/local/sbin/gvmd
sudo chmod 6750 /usr/local/sbin/gvmd

# GPG Key setup for /etc/openvas/gnupg
curl -f -L https://www.greenbone.net/GBCommunitySigningKey.asc -o /tmp/GBCommunitySigningKey.asc
export GNUPGHOME=/tmp/openvas-gnupg
mkdir -p $GNUPGHOME
gpg --import /tmp/GBCommunitySigningKey.asc
echo "8AE4BE429B60A59B311C2E739823FAA60ED1E580:6:" | gpg --import-ownertrust
export OPENVAS_GNUPG_HOME=/etc/openvas/gnupg
sudo mkdir -p $OPENVAS_GNUPG_HOME
sudo cp -r /tmp/openvas-gnupg/* $OPENVAS_GNUPG_HOME/
sudo chown -R gvm:gvm $OPENVAS_GNUPG_HOME

# Add gvm sudo permissions
echo "%gvm ALL = NOPASSWD: /usr/local/sbin/openvas" | sudo tee /etc/sudoers.d/gvm
sudo chmod 0440 /etc/sudoers.d/gvm

# PostgreSQL setup
sudo apt install -y postgresql
sudo systemctl enable postgresql
sudo systemctl start postgresql

sudo -u postgres bash -c "createuser -DRS gvm"
sudo -u postgres bash -c "createdb -O gvm gvmd"
sudo -u postgres psql gvmd -c "create role dba with superuser noinherit; grant dba to gvm;"

# Create admin user for gvmd
/usr/local/sbin/gvmd --create-user=admin --password=mnc1!
/usr/local/sbin/gvmd --modify-setting 78eceaec-3385-11ea-b237-28d24461215b --value $(/usr/local/sbin/gvmd --get-users --verbose | grep admin | awk '{print $2}')

# Create systemd services
cat << EOF | sudo tee /etc/systemd/system/ospd-openvas.service
[Unit]
Description=OSPd Wrapper for the OpenVAS Scanner (ospd-openvas)
Documentation=man:ospd-openvas(8) man:openvas(8)
After=network.target networking.service redis-server@openvas.service openvasd.service
Wants=redis-server@openvas.service openvasd.service
ConditionKernelCommandLine=!recovery

[Service]
Type=exec
User=gvm
Group=gvm
RuntimeDirectory=ospd
RuntimeDirectoryMode=2775
PIDFile=/run/ospd/ospd-openvas.pid
ExecStart=/usr/local/bin/ospd-openvas --foreground --unix-socket /run/ospd/ospd-openvas.sock --pid-file /run/ospd/ospd-openvas.pid --log-file /var/log/gvm/ospd-openvas.log --lock-file-dir /var/lib/openvas --socket-mode 0o770 --notus-feed-dir /var/lib/notus/advisories
SuccessExitStatus=SIGKILL
Restart=always
RestartSec=60

[Install]
WantedBy=multi-user.target
EOF

cat << EOF | sudo tee /etc/systemd/system/gvmd.service
[Unit]
Description=Greenbone Vulnerability Manager daemon (gvmd)
After=network.target networking.service postgresql.service ospd-openvas.service
Wants=postgresql.service ospd-openvas.service
Documentation=man:gvmd(8)
ConditionKernelCommandLine=!recovery

[Service]
Type=exec
User=gvm
Group=gvm
PIDFile=/run/gvmd/gvmd.pid
RuntimeDirectory=gvmd
RuntimeDirectoryMode=2775
ExecStart=/usr/local/sbin/gvmd --foreground --osp-vt-update=/run/ospd/ospd-openvas.sock --listen-group=gvm
Restart=always
TimeoutStopSec=10

[Install]
WantedBy=multi-user.target
EOF

cat << EOF | sudo tee /etc/systemd/system/gsad.service
[Unit]
Description=Greenbone Security Assistant daemon (gsad)
Documentation=man:gsad(8) https://www.greenbone.net
After=network.target gvmd.service
Wants=gvmd.service

[Service]
Type=exec
User=gvm
RuntimeDirectory=gsad
RuntimeDirectoryMode=2775
PIDFile=/run/gsad/gsad.pid
ExecStart=/usr/local/sbin/gsad --foreground --listen=0.0.0.0 --port=9392 --http-only
Restart=always
TimeoutStopSec=10

[Install]
WantedBy=multi-user.target
Alias=greenbone-security-assistant.service
EOF

cat << EOF | sudo tee /etc/systemd/system/openvasd.service
[Unit]
Description=OpenVASD
Documentation=https://github.com/greenbone/openvas-scanner/tree/main/rust/openvasd
ConditionKernelCommandLine=!recovery

[Service]
Type=exec
User=gvm
RuntimeDirectory=openvasd
RuntimeDirectoryMode=2775
ExecStart=/usr/local/bin/openvasd --mode service_notus --products /var/lib/notus/products --advisories /var/lib/notus/advisories --listening 127.0.0.1:3000
SuccessExitStatus=SIGKILL
Restart=always
RestartSec=60

[Install]
WantedBy=multi-user.target
EOF

# Reload systemd and enable/start services
sudo systemctl daemon-reload
sudo /usr/local/bin/greenbone-feed-sync
sudo systemctl enable --now ospd-openvas gvmd gsad openvasd

sudo runuser -u gvm -- greenbone-feed-sync --type GVMD_DATA
sudo runuser -u gvm -- greenbone-feed-sync --type SCAP
sudo runuser -u gvm -- greenbone-feed-sync --type CERT
sudo runuser -u gvm -- greenbone-feed-sync --type NVT
```
