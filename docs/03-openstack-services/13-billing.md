# OpenStack Billing 기능 구축(Ceilometer, Cloudkitty)

---

---

# Ceilometer

```bash
. admin-openrc

openstack user create --domain default --password-prompt ceilometer
# User Password:
# Repeat User Password:

# admin 역할 부여
openstack role add --project service --user ceilometer admin

# 서비스 등록
openstack service create --name ceilometer --description "Telemetry" metering
```

Gnocchi (User / Service / Endpoint)

```bash
# 사용자 생성
openstack user create --domain default --password-prompt gnocchi
# 서비스 등록 
openstack service create --name gnocchi --description "Metric Service" metric
# admin 역할 부여
openstack role add --project service --user gnocchi admin

# 엔드포인트
openstack endpoint create --region RegionOne metric public http://controller:8041
openstack endpoint create --region RegionOne metric internal http://controller:8041
openstack endpoint create --region RegionOne metric admin http://controller:8041
```

## Gnocchi 설치·구성

패키지 설치·DB 생성

```bash
sudo apt-get install gnocchi-api gnocchi-metricd python3-gnocchiclient uwsgi uwsgi-plugin-python3 python3-redis

# DB생성
mysql -u root -p

CREATE DATABASE gnocchi;

GRANT ALL PRIVILEGES ON gnocchi.* TO 'gnocchi'@'localhost' IDENTIFIED BY 'mnc1!';
GRANT ALL PRIVILEGES ON gnocchi.* TO 'gnocchi'@'%' IDENTIFIED BY 'mnc1!';
```

`sudo vi /etc/gnocchi/gnocchi.conf`

```bash
[api]
auth_mode = keystone
port = 8041
uwsgi_mode = http-socket
archive_policy_rule = cloudkitty_rule

[keystone_authtoken]
auth_type = password
auth_url = http://mncsvrt06:5000/v3
project_domain_name = Default
user_domain_name = Default
project_name = service
username = gnocchi
password = mnc1!
interface = internalURL
region_name = RegionOne
www_authenticate_uri = http://mncsvrt06:5000

[indexer]
url = mysql+pymysql://gnocchi:mnc1!@mncsvrt06/gnocchi

[storage]
coordination_url = redis://192.168.73.171:6379
file_basepath = /var/lib/gnocchi
driver = file
```

Redis 설정

`sudo vi /etc/redis/redis.conf`

```bash
# 기존:
bind 127.0.0.1

# 변경:
bind 127.0.0.1 192.168.73.167
```

Redis 재시작

```bash
sudo systemctl restart redis-server
```

폴더 권한 설정

```bash
chown gnocchi:gnocchi gnocchi/*
```

 DB 업그레이드·서비스 재시작

```bash
sudo gnocchi-upgrade
sudo systemctl restart gnocchi-metricd.service
```

## Ceilometer 설치·구성

```bash
sudo apt-get install ceilometer-agent-notification ceilometer-agent-central
```

`sudo vi /etc/ceilometer/pipeline.yaml`

```bash
---
sources:
  - name: meter_source
    meters:
      - "*"
    sinks:
      - meter_sink
    interval: 600

sinks:
  - name: meter_sink
    publishers:
      - gnocchi://?archive_policy=high
```

`sudo vi /etc/ceilometer/ceilometer.conf`

```bash
[DEFAULT]
transport_url = rabbit://openstack:mnc_rabbit@mncsvrt06
pipeline_cfg_file = pipeline.yaml
dispatcher = gnocchi
rpc_conn_pool_size = 100

[service_credentials]
auth_type = password
auth_url = http://mncsvrt06:5000/v3
project_domain_id = default
user_domain_id = default
project_name = service
username = ceilometer
password = mnc1!
interface = internalURL
region_name = RegionOne
```

```bash
sudo ceilometer-upgrade
sudo service ceilometer-agent-central restart
sudo service ceilometer-agent-notification restart
```

---

## 데이터 수집 설정

### Cinder

`sudo vi /etc/cinder/cinder.conf` 

```bash
[oslo_messaging_notifications]
driver = messagingv2
transport_url = rabbit://openstack:mnc_rabbit@mncsvrt06
topics = notifications
```

cron 등록

```bash
sudo crontab -e

*/5 * * * * /path/to/cinder-volume-usage-audit --send_actions
```

```bash
sudo systemctl restart cinder-volume.service cinder-backup.service cinder-scheduler.service
```

### Glane

`sudo vi /etc/glance/glance-api.conf`

```bash
[DEFAULT]
enabled_backends=fs:file
transport_url = rabbit://openstack:mnc_rabbit@mncsvrt06
log_dir = /var/log/glance

[oslo_messaging_notifications]
driver = messagingv2
transport_url = rabbit://openstack:mnc_rabbit@mncsvrt06
topics = notifications

sudo service glance-api restart
```

### Neutron

`sudo vi /etc/neutron/neutron.conf`

```bash
[oslo_messaging_notifications]
driver = messagingv2
transport_url = rabbit://openstack:mnc_rabbit@mncsvrt06
topics = notifications

sudo service neutron-server restart
```

### Heat

`sudo vi /etc/heat/heat.conf`

```bash
[oslo_messaging_notifications]
driver = messagingv2
transport_url = rabbit://openstack:mnc_rabbit@mncsvrt06
topics = notifications

sudo systemctl restart heat-api.service heat-api-cfn.service heat-engine.service
```

### Swift

User 생성 및 권한 부여

```bash
openstack role create ResellerAdmin

openstack role add --project service --user ceilometer ResellerAdmin
```

패키지 설치

```bash
apt-get install python3-ceilometermiddleware
```

`sudo vi /etc/swift/proxy-server.conf`

```bash
[filter:keystoneauth]
use = egg:swift#keystoneauth
operator_roles = admin,user,ResellerAdmin

[pipeline:main]
pipeline = catch_errors gatekeeper healthcheck proxy-logging cache container_sync bulk ratelimit authtoken keystoneauth container-quotas account-quotas slo dlo versioned_writes proxy-logging ceilometer proxy-server

[filter:ceilometer]
paste.filter_factory = ceilometermiddleware.swift:filter_factory
control_exchange = swift
url = rabbit://openstack:mnc_rabbit@mncsvrt06:5672/
driver = messagingv2
topic = notifications
log_level = WARN
```

서비스 재시작 

`sudo service swift-proxy restart`

---

## Compute-Node (Nova) 추가 설정

패키지 설치

```bash
sudo apt-get install ceilometer-agent-compute ceilometer-agent-ipmi
```

`sudo vi /etc/ceilometer/ceilometer.conf`

```bash
# 아래 설정 복사 붙여넣기
[DEFAULT]
transport_url = rabbit://openstack:mnc_rabbit@mncsvrt06
pipeline_cfg_file = pipeline.yaml 
dispatcher = gnocchi
rpc_conn_pool_size = 100

[service_credentials]
auth_type = password
auth_url = http://mncsvrt06:5000/v3
project_domain_id = default
user_domain_id = default
project_name = service
username = ceilometer
password = mnc1!
interface = internalURL
region_name = RegionOne
```

`sudo vi **/**etc/nova/nova.conf`

```bash
[DEFAULT]
log_dir = /var/log/nova
lock_path = /var/lock/nova 
state_path = /var/lib/nova
transport_url = rabbit://openstack:mnc_rabbit@192.168.73.171:5672/
my_ip = 192.168.73.171
instance_usage_audit = True
instance_usage_audit_period = hour
notify_on_state_change = vm_and_task_state
notification_driver = messagingv2

[notifications]
notify_on_state_change = vm_and_task_state
notification_format = unversioned

[oslo_messaging_notifications]
driver = messagingv2
topics = notifications
transport_url = rabbit://openstack:mnc_rabbit@mncsvrt06
```

`sudo visudo`

```bash
ceilometer ALL = (root) NOPASSWD: /usr/bin/ceilometer-rootwrap /etc/ceilometer/rootwrap.conf *
```

`sudo vi /etc/ceilometer/polling.yaml` 

```bash
---
sources:
  - name: compute_polling
    interval: 300
    meters:
      - cpu
      - cpu_util
      - cpu_l3_cache
      - memory.usage
      - network.incoming.bytes
      - network.incoming.packets
      - network.outgoing.bytes
      - network.outgoing.packets
      - disk.device.read.bytes
      - disk.device.read.requests
      - disk.device.write.bytes
      - disk.device.write.requests
      - volume.size
      - volume.snapshot.size
      - volume.backup.size
      - instance
  - name: central_polling
    interval: 300
    meters:
      - disk.root.size
      - disk.ephemeral.size
      - vcpus
      - memory

  - name: ipmi
    interval: 300
    meters:
      - hardware.ipmi.temperature
```

```bash
sudo systemctl restart ceilometer-agent-compute.service ceilometer-agent-ipmi.service nova-compute.service
sudo systemctl enable --now ceilometer-polling.service
```

---

# CloudKitty 설치·구성

소스 설치 및 기본 파일 배포를 OpenStack에서 사용 중인 버전(`2024.1`)과 같은 버전 사용

```bash
git clone -b stable/2024.1 https://opendev.org/openstack/cloudkitty.git
cd cloudkitty
sudo python3 setup.py install
```

샘플 구성 파일 설치

```bash
mkdir /etc/cloudkitty

# tox 설치
sudo apt install tox

# cloudkitty 에서 실행
tox -e genconfig

sudo cp <cloudkitty clone 위치>/etc/cloudkitty/cloudkitty.conf.sample /etc/cloudkitty/cloudkitty.conf
sudo cp <cloudkitty clone 위치>/etc/cloudkitty/api_paste.ini /etc/cloudkitty
```

로그 디렉토리 생성

```bash
mkdir /var/log/cloudkitty/
```

클라이언트 설치

```bash
git clone -b stable/2024.1 https://opendev.org/openstack/python-cloudkittyclient.git
cd python-cloudkittyclient
sudo python3 setup.py install
```

대시보드 설치

```bash
git clone -b stable/2024.1 https://opendev.org/openstack/cloudkitty-dashboard.git
cd cloudkitty-dashboard
sudo python3 setup.py install
# cloudkitty dashborad 파일 horizon에 적용
ln -sf /usr/local/lib/python3.10/dist-packages/cloudkittydashboard/enabled/_[0-9]*.py /var/www/horizon/openstack_dashboard/enabled/
```

대시보드 적용

```bash
./manage.py collectstatic --noinput
./manage.py compress --force
systemctl restart apache2.service
```

---

create Databases

```bash
CREATE DATABASE cloudkitty;

GRANT ALL PRIVILEGES ON cloudkitty.* TO 'cloudkitty'@'localhost' IDENTIFIED BY 'mnc1!';

GRANT ALL PRIVILEGES ON cloudkitty.* TO 'cloudkitty'@'%' IDENTIFIED BY 'mnc1!';
```

Keystone 등록

```bash
# 사용자 생성
openstack user create --domain default --password-prompt cloudkitty
# admin 역할 부여 
openstack role add --project service --user cloudkitty admin
# 서비스 등록
openstack service create rating --name cloudkitty --description "OpenStack Rating Service"

# 엔드포인트 등록
openstack endpoint create rating --region RegionOne public http://mncsvrt06:8889
openstack endpoint create rating --region RegionOne admin http://mncsvrt06:8889
openstack endpoint create rating --region RegionOne internal http://mncsvrt06:8889
```

cloudkitty 시스템 계정 생성

```bash
sudo groupadd --system cloudkitty
sudo useradd --system --home-dir /var/lib/cloudkitty --shell /usr/sbin/nologin --gid cloudkitty cloudkitty
sudo chown -R cloudkitty:cloudkitty /etc/cloudkitty
```

---

**InfluxDB 설치**

```bash
sudo apt install influxdb influxdb-client
```

create InfluxDB

```bash
# DB 실행
influx

# 데이터베이스 생성
CREATE DATABASE cloudkitty;
SHOW DATABASES;
```

---

**`sudo vi /etc/cloudkitty/cloudkitty.conf`**

```bash
DEFAULT]
verbose = true
debug = false
log_dir = /var/log/cloudkitty
transport_url = rabbit://openstack:mnc_rabbit@mncsvrt06
auth_strategy = keystone
workers = 2

[database]
connection = mysql+pymysql://cloudkitty:mnc1!@mncsvrt06/cloudkitty

[keystone_authtoken]
www_authenticate_uri = http://192.168.73.171:5000/v3
auth_url = http://192.168.73.171:5000/v3
memcached_servers = 192.168.73.171:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = cloudkitty
password = mnc1!
service_token_roles_required = true
service_token_roles = admin

[collect]
collector = gnocchi
metrics_conf = /etc/cloudkitty/metrics.yml

[collector_gnocchi]
auth_section = keystone_authtoken
region_name = RegionOne
collect_period = 3600

[fetcher]
backend = gnocchi

[fetcher_gnocchi]
region_name = RegionOne
auth_url = http://192.168.73.171:5000/v3
memcached_servers = 192.168.73.171:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = cloudkitty
password = mnc1!
interface = internal
timeout = 30

[fetcher_keystone]
keystone_version = 3
auth_section = keystone_authtoken
region_name = RegionOne

[oslo_messaging_notifications]
driver = messagingv2
transport_url = rabbit://openstack:mnc_rabbit@mncsvrt06

[storage]
version = 2
backend = influxdb

[orchestrator]
coordination_url = mysql://cloudkitty:mnc1!@192.168.73.171/cloudkitty
rating_buffer = 3600
period = 3600
scope_key = project_id
# ↓ 추가 / 수정
workers           = 8            # 전체 green-thread 수 (CPU 수보다 작게)
workers_per_scope = 1            # 테넌트당 최대 1개 워커

[storage_influxdb]
username = cloudkitty
password = cloudkitty
database = cloudkitty
host = localhost

[rating]
backend = hashmap
```

Systemd 파일 등록

`sudo vi /etc/systemd/system/cloudkitty-api.service`

```bash
[Unit]
Description=CloudKitty API Service
After=network.target

[Service]
User=cloudkitty
Group=cloudkitty
Environment="CLOUDKITTY_CONFIG_FILE=/etc/cloudkitty/cloudkitty.conf"
ExecStart=/usr/bin/python3 /usr/local/bin/cloudkitty-api --host 0.0.0.0 --port 8889
Restart=on-failure
WorkingDirectory=/etc/cloudkitty
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```

`sudo vi /etc/systemd/system/cloudkitty-processor.service` 

```bash
[Unit]
Description=CloudKitty Processor Service
After=network.target

[Service]
ExecStart=/usr/local/bin/cloudkitty-processor
User=cloudkitty
Group=cloudkitty
Restart=on-failure
WorkingDirectory=/etc/cloudkitty
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```

폴더 권한 설정

```bash
sudo chmod 755 /etc/cloudkitty
sudo chown cloudkitty:cloudkitty cloudkitty/

sudo chmod 750 /var/log/cloudkitty
suychmod 644 api_paste.ini cloudkitty.conf 
```

추가 패키지 설치

```bash
sudo pip3 install datetimerange
```

```bash
# DB 업데이트
sudo cloudkitty-storage-init
sudo cloudkitty-dbsync upgrade

# 서비스 등록 재시작
sudo systemctl restart cloudkitty-api.service cloudkitty-processor.service
```

---

## 수집기 구성

`sudo vi /etc/cloudkitty/metrics.yml`

```yaml
metrics:
  vcpus:
    unit: instance
    groupby:
      - id
      - user_id
      - project_id
    metadata:
      - flavor_name
      - flavor_id
      - vcpus
    extra_args:
      aggregation_method: mean
      resource_type: instance
      force_granularity: 3600

  volume.size:
    unit: GiB
    groupby:
      - id
      - user_id
      - project_id
    metadata:
      - volume_type
    extra_args:
      aggregation_method: mean
      resource_type: volume
      force_granularity: 3600
```

| **항목** | **설명**  |
| --- | --- |
| metric_name | 과금 대상 메트릭 이름 (cpu, memory.usage, volume.size 등)
→ Gnocchi에서 정의된 메트릭 이름과 일치해야 함 |
| unit | 과금 시 사용할 단위 (core, MB, GiB 등)
→ 실제 요금 계산 시 적용되는 단위 |
| groupby | 과금 데이터를 구분할 기준
→ 예: `project_id`, `user_id`, `flavor_name` 등→ 하나 이상 설정 가능 |
| metadata | 메타데이터 기반 추가 요금 조건
→ 예: flavor의 `vcpus`, `memory`, 볼륨의 `volume_type` 등 |
| mutate | 원본 메트릭 값을 어떻게 변환할지 설정
→ 과금 방식 결정
`NONE`: 실제 사용량 기준 → 예: 5GiB 사용 → 5GiB 과금
`NUMBOOL`: 사용 여부 기준→ 값이 존재하면 무조건 1로 계산 (정액제) |
| aggregation_method | Gnocchi에서 사용할 집계 방식
→ 예: `mean`, `sum`, `max`, `last`, `count` 등 |
| resource_type | Gnocchi의 리소스 타입 지정
→ 예: `instance`, `volume`, `image`, `network` 등 |

---

## cloudkitty hashmap 등록

### **flavor 과금 설정**

```bash
# 서비스 등록(hashmap 이름이 cpu인 이유는 cpu가 flavor_name을 수집하기 때문임)
cloudkitty hashmap service create cpu

# 필드 등록 (예: flavor_name 기준)
cloudkitty hashmap field create <service id> flavor_name

# 매핑 등록
# 등록 기준
# GPU: T4 800원
# CPU(코어당): 40원
# RAM: 32GB 20원 64GB 40원, 128GB 80원
cloudkitty hashmap mapping create 50 --field-id <field id> --type flat --value 'm1.small'
cloudkitty hashmap mapping create 980 --field-id <field id> --type flat --value 'm1.larg'
```

### volume 과금 설정

```bash
cloudkitty hashmap service create volume.size

cloudkitty hashmap mapping create 0.006 --service-id <service id> --type flat
```

| **구분** | **설명** |
| --- | --- |
| --type rate | 사용량 비례 요금 (예: GiB당 단가 × 사용량) |
| --type flat | 정액 요금 (특정 값에 해당하면 고정 금액) |
| --value '*' | 모든 값에 동일한 요율 적용 |
| field-id | 리소스의 메타데이터 기준 필드 (예: vcpus, memory) |
| mapping create | 실제 요금과 매핑하는 단계 |

---

# Horizon Rating UI 수정

> NOTE
> 기존 Rating UI는 CPU, Instance, Root Disk, Memory 등 여러 항목을 포함하고 있었으나,
> Flavor에 CPU, RAM, Root Disk, GPU 정보가 포함되기 때문에 Flavor와 Volume 을 제외하고 나머지 항목을 UI에서 제거함.


## 프로젝트 → Rating UI

경로 이동 및 백업

```bash
cd /usr/local/lib/python3.10/dist-packages/cloudkittydashboard/dashboards/project/rating/
cp views.py views.py.back
```

`views.py` 수정

```bash
import json
from datetime import datetime
from dateutil.relativedelta import relativedelta

from django.conf import settings
from django import http
from django.utils.translation import gettext_lazy as _
from horizon import exceptions
from horizon import tables

from cloudkittydashboard.api import cloudkitty as api
from cloudkittydashboard.dashboards.project.rating import tables as rating_tables

# ────────────────────────────────────────────────────
# 공통 rename + 숨기기 필터링 함수
RENAME_TYPE = {
    "vcpus": "instance (cpu, memory, root_disk, gpu)",
    "ALL": _("Project Total"),
}
HIDE_TYPE = {
    "cpu",
    "disk.ephemeral.size",
    "disk.root.size",
    "instance",
    "memory.usage",
}

def _filter_summary(data):
    """필터링(숨기/이름변경) & 콤바르 통일
    """
    filtered = []
    for row in data:
        if 'type' not in row and 'res_type' in row:
            row['type'] = row.pop('res_type')
        t = row.get("type")
        if t in HIDE_TYPE:
            continue
        if t in RENAME_TYPE:
            row["type"] = RENAME_TYPE[t]
        filtered.append(row)
    return filtered
# ────────────────────────────────────────────────────

class IndexView(tables.DataTableView):
    table_class  = rating_tables.SummaryTable
    template_name = 'project/rating/index.html'

    def get_data(self):
        project_id = getattr(self.request.user, 'project_id', self.request.user.tenant_id)

        # 이번 달 1일 00시 ~ 현재 시간
        begin = datetime.utcnow().replace(day=1, hour=0, minute=0, second=0, microsecond=0)
        end = datetime.utcnow().replace(hour=23, minute=59, second=59, microsecond=999999)

        client = api.cloudkittyclient(self.request, version='1')
        summary = client.report.get_summary(
            begin=begin.isoformat(),
            end=end.isoformat(),
            tenant_id=project_id,
            groupby=['res_type']
        ).get('summary', [])

        # res_type → type 으로 통일
        for row in summary:
            if 'type' not in row and 'res_type' in row:
                row['type'] = row.pop('res_type')

        # 필터링 (숨김 리소스 제거 및 이름 변경)
        filtered_summary = _filter_summary(summary)

        # TOTAL 항은 필터링된 값만 기준으로 계산
        total_rate = sum(float(item['rate']) for item in filtered_summary)

        filtered_summary.append({
            'type': 'Project Total',
            'rate': total_rate,
        })

        return filtered_summary

def quote(request):
    pricing = 0.0
    if request.headers.get('x-requested-with') == 'XMLHttpRequest' and request.method == 'POST':
        json_data = json.loads(request.body)

        def __update_quotation_data(element, service):
            if isinstance(element, dict):
                element['service'] = service
            else:
                for elem in element:
                    __update_quotation_data(elem, service)

        try:
            service = getattr(settings, 'CLOUDKITTY_QUOTATION_SERVICE', 'instance')
            __update_quotation_data(json_data, service)
            pricing = float(
                api.cloudkittyclient(request).rating.get_quotation(res_data=json_data)
            )
        except Exception:
            exceptions.handle(request, _('Unable to retrieve price.'))

    return http.HttpResponse(json.dumps(pricing),
                             content_type='application/json')
```

---

## 프로젝트 → Reporting UI

경로 이동 및 백업

```bash
cd /usr/local/lib/python3.10/dist-packages/cloudkittydashboard/dashboards/project/reporting/templates/reporting/
cp this_month.html this_month.html.back
```

`this_month.html` 수정

```html
{% load i18n %}
{% load l10n %}
{% load static %}

<div class="container-fluid">
  <div class="row">
    <div class="col-md-3">
      <h4>{% trans "Legend" %}</h4>
      <ul>
        <li style="color:rgb(253, 141, 60)">instance&nbsp;(cpu, memory, root_disk, gpu)</li>
        <li style="color:rgb(49, 130, 189)">volume.size</li>
      </ul>
    </div>

    <div class="col-md-4">
      <h4>{% trans "Cumulative Cost Repartition" %}</h4>
      <canvas id="costPieChart" width="300" height="300"></canvas>
    </div>
  </div>

  <div class="row mt-5">
    <div class="col-md-6">
      <h4>Instance Cost Per Hour</h4>
      <canvas id="instanceChart" height="200"></canvas>
    </div>
    <div class="col-md-6">
      <h4>Volume Cost Per Hour</h4>
      <canvas id="volumeChart" height="200"></canvas>
    </div>
  </div>
</div>

<!-- Chart.js CDN -->
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>

<script>
  // ---------- 데이터 배열 ----------
  const pieLabels = [];
  const pieData   = [];
  const pieColors = [];

  const instanceLabels = [];
  const instanceData   = [];

  const volumeLabels   = [];
  const volumeData     = [];

  {% for service, data in repartition_data.items %}
    {% if service == 'vcpus' %}
      /* ---------- Pie ---------- */
      pieLabels.push('instance (cpu, memory, root_disk, gpu)');
      pieData.push({{ data.cumulated|unlocalize }});
      pieColors.push('rgb(253, 141, 60)');

      /* ---------- Bar (Instance) ---------- */
      {% for timestamp, rating in data.hourly.items %}
        {
          const date = new Date({{ timestamp }} * 1000);
          date.setHours(date.getHours() + 9); // UTC → KST
          const label = date.toLocaleString('ko-KR', {
            year: 'numeric',
            month: '2-digit',
            day:   '2-digit',
            hour:  '2-digit',
            minute:'2-digit',
            hour12:false
          }).replace(/\. /g, '-').replace(/\./g, '');
          instanceLabels.push(label);
          instanceData.push({{ rating|unlocalize }});
        }
      {% endfor %}
    {% elif service == 'volume.size' %}
      /* ---------- Pie ---------- */
      pieLabels.push('volume.size');
      pieData.push({{ data.cumulated|unlocalize }});
      pieColors.push('rgb(49, 130, 189)');

      /* ---------- Bar (Volume) ---------- */
      {% for timestamp, rating in data.hourly.items %}
        {
          const date = new Date({{ timestamp }} * 1000);
          date.setHours(date.getHours() + 9); // UTC → KST
          const label = date.toLocaleString('ko-KR', {
            year: 'numeric',
            month: '2-digit',
            day:   '2-digit',
            hour:  '2-digit',
            minute:'2-digit',
            hour12:false
          }).replace(/\. /g, '-').replace(/\./g, '');
          volumeLabels.push(label);
          volumeData.push({{ rating|unlocalize }});
        }
      {% endfor %}
    {% endif %}
  {% endfor %}

  /* ✂️ NEW ───────────────────────────────────────────────
     최근 5개(또는 배열 길이가 5 미만이면 전체)만 남기기
  ------------------------------------------------------- */
  function keepLastFive(labelsArr, dataArr) {
    if (labelsArr.length > 5) {
      const start = labelsArr.length - 5;
      labelsArr.splice(0, start);
      dataArr.splice(0, start);
    }
  }
  keepLastFive(instanceLabels, instanceData);
  keepLastFive(volumeLabels,   volumeData);
  /* ────────────────────────────────────────────────────── */

  // ---------- Pie Chart ----------
  new Chart(document.getElementById('costPieChart'), {
    type: 'doughnut',
    data: {
      labels: pieLabels,
      datasets: [{
        data: pieData,
        backgroundColor: pieColors
      }]
    },
    options: {
      responsive: true,
      plugins: { legend: { position: 'bottom' } }
    }
  });

  // ---------- Instance Bar Chart ----------
  new Chart(document.getElementById('instanceChart'), {
    type: 'bar',
    data: {
      labels: instanceLabels,
      datasets: [{
        label: 'Instance Cost (per hour)',
        data: instanceData,
        backgroundColor: 'rgba(253, 141, 60, 0.7)',
        borderColor:     'rgb(253, 141, 60)',
        borderWidth: 1
      }]
    },
    options: {
      scales: {
        y: {
          beginAtZero: true,
          title: { display: true, text: '₩ (KRW)' }
        }
      }
    }
  });

  // ---------- Volume Bar Chart ----------
  new Chart(document.getElementById('volumeChart'), {
    type: 'bar',
    data: {
      labels: volumeLabels,
      datasets: [{
        label: 'Volume Cost (per hour)',
        data:  volumeData,
        backgroundColor: 'rgba(49, 130, 189, 0.7)',
        borderColor:     'rgb(49, 130, 189)',
        borderWidth: 1
      }]
    },
    options: {
      scales: {
        y: {
          beginAtZero: true,
          title: { display: true, text: '₩ (KRW)' }
        }
      }
    }
  });
</script>
```

---

## 관리 → Rating → Rating Summary UI

경로 이동 및 백업

```bash
cd /usr/local/lib/python3.10/dist-packages/cloudkittydashboard/dashboards/admin/summary/
cp views.py views.py.back
```

`views.py` 수정

```bash
# Copyright 2018 Objectif Libre
#
#    Licensed under the Apache License, Version 2.0 (the "License"); you may
#    not use this file except in compliance with the License. You may obtain
#    a copy of the License at
#
#         http://www.apache.org/licenses/LICENSE-2.0
#
#    Unless required by applicable law or agreed to in writing, software
#    distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
#    WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
#    License for the specific language governing permissions and limitations
#    under the License.

from django.utils.translation import gettext_lazy as _
from horizon import tables

from openstack_dashboard.api import keystone as api_keystone

from cloudkittydashboard.api import cloudkitty as api
from cloudkittydashboard.dashboards.admin.summary import tables as sum_tables
from datetime import datetime 
# 맨 위에 추가
RENAME_TYPE = {
    "vcpus": "instance (cpu, memory, root_disk, gpu)",
}
HIDE_TYPE = {
    "cpu",
    "disk.ephemeral.size",
    "disk.root.size",
    "instance",
    "memory.usage",
}

def _filter_summary(summary):
    result = []
    for row in summary:
        res = row.get("res_type")
        if res in HIDE_TYPE:
            continue
        if res in RENAME_TYPE:
            row["res_type"] = RENAME_TYPE[res]
        result.append(row)
    return result

# dashboards/admin/summary/views.py  (혹은 동일 모듈 안의 위치)

class IndexView(tables.DataTableView):
    template_name = 'admin/rating_summary/index.html'
    table_class = sum_tables.SummaryTable

    def get_data(self):
        # ―― 월 기준 기간 설정 ―――――――――――――――――――――――――
        begin = datetime.utcnow().replace(day=1, hour=0, minute=0, second=0,
                                          microsecond=0)
        end   = datetime.utcnow().replace(hour=23, minute=59, second=59,
                                          microsecond=999999)

        client = api.cloudkittyclient(self.request, version='1')

        # ── 1) 테넌트·리소스별 요약을 한꺼번에 가져온다 ────────────────
        raw = client.report.get_summary(
            begin=begin.isoformat(),
            end=end.isoformat(),
            groupby=['tenant_id', 'res_type'],
            all_tenants=True,
        ).get('summary', [])
        # ── 2) 프로젝트-별로 rate 합산(숨길 리소스 제거 + 이름 변경) ───────
        tenant_rates = {}
        for row in raw:
            res_type = row['res_type']
            if res_type in HIDE_TYPE:          # cpu, memory.usage … 등 제거
                continue
            if res_type in RENAME_TYPE:        # vcpus → instance(…)
                res_type = RENAME_TYPE[res_type]

            tenant_id = row['tenant_id']
            tenant_rates.setdefault(tenant_id, 0.0)
            tenant_rates[tenant_id] += float(row['rate'])

        # ── 3) 화면에 표시할 summary 리스트 생성 ───────────────────────
        summary = [{'tenant_id': tid, 'rate': r} for tid, r in tenant_rates.items()]
        summary.append({                       # Cloud Total 행
            'tenant_id': 'ALL',
            'rate': sum(tenant_rates.values()),
        })

        # ── 4) 테넌트 이름 매핑 ────────────────────────────────────────
        tenants, _ = api_keystone.tenant_list(self.request)
        name_map = {t.id: t.name for t in tenants}
        for row in summary:
            if row['tenant_id'] == 'ALL':
                row['name'] = 'Cloud Total'
            else:
                row['name'] = name_map.get(row['tenant_id'], '-')

        # 테이블에 id·name 속성을 기대하므로 identify 처리는 유지
        summary = api.identify(summary, key='tenant_id')
        return summary

# 기존 TenantDetailsView 수정
class TenantDetailsView(tables.DataTableView):
    template_name = 'admin/rating_summary/details.html'
    table_class   = sum_tables.TenantSummaryTable
    page_title    = _("Script Details : {{ table.project_id }}")

    def _get_tenant_summary(self, tenant_id):
        # 월 기준 기간 설정 ---------------------------
        begin = datetime.utcnow().replace(day=1, hour=0, minute=0, second=0,
                                          microsecond=0)
        end   = datetime.utcnow().replace(hour=23, minute=59, second=59,
                                          microsecond=999999)

        client = api.cloudkittyclient(self.request, version='1')

        if tenant_id == 'ALL':
            return client.report.get_summary(
                begin=begin.isoformat(),
                end=end.isoformat(),
                groupby=['res_type'],
                all_tenants=True)['summary']

        return client.report.get_summary(
            begin=begin.isoformat(),
            end=end.isoformat(),
            groupby=['res_type'],
            tenant_id=tenant_id)['summary']

    def get_data(self):
        tenant_id = self.kwargs['project_id']

        # ── 1) 원본 메트릭 조회 ───────────────────────────────
        raw = self._get_tenant_summary(tenant_id)

        # ── 2) 숨김/이름변경 필터 적용 ───────────────────────
        filtered = _filter_summary(raw)          # HIDE_TYPE · RENAME_TYPE 적용

        # ── 3) 필터링된 값 기준으로 TOTAL 재계산 ─────────────
        total_rate = sum(float(item['rate']) for item in filtered)
        filtered.append({
            'tenant_id': tenant_id,
            'res_type': 'TOTAL',
            'rate': total_rate,
        })

        # ── 4) identify → 테이블 row 객체화 ─────────────────
        return api.identify(filtered, key='res_type', name=True)
```

대시보드 설정 적용

```bash
systemctl restart apache2.service
```

---

# cloudkitty Error 대응

- cloudkitty 데이터 수집 시 wsgi 과부화 오류 해결
    
    ```bash
    sudo vi /etc/apache2/mods-enabled/mpm_event.conf
    
    <IfModule mpm_event_module>
            StartServers              4
            MinSpareThreads          50
            MaxSpareThreads         150
            ThreadLimit              64
            ThreadsPerChild          32
            MaxRequestWorkers      1024
            ServerLimit              32
            MaxConnectionsPerChild    0
    </IfModule>
    
    ---------------------------------------------------
    
    sudo vi /etc/apache2/sites-enabled/keystone.conf
    
    WSGIDaemonProcess keystone-public processes=5 threads=15 listen-backlog=1024 queue-timeout=45 inactivity-timeout=900  user=keystone group=keystone display-name=%{GROUP}
    
    sudo vi /etc/apache2/sites-enabled/gnocchi-api.conf
    
    WSGIDaemonProcess gnocchi-api processes=8 threads=4 listen-backlog=2048 request-timeout=180 inactivity-timeout=600 maximum-requests=5000 user=gnocchi group=gnocchi display-name=%{GROUP}
    
    # 시스템 재시작
    sudo systemctl restart apache2 gnocchi-metricd.service ceilometer-agent-notification ceilometer-polling cloudkitty-processor
    ```
    
- CloudKitty에서 Gnocchi metric을 수집하는 과정에서 아래와 같은 오류가 반복적으로 발생
    
    ## 원인
    
    - CloudKitty는 기본적으로 `3600초 (1시간)` 단위의 집계를 요구함.
    - 하지만 Gnocchi에 등록된 메트릭이 `low`, `ceilometer-low` 등의 archive policy를 사용할 경우, 해당 granularity(3600.0)를 지원하지 않음.
    - 그 결과 `granularities are missing` 에러 발생.
    
    ```bash
    gnocchiclient.exceptions.BadRequest: {'cause': "Metrics can't being aggregated", 'reason': 'Granularities are missing', 
    'detail': [['volume.size', 'mean', 3600.0]]} (HTTP 400)
    ```
    
    해결 방법 
    1. 모든 리소스 타입에서 `low` 계열 정책을 사용하는 메트릭을 찾아 `high`로 재생성합니다.
    
    ```bash
    #!/bin/bash
    ###############################################################################
    #   gnocchi_change_metrics_to_high.sh
    #   모든 리소스-타입의 메트릭 중 archive-policy 가 low* 계열이면 high 로 재생성
    ###############################################################################
    DRY_RUN=false        # true  ➜ 삭제/생성 안 하고 echo 만
                        # false ➜ 실제로 delete → create 수행
    
    # ── ① high 정책 이름 (필요하면 다른 정책으로 변경)
    TARGET_POLICY="high"
    
    # ── ② 전체 리소스-타입 배열 (2025-06 기준)
    RESOURCE_TYPES=(
      ceph_account generic host host_disk host_network_interface identity image
      instance instance_disk instance_network_interface ipmi ipmi_sensor
      manila_share network nova_compute port stack swift_account
      switch switch_port switch_table
      volume volume_provider volume_provider_pool
    )
    
    ###############################################################################
    echo "=== Starting: switch all *low* metrics to '$TARGET_POLICY' ==="
    for RTYPE in "${RESOURCE_TYPES[@]}"; do
      echo -e "\n🔎  Processing resource-type: \e[33m$RTYPE\e[0m"
      # 리소스 ID 목록
      gnocchi resource list -t "$RTYPE" -f value -c id 2>/dev/null | while read RID; do
        [[ -z "$RID" ]] && continue
        echo "  ▸ Resource: $RID"
        # 메트릭 키:ID 목록
        gnocchi resource show "$RID" -f json 2>/dev/null |
          jq -r '.metrics | to_entries[] | "\(.key):\(.value)"' |
        while IFS=: read MNAME MID; do
          MID=$(echo "$MID" | tr -d ' ')
          [[ -z "$MID" ]] && continue
    
          # 현재 정책
          POLICY=$(gnocchi metric show "$MID" -f json 2>/dev/null |
                   jq -r '."archive_policy/name"')
    
          echo "    • $MNAME ($MID) → policy=$POLICY"
          if [[ "$POLICY" =~ ^low ]]; then          # low, low-rate, ceilometer-low 등
            if $DRY_RUN; then
              echo "      ↳ [DRY-RUN] would delete & recreate with '$TARGET_POLICY'"
            else
              echo "      ↳ deleting & recreating with '$TARGET_POLICY' ..."
              gnocchi metric delete "$MID"
              gnocchi metric create \
                    --resource-id "$RID" \
                    --archive-policy-name "$TARGET_POLICY" \
                    "$MNAME"
            fi
          fi
        done
      done
    done
    echo -e "\n=== Finished.  DRY_RUN=$DRY_RUN ==="
    ```
    
    1. granularity를 3600초로 명확히 하기 위해 별도 archive-policy (`cloudkitty_1h`) 생성 후 적용
    
    ```bash
    #!/bin/bash
    ###############################################################################
    #   change_metrics_1h.sh
    #   기존 정책 (low*, high, cloudkitty_300s) → cloudkitty_1h 로 재생성
    ###############################################################################
    DRY_RUN=false
    TARGET_POLICY="cloudkitty_1h"
    
    RESOURCE_TYPES=(
      volume instance cpu host # 필요시 조정
    )
    
    echo "=== Switch metrics → ${TARGET_POLICY}  (DRY_RUN=$DRY_RUN) ==="
    for RTYPE in "${RESOURCE_TYPES[@]}"; do
      echo -e "\n🔎  Resource-type: \e[34m$RTYPE\e[0m"
      gnocchi resource list -t "$RTYPE" -f value -c id 2>/dev/null | while read RID; do
        [[ -z "$RID" ]] && continue
        gnocchi resource show "$RID" -f json |
          jq -r '.metrics | to_entries[] | "\(.key):\(.value)"' |
        while IFS=: read MNAME MID; do
          MID=$(echo "$MID" | tr -d ' ')
          [[ -z "$MID" ]] && continue
    
          POLICY=$(gnocchi metric show "$MID" -f json |
                   jq -r '."archive_policy/name"')
    
          echo "  • $MNAME  ($MID)  policy=$POLICY"
          if [[ "$POLICY" != "$TARGET_POLICY" ]]; then
            if $DRY_RUN; then
              echo "    ↳ [DRY-RUN] would recreate with '$TARGET_POLICY'"
            else
              echo "    ↳ deleting & recreating with '$TARGET_POLICY' ..."
              gnocchi metric delete "$MID"
              gnocchi metric create \
                --resource-id "$RID" \
                --archive-policy-name "$TARGET_POLICY" \
                "$MNAME"
            fi
          fi
        done
      done
    done
    echo -e "\n=== Done ==="
    ```
    
    ## 결론
    
    CloudKitty에서 1시간 단위의 과금/수집이 기본 전제로 되어 있으므로, Gnocchi 메트릭의 archive-policy는 반드시 `3600초 granularity`를 포함해야 함.
    
- Gnocchi 메트릭 수집은 정상이나 CloudKitty 과금 계산 미수행
    
    ## 현상
    
    ---
    
    - `Gnocchi`에서는 메트릭 데이터가 정상 수집되고 있음 (예: `vcpus`, `cpu`, `memory.usage` 등)
    - 그러나 `CloudKitty`에서는 해당 데이터를 기반으로 과금 계산이 진행되지 않음
        
        (→ `orchestrator` 로그에 `loaded [0] schedules to process` 지속 발생)
        
    
    ---
    
    ## 원인
    
    - `cloudkitty.conf`의 `fetcher_gnocchi` 설정 오류
    - auth_section = keystone_authtoken 로 keystone_authtoken의 설정을 가져와서 사용
    - 아래와 같이 설정된 경우:
        
        ```bash
        [fetcher_gnocchi]
        auth_section = keystone_authtoken
        region_name = RegionOne
        ```
        
    - CloudKitty는 `[keystone_authtoken]` 섹션에서 인증 정보를 가져오려 하지만,
        
        내부 fetcher 코드에서 `keystoneauth1.exceptions.auth_plugins.MissingAuthPlugin` 에러 발생
        
    
    ---
    
    ## 해결 방법
    
    - `auth_section` 대신 직접 인증 정보 명시
    
    ```bash
    [fetcher_gnocchi]
    region_name = RegionOne
    auth_url = http://192.168.73.171:5000/v3
    memcached_servers = 192.168.73.171:11211
    auth_type = password
    project_domain_name = Default
    user_domain_name = Default
    project_name = service
    username = cloudkitty
    password = mnc1!
    interface = internal
    timeout = 30
    ```
    
- TypeError: AggregatesManager.fetch() got an unexpected keyword argument 'use_history’
    
    > NOTE
> CloudKitty 20.1.0은 `gnocchiclient.aggregates.fetch()` 호출 시 `use_history` 인자를 사용함.
>
> 하지만 Ubuntu에 기본 설치된 `gnocchiclient==7.0.7`은 `use_history` 파라미터를 **지원하지 않음**.
>
> 즉, CloudKitty 버전과 Gnocchi client 버전이 **호환되지 않음**.

    
    **해결 방법**
    
    ```bash
    sudo pip3 install gnocchiclient==7.2.0 --force-reinstall
    ```
    
    **설정 적용**
    
    ```bash
    sudo systemctl restart cloudkitty-api.service cloudkitty-processor.service
    ```
