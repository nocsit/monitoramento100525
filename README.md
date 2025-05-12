# Guía Zabbix 7.2 LTS en Ubuntu 24.04 con NGINX y PostgreSQL

Zabbix es una solución empresarial de código abierto que monitorea diversos parámetros de red, servidores y la salud de los servicios. Utiliza un mecanismo flexible de notificación que permite configurar alertas por correo electrónico y otros medios como Telegram, facilitando una respuesta rápida a problemas. Además, ofrece recursos avanzados para informes y visualización de datos almacenados, siendo ideal para la planificación de capacidad.

---

## 🛠️ Requisitos para Ubuntu Server 24.04
Para una maquina de pruebas (puede ser una maquina virtual en Virtual Box, Proxmox, VMware, Hyper-v, Xenserver, XCP-NG, etc)
- Recomendado 4Gb de memoria RAM
- 2 Nucleos
- 50 Gb disco 
---

## 🚀 Instalación de NGINX (SERVIDOR WEB)
Procederemos con la instalación de nginx, ocultando su versión como una buena práctica.

```bash
sudo -s
apt update
apt -y install nginx
sed -i 's/# server_tokens/server_tokens/' /etc/nginx/nginx.conf
systemctl restart nginx
```

---

## ⚙️ Instalación de PHP 8
Vamos a instalar extensiones de PHP utilizadas por Zabbix.
```bash
apt -y install --no-install-recommends php php-{fpm,cli,mysql,pear,gd,gmp,bcmath,mbstring,curl,xml,zip,json,pgsql}

```

Vamos a modificar el archivo php.ini:
```bash
nano /etc/php/8.3/fpm/php.ini
#Para salir del nano presionamos la combinacion de Ctrl+X nos va a preguntar si queremos guardar ponemos y seguido presionamos ENTER
```
Hay que buscar estas lineas y modificarlas con estos valores para buscar esas lineas en nano hay que presionar Ctrl+w y escribir en el campo
lo que queremos buscar y presionar enter.
```ini
max_execution_time = 600
upload_max_filesize = 100M
```
Reiniciamos el servicio con este comando
```bash
systemctl restart php8.3-fpm
```

---

## 🐘 Instalación de PostgreSQL
Vamos a instalar el motor de base de datos con los siguientes comandos:
```bash
apt -y install postgresql postgresql-contrib
# Aqui vamos a entrar en el usuario postgres y a la aplicacion postgress
su - postgres
psql
```

En la consola de PostgreSQL:
Luego dentro de la aplicacion vamos a colocar una contraseña para el usuario master de la base 
en este tutorial estamos usando la clave cursobase123
```sql
\password postgres
#Nos va a mostrar los siguentes mensajes escribimos la clave que queremos el sistema no va a mostrar nada caso de un error
#el mismo lo va a informar.
#Enter new password for user "postgres": 
#Enter it again: 

CREATE EXTENSION adminpack;
\q
exit
```

Configurar autenticación:
Vamos a configurar para que cada alteracion con la base de datos nos pida la contraseña de master
```bash
sed -i '/postgres.*peer/s/peer/md5/' /etc/postgresql/16/main/pg_hba.conf
sed -i '0,/local\s*all\s*all\s*peer/s/peer/md5/' /etc/postgresql/16/main/pg_hba.conf
```
A hora vamos a editar el archivo de configuracion de postgressql elegir en base a la cantidad de memoria
RAM que estan usando en su servidor.
Editar `nano /etc/postgresql/16/main/postgresql.conf`:
Dentro del archivo buscar las lineas y modificar acorde su servidor.
```conf
#-----------------------------------------
 # 4GB de Memoria RAM
 #-----------------------------------------
 max_connections = 500
 shared_buffers = 1GB
 work_mem = 16MB
 maintenance_work_mem = 128MB
 max_wal_size = 1GB
 min_wal_size = 256MB
 effective_cache_size = 2GB
 
 #-----------------------------------------
 # 8GB de Memoria RAM
 #-----------------------------------------
 max_connections = 1000
 shared_buffers = 2GB
 work_mem = 32MB
 maintenance_work_mem = 512MB
 max_wal_size = 2GB
 min_wal_size = 512MB
 effective_cache_size = 4GB
 
 #-----------------------------------------
 # 16GB de Memoria RAM
 #-----------------------------------------
 max_connections = 2000
 shared_buffers = 4GB
 work_mem = 64MB
 maintenance_work_mem = 1GB
 max_wal_size = 4GB
 min_wal_size = 1GB
 effective_cache_size = 8GB
```
A hora reiniciamos el servicio
```bash
systemctl restart postgresql
```

---

## 📦 Instalación de Zabbix 7.2 LTS

```bash
cd /tmp/
wget https://repo.zabbix.com/zabbix/7.2/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.2+ubuntu24.04_all.deb
dpkg -i zabbix-release_latest_7.2+ubuntu24.04_all.deb
apt update
# Luego vamos a instalar los paquetes con el siguiente comando

apt -y install zabbix-server-pgsql zabbix-frontend-php \
  zabbix-nginx-conf zabbix-sql-scripts zabbix-agent traceroute snmp
```

---

## 🔐 Crear base de datos y usuario
Vamos a crear la base de datos zabbix y el usuario zabbix para ellos nos va a pedir crear una clave vamos a usar cursozabbix123 y para aplicar los cambios nos va a pedir la de master 
que seria cursobase123
```bash
su - postgres
createuser --pwprompt zabbix
createdb -O zabbix zabbix
exit
```

Importar datos:
A hora vamos a importar la estructura de la base de datos para el usuario zabbix 
```bash
zcat /usr/share/zabbix/sql-scripts/postgresql/server.sql.gz | psql -U zabbix -d zabbix
#va a pedir una clave la misma es cursozabbix123
```

---

## ⚙️ Configuración
Vamos a editar el archivo del server de zabbix indicandole la clave de la base de datos.
Editar `nano /etc/zabbix/zabbix_server.conf`:
Buscar la linea que dice DBPassword descomentar 
```conf
DBPassword=cursozabbix123
```
Luego vamos a editar el archivo de php agregando la localizacion de nuestro server.
Editar `nano /etc/zabbix/php-fpm.conf`:

```conf
#esta linea agregamos al final del archivo
php_value[date.timezone] = America/Argentina/Buenos_Aires
#esta linea solo modificar el valor original por 100M
php_value[upload_max_filesize] = 100M
```

---

## 🌐 Configuración de NGINX
Aqui vamos a configurar el servidor web indicando la puerta que vamos a usar y la ip o dominio puse de ejemplo la ip 10.11.104.11 pueden poner la que ustedes tienen en su red.
y hay que agregar client_max_body_size 100M;
Archivo: `nano /etc/nginx/conf.d/zabbix.conf`

```nginx
server {
    listen 80;
    server_name 10.11.104.11 localhost zabbix.tudominio.com;
    client_max_body_size 100M;
    root /usr/share/zabbix;
    index index.php;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

---
Con todos los pasos realizados vamos a reinciar los servicios para probar
## 🟢 Iniciar servicios

```bash
#estamos activando para que inicie automaticamente cuando prender el servidor o reiniciar
systemctl enable zabbix-server
systemctl restart zabbix-server zabbix-agent nginx
```

---
Para finalizar la instalacion a hora vamos a nuestro navegador y ponemos la ip del servidor.
## 🌐 Acceder vía Web

Accede a:

```
http://tu_ip
http://zabbix.tudominio.com
```

Y sigue la configuración web.
