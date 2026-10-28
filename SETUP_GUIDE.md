# Guía de preparación de equipos · Automatización de procesos con Python

| Dato | Detalle |
| :--- | :--- |
| **Curso** | Automatización de procesos con Python (PYT EFF AUTO AZ Priv) |
| **Alcance** | Prácticas 1 a 7 |
| **Dirigida a** | Equipo de operaciones que prepara los equipos de los participantes |
| **Kit** | `kit_operaciones.zip`: verificador de equipos y los paquetes de cada práctica |
| **Versión** | Octubre de 2026 |

Esta guía dice qué instalar en cada equipo, qué permisos y accesos de red hacen falta y cómo comprobar, antes del curso, que todo funciona. Al final de la preparación, cada equipo debe pasar el verificador del kit con el resultado **LISTO**.

---

## 1. Resumen: qué necesita cada equipo

| # | Componente | Versión | Prácticas | Obligatorio |
| :---: | :--- | :--- | :---: | :---: |
| 1 | Python de 64 bits | **3.14** (mínimo 3.12) | 1-7 | Sí |
| 2 | Visual Studio Code + extensión **Python** (`ms-python.python`) | Actual | 1-7 | Sí |
| 3 | Git (en Windows, **Git for Windows**, que incluye Git Bash) | 2.40 o superior | 6, 7 | Sí |
| 4 | Docker Desktop (Windows, macOS) o Docker Engine (Linux) | Actual (Docker 25 o superior) | 6 | Sí |
| 5 | WSL 2 con Ubuntu 24.04 (solo Windows) | Actual | 6 | Sí en Windows (lo usa Docker Desktop) |
| 6 | Asistente de IA: **Claude Code** o **GitHub Copilot** (el que elija el instructor) | Actual | 7 | Sí, uno de los dos |
| 7 | Imágenes precargadas `python:3.14-slim` y `docker/dockerfile:1` | — | 6 | Recomendado |
| 8 | Caché de pip con los paquetes de las 7 prácticas | — | 1-7 | Recomendado |

No se necesitan bases de datos, cuentas de AWS ni de Snowflake: todas las prácticas usan emuladores o servicios simulados en el propio equipo.

### Qué usa cada práctica

| Práctica | Duración | Software | Red durante la práctica | Notas |
| :--- | :---: | :--- | :--- | :--- |
| 1. Automatización con Python | 90 min | Python, VS Code | PyPI | El participante genera sus versiones con `pip-compile` |
| 2. Calidad y contratos de datos | 120 min | Python, VS Code | PyPI | Incluye `pyarrow` (macOS 12 o superior) |
| 3. Integración con API y Snowflake | 120 min | Python, VS Code, **dos terminales** | PyPI | Emulador local de Snowflake; puerto **8000** (alterno 8010); macOS **15** o superior |
| 4. Monitoreo y remediación | 90 min | Python, VS Code | PyPI | Sin AWS |
| 5. Pruebas y escalamiento | 120 min | Python 3.12+, VS Code, **dos terminales** | PyPI | Puerto **8000** (alterno 8010) |
| 6. Despliegue y orquestación | 180 min | Python 3.12+, VS Code, Git, **Docker** | PyPI, Docker Hub, GitHub | Parte opcional con Airflow: Linux, macOS o WSL 2. GitHub opcional |
| 7. Productividad con IA | 120 min | Python 3.12+, VS Code, Git, **Claude Code o Copilot** | PyPI, servicio de IA | Una cuenta de IA **por participante** |

---

## 2. Hardware y sistema operativo

| Elemento | Mínimo | Recomendado |
| :--- | :--- | :--- |
| Sistema operativo | Windows 10 22H2 o Windows 11 (x64) · macOS 15 Sequoia o superior (Apple Silicon o Intel) · Ubuntu 22.04 o 24.04 (x64) | Windows 11 · macOS 15 o superior · Ubuntu 24.04 |
| Procesador | 4 núcleos, 64 bits, **virtualización activa en BIOS/UEFI** (Intel VT-x o AMD-V) | 8 núcleos |
| Memoria RAM | 8 GB | 16 GB (Docker Desktop y VS Code abiertos al mismo tiempo) |
| Disco libre | 10 GB | 20 GB en la unidad del perfil del usuario (SSD) |
| Pantalla | 1366 × 768 | 1920 × 1080: las prácticas 3 y 5 usan dos terminales lado a lado |

> **Por qué macOS 15.** El conector oficial de Snowflake que usa la Práctica 3 no publica paquetes para macOS 14 o anterior. En equipos con macOS 14, la Práctica 3 no se puede instalar.

> **Windows en ARM** (por ejemplo, Surface con Snapdragon) no está probado. Use equipos x64.

---

## 3. Cuentas y licencias

| Cuenta | Para qué | Quién la aporta | Obligatoria |
| :--- | :--- | :--- | :---: |
| **GitHub** (una por participante) | Copilot en la Práctica 7 (si el grupo usa Copilot) y, de forma opcional, subir el repositorio en la Práctica 6 | El participante, o la organización | Solo si se usa Copilot |
| **GitHub Copilot** | Práctica 7 | El plan **Copilot Free** sirve (tiene límites mensuales); para grupos grandes conviene una licencia Business de la organización | Solo si se usa Copilot |
| **Claude** (Pro, Max, Team o Enterprise, o Claude Console) | Claude Code en la Práctica 7 | El participante o la organización. El plan gratuito de claude.ai **no** incluye Claude Code | Solo si se usa Claude Code |
| **Docker Hub** | No es necesaria, pero evita el límite de descargas anónimas (ver sección 5.2) | Cuenta gratuita | No |

> **Confirme con el instructor**, al menos una semana antes, cuál asistente de IA usará el grupo en la Práctica 7. Las cuentas de IA son **personales**: no se comparten entre participantes ni se dejan abiertas en el equipo al terminar el curso (sección 9).

### Licencia de Docker Desktop

Docker Desktop es gratuito para uso personal, educativo, proyectos de código abierto no comerciales y empresas con **menos de 250 empleados y menos de 10 millones de dólares** de ingresos anuales. Fuera de esos casos requiere una suscripción de pago. Si su organización no califica, use una suscripción de Docker o, en Linux, **Docker Engine**, que es de código abierto.

---

## 4. Instalación por sistema operativo

Los comandos usan los gestores de paquetes oficiales. Si su organización instala software con Intune, SCCM, Jamf u otra herramienta, use los mismos productos y versiones.

### 4.1 Windows 10/11

Ejecute en **PowerShell como administrador**.

**1. WSL 2 y Ubuntu** (requiere reiniciar):

```powershell
wsl --install -d Ubuntu-24.04
```

Después del reinicio, abra **Ubuntu** una vez para crear el usuario de Linux.

**2. Python 3.14 para todos los usuarios, en el PATH y con el lanzador `py`:**

```powershell
winget install -e --id Python.Python.3.14 --scope machine --override "/quiet InstallAllUsers=1 PrependPath=1 Include_launcher=1 Include_test=0"
```

> Alternativa: el instalador de python.org (marque **Add python.exe to PATH**) o el *Python install manager* (`winget install -e --id Python.PythonInstallManager` y luego `py install 3.14`). Cualquiera sirve si, en una terminal nueva, `python --version` responde `Python 3.14.x`.

**3. Git for Windows, VS Code y Docker Desktop:**

```powershell
winget install -e --id Git.Git
winget install -e --id Microsoft.VisualStudioCode --scope machine
winget install -e --id Docker.DockerDesktop --override "install --quiet --accept-license --backend=wsl-2"
```

**4. Permisos para Docker.** Agregue la cuenta de cada participante al grupo `docker-users` (sustituya el nombre de usuario) y cierre e inicie sesión de nuevo:

```powershell
net localgroup docker-users "NOMBRE_DE_USUARIO" /add
```

**5. Activación de entornos virtuales en PowerShell.** Las prácticas activan `.venv\Scripts\Activate.ps1`. Si la política del equipo lo bloquea, permita scripts locales para el usuario:

```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```

(o aplique la política equivalente por GPO).

**6. Asistente de IA para la Práctica 7** (el que elija el instructor). Como el **usuario del participante**, no como administrador:

```powershell
# Claude Code (no requiere administrador)
irm https://claude.ai/install.ps1 | iex
# o bien, administrado por winget:  winget install -e --id Anthropic.ClaudeCode
```

Copilot se instala en VS Code (paso 4.4).

### 4.2 macOS 15 o superior

1. **Python 3.14:** instalador oficial `.pkg` de python.org (universal2). Al terminar, ejecute **una vez** `Install Certificates.command`, en `/Applications/Python 3.14/`. Sin ese paso, los programas que usan `urllib` (el verificador del kit y un ejercicio de la Práctica 7) no pueden validar certificados HTTPS.
2. **Git:** `xcode-select --install` (o `brew install git`).
3. **VS Code:** `brew install --cask visual-studio-code` (o descarga de code.visualstudio.com). Luego, en VS Code: **Command Palette → Shell Command: Install 'code' command in PATH**.
4. **Docker Desktop:** `brew install --cask docker-desktop` (o descarga de docker.com). Ábralo una vez para aceptar la licencia y conceder permisos.
5. **Claude Code** (si aplica), como el usuario del participante: `curl -fsSL https://claude.ai/install.sh | bash` (o `brew install --cask claude-code`).

### 4.3 Ubuntu 22.04 / 24.04

```bash
# Python 3.14 (Ubuntu trae 3.12; 3.14 se instala desde el PPA deadsnakes)
sudo apt update
sudo apt install -y software-properties-common git curl
sudo add-apt-repository -y ppa:deadsnakes/ppa
sudo apt install -y python3.14 python3.14-venv

# Docker Engine (repositorio oficial de Docker) y permisos para el participante
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker NOMBRE_DE_USUARIO     # cerrar e iniciar sesión después

# VS Code: paquete .deb de code.visualstudio.com o:  sudo snap install code --classic
```

> En Ubuntu, el participante crea entornos con `python3.14 -m venv .venv`. Dentro del entorno activo, `python` ya apunta a 3.14.

### 4.4 Extensiones de VS Code (todos los sistemas)

Como el usuario del participante:

```bash
code --install-extension ms-python.python
code --install-extension GitHub.copilot-chat     # solo si el grupo usa Copilot en la Práctica 7
```

Para Copilot, el participante inicia sesión con **su** cuenta de GitHub durante la Práctica 7 (icono de Copilot en la barra de estado → *Use AI Features*). Sin un plan de pago, su cuenta queda en Copilot Free.

### 4.5 Parte opcional de Airflow (Práctica 6)

La Práctica 6 incluye una sección opcional con Airflow 3.3.2 que **no corre de forma nativa en Windows**: en Windows se usa dentro de WSL (Ubuntu). Si el instructor la va a hacer, preinstale Airflow en Ubuntu/WSL (algunos minutos por equipo y unos 300 MB):

```bash
sudo apt install -y python3-venv          # Ubuntu 24.04 trae Python 3.12, que Airflow 3.3.2 soporta
python3 -m venv ~/venv-airflow
~/venv-airflow/bin/python -m pip install --upgrade pip
~/venv-airflow/bin/python -m pip install "apache-airflow==3.3.2" \
  --constraint https://raw.githubusercontent.com/apache/airflow/constraints-3.3.2/constraints-3.12.txt
~/venv-airflow/bin/airflow version        # debe responder 3.3.2
```

En Docker Desktop, active **Settings → Resources → WSL integration** para la distribución Ubuntu: así `docker` funciona también dentro de WSL.

---

## 5. Red

### 5.1 Sitios que deben estar permitidos (HTTPS, puerto 443)

| Sitio | Para qué | Prácticas |
| :--- | :--- | :---: |
| `pypi.org`, `files.pythonhosted.org` | Instalar paquetes con pip; la Práctica 7 consulta además la API JSON de `pypi.org` | 1-7 |
| `registry-1.docker.io`, `auth.docker.io`, `production.cloudflare.docker.com` | Descargar `python:3.14-slim` y `docker/dockerfile:1` | 6 |
| `github.com`, `release-assets.githubusercontent.com`, `objects.githubusercontent.com` | `actionlint-py` descarga su binario de GitHub Releases al instalarse; subir el repositorio (opcional) | 6 |
| `raw.githubusercontent.com` | Archivo de restricciones de Airflow (parte opcional) | 6 |
| `marketplace.visualstudio.com`, `*.vscode-cdn.net`, `update.code.visualstudio.com` | Extensiones y actualizaciones de VS Code | 1-7 |
| **Claude Code:** `api.anthropic.com`, `claude.ai`, `claude.com`, `platform.claude.com`, `downloads.claude.ai` | Modelo, inicio de sesión e instalador o actualizaciones | 7 |
| **GitHub Copilot:** `github.com/login`, `api.github.com`, `copilot-proxy.githubusercontent.com`, `origin-tracker.githubusercontent.com`, `*.individual.githubcopilot.com` (o `*.business.githubcopilot.com` con licencia Business) | Inicio de sesión y sugerencias | 7 |

Las listas completas y actualizadas de los asistentes están en la documentación de cada proveedor: *Network access requirements* de Claude Code (code.claude.com) y *Copilot allowlist reference* (docs.github.com).

### 5.2 Límite de descargas de Docker Hub

Docker Hub permite **100 descargas cada 6 horas por dirección IP** sin iniciar sesión (200 con una cuenta gratuita). Un salón completo detrás de una sola IP pública puede alcanzar el límite en la Práctica 6. Para evitarlo:

1. **Precargue** las imágenes en cada equipo durante la preparación (sección 6.3). Con la imagen ya presente, la práctica no vuelve a descargarla.
2. O configure un *mirror* o proxy de registro de la organización.
3. O pida a los participantes que inicien sesión con una cuenta gratuita (`docker login`).

### 5.3 Proxy corporativo e inspección TLS

Si la red usa un proxy o inspecciona TLS con un certificado propio, configúrelo para cada herramienta (el verificador lo detecta como *certificado no confiable*):

| Herramienta | Configuración |
| :--- | :--- |
| pip | `python -m pip config set global.proxy http://proxy:puerto` y `python -m pip config set global.cert C:\ruta\ca-corporativa.pem` |
| Python (`requests`, `urllib`) | Variables `HTTPS_PROXY`, `NO_PROXY=localhost,127.0.0.1` y `REQUESTS_CA_BUNDLE` / `SSL_CERT_FILE` con la CA corporativa |
| Git | `git config --global http.proxy http://proxy:puerto` |
| Docker Desktop | **Settings → Resources → Proxies**; la CA corporativa en el almacén del sistema |
| VS Code | Ajuste `http.proxy`, o el proxy del sistema |
| Claude Code | Variables `HTTPS_PROXY` y `NODE_EXTRA_CA_CERTS` (también en `~/.claude/settings.json`). No admite proxies SOCKS |

> **Importante:** `NO_PROXY` debe incluir `localhost` y `127.0.0.1`. Las prácticas 3, 5 y 7 hablan con servicios simulados en `127.0.0.1`; si esas llamadas pasan por el proxy, fallan.

### 5.4 Puertos locales

Las prácticas 3 y 5 levantan una API simulada en `127.0.0.1:8000` (alterno: `8010`); la Práctica 7, un webhook en un puerto libre que elige el sistema. Todo es local (no se abren puertos hacia la red), pero:

- Ningún otro programa debe ocupar el puerto **8000** (servidores de desarrollo, agentes de software). El verificador lo comprueba.
- Si el firewall de Windows pregunta por `python.exe` la primera vez, basta con **Redes privadas**; también puede preautorizarse por política.

---

## 6. Preparación con el kit

### 6.1 Contenido del kit

```text
kit_operaciones/
├── verificar_equipo.py          verificador del equipo (solo biblioteca estándar)
└── requisitos/
    ├── practica1/ ... practica7/
    │   ├── requirements.txt      paquetes de ejecución de cada práctica
    │   └── requirements-dev.txt  herramientas de desarrollo y pruebas
```

Son exactamente las versiones de las guías de laboratorio (en las prácticas 1 y 2, el participante las genera en clase; las del kit son una referencia para precargar la caché).

### 6.2 Precargar los paquetes de Python (recomendado)

Copie `kit_operaciones` al equipo y, **como el usuario del participante** (la caché de pip es por usuario), ejecute:

**Windows**
```powershell
cd kit_operaciones
python verificar_equipo.py --instalar
```

**macOS / Linux**
```bash
cd kit_operaciones
python3.14 verificar_equipo.py --instalar
```

`--instalar` crea un entorno temporal por práctica, instala sus paquetes y lo borra. Así comprueba que **todas** las prácticas instalan en ese equipo y deja los paquetes en la caché de pip, de modo que en clase la instalación tarda segundos aunque la red esté saturada. Toma de 2 a 5 minutos por equipo (unos 2 minutos en el equipo de referencia).

### 6.3 Precargar las imágenes de Docker (recomendado)

```bash
docker pull python:3.14-slim
docker pull docker/dockerfile:1
```

O, con el verificador: `python verificar_equipo.py --docker` (descarga y ejecuta `python:3.14-slim`).

### 6.4 Verificación final de cada equipo

Con el usuario del participante y Docker Desktop abierto:

```bash
python verificar_equipo.py --docker
```

Ejemplo ilustrativo de un equipo Windows listo (versiones, rutas y tamaños cambian por equipo):

```text
Verificación del equipo AULA-PC07 · 2026-11-03 08:10

== Sistema
  ✓ sistema operativo                            Windows 11 (AMD64)
  ✓ memoria RAM                                  15.8 GB (mínimo 8, recomendado 16)
  ✓ disco libre (carpeta de usuario)             112 GB (mínimo 10, recomendado 20)

== Python
  ✓ versión de Python                            3.14.7 en C:\Program Files\Python314\python.exe
  ✓ entornos virtuales (venv + pip)              se crean sin errores
  ✓ pip                                          pip 26.1.2  [prácticas 1-7]

== Herramientas
  ✓ Git                                          git version 2.51.0.windows.1
  ! identidad de Git                             sin user.name/user.email (se configura en la práctica)  [prácticas 6-7]
  ✓ VS Code                                      1.105.1
  ✓ extensión Python (ms-python.python)          instalada  [prácticas 1-7]
  ...
== Docker (Práctica 6)
  ✓ Docker (servidor)                            29.0.1
  ✓ imagen python:3.14-slim                      ya descargada
  ✓ docker run python:3.14-slim                  Python 3.14.7  [prácticas 6]
...
LISTO: 0 fallas, 3 avisos. Reporte: reporte_equipo_AULA-PC07.json
```

| Marca | Significado | Acción |
| :---: | :--- | :--- |
| ✓ | Correcto | — |
| ! | **AVISO**: recomendado, o solo necesario en un caso (por ejemplo, el asistente de IA que el grupo no usa) | Revise si aplica al grupo |
| ✗ | **FALLA**: bloquea al menos una práctica (indicada entre corchetes) | Corrija y vuelva a ejecutar |

El código de salida es 0 si el equipo está **LISTO** y 1 si hay fallas. El archivo `reporte_equipo_<equipo>.json` sirve para consolidar el estado de todo el salón.

**Avisos esperados y aceptables:** *identidad de Git* (cada participante la configura en la Práctica 6), *extensión GitHub Copilot Chat* o *Claude Code* (si el grupo usa el otro asistente) y *WSL 2* en equipos que no harán la parte opcional de Airflow, siempre que Docker funcione.

---

## 7. Lista de comprobación por equipo

- [ ] Virtualización activa en BIOS/UEFI; en Windows, WSL 2 con Ubuntu inicializado
- [ ] Python 3.14 de 64 bits en el PATH (`python --version` en una terminal nueva)
- [ ] macOS: `Install Certificates.command` ejecutado
- [ ] VS Code con la extensión Python; comando `code` en el PATH
- [ ] Git (Git for Windows en Windows)
- [ ] Docker Desktop instalado, licencia aceptada, participante en `docker-users` (Windows) o `docker` (Linux)
- [ ] PowerShell permite activar entornos (`RemoteSigned` para el usuario)
- [ ] Asistente de IA de la Práctica 7 instalado (Claude Code o la extensión de Copilot)
- [ ] Sitios de la sección 5.1 permitidos; proxy y CA configurados si aplica
- [ ] Puerto 8000 libre
- [ ] `python verificar_equipo.py --instalar` sin fallas (precarga de pip)
- [ ] `python verificar_equipo.py --docker` en **LISTO** (precarga de imágenes)
- [ ] Opcional: Airflow 3.3.2 en `~/venv-airflow` dentro de Ubuntu/WSL

---

## 8. Equipo del instructor

Además de todo lo anterior:

- Las guías de laboratorio y la guía del instructor de la Práctica 7, que trae los *checkpoints* de solución para quien se atore.
- El asistente de IA del grupo con una cuenta propia, para ensayar los prompts de la Práctica 7 antes de la clase.
- Si hará la parte opcional de la Práctica 6, Airflow 3.3.2 preinstalado (sección 4.5).
- Proyector con resolución de al menos 1920 × 1080: las salidas de terminal son anchas.

---

## 9. Al terminar cada curso (limpieza entre grupos)

Para que el siguiente grupo empiece con un equipo limpio y sin datos de los participantes anteriores:

1. Borrar las carpetas de las prácticas (`practica1_compras` … `practica7_ia`) y los archivos `.env`.
2. **Cerrar sesión** en las cuentas personales: GitHub en VS Code (**Accounts → Sign out**), Copilot, Claude Code (`claude` y luego `/logout`, o borrar la carpeta `~/.claude`) y Docker Hub (`docker logout`).
3. Quitar la identidad de Git: `git config --global --unset user.name` y `git config --global --unset user.email`.
4. Docker: `docker volume rm cargaventas-salida` y `docker image rm cargaventas:1.0.0`. **Conserve** `python:3.14-slim` para el siguiente grupo.
5. Opcional: borrar `~/venv-airflow` y `airflow_home` si se usó Airflow.

No borre la caché de pip ni las imágenes base: aceleran el siguiente curso.

---

## 10. Problemas frecuentes

| Síntoma | Causa probable | Solución |
| :--- | :--- | :--- |
| `python` abre la Microsoft Store o "no se reconoce" | Python no está en el PATH; alias de la Store activos | Reinstale con `PrependPath=1`; desactive **Configuración → Aplicaciones → Alias de ejecución** para `python.exe` |
| `python --version` muestra 3.11 o anterior | Otra versión queda primero en el PATH | Desinstálela o ajuste el orden del PATH |
| ✗ *entornos virtuales* en Ubuntu | Falta el paquete de venv | `sudo apt install python3.14-venv` |
| ✗ *certificado no confiable* | Proxy con inspección TLS, o en macOS falta `Install Certificates.command` | Sección 5.3; en macOS, ejecute el comando de la sección 4.2 |
| ✗ *Docker: sin servidor* | Docker Desktop cerrado, licencia sin aceptar o usuario fuera de `docker-users` | Abra Docker Desktop; agregue el usuario al grupo y vuelva a iniciar sesión |
| Docker Desktop: *WSL 2 installation is incomplete* o *Virtualization not enabled* | WSL 2 sin instalar o virtualización apagada en BIOS | `wsl --install`, reinicio; active VT-x/AMD-V en BIOS |
| `toomanyrequests: You have reached your pull rate limit` | Límite de Docker Hub por IP | Sección 5.2: precarga, mirror o `docker login` |
| ✗ *127.0.0.1:8000 ocupado* | Otro programa usa el puerto | Identifíquelo (`netstat -ano \| findstr :8000` en Windows, `lsof -i :8000` en macOS/Linux) y deténgalo; las prácticas también aceptan el 8010 |
| `.venv\Scripts\Activate.ps1 cannot be loaded` | Política de ejecución de PowerShell | `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` |
| ✗ *paquetes de la Práctica N* | Sin acceso a PyPI o un paquete sin versión para ese sistema | Revise la red (5.1); en macOS, que sea 15 o superior |
| La instalación de la Práctica 6 se queda en `actionlint-py` | No puede descargar el binario de GitHub Releases | Permita `github.com` y `*.githubusercontent.com` (5.1) |
| Claude Code: *Unable to connect to Anthropic services* | Dominios de Anthropic bloqueados o proxy sin configurar | Sección 5.1 y variables de la 5.3 |
| Copilot no responde o pide iniciar sesión siempre | Dominios de GitHub Copilot bloqueados | Sección 5.1 |

---

## 11. Qué se probó

- Las siete prácticas, completas, en Linux con Python 3.12, 3.13 y 3.14. Las prácticas 1 y 2 se revalidaron con Python 3.14 para esta guía.
- Que todas las dependencias publiquen paquetes para Windows x64, macOS (Apple Silicon e Intel) y Linux x64. El límite de macOS 15 lo impone la Práctica 3.
- Airflow 3.3.2 en Python 3.12, 3.13 y 3.14.
- El verificador del kit, en Linux con Python 3.12, 3.13 y 3.14.

**No se ejecutó en equipos Windows ni macOS reales.** Antes del primer curso, conviene preparar **un equipo piloto** de cada sistema con esta guía, pasar el verificador y hacer las prácticas 3, 6 y 7 completas (las más exigentes en red, Docker e IA).

