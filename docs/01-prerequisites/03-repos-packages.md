# OpenStack 패키지/리포 설치

OpenStack packages 설치
        - Controller Node, Compute Node 동일 → 25.04 기준으로 24.01애 출시된 Caracal 버전을 설치
            
            ```bash
            # OpenStack 2024.1 Caracal for Ubuntu 22.04 LTS:
            sudo add-apt-repository cloud-archive:caracal
            
            # Sample Installation¶
            sudo apt install -y nova-compute
            sudo apt install -y python3-openstackclient
            ```
