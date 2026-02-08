- Horizon - Dashboard service
        
        ### Controller Node에 수동 설치
        
        - **수동 설치** **(수동 설치로 진행)**
            - 패키지 설치 보다는 복잡하지만, 커스터마이징이 가능
        - 순서
            - git repository에서 clone
            - 관련 패키지 설치, 설정하고, 컴파일
            - 아파치 웹서버 등록
        
        ### Horizon 소스 다운로드
        
        ```bash
        cd /var/www/
        
        # release: 본인이 사용하는 버전에 맞게 설정
        git clone https://opendev.org/openstack/horizon -b stable/<release> --depth=1
        
        cd horizon
        
        # upper-constraints 파일 다운로드
        wget https://opendev.org/openstack/requirements/raw/branch/stable/<release>/upper-constraints.txt
        
        # horizon 버전 주석 처리
        vi upper-constraints.txt
        #horizon===24.1.1
        ```
        
        ### 패키지 설치
        
        ```bash
        pip3 install -c upper-constraints.txt .
        ```
        
        ### local_settings.py 설정
        
        ```bash
        cd openstack_dashboard/local
        cp local_settings.py.example local_settings.py
        vi local_settings.py
        
        DEBUG = True
        WEBROOT = "/horizon/"
        ALLOWED_HOSTS = ['192.168.73.171']  # Controller Node IP
        SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
        CACHES = {
            'default': {
                'BACKEND': 'django.core.cache.backends.memcached.PyMemcacheCache',
                'LOCATION': '192.168.73.171:11211',  # Controller Node IP
            },
        }
        COMPRESS_OFFLINE = True
        
        OPENSTACK_HOST = "192.168.73.171"
        OPENSTACK_KEYSTONE_URL = "http://192.168.73.171:5000/v3"
        
        TIME_ZONE = "Asia/Seoul"
        ```
        
        ### Django 관련 작업
        
        ```bash
        cd /var/www/horizon/
        
        # Translations
        sudo apt install gettext
        ./manage.py compilemessages
        
        # Static Assets
        ./manage.py collectstatic --noinput
        ./manage.py compress --force
        ```
        
        ### Apache 웹서버 설정
        
        ```bash
        apt install apache2 libapache2-mod-wsgi-py3
        
        # fail message 무시해도 됨
        ./manage.py make_web_conf --wsgi
        
        # fail message 무시해도 됨
        ./manage.py make_web_conf --apache > /etc/apache2/sites-available/horizon.conf
        
        # 위 명령어 Permission denied시 아래 명령어 실행
        sudo ./manage.py make_web_conf --apache | sudo tee /etc/apache2/sites-available/horizon.conf > /dev/null
        
        # https 사용 시 아래 코드로 진행
        ./manage.py make_web_conf --apache --ssl --sslkey=/path/to/ssl/key --sslcert=/path/to/ssl/cert > /etc/apache2/sites-available/horizon.conf
        
        a2ensite horizon
        service apache2 restart
        ```
        
        ### Horizon 접속 (브라우저)
        
        - URL: [http://192.168.73.1](http://192.168.73.181)71
        
        ![image.png](assets/images/OpenStack_image_eb4350ed.png)
        
        ```bash
        cd /etc/apache2/sites-available/
        
        # 기본 HTTP 서비스 제거
        a2dissite 000-default.conf
        
        # Horizon 전용 가상 호스트 활성화
        a2ensite horizon.conf
        
        # 설정 적용
        service apache2 restart
        
        # 적용된 설정 확인
        ls -al ../sites-enabled/
        ```
        
        - [http://192.168.73.1](http://192.168.73.181)71 접속
            
            ![image.png](assets/images/OpenStack_image_1_0c7950a1.png)
            
            ```bash
            cd /var/log/apache2/
            
            # 로그 확인 -> PermissionError 발생
            tail -f openstack_dashboard-error.log 
            
            cd /var/www/
            
            # 권한 변경
            chown www-data:www-data horizon/ -R
            
            # 적용
            sudo python3 /var/www/horizon/manage.py collectstatic --noinput
            sudo python3 /var/www/horizon/manage.py compress --force
            ```
            
        
        ### 인증 메뉴 작동 오류 시 horizon.conf 수정
        
        ```bash
        cd /etc/apache2/sites-available/
        
        vi horizon.conf
        
        # 원본 백업
        cp horizon.conf horizon.conf.v0
        
        # 수정
        <VirtualHost *:80>
        
            ServerAdmin webmaster@openstack.org
            ServerName  openstack_dashboard
        
            DocumentRoot /var/www/horizon/
        
            LogLevel warn
            ErrorLog /var/log/apache2/openstack_dashboard-error.log
            CustomLog /var/log/apache2/openstack_dashboard-access.log combined
        
            WSGIScriptReloading On
            WSGIDaemonProcess openstack_dashboard_website processes=105
            WSGIProcessGroup openstack_dashboard_website
            WSGIApplicationGroup %{GLOBAL}
            WSGIPassAuthorization On
        
            **RewriteEngine on
            RewriteCond %{REQUEST_URI} ^/$
            RewriteRule ^/$ /horizon/ [R=301,L]**
        
            **WSGIScriptAlias /horizon /var/www/horizon/openstack_dashboard/horizon_wsgi.py
        
            <Location "/horizon">
                Require all granted
            </Location>
        
            Alias /horizon/static /var/www/horizon/static
            <Location "/var/www/horizon/static">
                SetHandler None
                Require all granted
            </Location>**
        </Virtualhost>
        
        # ------------------------
         a2enmod rewrite
        # ------------------------
         systemctl restart apache2
        # ------------------------
        # 적용
        sudo python3 /var/www/horizon/manage.py collectstatic --noinput
        sudo python3 /var/www/horizon/manage.py compress --force
        ```
        
        ### Horizon Dashboard 로고 변경 (선택)
        
        - 경로: `/var/www/horizon/static/dashboard/img/`
        - **로그인 화면 로고**: `logo-splash.svg`
        - **로그인 후 좌상단 로고**: `logo.svg`
        
        ```bash
        # Horizon Dashboard 로고 변경 절차
        
        # 원본 로고 백업
        cd /var/www/horizon/static/dashboard/img
        cp logo.svg logo.svg.origin
        
        # 새로운 로고 파일(SFTP 등으로 업로드한)로 교체
        mv upload_logo.svg logo.svg
        
        # Django static 파일 갱신
        cd /var/www/horizon/
        ./manage.py collectstatic
        ./manage.py compress --force
        
        # 실행 후 로고 변경확인
        ```
        
        ### Horizon Dashboard 로고 변경 (인터넷 창)
        
        ```bash
        # 텍스트 변경 
        vi /var/www/horizon/openstack_dashboard/local/local_settings.py
        # 아래 추가
        SITE_BRANDING = "GenCloud"
        
        # 로고 뱐경
        sudo vi /var/www/horizon/openstack_dashboard/templates/_stylesheets.html
        # 아래와 같이 변경 (favicon.ico -> favicon.svg)
        <link rel="shortcut icon" href="{% themable_asset 'img/favicon.svg' %}"/>
        ```
        
        ### Horizon Dashboard Material 로고 변경
        
        ```bash
        cd /var/www/horizon/openstack_dashboard/themes/material/templates/material
        
        cp openstack-one-color.svg openstack-one-color.svg.back
        
        vi openstack-one-color.svg
        # 기존 내용 다 지우고 아래 내용 복붙
        
        <?xml version="1.0" encoding="UTF-8"?>
        <svg id="Layer_1" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" viewBox="0 0 189.92 39.69">
          <defs>
            <style>
              .cls-1 {
                fill: url(#New_Gradient_Swatch_3-5);
              }
        
              .cls-2 {
                fill: url(#New_Gradient_Swatch_3-6);
              }
        
              .cls-3 {
                fill: url(#New_Gradient_Swatch_3-2);
              }
        
              .cls-4 {
                fill: url(#New_Gradient_Swatch_3-4);
              }
        
              .cls-5 {
                fill: url(#New_Gradient_Swatch_3-9);
              }
        
              .cls-6 {
                fill: url(#New_Gradient_Swatch_3-8);
              }
        
              .cls-7 {
                fill: url(#New_Gradient_Swatch_3-3);
              }
        
              .cls-8 {
                fill: url(#New_Gradient_Swatch_3-7);
              }
        
              .cls-9 {
                fill: url(#New_Gradient_Swatch_3);
              }
        
              .cls-10 {
                fill: url(#New_Gradient_Swatch_3-10);
              }
            </style>
            <linearGradient id="New_Gradient_Swatch_3" data-name="New Gradient Swatch 3" x1="2.72" y1="30.06" x2="25.69" y2=".71" gradientUnits="userSpaceOnUse">
              <stop offset=".19" stop-color="#8a49ff"/>
              <stop offset=".34" stop-color="#9862ff"/>
              <stop offset=".6" stop-color="#ae8bff"/>
              <stop offset=".74" stop-color="#b79bff"/>
            </linearGradient>
            <linearGradient id="New_Gradient_Swatch_3-2" data-name="New Gradient Swatch 3" x1="-.99" y1="27.16" x2="21.98" y2="-2.19" xlink:href="#New_Gradient_Swatch_3"/>
            <linearGradient id="New_Gradient_Swatch_3-3" data-name="New Gradient Swatch 3" x1="-1.19" y1="27.01" x2="21.78" y2="-2.35" xlink:href="#New_Gradient_Swatch_3"/>
            <linearGradient id="New_Gradient_Swatch_3-4" data-name="New Gradient Swatch 3" x1="2.96" y1="30.25" x2="25.93" y2=".9" xlink:href="#New_Gradient_Swatch_3"/>
            <linearGradient id="New_Gradient_Swatch_3-5" data-name="New Gradient Swatch 3" x1="9.13" y1="35.08" x2="32.1" y2="5.73" xlink:href="#New_Gradient_Swatch_3"/>
            <linearGradient id="New_Gradient_Swatch_3-6" data-name="New Gradient Swatch 3" x1="15" y1="39.67" x2="37.97" y2="10.32" xlink:href="#New_Gradient_Swatch_3"/>
            <linearGradient id="New_Gradient_Swatch_3-7" data-name="New Gradient Swatch 3" x1="18.71" y1="42.58" x2="41.68" y2="13.22" xlink:href="#New_Gradient_Swatch_3"/>
            <linearGradient id="New_Gradient_Swatch_3-8" data-name="New Gradient Swatch 3" x1="18.91" y1="42.73" x2="41.88" y2="13.38" xlink:href="#New_Gradient_Swatch_3"/>
            <linearGradient id="New_Gradient_Swatch_3-9" data-name="New Gradient Swatch 3" x1="14.76" y1="39.48" x2="37.73" y2="10.13" xlink:href="#New_Gradient_Swatch_3"/>
            <linearGradient id="New_Gradient_Swatch_3-10" data-name="New Gradient Swatch 3" x1="8.59" y1="34.66" x2="31.56" y2="5.31" xlink:href="#New_Gradient_Swatch_3"/>
          </defs>
          <g>
            <path d="M66.5,22.34h4.34c-.79,3.42-3.86,5.77-7.76,5.77-4.65,0-8.2-3.55-8.2-8.27s3.55-8.27,8.2-8.27c3.04,0,5.57,1.47,6.9,3.72h4.54c-1.44-4.48-5.98-7.65-11.45-7.65-7.04,0-12.4,5.26-12.4,12.2s5.36,12.2,12.4,12.2,12.1-5.16,12.1-11.99v-1.37h-8.68v3.66Z"/>
            <polygon points="87.55 21.69 98.57 21.69 98.57 17.86 87.55 17.86 87.55 11.88 99.69 11.88 99.69 8.05 83.49 8.05 83.49 31.63 99.86 31.63 99.86 27.8 87.55 27.8 87.55 21.69"/>
            <path d="M149.21,7.65c-7.04,0-12.4,5.26-12.4,12.2s5.36,12.2,12.4,12.2,12.4-5.26,12.4-12.2-5.33-12.2-12.4-12.2ZM149.21,28.11c-4.65,0-8.2-3.55-8.2-8.27s3.55-8.27,8.2-8.27,8.23,3.55,8.23,8.27-3.55,8.27-8.23,8.27Z"/>
            <g>
              <rect x="109.28" y="8.05" width="4.07" height="23.58"/>
              <rect x="125.79" y="8.05" width="4.07" height="23.58"/>
              <polygon points="113.34 8.05 121.95 31.63 125.79 31.63 117.18 8.05 113.34 8.05"/>
            </g>
            <g>
              <rect x="168.56" y="8.05" width="4.07" height="23.58"/>
              <rect x="185.07" y="8.05" width="4.07" height="23.58"/>
              <polygon points="172.63 8.05 181.23 31.63 185.07 31.63 176.46 8.05 172.63 8.05"/>
            </g>
          </g>
          <g>
            <path class="cls-9" d="M26.21.8c-2.53.92-4.86,2.26-6.89,3.97-2.03,1.71-3.76,3.77-5.1,6.1l2.58,1.49c1.17-2.02,2.67-3.82,4.44-5.3,1.77-1.48,3.8-2.65,5.99-3.46l-1.02-2.8Z"/>
            <path class="cls-3" d="M13.96,1.15c-1.5,2.23-2.6,4.69-3.24,7.26-.64,2.58-.82,5.26-.54,7.93l2.96-.31c-.24-2.32-.08-4.66.48-6.9.56-2.24,1.51-4.38,2.82-6.32l-2.47-1.66Z"/>
            <path class="cls-7" d="M4.24,8.64c.1,2.69.65,5.32,1.65,7.78,1,2.46,2.43,4.74,4.22,6.74l2.21-1.99c-1.56-1.74-2.81-3.72-3.67-5.86-.87-2.14-1.35-4.43-1.44-6.77l-2.97.1Z"/>
            <path class="cls-4" d="M.78,20.4c1.66,2.12,3.66,3.92,5.91,5.33,2.25,1.41,4.75,2.41,7.38,2.97l.62-2.91c-2.28-.49-4.46-1.36-6.42-2.58-1.96-1.22-3.7-2.79-5.14-4.63l-2.35,1.83Z"/>
            <path class="cls-1" d="M4.9,31.96c2.58.74,5.26,1.02,7.91.84,2.65-.19,5.26-.84,7.71-1.94l-1.21-2.72c-2.14.95-4.4,1.52-6.71,1.68-2.3.16-4.63-.08-6.88-.73l-.82,2.86Z"/>
            <path class="cls-2" d="M15.03,38.88c2.53-.92,4.86-2.26,6.89-3.97,2.03-1.71,3.76-3.77,5.1-6.1l-2.58-1.49c-1.17,2.02-2.67,3.82-4.44,5.3-1.77,1.48-3.8,2.65-5.99,3.46l1.02,2.8Z"/>
            <path class="cls-8" d="M27.28,38.53c1.5-2.23,2.6-4.69,3.24-7.26.64-2.58.82-5.26.54-7.93l-2.96.31c.24,2.32.08,4.66-.48,6.9-.56,2.24-1.51,4.38-2.82,6.32l2.47,1.66Z"/>
            <path class="cls-6" d="M37,31.05c-.1-2.69-.65-5.32-1.65-7.78-1-2.46-2.43-4.74-4.22-6.74l-2.21,1.99c1.56,1.74,2.81,3.72,3.67,5.86.87,2.14,1.35,4.43,1.44,6.77l2.97-.1Z"/>
            <path class="cls-5" d="M40.46,19.28c-1.66-2.12-3.66-3.92-5.91-5.33-2.25-1.41-4.75-2.41-7.38-2.97l-.62,2.91c2.28.49,4.46,1.36,6.42,2.58,1.96,1.22,3.7,2.79,5.14,4.63l2.35-1.83Z"/>
            <path class="cls-10" d="M36.34,7.73c-2.58-.74-5.26-1.02-7.91-.84-2.65.19-5.26.84-7.71,1.94l1.21,2.72c2.14-.95,4.4-1.52,6.71-1.68,2.3-.16,4.63.08,6.88.73l.82-2.86Z"/>
          </g>
        </svg>
        ```
