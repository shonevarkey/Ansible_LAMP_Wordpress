# Ansible_LAMP_Wordpress

- The playbook is used to install a Wordpress on a LAMP environment.

- The templates of the files used for the setup are provided below.

### Playbook

```
---
- name: "LampStack Installation & Configuration"
  hosts: all
  become: true
  vars:
    domain: www.example.com
    httpd_port: 80
    httpd_owner: apache
    httpd_group: apache
    mysql_root: "mysqlroot123"
    mysql_user: "wpuser"
    mysql_password: "wpuser123"
    mysql_database: "wordpress"
    wordpress_url: "https://wordpress.org/wordpress-4.7.7.tar.gz"
  tasks:
    
    - name: "Apache  - Installing httpd"
      yum:
        name: httpd
        state: present
   
    - name: "Apache  - Installing php & Extensions"
      yum:
        name:
          - php
          - php-mysql

    - name: "Apache  - Creating httpd.conf"
      template:
        src: httpd.conf.tmpl
        dest: /etc/httpd/conf/httpd.conf
        owner: "{{ httpd_owner }}"
        group: "{{ httpd_group }}"
            
            
    - name: "Apache  - Creating Virtualhost"
      template:
        src: virtualhost.conf.tmpl
        dest: "/etc/httpd/conf.d/{{ domain }}.conf"
        owner: "{{ httpd_owner }}"
        group: "{{ httpd_group }}"
            
    - name: "Apache  - DocumentRoot Creation"
      file:
        path: "/var/www/html/{{ domain }}"
        state: directory
        owner: "{{ httpd_owner }}"
        group: "{{ httpd_group }}"
            
    - name: "Apache  - Creating index.php"
      copy:
        content: "<?php phpinfo(); ?>"
        dest: "/var/www/html/{{ domain }}/index.php"
        owner: "{{ httpd_owner }}"
        group: "{{ httpd_group }}"
            
            
    - name: "Apache  - Service Restart/Enable"
      service:
        name: httpd
        state: restarted
        enabled: true
            
    - name: "Mariadb - Installation"
      yum:
        name:
          - mariadb-server
          - MySQL-python
        state: present
    
    - name: "Mariadb - Restart/Enable"
      service:
        name: mariadb
        state: restarted
        enabled: true
            
    - name: "Mariadb - Resetting Root Password"
      ignore_errors: true
      mysql_user:
        login_user: root
        login_password: ""
        user: root
        password: "{{ mysql_root }}"
        host_all: true
            
    - name: "Mariadb - Anonymous Users"
      mysql_user:
        login_user: root
        login_password: "{{ mysql_root }}"
        user: ""
        state: absent
        host_all: true
            
    - name: "Mariadb - Creating Extra Database"
      mysql_db:
        login_user: root
        login_password: "{{ mysql_root }}"
        name: "{{ mysql_database }}"
        state: present
            
    - name: "Mariadb - Creating Extra User"
      mysql_user:
        login_user: root
        login_password: "{{ mysql_root }}"
        user: "{{ mysql_user }}"
        password: "{{ mysql_password }}"
        state: present
        priv: "{{ mysql_database }}.*:ALL"

    - name: "Wordpress - Downloading Installation Files"
      get_url:
        url: "{{ wordpress_url }}"
        dest: /tmp/wordpress.tar.gz
            
    - name: "Wordpress - Extracting Installation File"
      unarchive:
        src: /tmp/wordpress.tar.gz
        dest: /tmp/
        remote_src: true
    
    - name: "Worpress  - Copying Contents To DocumentRoot"
      copy:
        src: /tmp/wordpress/
        dest: "/var/www/html/{{ domain }}"
        remote_src: true
            
    - name: "Wordpress - Creating wp-config.php"
      template:
        src: wp-config.php.tmpl
        dest: "/var/www/html/{{ domain }}/wp-config.php"
        owner: "{{ httpd_owner }}"
        group: "{{ httpd_group }}"
    
    - name: "Post-Installation Clean-up"
      file:
        path: "{{ item }}"
        state: absent
      with_items:
        - /tmp/wordpress/
        - /tmp/wordpress.tar.gz
        
    - name: "Post-Installation Restart"
      service:
        name: "{{ item }}"
        state: restarted
        enabled: true
      with_items:
        - httpd
        - mariadb
```

- The virtualhost.conf.tmpl and wp-config.php.tmpl files can be created from a default apache virtualhost and Wordpress wp-config entries respectively. 

### virtualhost.conf.tmpl

```
<virtualhost *:{{ httpd_port }}>
    
    servername  {{ domain }}
    documentroot /var/www/html/{{ domain }}
    directoryindex index.php index.html
    
</virtualhost>
```

### wp-config.php.tmpl

```

<?php

/** The name of the database for WordPress */
define( 'DB_NAME', '{{ mysql_database }}' );

/** MySQL database username */
define( 'DB_USER', '{{ mysql_user }}' );

/** MySQL database password */
define( 'DB_PASSWORD', '{{ mysql_password }}' );

/** MySQL hostname */
define( 'DB_HOST', 'localhost' );

/** Database Charset to use in creating database tables. */
define( 'DB_CHARSET', 'utf8' );

/** The Database Collate type. Don't change this if in doubt. */
define( 'DB_COLLATE', '' );

define( 'AUTH_KEY',         'put your unique phrase here' );
define( 'SECURE_AUTH_KEY',  'put your unique phrase here' );
define( 'LOGGED_IN_KEY',    'put your unique phrase here' );
define( 'NONCE_KEY',        'put your unique phrase here' );
define( 'AUTH_SALT',        'put your unique phrase here' );
define( 'SECURE_AUTH_SALT', 'put your unique phrase here' );
define( 'LOGGED_IN_SALT',   'put your unique phrase here' );
define( 'NONCE_SALT',       'put your unique phrase here' );


$table_prefix = 'wp_';


define( 'WP_DEBUG', false );

if ( ! defined( 'ABSPATH' ) ) {
	define( 'ABSPATH', __DIR__ . '/' );
}

require_once ABSPATH . 'wp-settings.php';
```

### httpd.conf.tmpl

```
ServerRoot "/etc/httpd"

Listen  {{ httpd_port }}

Include conf.modules.d/*.conf

User {{ httpd_owner }}
Group {{ httpd_group }}


ServerAdmin root@localhost

<Directory />
    AllowOverride none
    Require all denied
</Directory>
DocumentRoot "/var/www/html"
<Directory "/var/www">
    AllowOverride None
    # Allow open access:
    Require all granted
</Directory>

<Directory "/var/www/html">
    Options Indexes FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>

<IfModule dir_module>
    DirectoryIndex index.html
</IfModule>

<Files ".ht*">
    Require all denied
</Files>

ErrorLog "logs/error_log"
LogLevel warn

<IfModule log_config_module>
    LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" combined
    LogFormat "%h %l %u %t \"%r\" %>s %b" common
    <IfModule logio_module>
      # You need to enable mod_logio.c to use %I and %O
      LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\" %I %O" combinedio
    </IfModule>
    CustomLog "logs/access_log" combined
</IfModule>

<IfModule alias_module>
    ScriptAlias /cgi-bin/ "/var/www/cgi-bin/"
</IfModule>

<Directory "/var/www/cgi-bin">
    AllowOverride None
    Options None
    Require all granted
</Directory>

<IfModule mime_module>
    TypesConfig /etc/mime.types
    AddType application/x-compress .Z
    AddType application/x-gzip .gz .tgz
    AddType text/html .shtml
    AddOutputFilter INCLUDES .shtml
</IfModule>

AddDefaultCharset UTF-8
<IfModule mime_magic_module>
    MIMEMagicFile conf/magic
</IfModule>
EnableSendfile on
<IfModule mod_http2.c>
    Protocols h2 h2c http/1.1
</IfModule>
IncludeOptional conf.d/*.conf
```

- After running the playbook the Wordpress installation screen can be accessed by surfing the client server IP. 
