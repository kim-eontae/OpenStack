# OpenStack 인스턴스 백신 및 보안 설정

OpenStack 인스턴스 백신 및 보안 설정

- **ClamAV**  - **리눅스 무료 백신**
    - CentOS 9
        - install
            
            ```bash
            yum install -y epel-release
            yum install -y clamav clamd
            ```
            
        - 설정 파일 수정
            
            ```bash
            vi /etc/clamd.d/scan.conf
            
            #96번 라인 #LocalSocket /run/clamd.scan/clamd.sock -> LocalSocket /run/clamd.scan/clamd.sock  주석제거
            ```
            
        - **권한 등록 및 엔진업데이트**
            
            ```bash
            sudo chmod 640 /etc/freshclam.conf
            sudo freshclam
            ```
            
        - **서비스 등록**
            
            ```bash
            systemctl enable clamd@scan
            systemctl start clamd@scan
            systemctl status clamd@scan
            ```
            
        - Cronjob 설정 및 폴더 생성
            
            ```bash
            sudo vi /etc/cron.d/clamav-scan
            # ClamAV 전체 검사 (매일 새벽 3시) #sys, proc, dev 제외
            0 3 * * * root clamscan -r --exclude-dir="^/sys" --exclude-dir="^/proc" --exclude-dir="^/dev" / >> /var/log/clamav/daily-scan.log 2>&1
            
            mkdir -p /var/log/clamav
            ```
            
        - 로그 확인
            
            ```bash
            cat /var/log/clamav/daily-scan.log
            
            ----------- SCAN SUMMARY -----------
            Known viruses: 8707458
            Engine version: 1.0.8
            Scanned directories: 6740
            Scanned files: 36114
            Infected files: 0
            Data scanned: 3506.12 MB
            Data read: 2150.40 MB (ratio 1.63:1)
            Time: 464.594 sec (7 m 44 s)
            Start Date: 2025:05:29 03:00:01
            End Date:   2025:05:29 03:07:46
            ```
            
    - Ubuntu 22.04
        - install
            
            ```bash
            sudo apt install clamav clamav-daemon -y
            ```
            
        - 설치 확인
            
            ```bash
            clamscan --version
            ```
            
        - **권한 등록 및 엔진업데이트**
            
            ```bash
            # 1. 데몬 중지
            sudo systemctl stop clamav-freshclam
            
            # 2. 권한 수정
            sudo chown -R clamav:clamav /var/log/clamav
            sudo chmod 755 /var/log/clamav
            
            # 3. 잠금 파일 제거
            sudo rm -f /var/log/clamav/freshclam.log.lck
            
            # 4. 수동 업데이트
            sudo freshclam
            
            # 5. 완료되면 데몬 재시작 (필요시)
            sudo systemctl start clamav-freshclam
            ```
            
        - **서비스 등록**
            
            ```bash
            sudo systemctl enable clamav-daemon
            sudo systemctl start clamav-daemon
            sudo systemctl status clamav-daemon
            ```
            
        - 스크립트 생성
            
            ```bash
            sudo install -d -m 755 /usr/local/sbin
            sudo tee /usr/local/sbin/clamav-scan.sh >/dev/null <<'SH'
            #!/bin/sh
            set -eu
            
            LOGDIR=/var/log/clamav
            mkdir -p "$LOGDIR"
            
            LOGFILE="$LOGDIR/daily-scan-$(/usr/bin/date +%F).log"
            
            /usr/bin/clamscan -r \
              --exclude-dir="^/sys" \
              --exclude-dir="^/proc" \
              --exclude-dir="^/dev" \
              / > "$LOGFILE" 2>&1
            
            /usr/bin/find "$LOGDIR" -type f -name 'daily-scan-*.log' -mtime +14 -delete
            SH
            sudo chmod 755 /usr/local/sbin/clamav-scan.sh
            
            mkdir -p /var/log/clamav
            ```
            
        - 크론 파일 정리
            
            ```bash
            sudo tee /etc/cron.d/clamav-scan >/dev/null <<'CRON'
            SHELL=/bin/sh
            PATH=/usr/sbin:/usr/bin:/sbin:/bin
            MAILTO=""
            
            # 매일 03:00 실행
            0 3 * * * root /usr/local/sbin/clamav-scan.sh
            CRON
            sudo chmod 644 /etc/cron.d/clamav-scan
            sudo chown root:root /etc/cron.d/clamav-scan
            ```
            
        - 로그 확인
            
            ```bash
            cat /var/log/clamav/daily-scan.log
            
            ----------- SCAN SUMMARY -----------
            Known viruses: 8707458
            Engine version: 1.0.8
            Scanned directories: 6740
            Scanned files: 36114
            Infected files: 0
            Data scanned: 3506.12 MB
            Data read: 2150.40 MB (ratio 1.63:1)
            Time: 464.594 sec (7 m 44 s)
            Start Date: 2025:05:29 03:00:01
            End Date:   2025:05:29 03:07:46
            ```
            
- **~~Snort - 침입 탐지/방지 시스템(이제 설치 안함 - 방화벽으로 대체(OPNsense))~~**
    
    ## Snort 3.0 설치 절차 (CentOS 9 기준)
    
    ---
    
    ### 시스템 업데이트 및 필수 패키지 설치
    
    ```bash
    sudo dnf update -y
    sudo dnf groupinstall "Development Tools" -y
    sudo dnf install epel-release -y
    sudo dnf install cmake libpcap-devel pcre-devel libdnet-devel zlib-devel \
    luajit-devel openssl-devel bison flex git wget -y
    ```
    
    ---
    
    ### Snort 3 의존성 라이브러리 설치
    
    Snort 3는 빠른 패턴 매칭을 위한 Hyperscan, Boost, Ragel, gperftools, flatbuffers 등 다양한 라이브러리를 필요로 합니다. 아래는 주요 라이브러리 설치 예시입니다.
    
    ---
    
    ### PCRE 설치 (정규식 라이브러리)
    
    ```bash
    cd ~/snort_src
    wget https://sourceforge.net/projects/pcre/files/pcre/8.45/pcre-8.45.tar.gz
    tar -xzvf pcre-8.45.tar.gz
    cd pcre-8.45
    ./configure        # 빌드 설정
    make               # 컴파일
    sudo make install  # 설치
    cd ..
    ```
    
    ---
    
    ### gperftools 설치 (성능 분석 도구)
    
    ```bash
    wget https://github.com/gperftools/gperftools/releases/download/gperftools-2.9.1/gperftools-2.9.1.tar.gz
    tar xzvf gperftools-2.9.1.tar.gz
    cd gperftools-2.9.1
    ./configure && make && sudo make install
    cd ..
    ```
    
    ---
    
    ### Ragel 설치 (상태 머신 생성기)
    
    ```bash
    wget http://www.colm.net/files/ragel/ragel-6.10.tar.gz
    tar -xzvf ragel-6.10.tar.gz
    cd ragel-6.10
    ./configure && make && sudo make install
    cd ..
    ```
    
    ---
    
    ### Boost 설치 (C++ 라이브러리 집합)
    
    ```bash
    wget https://archives.boost.io/release/1.88.0/source/boost_1_88_0.tar.gz
    tar -xvzf boost_1_88_0.tar.gz
    cd boost_1_88_0
    ./bootstrap.sh     # 빌드 준비
    ./b2               # 빌드
    sudo ./b2 install  # 설치
    cd ..
    ```
    
    ---
    
    ### Hyperscan 설치 (고속 패턴 매칭 라이브러리)
    
    ```bash
    git clone https://github.com/intel/hyperscan.git
    cd hyperscan
    mkdir build && cd build
    cmake -DBUILD_TOOLS=OFF ..  # 도구 빌드 제외
    make && sudo make install
    cd ../..
    ```
    
    ---
    
    ### flatbuffers 설치 (데이터 직렬화 라이브러리)
    
    ```bash
    wget https://github.com/google/flatbuffers/archive/refs/tags/v2.0.0.tar.gz -O flatbuffers-v2.0.0.tar.gz
    tar -xzvf flatbuffers-v2.0.0.tar.gz
    mkdir flatbuffers-build && cd flatbuffers-build
    cmake ../flatbuffers-2.0.0
    make && sudo make install
    cd ..
    ```
    
    **LibDAQ 설치 (Snort 3 전용 패킷 캡처 라이브러리)**
    
    ```bash
    cd ~/snort_src
    git clone https://github.com/snort3/libdaq.git
    cd libdaq
    ./bootstrap
    ./configure    # 빌드 설정
    make
    sudo make install
    sudo ldconfig  # 라이브러리 캐시 갱신
    ```
    
    **Snort 3 다운로드 및 빌드**
    
    ```bash
    cd ~/snort_src
    git clone https://github.com/snort3/snort3.git
    cd snort3
    
    # 필수 패키지 설치 (이미 설치했다면 생략)
    sudo dnf install -y gcc-c++ cmake hwloc-devel pcre-devel \
      libdnet-devel zlib-devel luajit-devel openssl-devel  
    sudo dnf install -y jemalloc jemalloc-devel
    sudo dnf install -y pcre2 pcre2-devel
    
    # Snort 3 빌드 설정
    ./configure_cmake.sh --prefix=/usr/local --enable-jemalloc \
      --with-daq-includes=/usr/local/include \
      --with-daq-libraries=/usr/local/lib
    
    # 빌드 및 설치
    cd build
    make -j$(nproc)    # 병렬 빌드
    sudo make install
    ```
    
    **설치 라이브러리 경로 등록**
    
    ```bash
    sudo find / -name libdaq.so.3   # 라이브러리 위치 확인
    echo "/usr/local/lib" | sudo tee /etc/ld.so.conf.d/snort.conf
    sudo ldconfig                  # 공유 라이브러리 캐시 갱신
    ```
    
    **설치 확인**
    
    ```bash
    snort --version  # Snort 버전 확인
    ```
    
    **systemd 서비스 파일 수정** 
    
    ```bash
    [Unit]
    Description=Snort 3 Intrusion Detection/Prevention Service
    After=network.target
    
    [Service]
    ExecStart=/usr/local/bin/snort \
      -c /usr/local/etc/snort/snort.lua \
      --daq-dir /usr/local/lib/daq \
      --daq pcap --daq-mode passive \
      -i eth0 -A unified2 -s 65535 -k none \
      -l /var/log/snort3
    ExecReload=/bin/kill -HUP $MAINPID
    Restart=on-failure
    
    [Install]
    WantedBy=multi-user.target
    ```
    
    적용
    
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl restart snort3
    ```
    
    로그파일 이름 변경 스크립트 (날짜별로 보기 좋게)
    
    ```bash
    vi /usr/local/bin/snort-log-rename.sh
    
    #!/bin/bash
    
    cd /var/log/snort3
    for file in unified2.log.*; do
      if [[ -f "$file" ]]; then
        # epoch timestamp → formatted date (YYYY-MM-DD)
        ts=$(echo "$file" | cut -d'.' -f3)
        formatted_date=$(date -d "@$ts" "+%Y-%m-%d")
        new_file="snort_${formatted_date}.u2"
        mv "$file" "$new_file"
      fi
    done
    ```
    
    권한 부여
    
    ```bash
    sudo chmod +x /usr/local/bin/snort-log-rename.sh
    ```
    
    cron으로 매일 00시에 자동 실행
    
    ```bash
    vi /etc/cron.d/snort-log-rename
    
    0 0 * * * root /usr/local/bin/snort-log-rename.sh
    ```
    

[OPNsense 및 OpenVAS 구축](./06-opnsense-openvas.md)

