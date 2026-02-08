# Memcached

- 4.  Memcached (Controller Node만 설치)
    - Install packages
        
        ```bash
        #For Ubuntu versions prior to 18.04 use:
        sudo apt install -y memcached python-memcache
        
        #For Ubuntu 18.04 and newer versions use:
        sudo apt install -y memcached python3-memcache
        ```
        
    - config 파일 수정
        - vi /etc/memcached.conf
            
            ```bash
            # :35 입력해서 35번 라인으로 이동해서 수정 -> **ip는 본인에게 맞게 변경**
            -ㅣ 192.168.73.171
            ```
            
        - `service memcached restart`
