# LAMP

# Prerequisites

Ansible need to be installed in Ansible Controller. 

# Ansible Installation

If Ansible Master is RedHat 
```bash
sudo yum install amazon-linux-extras ansible2
```
If Ansible Master is Debian
```bash
sudo apt update -y
sudo apt install ansible -y
```

# Hosts file
```bash
[amazon]
172.31.14.3 ansible_ssh_user="ec2-user" ansible_ssh_port=22 ansible_ssh_private_key_file="key.pem"

[ubuntu]
172.31.14.33 ansible_ssh_user="ubuntu" ansible_ssh_port=22 ansible_ssh_private_key_file="key.pem"
```

# common_package.yml
```yaml
---
common_packages:
  - MySQL-python
  - mariadb-server
```
# lamp.yml
```yaml
---
- name: "Installing LAMP wordpress in Amazon Linux and Ubuntu"
  become: true
  hosts: all
  vars:
    - wp_url: "https://wordpress.org/latest.tar.gz"
    - hostname: "www.adi.ml"
    - php_package: php
    - amazon_packages:
        - mariadb-server
        - MySQL-python
    - ubuntu_packages:
        - mariadb-server
        - python3-pymysql
    - db_name: "wp"
    - db_user: "wp"
    - db_user_password: "wp@123"
    - mysql_root_password: "mysqlroot@123"

  tasks:
    - name: "Set fact variables for Amazon"
      when: ansible_distribution == "Amazon"
      set_fact:
        package: "httpd"
        httpd_user: "apache"
        httpd_group: "apache"
        httpd_service: "httpd"
    
    - name: "Set fact variables for Ubuntu"
      when: ansible_distribution == "Ubuntu"
      set_fact:
        package: "apache2"
        httpd_user: "www-data"
        httpd_group: "www-data"
        httpd_service: "apache"

    - name: "Creating Virtual Config in Amazon"
      when: ansible_distribution == "Amazon"
      template:
        src: ~/virtualhost.conf.tmpl
        dest: "/etc/httpd/conf.d/{{hostname}}.conf"

    - name: "Creating Virtualhost Config in Ubuntu"
      when: ansible_distribution == "Ubuntu"
      template:
        src: ~/virtualhost.conf.tmpl
        dest: "/etc/apache2/sites-available/{{hostname}}.conf"
    
    - name: "Creating Softlink in sites-enabled for Ubuntu"
      when: ansible_distribution == "Ubuntu"
      file:
        src: "/etc/apache2/sites-available/{{hostname}}.conf"
        dest: "/etc/apache2/sites-enabled/{{hostname}}.conf"
        state: link

    - name: "Installing PHP in Amazon"
      when: ansible_distribution == "Amazon"
      shell: amazon-linux-extras install php7.4 -y

    - name: "Install PHP in Ubuntu"
      when: ansible_distribution == "Ubuntu"
      apt: 
        name: "{{php_package}}"
        state: present
        update_cache: true
        
    - name: "Installing httpd in Amazon"
      when: ansible_distribution == "Amazon"
      yum: 
        name: "{{package}}"
        state: present
      notify:
        - restart-httpd

    - name: "Installing apache in ubuntu"
      when: ansible_distribution == "Ubuntu"
      apt:
        name: "{{package}}"
        state: present
        update_cache: yes
      notify:
        - restart-apache

    - name: "Download Wordpress in Amazon as well as Ubuntu"
      get_url: 
        url: "{{wp_url}}"
        dest: "/tmp/wordpress.tar"

    - name: "Decompress Wordpress files"
      unarchive: 
        src: "/tmp/wordpress.tar"
        dest: "/tmp/"
        remote_src: yes

    - name: "Copying contents from tmp folder to root folder - Amazon"
      when: ansible_distribution == "Amazon"
      copy:
        src: "/tmp/wordpress/"
        dest: "/var/www/html/{{hostname}}"
        group: "{{httpd_group}}"
        owner: "{{httpd_user}}"
        remote_src: yes

    - name: "Creating wp-config using template - Amazon"
      when: ansible_distribution == "Amazon"
      template:
        src: ~/wp-config.php.tmpl
        dest: "/var/www/html/{{hostname}}/wp-config.php"
        group: "{{httpd_group}}"
        owner: "{{httpd_user}}"

    - name: "Copying contents from tmp folder to root folder - Ubuntu"
      when: ansible_distribution == "Ubuntu"
      copy:
        src: "/tmp/wordpress/"
        dest: "/var/www/html/{{hostname}}/"
        group: "{{httpd_group}}"
        owner: "{{httpd_user}}"
        remote_src: yes

    - name: "Creating wp-config using template - Ubuntu"
      when: ansible_distribution == "Ubuntu"
      template:
        src: ~/wp-config.php.tmpl
        dest: "/var/www/html/{{hostname}}/wp-config.php"
        group: "{{httpd_group}}"
        owner: "{{httpd_user}}"
      
    - name: "Install Mariadb-Server and MySQL Python in Amazon"
      when: ansible_distribution == "Amazon"
      yum: 
        name: "{{amazon_packages}}"
        state: present
          
    - name: "Install Mariadb-Server and PyMySQL in Ubuntu"
      when: ansible_distribution == "Ubuntu"
      apt: 
        name: "{{ubuntu_packages}}"
        state: present
        update_cache: true
      
    - name: "Restarting Mariadb Service in Amazon and Ubuntu"
      service:
        name: mariadb
        state: restarted
        enabled: true
      
    - name: "Setting root password in Amazon"
      when: ansible_distribution == "Amazon"
      ignore_errors: true
      mysql_user:
        login_user: "root"
        login_password: ""
        user: "root"
        password: "{{ mysql_root_password }}"
        host_all: true

    - name: "Setting root password in Ubuntu"
      when: ansible_distribution == "Ubuntu"
      ignore_errors: true
      mysql_user:
        login_unix_socket: /var/run/mysqld/mysqld.sock
        login_user: "root"
        login_password: ""
        user: "root"
        password: "{{ mysql_root_password }}"
        host_all: true
    
    - name: "Removing anonymous users in Amazon and Ubuntu"
      mysql_user:
        login_user: "root"
        login_password: "{{ mysql_root_password }}"
        user: ""
        host_all: true 
        state: absent
            
    - name: "Creating DB in Amazon and Ubuntu"
      mysql_db:
        login_user: "root"
        login_password: "{{ mysql_root_password }}"
        name: "{{db_name}}"
        state: present
            
    - name: "Creating DB user in Amazon and Ubuntu"
      mysql_user:
        login_user: "root"
        login_password: "{{ mysql_root_password }}"
        user: "{{db_user}}"
        password: "{{db_user_password}}"
        priv: '{{db_name}}.*:ALL'
        
    - name: "Post installation Clean up in Amazon and Ubuntu"
      file:
        path: "/tmp/wordpress"
        state: absent

  handlers:
  
    - name: "restart-httpd"
      when: ansible_distribution == "Amazon"
      service:
        name: httpd
        state: restarted
        enabled: yes

    - name: "restart-apache"
      when: ansible_distribution == "Ubuntu"
      service:
        name: apache2
        state: restarted
        enabled: yes
```