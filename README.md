# ansible-iaw
Este repositorio es para la práctica04.4 de Ansible del módulo IAW

# Instalación en 2 niveles con Ansible
## Instalación pila LAMP
### ¿Qué es LAMP?
#### L → Linux (Sistema operativo).
#### A → Apache (Servidor web).
#### M → MySQL/MariaDB (Sistema gestor de base de datos).
#### P → PHP (Lenguaje de programación).

### Archivo inventory/inventory
```
[frontend]
34.199.151.166

[backend]
35.170.221.241

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=/home/ubuntu/ansible-iaw/wordpress/labsuser.pem
ansible_ssh_common_args='-o StrictHostKeyChecking=accept-new'
```

### Direcotorio templates
#### Archivo .htaccess
```
# BEGIN WordPress
<IfModule mod_rewrite.c>
RewriteEngine On
RewriteBase /
RewriteRule ^index\.php$ - [L]
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule . /index.php [L]
</IfModule>
# END WordPress
```
#### Archivo 000-default.conf
```
ServerSignature Off
ServerTokens Prod

<VirtualHost *:80>
        DocumentRoot /var/www/html
        DirectoryIndex index.php index.html

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        <Directory "/var/www/html">
            AllowOverride All
        </Directory>
</VirtualHost>
```

### Archivo vars/variables.yml
```
installation_path: /var/www/html
db:
    name: wordpress_db
    user: wordpress_user
    password: wordpress_passwd
    MYSQL_PRIVATE: 172.31.85.162

wordpress:
    wordpress_title: "Sitio web de IAW"
    wordpress_admin_user: admin
    wordpress_admin_pass: admin
    wordpress_admin_email: demo@demo.es
    WORDPRESS_DB_HOST: 172.31.85.162
    WORDPRESS_HIDE_LOGIN: acceso

certbot:
    email: admin_wordpress@email.es
    domain: ansible-iaw.ddns.net

phpmyadmin:
    user: pma_user
    password: pma_password
    db_name: db_name
```

### Directorio playbooks
#### Archivo install_tools.yml
```
---
- name: Playbook para instalar la pila LAMP en el FrontEnd
  hosts: frontend
  become: yes

  tasks:

    - name: Actualizar los repositorios
      apt:
        update_cache: yes

    - name: Instalar el servidor web Apache
      apt:
        name: apache2
        state: present
      
    - name: Copiar el archivo de configuración de Apache
      template: # Si se hace con template se modifica el contenido con variables y además manda el contenido a la máquina destino.
        src: ../templates/000-default.conf
        dest: /etc/apache2/sites-available/
        mode: 0755

    - name: Instalar PHP y los módulos necesarios
      apt: 
        name:
          - php
          - php-mysql
          - libapache2-mod-php
        state: present

    - name: Habilitar el módulo rewrite de Apache
      apache2_module:
        name: rewrite
        state: present

    - name: Reiniciar el servidor web Apache
      service:
        name: apache2
        state: restarted
```

#### Archivo install_lamp_backend.yml
```
---
- name: Playbook para instalar la pila LAMP en el Backend
  hosts: backend
  become: yes

  vars_files:
    - ../vars/variables.yml

  tasks:

    - name: Actualizar los repositorios
      apt:
        update_cache: yes

    - name: Instalar el sistema gestor de bases de datos MySQL
      apt:
        name: mysql-server
        state: present

    - name: Instalamos el gestor de paquetes de Python pip3
      apt: 
        name: python3-pip
        state: present

    - name: Instalamos el módulo de pymysql
      pip:
        name: pymysql
        state: present

    - name: Crear una base de datos
      mysql_db:
        name: "{{ db.name }}"
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock 

    - name: Crear el usuario de la base de datos
      mysql_user:         
        name: "{{ db.user }}"
        password: "{{ db.password }}"
        priv: "{{ db.name }}.*:ALL"
        host: "%"
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock 

    - name: Configuramos MySQL para permitir conexiones desde cualquier interfaz
      replace:
        path: /etc/mysql/mysql.conf.d/mysqld.cnf
        regexp: 127.0.0.1
        replace: "{{ db.MYSQL_PRIVATE }}" # Ip privada del backend.

    - name: Reiniciamos el servicio de base de datos
      service:
        name: mysql
        state: restarted
```

#### Archivo install_tools.yml
```
---
- name: Playbook para instalar herramientas adicionales
  hosts: aws
  become: yes

  vars_files:
    - ../vars/variables.yml

  tasks:

    - name: Descargar phpMyAdmin
      get_url:
        url: https://files.phpmyadmin.net/phpMyAdmin/5.2.0/phpMyAdmin-5.2.0-all-languages.zip
        dest: /tmp/phpMyAdmin-5.2.0-all-languages.zip

    - name: Instalar unzip
      apt:
        name: unzip
        state: present

    - name: Descomprimir phpMyAdmin
      unarchive:
        src: /tmp/phpMyAdmin-5.2.0-all-languages.zip
        dest: /tmp/
        remote_src: yes

    - name: Copiar phpMyAdmin
      copy:
        src: /tmp/phpMyAdmin-5.2.0-all-languages/
        dest: /var/www/html/phpmyadmin
        remote_src: yes
        mode: 0755

    - name: Cambiar el propietario y el grupo de phpMyAdmin
      file:
        path: /var/www/html/phpmyadmin
        state: directory
        owner: www-data
        group: www-data
        recurse: yes

    # Para utilizar el módulo mysql_db necesitamos tener instalado el paquete python3-mysqldb.
    - name: Instalar pip3
      apt:
        name: python3-pip
        state: present

    - name: Instalar PyMySQL
      pip:
        name: pymysql
        state: present

    - name: Crear la base de datos para phpMyAdmin
      mysql_db:
        name: "{{ phpmyadmin.db_name }}"
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock

    - name: Importar la base de datos de phpMyAdmin
      mysql_db:
        name: "{{ phpmyadmin.db_name }}"
        state: import
        target: /var/www/html/phpmyadmin/sql/create_tables.sql
        login_unix_socket: /var/run/mysqld/mysqld.sock

    - name: Crear el usuario de phpMyAdmin
      mysql_user:
        name: "{{ phpmyadmin.user }}"
        password: "{{ phpmyadmin.password }}"
        priv: "{{ phpmyadmin.db_name }}.*:ALL"
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock
```

#### Archivo config_https.yml
```
---
- name: Playbook para configurar HTTPS
  hosts: frontend
  become: yes

  vars_files:
    - ../vars/variables.yml

  tasks:

    - name: Desinstalar instalaciones previas de Certbot
      snap:
        name: certbot
        state: absent

    - name: Instalar Certbot con snap
      snap:
       name: certbot
       classic: yes
       state: present
      

    - name: Crear un alias para el comando certbot
      command: ln -s -f /snap/bin/certbot /usr/bin/certbot

    - name: Solicitar y configurar certificado SSL/TLS a Let's Encrypt con certbot
      command:
        certbot --apache \
        -m {{ certbot.email }} \
        --agree-tos \
        --no-eff-email \
        --non-interactive \
        -d {{ certbot.domain }}
```

#### Archivo deploy_web.yml
```
---
- name: Playbook para hacer el deploy de la aplicación web PrestaShop
  hosts: frontend
  become: yes

  vars_files:
    - ../vars/variables.yml

  tasks:

    - name: Borrar archivos previos de wordpress.
      file:
        path: /tmp/wp-cli.phar
        state: absent

    - name: Descargar el código fuente de Wordpress.
      get_url:
        url: https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
        dest: /tmp
        mode: 0755

    - name: Movemos el fichero a bin para incluirlo en la lista de comandos.
      command: mv /tmp/wp-cli.phar /usr/local/bin/wp 

    - name: Eliminamos instalaciones previas de wordpress.
      shell: rm -rf /var/www/html/*

    - name: Descargamos el codigo fuente de wordpress en /var/www/html 
      command: wp core download --path=/var/www/html --locale=es_ES --allow-root
    
    - name: Creación del archivo wp-config
      command: wp config create \
              --dbname="{{ db.name }}" \
              --dbuser="{{ db.user }}"  \
              --dbpass="{{ db.password }}" \
              --dbhost="{{ db.MYSQL_PRIVATE }}" \
              --path=/var/www/html \
              --allow-root

    - name: Instalo wordpress
      command: wp core install \
                --url="{{ certbot.domain }}" \
                --title="{{ wordpress.wordpress_title }}" \
                --admin_user="{{ wordpress.wordpress_admin_user }}" \
                --admin_password="{{ wordpress.wordpress_admin_pass }}" \
                --admin_email="{{ wordpress.wordpress_admin_email }}" \
                --path=/var/www/html \
                --allow-root
                
    - name: Actualizamos el core de wordpress.
      command: wp core update --path=/var/www/html --allow-root

    - name: Instalamos un tema para wordpress
      command: wp theme install sydney --activate --path=/var/www/html --allow-root
   
    - name: Instalamos un plugin para wordpress. 
      command: wp plugin install bbpress --activate --path=/var/www/html --allow-root
   
    - name: Instalación del plugin para ocultar login.
      command: wp plugin install wps-hide-login --activate --path=/var/www/html --allow-root

    - name: Habilitamos reescritura para mejorar el seo.
      command:  wp rewrite structure '/%postname%/' \
               --path=/var/www/html \
               --allow-root
               
    - name: Actualizar el plugin de wordpress para cambiar el fichero de la página de logín a otro y no aparezca en el navegador.            
      command: wp option update whl_page "{{ wordpress.WORDPRESS_HIDE_LOGIN }}" --path=/var/www/html --allow-root
    
    - name: Copiar el archivo .htacces
      template: # Si se hace con template se modifica el contenido con variables y además manda el contenido a la máquina destino.
         src: ../templates/.htaccess
         dest: /var/www/html/
         mode: 0755

    - name: Cambiar el propietario de html. 
      file: 
        dest: /var/www/html
        owner: www-data
        group: www-data
        recurse: yes
```

## Como ejecutar el playbook
#### Hacemos uso del archivo main.yml
```
---
- import_playbook: playbooks/install_lamp_frontend.yml
- import_playbook: playbooks/install_lamp_backend.yml
- import_playbook: playbooks/install_tools.yml
- import_playbook: playbooks/config_https.yml
- import_playbook: playbooks/deploy_web.yml
```
### En el directorio wordpress ejecutamos este comando:
```
ansible-playbook -i /home/ubuntu/ansible-iaw/wordpress/inventory/inventory main.yml

```
