# OpenStack 모니터링 및 알람 설정

OpenStack 모니터링 및 알람 설정
        
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
    
    #
