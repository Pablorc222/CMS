# Proyecto OwnCloud con NFS, Nginx y Balanceador de Carga

Este repositorio configura un entorno de servidores con **OwnCloud**, **NFS** (Network File System), y un **Balanceador de Carga** utilizando **Vagrant** y **VirtualBox**. Los servidores están divididos en varias máquinas virtuales, cada una con tareas específicas.

## Índice

1. [Descripción del Proyecto](#descripción-del-proyecto)
2. [Requisitos](#requisitos)
3. [Estructura del Proyecto](#estructura-del-proyecto)
4. [Vagrantfile](#vagrantfile)
5. [Scripts de Aprovisionamiento](#scripts-de-aprovisionamiento)
   - [BDD.sh](#bddsh)
   - [NFS.sh](#nfssh)
   - [Webs.sh](#webssh)
   - [Balanceador.sh](#balanceadorsh)
6. [Configuración Nginx](#configuración-nginx)
7. [Solución de Problemas](#solución-de-problemas)

---

## Descripción del Proyecto

Este proyecto configura un entorno multi-servidor que incluye:

- **Servidor de Base de Datos (MySQL)**
- **Servidor NFS** (para compartir archivos)
- **Dos Servidores Web** (para alojar OwnCloud)
- **Balanceador de Carga** (para distribuir el tráfico entre los servidores web)

Las máquinas están configuradas utilizando **Vagrant** y **VirtualBox**, y los **scripts de aprovisionamiento** se encargan de la instalación y configuración de todos los servicios necesarios para hacer funcionar el entorno de manera automatizada.

---

## Requisitos

Antes de comenzar con la implementación, asegúrate de tener los siguientes programas instalados:

1. **Vagrant**: Herramienta para gestionar entornos virtuales.
2. **VirtualBox**: Proveedor de virtualización utilizado por Vagrant.
3. **NFS**: Sistema de archivos para compartir datos entre máquinas.
4. **NGINX**: Servidor web y balanceador de carga.
5. **OwnCloud**: Plataforma de almacenamiento y colaboración.

---

## Estructura del Proyecto

La estructura de archivos del proyecto es la siguiente:

/provision ├── BDD.sh ├── NFS.sh ├── Webs.sh └── Balanceador.sh Vagrantfile README.md


- **Vagrantfile**: Archivo de configuración de Vagrant que define las máquinas virtuales.
- **/provision**: Directorio que contiene los scripts de aprovisionamiento para cada uno de los servidores.

---

## Vagrantfile

Este archivo configura las máquinas virtuales, redes privadas y públicas, y define los scripts de aprovisionamiento para cada uno de los servidores.

```ruby
Vagrant.configure("2") do |config|

  # Servidor de base de datos
  config.vm.define "dbPabloRdgz" do |db|
    db.vm.box = "debian/bullseye64"
    db.vm.network "private_network", ip: "192.168.8.14", virtualbox_intnet: "prnetwork_db"
    db.vm.provision "shell", path: "provision/BDD.sh"
  end

  # Servidor NFS
  config.vm.define "NFSPabloRdgz" do |nfs|
    nfs.vm.box = "debian/bullseye64"
    nfs.vm.network "private_network", ip: "192.168.9.13", virtualbox_intnet: "prnetwork"
    nfs.vm.network "private_network", ip: "192.168.8.13", virtualbox_intnet: "prnetwork_db"
    nfs.vm.provision "shell", path: "provision/nfs.sh"
  end

  # Servidores web
  config.vm.define "web1PabloRdgz" do |serverweb1|
    serverweb1.vm.box = "debian/bullseye64"
    serverweb1.vm.network "private_network", ip: "192.168.9.11", virtualbox_intnet: "prnetwork"
    serverweb1.vm.network "private_network", ip: "192.168.8.11", virtualbox_intnet: "prnetwork_db"
    serverweb1.vm.provision "shell", path: "provision/webs.sh"
  end

  config.vm.define "web2PabloRdgz" do |serverweb2|
    serverweb2.vm.box = "debian/bullseye64"
    serverweb2.vm.network "private_network", ip: "192.168.9.12", virtualbox_intnet: "prnetwork"
    serverweb2.vm.network "private_network", ip: "192.168.8.12", virtualbox_intnet: "prnetwork_db"
    serverweb2.vm.provision "shell", path: "provision/webs.sh"
  end

  # Maquina balanceador
  config.vm.define "balanceadorPabloRdgz" do |balanceador|
    balanceador.vm.box = "debian/bullseye64"
    balanceador.vm.network "public_network"
    balanceador.vm.network "forwarded_port", guest: 80, host: 9090  # Cambié el puerto de 8080 a 9090
    balanceador.vm.network "private_network", ip: "192.168.9.10", virtualbox_intnet: "prnetwork"
    balanceador.vm.provision "shell", path: "provision/balanceador.sh"
  end
end
Scripts de Aprovisionamiento
A continuación, se detallan los scripts que se utilizan para la configuración de los servidores. Los scripts están en el directorio provision/.

BDD.sh
Este script configura la base de datos MySQL para OwnCloud.

bash
Copiar código
#!/bin/bash
apt-get update
apt-get install -y mysql-server

# Configurar base de datos para OwnCloud
mysql -u root <<EOF
CREATE DATABASE owncloud;
CREATE USER 'owncloud'@'%' IDENTIFIED BY '1234';
GRANT ALL PRIVILEGES ON owncloud.* TO 'owncloud'@'%';
FLUSH PRIVILEGES;
EOF
NFS.sh
Este script configura el servidor NFS para compartir el directorio de OwnCloud.

bash
Copiar código
#!/bin/bash
apt-get update
apt-get install -y nfs-kernel-server

# Crear directorio y exportar
mkdir -p /var/www/html/owncloud/data
chown -R www-data:www-data /var/www/html/owncloud/data

# Configurar el archivo de exportaciones
echo "/var/www/html/owncloud/data 192.168.9.11(rw,sync,no_subtree_check)" >> /etc/exports
echo "/var/www/html/owncloud/data 192.168.9.12(rw,sync,no_subtree_check)" >> /etc/exports

# Reiniciar NFS
exportfs -a
systemctl restart nfs-kernel-server
Webs.sh
Este script configura los servidores web con Nginx y PHP-FPM para OwnCloud.

bash
Copiar código
#!/bin/bash
apt-get update
apt-get install -y nginx php-fpm php-mysql

# Configurar el archivo Nginx
cat <<EOF > /etc/nginx/sites-available/default
server {
    listen 80;
    server_name localhost;

    root /var/www/html/owncloud;
    index index.php index.html index.htm;

    location / {
        try_files \$uri \$uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
        fastcgi_param SCRIPT_FILENAME \$document_root\$fastcgi_script_name;
        include fastcgi_params;
    }
}
EOF

# Reiniciar servicios
systemctl restart nginx
systemctl restart php7.4-fpm
Balanceador.sh
Este script configura el balanceador de carga utilizando Nginx.

bash
Copiar código
#!/bin/bash
apt-get update
apt-get install -y nginx

# Configurar el balanceador de carga
cat <<EOF > /etc/nginx/sites-available/default
upstream backend_servers {
    server 192.168.9.11;
    server 192.168.9.12;
}

server {
    listen 80;
    server_name localhost;

    location / {
        proxy_pass http://backend_servers;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
    }
}
EOF

# Reiniciar el servicio de Nginx
systemctl restart nginx
Configuración Nginx
Para configurar Nginx como balanceador de carga, crea el archivo de configuración en el directorio /etc/nginx/sites-available/default con el siguiente contenido:

nginx
Copiar código
upstream backend_servers {
    server 192.168.9.11;
    server 192.168.9.12;
}

server {
    listen 80;
    server_name localhost;

    location / {
        proxy_pass http://backend_servers;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
    }
}
Solución de Problemas
Si encuentras problemas con la configuración de Vagrant o alguna de las máquinas virtuales, verifica los siguientes puntos:

Vagrant no arranca las máquinas: Asegúrate de tener VirtualBox instalado y funcionando correctamente.
Problemas con la red: Verifica las configuraciones de red en el archivo Vagrantfile.
Errores de Nginx o MySQL: Revisa los logs de los servicios (/var/log/nginx/error.log, /var/log/mysql/error.log) para obtener más información sobre posibles errores.
