# NTP(Chrony) 설정

Network Time Protocol (NTP) 설정**
        - Controller Node 구성
            
            ```bash
            # install components
            sudo apt install -y chrony
            
            # configure edit
            sudo vi /etc/chrony/chrony.conf
            
            # 젤 아래에 아래 내용 입력
            server 1.kr.pool.ntp.org
            server 1.asia.pool.ntp.org
            server 3.asia.pool.ntp.org
            
            allow 192.168.73.0/24
            
            # service restart
            service chrony restart
            ```
            
        - Compute Node
            
            ```bash
            # install components
            sudo apt install -y chrony
            
            # configure edit
            sudo vi /etc/chrony/chrony.conf
            
            # 젤 아래에 아래 내용 입력 (controller node)와 시간 동기화
            server 192.168.73.171 iburst
            
            # service restart
            service chrony restart
            ```
            
    -
