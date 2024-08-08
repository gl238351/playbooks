# Automatización con Ansible

Instalación del entorno de ansible

## Installation

Crear un usuario en el servidor ansible llamado soporte con permisos de ejecución. Crear un directorio nombrado playbooks en /home/soporte/.
Dentro de playbooks se crea otro directorio nombrado inventory, este contendrá un archivo nombrado hosts que tiene las IP y hostname de los remotos (ubuntu y centos).
## Crear directorio playbooks
```bash
home/soporte/ mkdir playbooks
cd playbooks/
playbooks/mkdir inventory
$ vi hosts
[ubuntu]
ubuntu-db ansible_host=192.168.1.100

[centos]
centos-app ansible_host=192.168.1.200
centos-app_2 ansible_host=192.168.1.101
[linux:children]
ubuntu
centos
```

Dentro del directorio playbooks se crea el archivo app.properties. 
Esto permite la conexión entre la aplicación con la base de datos. 
```bash
$vi app.properties
tipoDB=mysql
jdbcURL=jdbc:mysql://192.168.1.100:3306/todo
jdbcUsername=todo
jdbcPassword=Prueba2024!
```
Copiar en este mismo directorio el archivo todo.war que se debe descargar de aulas.

```bash
/home/soporte/playbooks/ ls
app.properties
todo.war
```
## Crear la Base de Datos en Ubuntu
Dentro del directorio /playbooks se crea un archivo YAML nombrado mysql-completo.yml. Este debe contener el siguiente script  

```python
---
- name: MySQL
  hosts: ubuntu
  become: yes
  tasks:
    - name: Crear archivo /root/.my.cnf con las credenciales de MySQL
      copy:
        dest: /root/.my.cnf
        content: |
          [client]
          user=root
          password=Laboratorio2024
        owner: root
        group: root
        mode: '0600'

    - name: Instalar dependencias de Python
      apt:
        name: python3-pip
        state: present

    - name: Instalar MySQL
      apt:
        name: mysql-server
        state: present

    - name: Instalar MySQLClient
      apt:
        name: mysql-client
        state: present

    - name: Instalar Python
      apt:
        name: python3-mysqldb
        state: present

    - name: Iniciar el servicio MySQL y asegurarse de que esté habilitado
      service:
        name: mysql
        state: started
        enabled: yes

    - name: Establecer contraseña de root para MySQL
      mysql_user:
        name: root
        password: "Laboratorio2024"
        login_unix_socket: /var/run/mysqld/mysqld.sock
        host: localhost
        state: present

    - name: Asegurarse de que la cuenta root tenga todos los privilegios
      mysql_user:
        name: root
        host: localhost
        priv: '*.*:ALL,GRANT'
        login_unix_socket: /var/run/mysqld/mysqld.sock
        state: present

    - name: Crear base de datos 'todo'
      mysql_db:
        name: todo
        state: present
        login_user: root
        login_password: "Laboratorio2024"

    - name: Crear tabla 'users'
      mysql_query:
        login_user: root
        login_password: "Laboratorio2024"
        query: |
          USE todo;
          CREATE TABLE IF NOT EXISTS `users` (
            `id` int(3) NOT NULL AUTO_INCREMENT,
            `first_name` varchar(20) DEFAULT NULL,
            `last_name` varchar(20) DEFAULT NULL,
            `username` varchar(250) DEFAULT NULL,
            `password` varchar(20) DEFAULT NULL,
            PRIMARY KEY (`id`)
          ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

    - name: Crear tabla 'todos'
      mysql_query:
        login_user: root
        login_password: "Laboratorio2024"
        query: |
          USE todo;
          CREATE TABLE IF NOT EXISTS `todos` (
            `id` bigint(20) NOT NULL AUTO_INCREMENT,
            `description` varchar(255) DEFAULT NULL,
            `is_done` bit(1) NOT NULL,
            `target_date` datetime(6) DEFAULT NULL,
            `username` varchar(255) DEFAULT NULL,
            `title` varchar(255) DEFAULT NULL,
            PRIMARY KEY (`id`)
          ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

    - name: Crear usuario 'todo' y otorgar privilegios
      mysql_user:
        name: todo
        host: '%'
        password: "Prueba2024!"
        priv: 'todo.*:ALL'
        login_user: root
        login_password: "Laboratorio2024"
        state: present

    - name: Refrescar privilegios de MySQL
      mysql_query:
        login_user: root
        login_password: "Laboratorio2024"
        query: "FLUSH PRIVILEGES;"

    - name: Configure MySQL my.cnf
      blockinfile:
        path: /etc/mysql/my.cnf
        block: |
          # Archivo de configuración global de MySQL

          [mysqld]
          bind-address = 0.0.0.0
        marker: "# {mark} ANSIBLE MANAGED BLOCK"
      notify: restart mysql

    - name: Open MySQL port 3306 in UFW
      ufw:
        rule: allow
        port: 3306
        proto: tcp

  handlers:
    - name: restart mysql
      service:
        name: mysql
        state: restarted
...
```
## Crear contenedor utilizando Podman y la imagen de tomcat 9.0
Dentro del directorio /playbooks se crea un archivo YAML nombrado instalacion-tomcat-G.yml. Este debe contener el siguiente script:

```python
---
- name: Playbook para instalar Podman y configurar Tomcat en CentOS
  hosts: centos
  become: yes
  tasks:

    - name: Instalar Podman (DNF Backend)
      dnf:
        name: podman
        state: present

    - name: Crear directorio para almacenar archivos
      file:
        path: /temporal/contenedor
        state: directory
        owner: soporte
        group: soporte
        mode: '0755'

    - name: Copiar archivo todo.war
      copy:
        src: /home/soporte/playbooks/todo.war
        dest: /temporal/contenedor/todo.war
        owner: soporte
        group: soporte
        mode: '0644'

    - name: Copiar archivo app.properties
      copy:
        src: /home/soporte/playbooks/app.properties  
        dest: /temporal/contenedor/app.properties
        owner: soporte
        group: soporte
        mode: '0644'
        
    - name: Configurar permisos de archivos
      file:
        path: /temporal/contenedor
        state: directory
        recurse: yes
        mode: '0755'
        owner: soporte
        group: soporte

    - name: Configurar permisos de archivos en /temporal/contenedor
      file:
        path: /temporal/contenedor/todo.war
        state: file
        mode: '0644'
        owner: soporte
        group: soporte

    - name: Configurar permisos de archivo app.properties
      file:
        path: /temporal/contenedor/app.properties
        state: file
        mode: '0644'
        owner: soporte
        group: soporte

    - name: Crear archivo de servicio para Podman
      copy:
        dest: /etc/systemd/system/miservicio.service
        content: |
          [Unit]
          Description=Servicio para ejecutar Tomcat con Podman
          After=network.target

          [Service]
          Type=simple
          ExecStart=/usr/bin/podman run --rm -p 7070:8080 \
            -v /temporal/contenedor/todo.war:/usr/local/tomcat/webapps/todo.war \
            -v /temporal/contenedor/app.properties:/opt/config/app.properties \
            docker.io/library/tomcat:9.0
          ExecStop=/usr/bin/podman stop -t 2 tomcat-container
          Restart=always
          RestartSec=10

          [Install]
          WantedBy=multi-user.target

    - name: Recargar configuración de systemd
      systemd:
        daemon_reload: yes

    - name: Habilitar el servicio miservicio
      systemd:
        name: miservicio.service
        enabled: yes
        state: started
...
```
## Ejecución de Ansible para aprovisionar los dos servidores Centos y Ubuntu.
Se debe ejecutar desde el directorio /home/soporte/
```bash
ansible-playbooks -i inventory/hosts playbooks/mysql-completo.yml --ask-become-pass
```
```bash
ansible-playbooks -i inventory/hosts playbooks/instalacion-tomcat-G.yml --ask-become-pass
```

## License
Taller Linux 2024
[ORT](https://www.ort.edu.uy)