# Guía Zabbix 7.2 LTS en Ubuntu 24.04 con NGINX y PostgreSQL

Zabbix es una solución empresarial de código abierto que monitorea diversos parámetros de red, servidores y la salud de los servicios. Utiliza un mecanismo flexible de notificación que permite configurar alertas por correo electrónico y otros medios como Telegram, facilitando una respuesta rápida a problemas. Además, ofrece recursos avanzados para informes y visualización de datos almacenados, siendo ideal para la planificación de capacidad.

---

## 🛠️ Requisitos para Ubuntu Server 24.04
Para una maquina de pruebas (puede ser una maquina virtual en Virtual Box, Proxmox, VMware, Hyper-v, Xenserver, XCP-NG, etc)
- Recomendado 4Gb de memoria RAM
- 2 Nucleos
- 50 Gb disco 
---

## 🚀 Instalación de NGINX

```bash
apt update
apt -y install nginx
sed -i 's/# server_tokens/server_tokens/' /etc/nginx/nginx.conf
systemctl restart nginx
```

---

## ⚙️ Instalación de PHP 8

```bash
apt -y install --no-install-recommends php php-{fpm,cli,mysql,pear,gd,gmp,bcmath,mbstring,curl,xml,zip,json,pgsql}
nano /etc/php/8.3/fpm/php.ini
```

Modificar:
```ini
max_execution_time = 600
upload_max_filesize = 100M
```

```bash
systemctl restart php8.3-fpm
```

---

## 🐘 Instalación de PostgreSQL

```bash
apt -y install postgresql postgresql-contrib
su - postgres
psql
```

En la consola de PostgreSQL:

```sql
\password postgres
CREATE EXTENSION adminpack;
\q
exit
```

Configurar autenticación:

```bash
sed -i '/postgres.*peer/s/peer/md5/' /etc/postgresql/16/main/pg_hba.conf
sed -i '0,/local\s*all\s*all\s*peer/s/peer/md5/' /etc/postgresql/16/main/pg_hba.conf
```

Editar `/etc/postgresql/16/main/postgresql.conf`:

```conf
max_connections = 1000
shared_buffers = 2GB
work_mem = 32MB
maintenance_work_mem = 512MB
max_wal_size = 2GB
min_wal_size = 512MB
effective_cache_size = 4GB
```

```bash
systemctl restart postgresql
```

---

## 📦 Instalación de Zabbix 7 LTS

```bash
cd /tmp/
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu24.04_all.deb
dpkg -i zabbix-release_latest_7.0+ubuntu24.04_all.deb
apt update

apt -y install zabbix-server-pgsql zabbix-frontend-php \
  zabbix-nginx-conf zabbix-sql-scripts zabbix-agent traceroute
```

---

## 🔐 Crear base de datos y usuario

```bash
su - postgres
createuser --pwprompt zabbix
createdb -O zabbix zabbix
exit
```

Importar datos:

```bash
zcat /usr/share/zabbix-sql-scripts/postgresql/server.sql.gz | psql -U zabbix -d zabbix
```

---

## ⚙️ Configuración

Editar `/etc/zabbix/zabbix_server.conf`:

```conf
DBPassword=cursozabbix123
```

Editar `/etc/zabbix/php-fpm.conf`:

```conf
php_value[date.timezone] = America/Argentina/Buenos_Aires
php_value[upload_max_filesize] = 100M
```

---

## 🌐 Configuración de NGINX

Archivo: `/etc/nginx/conf.d/zabbix.conf`

```nginx
server {
    listen 80;
    server_name zabbix.tudominio.com;

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

## 🟢 Iniciar servicios

```bash
systemctl enable zabbix-server
systemctl restart zabbix-server zabbix-agent nginx
```

---

## 🌐 Acceder vía Web

Accede a:

```
http://tu_ip
http://zabbix.tudominio.com
```

Y sigue la configuración web.
