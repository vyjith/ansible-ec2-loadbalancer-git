# ansible-ec2-loadbalancer-git
```
---
- name: "Creating Aws Infra Using Ansible"
  hosts: localhost
  vars_files:
    - awscredentials.vars
  vars:
    region: "ap-south-1"
    project: "instgram"
    webserver_ports:
      - 80
      - 443
      - 8080  
    remote_ports:
      - 22
  tasks:
    
    - name: "Amazon - Creating Ssh-Key Pair"
      amazon.aws.ec2_key:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        name: "{{project}}"
        state: present
        tags:
          Name: "{{ project }}"
          project: "{{ project }}"
      register: keypair
        
    - name: "Amazon - Saving KeyPair"
      when: keypair.changed 
      copy:
        content: "{{ keypair.key.private_key }}"
        dest: "{{ project }}.pem"
        mode: 0400
            
            
    - name: "Amazon - Creating Security Group webserver"
      amazon.aws.ec2_group:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        name: "webserver-{{project}}"
        description: "webserver-{{project}}"
        state: present
        tags:
          Name: "webserver-{{project}}"
          project: "{{ project }}"
            
        rules:
          - proto: tcp
            ports: "{{ webserver_ports }}"
            cidr_ip: 0.0.0.0/0
      register: sg_webserver 
       
    - name: "Amazon - Creating Security Group remote"
      amazon.aws.ec2_group:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        name: "remote-{{project}}"
        description: "remote-{{project}}"
        state: present
        tags:
          Name: "remote-{{project}}"
          project: "{{ project }}"
            
        rules:
          - proto: tcp
            ports: "{{ remote_ports }}"
            cidr_ip: 0.0.0.0/0
      register: sg_remote
        
        
    - name: "Amazon - Creating Ec2 Instance"
      amazon.aws.ec2_instance:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        key_name: "{{ keypair.key.name }}"
        instance_type: t2.micro
        image_id: "ami-0a3277ffce9146b74"
        wait: yes
        security_groups:
           - "{{ sg_webserver.group_id  }}" 
           - "{{ sg_remote.group_id  }}" 
        tags:
          Name: "webserver-{{project}}"
          project: "{{ project }}"
        exact_count: 4
        filters:
            "tag:Name=webserver-{{project}}"
      register: ec2
    
    - name: "Amazon - Waiting For Ec2 Instacne To Be Online"
      when: ec2.changed 
      wait_for:
        timeout: 60
            
    - name: "Amazon - Fetching Ec2 Info"
      amazon.aws.ec2_instance_info:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        filters:
            "tag:Name=webserver-{{project}}"
      register: ec2
       
    - name: "classic Loadbalancer"
      amazon.aws.elb_classic_lb:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        name: "ansible-classic-loadbalncer"
        state: present
        instance_ids: "{{ item.instance_id }}"
        zones:
          - ap-south-1a
          - ap-south-1b
        listeners:
          - protocol: http 
            load_balancer_port: 80
            instance_port: 80
        security_group_ids:
           - "{{ sg_webserver.group_id }}" 
           - "{{ sg_remote.group_id }}" 
        health_check:   
          ping_protocol: http # options are http, https, ssl, tcp
          ping_port: 80
          ping_path: "/health.html" # not required for tcp or ssl
          response_timeout: 5 # seconds
          interval: 15 # seconds
          unhealthy_threshold: 2
          healthy_threshold: 2
      with_items:
        - "{{ ec2.instances }}"
        
    - name: "Amazon - Creating Dynamic Inventory"
      add_host:
        hostname: '{{ item.public_ip_address }}'
        ansible_ssh_host: '{{ item.public_ip_address }}'
        ansible_ssh_port: 22
        groups:
          - webservers
        ansible_ssh_private_key_file: "{{ project }}.pem"
        ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
      with_items: "{{ ec2.instances }}"

- name: "Deployment From GitHub"
  hosts: webservers
  become: true
  serial: 1
  vars:
    packages:
      - httpd
      - php
      - git
    repo: https://github.com/vyjith/aws-elb-site
  tasks:
    
    - name: "Package Installation"
      yum:
        name: "{{ packages }}"
        state: present
        
    - name: "Clonning Github Repository {{ repo }}" 
      git:
        repo: "{{ repo }}"
        dest: "/var/website/"
      register: gitstatus
        
    - name: "Backend off loading from elb"
      when: gitstatus.changed
      file:
        path: "/var/www/html/health.html"
        mode: 0000
            
    - name: "waiting for connection draining"
      when: gitstatus.changed
      wait_for:
        timeout: 30
            
    - name: "updating site contents"
      when: gitstatus.changed
      copy:
        src: "/var/website/"
        dest: "/var/www/html/"
        remote_src: true
        owner: apache
        group: apache
    - name: "Service httpd"
      service: 
         name: httpd
         state: restarted
         enabled: true
            
    - name: "loading backend to elb"
      when: gitstatus.changed
      file:
        path: "/var/www/html/health.html"
        mode: 0644
            
    - name: "waiting for connection draining"
      when: gitstatus.changed
      wait_for:
        timeout: 20
```

