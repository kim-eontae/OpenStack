# Message queue (RabbitMQ)

- 3. Message queue for Ubuntu (Controller Node만 설치)
    - Install packages
        
        ```bash
        sudo apt install -y rabbitmq-server
        ```
        
    - Add the **`openstack`** user
        
        ```bash
        sudo rabbitmqctl add_user openstack RABBIT_PASS
        
        # Creating user "openstack" ...
        ```
        
    - Permit configuration, write, and read access for the **`openstack`** user:
        
        ```
        sudo rabbitmqctl set_permissions openstack ".*" ".*" ".*"
        
        # Setting permissions for user "openstack" in vhost "/" ...
        ```
