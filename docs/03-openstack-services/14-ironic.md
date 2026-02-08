# OpenStack ironic(베어메탈 서비스) 구축 - 일시 중지

---

---

## **베어 메탈 서비스 설치 및 구성**

Bare Metal용 데이터베이스 설정 

```sql
sudo mysql -uroot -pmnc1!

CREATE DATABASE ironic CHARACTER SET utf8mb3;

GRANT ALL PRIVILEGES ON ironic.* TO 'ironic'@'localhost' IDENTIFIED BY 'mnc1!';
GRANT ALL PRIVILEGES ON ironic.* TO 'ironic'@'%' IDENTIFIED BY 'mnc1!';
```

패키지 설치

```bash
sudo apt-get install ironic-api ironic-conductor python3-ironicclient
```

### **ironic-api 구성**

```bash
sudo vi /etc/ironic/ironic.conf

[DEFAULT]
log_dir = /var/log/ironic
transport_url = rabbit://openstack:mnc_rabbit@192.168.73.171:5672/
rpc_transport = json-rpc
auth_strategy=keystone
default_boot_interface=ipxe
enabled_boot_interfaces=ipxe,pxe

[database]
connection = mysql+pymysql://ironic:mnc1!@192.168.73.171/ironic?charset=utf8

[json_rpc]
auth_type = password
auth_url=http://192.168.73.171:5000/
username=ironic
password=mnc1!
project_name=service
project_domain_id=default
user_domain_id=default

[keystone_authtoken]
www_authenticate_uri = http://mncsvrt06:5000/
auth_url = http://mncsvrt06:5000/
memcached_servers = 192.168.73.171:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = ironic
password = mnc1!
```

DB 초기화

```bash
sudo ironic-dbsync --config-file /etc/ironic/ironic.conf create_schema
```

서비스 재시작

```bash
sudo service ironic-api restart
```

### **Configuring ironic-api behind mod_wsgi**

```bash
sudo vi /etc/apache2/sites-available/ironic.conf

# Licensed under the Apache License, Version 2.0 (the "License"); you may
# not use this file except in compliance with the License. You may obtain
# a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
# WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
# License for the specific language governing permissions and limitations
# under the License.

# This is an example Apache2 configuration file for using the
# Ironic API through mod_wsgi.  This version assumes you are
# running devstack to configure the software, and PBR has generated
# and installed the ironic-api-wsgi script while installing ironic.

Listen 6385

<VirtualHost *:6385>
    WSGIDaemonProcess ironic user=www-data group=www-data threads=10 display-name=%{GROUP}
    WSGIScriptAlias / /usr/bin/ironic-api-wsgi

    SetEnv APACHE_RUN_USER www-data
    SetEnv APACHE_RUN_GROUP www-data
    WSGIProcessGroup ironic

    ErrorLog /var/log/apache2/ironic_error.log
    LogLevel info
    CustomLog /var/log/apache2/ironic_access.log combined

    <Directory />
        WSGIProcessGroup ironic
        WSGIApplicationGroup %{GLOBAL}
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

> NOTE
> > `WSGIDaemonProcess`, `APACHE_RUN_USER`, `APACHE_RUN_GROUP` 지시문을 수정하여 **서버에서 적절한 사용자 및 그룹**으로 설정하세요.
> `WSGIScriptAlias` 지시문을 수정하여 **IRONIC_BIN 디렉토리에 자동으로 생성된 `ironic-api-wsgi` 스크립트**를 가리키도록 설정하세요.
> `Directory` 지시문을 수정하여 **Ironic API 코드가 위치한 경로**로 설정하세요.
> >


ironic service stop

```bash
sudo service ironic-api stop
sudo service ironic-api disable
```

start apache

```bash
sudo a2ensite ironic
sudo service apache2 reload
systemctl reload apache2
```

---

## **Bare Metal 서비스에 대한 Identity 서비스 구성**

```bash
# user 생성
openstack user create --domain default --password-prompt ironic

# 권한 부여
openstack role add --project service --user ironic admin

# 서비스 등록
openstack service create --name ironic --description "Ironic baremetal provisioning service" baremetal
```

 endpoint 생성

```bash
openstack endpoint create --region RegionOne baremetal admin http://mncsvrt06:6385
openstack endpoint create --region RegionOne baremetal public http://mncsvrt06:6385
openstack endpoint create --region RegionOne baremetal internal http://mncsvrt06:6385
```

---

## **베어 메탈 서비스를 사용하도록 컴퓨팅 서비스 구성**

`sudo vi /etc/nova/nova.conf`

```bash
# 기존 nova.conf에 아래 내용 추가

[DEFAULT]
# baremetal
compute_driver=ironic.IronicDriver

[filter_scheduler]
# baremetal
track_instance_changes=False
shuffle_best_same_weighed_hosts = true

[scheduler]
# baremetal
discover_hosts_in_cells_interval=120

[compute]
# baremetal
consecutive_build_service_disable_threshold = 0

[ironic]
auth_type = password
auth_url = http://mncsvrt06:5000/v3
project_name = service
username = ironic
password = mnc1!
project_domain_name = Default
user_domain_name = Default
```

서비스 재 시작

```bash
systemctl restart nova-compute.service 
systemctl restart nova-scheduler.service
```

---

## **베어 메탈 프로비저닝을 위한 네트워킹 서비스 구성**

> NOTE
> 베어메탈 네트워크 서비스 구성을 위해 기존 Neutron 네트워크와는 별도로 추가 네트워크 구성이 필요합니다. 현재 해당 추가 네트워크는 구성되어 있지 않으며, 추후 추가 네트워크를 할당한 후 관련 네트워크 설정 작업을 진행할 예정입니다.


---
