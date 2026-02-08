# SQL database (MariaDB)

- 2. SQL database for Ubuntu (Controller Node만 설치)
    - Install packages
        
        ```bash
        #As of Ubuntu 20.04, install the packages:
        #22.04 버전을 사용중이므로 이 버전으로 설치
        sudo apt install -y mariadb-server python3-pymysql
        
        #As of Ubuntu 18.04 or 16.04, install the packages:
        sudo apt install -y mariadb-server python-pymysql
        ```
        
    - create config
        - **sudo vi /etc/mysql/mariadb.conf.d/99-openstack.cnf**
        
        ```bash
        # 아래 내용 복붙
        # ip는 환경에 맞게 변경
        [mysqld]
        bind-address = 192.168.73.171
        
        default-storage-engine = innodb
        innodb_file_per_table = on
        max_connections = 4096
        collation-server = utf8_general_ci
        character-set-server = utf8
        ```
        
        - `service mysql restart`
        - `mysql_secure_installation`
