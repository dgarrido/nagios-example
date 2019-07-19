# Nagios

## Pasos previos

1. Levantar una instancia EC2 en AWS, con al menos 2 GB de RAM. Levantarla instalando un Ubuntu 18.04. Esta será nuestra máquina que usaremos como instancia **Servidor**.
2. Una vez levantada la instancia, conectarnos a ella por SSH.

## Instalación de requisitos

```
sudo apt-get update
```

```
sudo apt install build-essential libgd-dev openssl libssl-dev unzip apache2 php libapache2-mod-php
```

## Creación de usuario Nagios

Crear el usuario nagios:
```
sudo useradd nagios
```
Crear el grupo nagcmd
```
sudo groupadd nagcmd
```

Añadir el usuario nagios al grupo:

```
sudo usermod -a -G nagcmd nagios
```

## Descarga de Nagios Core

Las últimas releases se puede ver en:

[Thanks for Downloading Nagios Core - Nagios](https://www.nagios.org/downloads/nagios-core/thanks/?skip=1&t=1524771419)

Descargamos la última versión:

```
wget <https://assets.nagios.com/downloads/nagioscore/releases/nagios-4.4.3.tar.gz
```

## Compilación de Nagios

Descomprimir el fichero descargado:
```
tar xpf nagios-*.tar.gz
```

Entrar al directorio donde se ha descomprimido:

```
cd nagios-4.4.3/
```

Preparar el código fuente para su compilación:

```
./configure --with-nagios-group=nagios --with-command-group=nagcmd
```

Hacer el make (compilar) indicando con el parámetro `j` el número de procesadores:

```
make -j1 all
```

Instalar los componentes compilados:

```
sudo make install
sudo make install-commandmode
sudo make install-init
sudo make install-config
```

## Configurando Apache

Necesitaremos copiar la configuración del servidor Apache:

```
sudo /usr/bin/install -c -m 644 sample-config/httpd.conf /etc/apache2/sites-available/nagios.conf
```

Y añadir el usuario de Apache al grupo que creamos para Nagios:

```
sudo usermod -a -G nagcmd www-data
```

## Instalación de los plugins

Descargar el paquete:

```
cd .. && wget https://nagios-plugins.org/download/nagios-plugins-2.2.1.tar.gz
```

Descomprimir el fichero:

```
tar xpf nagios-plugins-*.tar.gz
```

Entrar al directorio descomprimido:

```
cd nagios-plugins-2.2.1
```

Preparar la compilación:

```
./configure --with-nagios-user=nagios --with-nagios-group=nagcmd --with-openssl
```

Y hacer la compilación e instalación como tal:

```
make -j1
sudo make install
```

## Configuración mínima

Editar el fichero `/usr/local/nagios/etc/nagios.cfg` y descomentar la línea que diga:

```
cfg_dir=/usr/local/nagios/etc/servers
```

Ese directorio, en realidad, no existe. Crearlo:

```
sudo mkdir /usr/local/nagios/etc/servers
```

Abrir la libreta de contactos y cambiar el email de contacto. El fichero está en `/usr/local/nagios/etc/objects/contacts.cfg`. Buscar la línea que diga:

```
email     nagios@localhost; <<***** CHANGE THIS TO YOUR EMAIL ADDRESS ******
```

## Habilitar módulos Apache

Para que Nagios pueda ser servido por Apache de forma correcta, necesitamos habilitar algunos módulos:

```
sudo a2enmod rewrite
sudo a2enmod cgi
```

Ahora, crear un usuario y contraseña para el usuario administrador de Nagios:

```
sudo htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin
```

Crear enlace simbólico a la configuración de Apache que ya copiamos:

```
sudo ln -s /etc/apache2/sites-available/nagios.conf /etc/apache2/sites-enabled/
```

Reiniciar el servicio de Apache:

```
sudo systemctl start apache2
```

## Creación del servicio

Crear el fichero `/etc/systemd/system/nagios.service` con el siguiente contenido:

```
[Unit]
Description=Nagios
BindTo=network.target

[Install]
WantedBy=multi-user.target

[Service]
Type=simple
User=nagios
Group=nagcmd
ExecStart=/usr/local/nagios/bin/nagios /usr/local/nagios/etc/nagios.cfg
```

Habilitar el servicio:

```
sudo systemctl enable /etc/systemd/system/nagios.service
```

Y arrancarlo:

```
sudo systemctl start nagios
```

## Probando el servidor de Nagios

Acceder a `http://<ip>/nagios`


----------------------------
----------------------------
----------------------------

# Instalación del cliente Nagios (NRPE)

Creamos otra instancia EC2 en AWS.

## Instalación de NRPE y plugins

Actualizar índice de paquetes:

```
sudo apt-get update
```
Instalar paquetes en sí:
```
sudo apt-get install nagios-nrpe-server nagios-plugins
```

## Configuración de NRPE

Lo primero de todo, deberemos habilitar la dirección IP del servidor Nagios. Recomendable habilitar la privada. Editamos el fichero `/etc/nagios/nrpe.cfg` y editamos:

```
allowed_hosts=127.0.0.1, <ip server Nagios>
```

Ahora, hay que reiniciar el servicio NRPE:

```
sudo /etc/init.d/nagios-nrpe-server restart
```

## Verificar que servidor y cliente se ven

Antes de proseguir, vamos a asegurarnos de que ambas máquinas se pueden ver. 

Para esto, nos conectamos al servidor Nagios y desde allí, lanzamos:

```
/usr/local/nagios/libexec/check_ping -H <ip del cliente Nagios> -w 70,85% -c 85,100%
```

## Añadir comandos de check al NRPE

Para esto, volvemos al cliente de Nagios, y editamos el fichero `/etc/nagios/nrpe.cfg`

Y añadimos, por ejemplo:

```
command[check_users]=/usr/lib/nagios/plugins/check_users -w 5 -c 10
command[check_load]=/usr/lib/nagios/plugins/check_load -w 15,10,5 -c 30,25,20
command[check_sda1]=/usr/lib/nagios/plugins/check_disk -w 20% -c 10% -p /dev/sda1
command[check_zombie_procs]=/usr/lib/nagios/plugins/check_procs -w 5 -c 10 -s Z
command[check_total_procs]=/usr/lib/nagios/plugins/check_procs -w 150 -c 200
```

# Añadir Cliente al Servidor

Ahora, volvemos a la máquina Servidor, y nos vamos al directorio:

```
cd /usr/local/nagios/etc/servers/
```

Aquí creamos el fichero `hosts.cfg`, con el contenido:

```
define host {
  use         linux-server
  host_name   webserver
  alias       Servidor web
  address     172.31.46.249
  max_check_attempts    5
  check_period      24x7
  notification_interval   30
  notification_period     24x7
}
```

Y reiniciar el servicio:

```
sudo systemctl restart nagios.service
```

Pero si vamos a mirar los servicios no están... 

Volvemos a editar el fichero `hosts.cfg` con, por ejemplo:

```
define service{
  use                     generic-service
  host_name               webserver.com
  service_description     http
  check_command           check_http
  check_period            24×7
  check_interval          5
  max_check_attempts      2
  normal_check_interval   5
  retry_check_interval    1
  contact_groups          admins
  notification_interval   15
  notification_period     24x7
  notification_options    w,c,u,r
}
```