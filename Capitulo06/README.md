# Práctica 6. De automatización local a solución desplegable y orquestada

| Dato | Detalle |
| :--- | :--- |
| **Curso** | Automatización de procesos con Python |
| **Capítulo** | 6. Programación de ejecuciones, orquestación, seguridad y despliegue |
| **Duración** | 180 minutos (más una parte opcional con Airflow local) |
| **Modalidad** | Individual |
| **Nivel de Bloom** | Aplicar / Crear |

---

## Objetivo

Preparar una automatización Python para su ejecución reproducible:

- **Empaquetarla** como paquete interno con CLI, versión única, lockfile con hashes y un wheel verificado.
- **Contenerizarla con Docker** en una imagen multi-stage que corre sin root y recibe sus secretos desde fuera.
- **Configurar CI/CD con GitHub Actions** con pruebas, calidad (Ruff, mypy, cobertura), seguridad (pip-audit, actionlint) y una entrega controlada con prueba de humo y reversión.
- **Definir un DAG para Amazon MWAA** que coordine las tareas, sus dependencias, sus estados y el manejo de fallos con base en el código de salida.

No necesitas cuenta de AWS: todo se ejecuta en tu equipo. La parte de la nube (ECR, ECS Fargate, MWAA) queda escrita, validada y lista para conectarse.

## Lo que vas a construir

```text
 datos/ventas_AAAA-MM-DD.csv  (sintético: 10% de filas inválidas, columnas nombre_cliente y rfc de más)
        │
        ▼
 cargaventas cargar --fecha 2026-10-26        (CLI con click; mismo comando en todos lados)
   config.toml + variables de entorno ─► Pydantic estricto ─► falla rápido (código 3)
   CLAVE_PSEUDONIMOS ─────────────────► solo del entorno; nunca en logs (redacción)
   ruta dentro de datos/ ─────────────► si sale de la carpeta, error de uso (código 2)
   cada fila ─► VentaEntrada (extra="forbid") ─► >20% inválidas: lote rechazado (código 10)
   minimización: sin nombre ni RFC; correo -> seudónimo HMAC
   SQLite: UPSERT en UNA transacción ─► si falla, rollback (código 20, reintentable)
   stdout: JSON de métricas (última línea)   stderr: logs con versión y commit
        │
        ├── wheel cargaventas-1.0.0  ─► twine check + verificador + SHA-256 + pip-audit
        ├── imagen cargaventas:1.0.0 ─► multi-stage, uid 10001, etiquetas OCI, --read-only
        │
        ├── GitHub Actions
        │     ci.yml ──────── calidad (3.12, 3.13, 3.14) · pip-audit · wheel · imagen + humo
        │     entrega.yml ─── etiqueta v1.0.0 ─► ECR (una sola vez) ─► ECS + humo ─► DAG a MWAA
        │                     └─ si algo falla: revierte la task definition y el DAG
        │
        └── DAG carga_ventas (Airflow 3 / MWAA)
              validar (dry-run) ─► cargar ─► publicar_resumen
                    └──────┴──── one_failed ─► alertar          cerrar (teardown: siempre)
              código 0: success · 20: up_for_retry · 10, 3, 2, 1: failed sin reintentos
```

### Relación con el capítulo

| Componente del laboratorio | Lección del capítulo |
| :--- | :--- |
| `schedule="30 2 * * 1-5"`, `catchup=False`, `max_active_runs=1` | 6.1 Ejecución programada |
| Dependencias `validar >> cargar >> publicar_resumen` | 6.3 Dependencias entre flujos |
| `contrato.py`: códigos 0/1/2/3/10/20 (sin 99), logs a stderr, JSON en stdout, `XCOM_FILE` | 6.4 Integración con plataformas de orquestación |
| Estados `success`, `failed`, `up_for_retry`, `upstream_failed`, `skipped` | 6.5 Estados de ejecución |
| `--dry-run`, `execution_timeout`, contenedor con `--rm` y nombre único | 6.6 Controles operativos |
| `retries` con espera exponencial, `AirflowFailException`, `one_failed`, `teardown` | 6.7 Dependencias y fallos |
| Variable JSON, rol de ejecución, DAG en S3, ECS Fargate con `boto3` | 6.8 Amazon MWAA |
| `.venv`, `requirements.in` y lockfile con hashes | 6.10 Entornos, 6.11 Paquetes |
| `pyproject.toml`, `src/`, `[project.scripts]`, clasificador `Private` | 6.12 Paquetes internos |
| CLI con `click`: grupo, `--version`, `BadParameter`, códigos de salida | 6.13 CLI |
| SemVer en un solo lugar; versión y commit en cada línea de log | 6.14 Gestión de versiones |
| `verificar_wheel.py`: archivos prohibidos, versión y comando | 6.15 Preparación para despliegue |
| `Dockerfile` multi-stage, `.dockerignore`, uid 10001, etiquetas OCI | 6.16 Docker |
| `config.toml` versionado, entorno que sobrescribe, secretos solo del entorno | 6.17 Configuración y secretos |
| Ejecutor ECS Fargate en el DAG; rol en lugar de llaves | 6.18 Entornos administrados, 6.20 AWS |
| `git init`, etiquetas anotadas, `.gitignore`, `.gitattributes` | 6.22 Control de versiones |
| `ci.yml`: Ruff, mypy, pytest con umbral de cobertura, matriz de Python | 6.23, 6.24 CI y calidad |
| `build_package.sh`: `python -m build`, `twine check --strict`, huellas SHA-256 | 6.25 Empaquetado |
| `pip-audit` sobre el lockfile con hashes | 6.26 Verificación de dependencias, 6.35 Análisis de dependencias |
| La imagen se construye una vez y se promueve con la etiqueta `versión-SHA` | 6.27 Construcción de artefactos |
| `environment: produccion` con revisores | 6.28 Entrega controlada |
| Nueva revisión de la task definition, prueba de humo, reversión automática | 6.29 Despliegue |
| Etiqueta → imagen → revisión de ECS → logs; reentrega manual con motivo | 6.30 Trazabilidad y reversión |
| OIDC, `permissions` mínimos, entradas vía `env` (sin inyección), actionlint | 6.32, 6.33 GitHub Actions y secretos |
| `VentaEntrada` con `extra="forbid"` y patrones; ruta segura con `resolve()` | 6.34 Validación de entradas |
| Roles separados para publicar, desplegar y ejecutar | 6.36, 6.37 Control de acceso y mínimo privilegio |
| `RedactorSecretos` en el handler; nunca valores en los mensajes de error | 6.38 Datos sensibles en registros |
| Columnas descartadas y correo convertido en seudónimo HMAC | 6.39 Minimización de datos |

> Fuera del alcance de esta práctica (tienen demo propia en el capítulo): eventos con watchdog (6.2), systemd y APScheduler (6.1), Lambda (6.19), canario por porcentajes (6.28), integración con dbt o Fivetran (6.9), cliente seguro de API (6.40) y cifrado con Fernet (6.41).

## Criterios de evaluación

| Criterio | Evidencia en tu entrega |
| :--- | :--- |
| **Corrección** | El componente empaquetado se ejecuta igual con la CLI, en el contenedor y desde el ejecutor del DAG: 45 filas cargadas de 50, mismo total; la carga repetida no duplica |
| **Resiliencia** | Entradas validadas (códigos 2 y 10); un destino bloqueado devuelve 20 y se reintenta; la transacción se revierte completa; `entrega.yml` revierte la task definition y el DAG si la prueba de humo falla |
| **Mantenibilidad** | Una sola versión (`pyproject.toml`), lockfile con hashes, configuración versionada y separada del código, wheel verificado, imagen etiquetada con versión y commit |
| **Seguridad** | Secretos fuera del código y de la imagen; contenedor sin root y de solo lectura; `permissions` mínimos y OIDC; pip-audit y actionlint sin hallazgos; logs sin secretos y datos personales minimizados |
| **Valor operativo** | El mismo flujo, repetible y controlado: CI en cada cambio, entrega por etiqueta con aprobación, reentrega manual con motivo, y un DAG cuyos estados explican qué pasó |

---

## Requisitos

| Elemento | Requisito |
| :--- | :--- |
| Python | **3.12 o superior** (probado con 3.12, 3.13 y 3.14) |
| Docker | Docker Desktop (Windows o macOS) o Docker Engine (Linux). No se usa Docker Compose |
| Git | Cualquier versión reciente (`git --version`) |
| Sistema operativo | Windows 10/11, macOS o Linux. La parte opcional de Airflow requiere Linux, macOS o WSL2 |
| Editor | Visual Studio Code con la extensión de Python |
| Terminal | Terminal integrada de VS Code (PowerShell en Windows; bash/zsh en macOS o Linux) |
| Red | Acceso a PyPI y a Docker Hub (para la imagen base `python:3.14-slim`) |
| Cuentas | Ninguna obligatoria. GitHub es opcional (Parte 6.6); AWS no se usa |

> **Docker Desktop y su licencia.** Es gratuito para uso personal, educativo, proyectos de código abierto no comerciales y empresas con menos de 250 empleados **y** menos de 10 millones de dólares de ingresos anuales. En organizaciones más grandes requiere una suscripción de pago. Docker Engine en Linux es de código abierto.

> **Convenciones**
> - Todos los comandos se ejecutan desde la **carpeta raíz del proyecto** (`practica6_despliegue`) con el entorno virtual activo.
> - Cuando un comando cambia según el sistema operativo, verás la versión **PowerShell** y la versión **bash**.
> - Para ver el código de salida del último comando: `$LASTEXITCODE` en PowerShell, `echo $?` en bash.
> - Las fechas y horas de los logs cambian; los conteos, los códigos y los totales, no.

### Distribución del tiempo

| Parte | Contenido | Minutos |
| :--- | :--- | :---: |
| 0. Preparación | Docker, estructura, lockfile con hashes, entorno y Git | 15 |
| 1. Paquete y CLI | Contrato con el orquestador, configuración, secretos, validación y minimización | 30 |
| 2. Contenedor | Imagen multi-stage, ejecución sin root, volúmenes y códigos de salida | 35 |
| 3. DAG para MWAA | Ejecutores, contrato y el DAG con reintentos, `one_failed` y `teardown` | 25 |
| 4. Pruebas y calidad | Suite completa, prueba de contrato DAG-paquete, Ruff, mypy y cobertura | 25 |
| 5. Artefacto | Wheel, verificación, wheel manipulado y auditoría de dependencias | 15 |
| 6. CI/CD | `ci.yml`, `entrega.yml`, actionlint y mínimo privilegio | 25 |
| Cierre | Preguntas y entregables | 10 |
| 7. Opcional | Airflow 3 local: éxito, fallo sin reintento y reintento que se recupera | +30 |

> **Checkpoints.** Al final de las partes 1 a 6 hay un punto de control. Si te atrasas, el instructor puede darte la carpeta del checkpoint.

---

## Parte 0. Preparación (15 min)

**0.1** Comprueba Docker y descarga la imagen base (tarda un poco la primera vez, así no esperas en la Parte 2):

```bash
docker version --format "cliente {{.Client.Version}} / servidor {{.Server.Version}}"
docker pull python:3.14-slim
git --version
```

Resultado esperado: las versiones del cliente y del servidor (si falta el servidor, abre Docker Desktop) y `Status: Downloaded newer image for python:3.14-slim` (o `Image is up to date`).

**0.2** Crea la carpeta `practica6_despliegue`, ábrela en VS Code (**File → Open Folder**) y crea la estructura:

**PowerShell**
```powershell
New-Item -ItemType Directory -Force -Path src/cargaventas, tests, dags, scripts, .github/workflows
```

**bash**
```bash
mkdir -p src/cargaventas tests dags scripts .github/workflows
```

**0.3** Crea `pyproject.toml`. Es la única fuente de la versión (6.14) y la configuración de pytest, cobertura, Ruff y mypy vive aquí (6.12): el mismo umbral en local y en CI.

**Archivo:** `pyproject.toml`

```toml
[build-system]
requires = ["setuptools>=77"]
build-backend = "setuptools.build_meta"

[project]
name = "cargaventas"
version = "1.0.0"                 # única fuente de la versión (SemVer)
description = "Carga diaria de ventas: validación, minimización y carga idempotente"
readme = "README.md"
requires-python = ">=3.12"
license = "LicenseRef-Proprietary"
authors = [{ name = "Equipo de Automatización" }]
classifiers = ["Private :: Do Not Upload to PyPI"]   # PyPI rechaza la subida
dependencies = [
    "click>=8.2,<9",
    "pydantic>=2.12,<3",
]

[project.scripts]
cargaventas = "cargaventas.cli:main"   # comando que queda en el PATH del entorno

[tool.setuptools.packages.find]
where = ["src"]

[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = [".", "dags"]       # dags/: el DAG importa sus ejecutores desde ahí
addopts = "-ra --strict-markers --strict-config"

[tool.coverage.run]
branch = true
source = ["cargaventas", "cargaventas_ejecutores"]

[tool.coverage.report]
fail_under = 85                   # el mismo umbral en local y en CI
show_missing = true

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "S"]   # S: reglas de seguridad (bandit)

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101", "S106"]     # assert y credenciales falsas en pruebas

[tool.ruff.lint.isort]
known-first-party = ["cargaventas", "cargaventas_ejecutores"]

[tool.mypy]
strict = true
files = ["src", "dags/cargaventas_ejecutores.py"]   # el DAG necesita Airflow: lo valida airflow

[[tool.mypy.overrides]]
module = ["boto3"]
ignore_missing_imports = true
```

**0.4** Crea los archivos de dependencias. `requirements.in` tiene las dependencias directas con rangos; `requirements.txt` es el lockfile generado con `pip-compile --generate-hashes`: cada paquete con su versión exacta y sus hashes SHA-256 (6.11). Se entregan resueltos para que todos instalen exactamente lo que se probó; el lockfile **se regenera, no se edita**.

**Archivo:** `requirements.in`

```text
# Dependencias directas de ejecución (las mismas de pyproject.toml).
# Lockfile con hashes:
#   pip-compile --generate-hashes --strip-extras --no-annotate -o requirements.txt requirements.in
click>=8.2,<9
pydantic>=2.12,<3
# click la usa en Windows; se fija sin marcador para que el MISMO lockfile con
# hashes sirva en Windows, macOS, Linux y la imagen (en Linux no se usa)
colorama>=0.4
```

**Archivo:** `requirements.txt`

```text
# Versiones fijadas para la Práctica 6 (hay ruedas para Windows, macOS 10.15+ y Linux; Python 3.12-3.14).
# Lockfile con hashes: pip install -r requirements.txt los verifica todos.
# Generado con:
#   pip-compile --generate-hashes --strip-extras --no-annotate -o requirements.txt requirements.in
annotated-types==0.8.0 \
    --hash=sha256:13b2beaad985e05e2d6407ee4c4f35590b11f8d693a258a561055cac8f64cab7 \
    --hash=sha256:f072f4d804ea359e4eaf198b1af7a8b0943881a87f31bb764f8bf219bb9419e0
click==8.5.0 \
    --hash=sha256:255bc9599cf7748b4b1a446ccc735421bd08a2ae529a8b88597d3de5664ee360 \
    --hash=sha256:ba0d2089de75ea0310e2dde03160e6ca10009947fb95a182f9b54021bb272e34
colorama==0.4.6 \
    --hash=sha256:08695f5cb7ed6e0531a20572697297273c47b8cae5a63ffc6d6ed5c201be6e44 \
    --hash=sha256:4f1d9991f5acc0ca119f9d443620b77f9d6b33703e51011c16baf57afb285fc6
pydantic==2.14.1 \
    --hash=sha256:9195d967ec791692a04438115466764fb8b9a27b31f14a760437694f40d6b454 \
    --hash=sha256:94f478203dd03404682a1ada216965651dd74b1d2d5ffd62e00e0837caab5c26
pydantic-core==2.50.1 \
    --hash=sha256:0036473f5583e6a60e50b8b21651511564277a3f05cc5dab8cf579f552cd5f6c \
    --hash=sha256:0048b6dddc8ef4b64fccaad878bd143b0c3882ea9936279dc11d613f6b7dd1bc \
    --hash=sha256:009634b83993777ddcd69cad0ffcace43dabde692109528e35f0fde91e386a8b \
    --hash=sha256:028e2f212273d4a39b1ec1e0de8166b1165a65fc0f1111452a9d94fc7c625c63 \
    --hash=sha256:062e891facce5ca296a1c37098e5e466780457f86413894b399f0cf22934f769 \
    --hash=sha256:06e01fbbfdb9be777b316a71b6c49efaf4a08b615d0a98d678cda3023f79d019 \
    --hash=sha256:06ead20d39ffd6f2f6f2a8f8a6de67ff8bb1b4f14a8a30e058502514ee2ac685 \
    --hash=sha256:0b3a6f334c6a2345ca15318ff894502a90012536404b37c844a976c76c846e0b \
    --hash=sha256:0c003c3b7f49debb893d2d85ae099ac5959c9839e2f330fadb1fcdf7a6594482 \
    --hash=sha256:131059670f1d2444269b8585cb888963994871932447c08b39ac6a51fcfef658 \
    --hash=sha256:132529c83901437ff642f585216831bf5fd7a91df66829907e155192ead62498 \
    --hash=sha256:17e722e156d0444ecaefbe640bdb60928752bf2013e2b7a11cdb099aaae19bec \
    --hash=sha256:1a9006395dece0e32e704c315eff8a00bede494f6108546cfc5539c89fef4f9a \
    --hash=sha256:1c8632d4ac04e6f91128fca584b3a8a507d81604c24eeaaad00d4be42765c32b \
    --hash=sha256:1c96fd793b73d1b92e65570132505498fe7b21eaef73cdf74e67e5dfba7ac9e4 \
    --hash=sha256:1cf41f1ae3fa155cf167a72689ad044bcc1e3c97e064123677149bdfb5dafc4a \
    --hash=sha256:1deeacb112d14d3f4fcb16b165f7dbaf76c70ba6e82f37ba042bdab51970a0b8 \
    --hash=sha256:1ef800dd7d85bcdadf4c3076e4c94e43939493558a3b69a1ea830c706d4617bb \
    --hash=sha256:1fa4c8bc12c1354c5550c0c35c1852c8c1901e89e06561724e03f8d0342e1f87 \
    --hash=sha256:2005207aafe1231315718bf6ed5d064a7300fb4772754af35ee72fc68159492e \
    --hash=sha256:23923ab9292c40da026330b1ecf4dc2618c8e86e0422e5d1fbf50d94d64ca4f8 \
    --hash=sha256:23edad659e8dbd8ca7e4e877fe6c81573abbdf215bd25a68b53e1272f58b80c7 \
    --hash=sha256:2ab756b72bd5054e4c7ef3ded331b35786cbd3cf931531a508f79a9537517064 \
    --hash=sha256:2cbd1b75b09e976ed0d6b6ca297675632ca35df86130088457cdc60ef36970ae \
    --hash=sha256:2cf91809d0721ab81592ba67bea7694821679c10b1a2e3c3460082b286c1918a \
    --hash=sha256:2df1ff41884de2bc4b307bafd7c40a691094fad2ff8e767e5b45a319257bcf4e \
    --hash=sha256:2eb75304506894a281d346220a4f7481a1b8729577c5ed2a05395991966a8396 \
    --hash=sha256:2eedf82ee4753cdab8e50044c6bd569577eebc3859b11fecf4eb9223761ff966 \
    --hash=sha256:30ddf019d082c117b5d309e5b86710c2a78909907ec1a9381feec3eec02eca0b \
    --hash=sha256:325c23f3e35cfbf0fe3486fa5f7260d1e45885173002d30a28ca019994124255 \
    --hash=sha256:32fad3a91e51b6d2039c572db04a5a873260b399f6bd62c3552671fa7a4a2899 \
    --hash=sha256:36c426eac0af8d1529ff8467e612b933346caec1fdc0d774f78f67a1a11e16c1 \
    --hash=sha256:3a5fce22f1e87d181e924e12da7d81cfe031fb3881a5ddf26ad28f141756ca43 \
    --hash=sha256:3aa9de446b793de2beb6fa2d9d0961803126c4e2a99c2f25ab59b9fd6ea125c0 \
    --hash=sha256:3e46a9eb0a0901dd6275e6b06ac3a464885ef350ec4121fe486869de8053e4bb \
    --hash=sha256:3fde4fdc6487a58d944ca87cf5adc95d5f266e872c19599f5f4c0a8a1b1f9f9f \
    --hash=sha256:409e0ea40ec30d9158f33574fd758e689f6045a0f2596701828c27816ca9687d \
    --hash=sha256:40f523349960fa30f3ea51404308ff50f9997a90df639590f47a057c1f32b415 \
    --hash=sha256:41bc8237121bd8dc8d888dfd6279fc166ffc88c1f1bf3a8bf00869680533ca4c \
    --hash=sha256:42b54c2c90ad348b5e3a85e03e715d572c1fde357ef104cdfe3b03b697a404ea \
    --hash=sha256:455a773617b5913bf5c20d0692e5787b119e52c4d40ea644ca31f5758fd31be2 \
    --hash=sha256:45b11cac094aa25725581d9304eee93c9028516b9ea80dd9e175e13a5a2c840e \
    --hash=sha256:45c6266d071c241f2a168d45bf8c54344f0effce35e7e6b73afdec11f3687568 \
    --hash=sha256:46b3301d3b5c886f77de7546e47274a5842c622ea2020b8c6524c6b66913b4a6 \
    --hash=sha256:476f6ed8e43cd1e0b460920e23571700872b284e77331cb30c4faf459cf48a4b \
    --hash=sha256:48569b0ade9edfbe065cad1d700175546592aebbb42f02adcebcc26e75b896fe \
    --hash=sha256:49c2cbb2397fe4d0987e84606e691af6cb87bc0ee1bd3e7b737f7e10b4c142f9 \
    --hash=sha256:4a53d13cdfbedbfa87f08b83c1a0a5efcc767d785a4b41934fa9cb672670493a \
    --hash=sha256:4be846f55c9477f5f3ddde8f2ce941137e16862a56d018ed885d422bb6ae02f2 \
    --hash=sha256:4df197990c15b5a37c5a277d131d9f2c67de6133f2e5dafd80d9bba4b99f46f9 \
    --hash=sha256:4e834f6a8e4ff772dcc34f58ef5504147a3ea5b0f4eeb13b0f8eb2ca75ac57f1 \
    --hash=sha256:57f51b31ff826e2859120cf4737c5a758a48d96f3e97da40ccee1796d58078ff \
    --hash=sha256:5958c72adb417c39b12ac87525ac60b0d73315fcdc59e21f44ee4a5e2512c9ef \
    --hash=sha256:5dfe41f232befddb9c4377f6cfc702b51595e2d78ed082672adf8758d2c4619f \
    --hash=sha256:5f3cae32fc46121f787cb2486de9cf95a8bf72aec5cc78f64c606fa1735a6ef5 \
    --hash=sha256:64f6047f62a6c5ae08d0a6afb035667aa2d97c3d20d69762e034c5ea144d92a5 \
    --hash=sha256:6a733778df2f7087ec1100ed0b41533e4f3001976e99570fa34f57c66e7f8e3e \
    --hash=sha256:6b20a4bffabdad0db2927ac034ae3b8a681b1f7a0182f3e60b479ad2fde21ebb \
    --hash=sha256:6dbcbee53bf17196a7f745aa9bf5a9603953a1e365b1f020be3207c676a3e7c4 \
    --hash=sha256:6ed4f3cef55164b026fefb41341b7754cc6b624c75dfe7142d2ecceb5ad21c87 \
    --hash=sha256:704075d10b74f2f3c6e15407c696d88701df35fc8953f434a431add0d0074db0 \
    --hash=sha256:739dc730e6be3bd5ec2f4ab5cfc7eb047cc45fc1497b3bafec74ff2ed07df597 \
    --hash=sha256:7456d699b13954e9c0164dcb267250a10ae0dfb03e6e26d6796ab0d46e189c84 \
    --hash=sha256:756d669f04e62ec4148ecfe22be6a4484d9b1181a6ef32e205ebfd200540858b \
    --hash=sha256:7689580e72a642ab5ec64d5f55b2e33636fa43b4ebe63c0c2c965ef307c7d1aa \
    --hash=sha256:76e2e83fa6ec8cdc972d438dafc2522b3a47bee4ec0ae668b29cfb1977ab5242 \
    --hash=sha256:7816e98acc08119dc0f340ab167048ecc54126316330c1f0caf7c6756c88e28f \
    --hash=sha256:79490e33c4c0fcb933bbbcfc3a62184d8803b99f535863dfbb925e1bcb6945ad \
    --hash=sha256:7f476456ac2bb0d937f75191494a09c83a30765fea4f70f3b404942fe25f6cdf \
    --hash=sha256:844b869f118e22a41a091bdcedda8a71bc1b0f62c38d1a0c3211cece47e1d8fc \
    --hash=sha256:84bc765b282a9d5b7fe0348b8648904f25a6a04b2139da52b1dd30c8ac3a2c8f \
    --hash=sha256:84f34323a61a365b4e9295de6028474754829aaddd59c7bf1a040e7487ef8f3c \
    --hash=sha256:8812592c85d0edf423f10eadcef42716d71e8219085ad9e85b775057b7306133 \
    --hash=sha256:88e492e8b9d0312e7dc13667c30222abf284dc3b79b5302b3607b41a5784ce61 \
    --hash=sha256:8a6791afa2245e6c6b180122d105941644f5bd410bb18623b408808cc41a3102 \
    --hash=sha256:8b4c3df25bd323bf1d36a648d563cf1fc69d717451569927151bdad7cad07a77 \
    --hash=sha256:8daa7ee75245d43ad7d747e5c9ecc1b1d06552f72b14887e9276f787d57375f4 \
    --hash=sha256:93ba4e9d8210d941c200431a56b2c0400b131865947903937ed3ec5404307d2e \
    --hash=sha256:94845ff54dc5193f228cab81b2662a04bfbb892e95bdc15edf7399000ce57d54 \
    --hash=sha256:94be440c03fede26969a5ce75468e0e6a9927a1b46d9b679ee8adc1b057b0350 \
    --hash=sha256:9572c1369e9c9da2d64a7b7992c786d90ff295abc93964cfe3125e4290768070 \
    --hash=sha256:983a662de2571cb2502fc8ff47b6770b03d025d2eb314c92f77b3f07c74720ed \
    --hash=sha256:992c3514ec891fa7858099183e4d64e6bd5a5d4ff452fae29df22faa77a006bb \
    --hash=sha256:99ba9bc2b8062ea0c326a990f7f00e6530c23579de66dd246e72c4cafef950a5 \
    --hash=sha256:9d1bed94af6a63835461f3cf7502058eb166c58c4778e11d0f433cfb1bd69e19 \
    --hash=sha256:9e4472072de0137ee0d8e72d6620e85939c271d2f90f6bbb4b15c24638b79f92 \
    --hash=sha256:a27c09d86600f1bf2fe3f37e1ae697faf3143931c09322cd799da94deee923b5 \
    --hash=sha256:a29a061fec0b4e2d714f277e70a3a18125ecff803f2fea6eade2f2e53711d112 \
    --hash=sha256:a3cda0e538208e5d722bbf3698b24f19c0a7d05bc8d5f8a7f9b121ea7fa243d9 \
    --hash=sha256:a44101320cfe99432db74237545a63057dc7a88dfe792cbcad0647f2af56cb81 \
    --hash=sha256:a4aaaa791bdae1c972a7e81765f4f3571c926b8e0b9b6e47346499fb80079665 \
    --hash=sha256:a51eee75939cf811ac09b278745a6cee7dc873ccfbc8b9af3cc88fe4b7ce25b5 \
    --hash=sha256:a7c58106de36ac6a56314182958de20db8d3a29dfd5db527192cc754e4f8e7fb \
    --hash=sha256:aa8224f10880d9bf1b5993988ba153d42a8b4f3f4f511f93b1f09c93ff613c72 \
    --hash=sha256:acbf31f37c53a5ac0c34706c80b4f5107ba20b05fdd3816124bf236ef0c57dd2 \
    --hash=sha256:adc06d218a1cadfd2ec4628424d7d79ce4eba69c2965e7e7b55106f0da5208c8 \
    --hash=sha256:ae28183297fb0d2b8dc46a1f01d51f5e45825fc5afe76a835a6cb7fb34821295 \
    --hash=sha256:b0135bcdcaa0f23573f286e4cb5e0fd2962700964ed13df085b85f2b97aeab9e \
    --hash=sha256:b087b1c5be7ac687cf22eabfe4b6b608d40df23610651e93611e1f49118baf84 \
    --hash=sha256:b0d955195bbbe489ad343fcc956eacea9357b79cb22192c66cacdefcbc14b32f \
    --hash=sha256:b281a3b0f0822618fe5e3e0d8a2048b6356b14388505dc9374ccffeb69989713 \
    --hash=sha256:b6d0c2183008c188e19f4906d426b293bdc4f67ab17df8e180fe16cda208fa71 \
    --hash=sha256:bbce99252ba3167b2b6277f1829d5bf4b43b754524bddf7f944707c3db7d2253 \
    --hash=sha256:bc1f08f68dac9f9e83845a8039880aba2ab553eb9b2259c3243a313182c253fe \
    --hash=sha256:bc94f474417604bd383d2cd445d071b07dd55fedceed3ce33407bf1fcc107290 \
    --hash=sha256:bed5163e03b98bc1fa2eb05d74c63d9c5c95d8ed6254985481640fbf5e237dea \
    --hash=sha256:c17799a62c142d61b8a3c51752a7cbc87fe2ad4ccfab10e628a77b405075c662 \
    --hash=sha256:c18db21573bd2c6489f9a544b7499f0df2853958c568e5e783536ee1f690af41 \
    --hash=sha256:c3ede305158e75510be50869b319550ab072008c13d64d4ab1e094fb286b6f44 \
    --hash=sha256:c516cc5367ca3448995d42cb994bf3f4c9002d2a7c22eac9622551269ad1b807 \
    --hash=sha256:c531166c42ea7bdfecc8c50049581f05dd1993b09cc7c52bb36a14e96deaec7d \
    --hash=sha256:c73622ef819328873b53109ee4f77ceb598bffedd02daf916102be3228866b78 \
    --hash=sha256:c8dce1f1e0e5358b682a6ad3fa5e31b31d4560997b8e61417e9217c8d60f8a0c \
    --hash=sha256:cb57f304525a5e3c13333b772bf9a473f36326e9c821b2e8e1b2fd36f80ae2c3 \
    --hash=sha256:ce8c25ca38cc0e3d7753ba180808de2c0c8cb24eae0df64491e40921454e9831 \
    --hash=sha256:ceff0acc940be2715bd6ad17b24c0e5304abf44f6efd0f81ee8499e640f9dc86 \
    --hash=sha256:cf356f70551d40374eaffb1aa63f1eb6d2006681cbd7a9faea173ce0f4dd7cd2 \
    --hash=sha256:d2d82aa62521c55ddfb000ae70f88cdd8de974078f6024e821dfe5addd0c818f \
    --hash=sha256:d32f3acc081cc3923386d88f422cde8892335e95f034e0104bb4cf9310d9915f \
    --hash=sha256:d4193206b6587047437f6f11d7e776df23e1c1e23af2a54d9347275614791e10 \
    --hash=sha256:d5c0e32fdbce7f1e8ef4d11f655694bf5f4175c757a9f1dc2be09b8864e5bcf5 \
    --hash=sha256:d5e062c01286d861fd6a1c4ff6e063547b3e713067f2df033c0ff97ac2ca006b \
    --hash=sha256:d8f9e8a6c4ab04b78d61f78627370d834eb004b2869dcb28cfffa647b4ea1980 \
    --hash=sha256:d939de9c82e2126f7f48a7e658f8a85ed46d57662d53f44c49b8895fe94a3eb7 \
    --hash=sha256:de531ce1e2a3364e8767878b58f4ff728a434b4fde089781fe30b1e08e2396e0 \
    --hash=sha256:e50d7b94baac6c7d09927fa5ca5800a0c7ee5015c7fcff65beb3a1931b5a6e09 \
    --hash=sha256:e5faeaee74a57d32b3ab3aebad2e348f06d3ba946fc5d28c1728455f00a3d13a \
    --hash=sha256:e6f0cc1bb9900dc558960894adeb30b0c083366fc1d69b856209fb2ca5c36fe5 \
    --hash=sha256:e8e1d6ce820aa23317e8209a86bd65a540973c12dc7552b48a4f6c8e9926815e \
    --hash=sha256:ed1e728b39a383c81035b2459cfcb35d99dfb01f7d6ebe3a913bc1cc5b81e459 \
    --hash=sha256:ee6db2fbed51a7991302e8fac498cd67e336246026d0dfa84cf5166ce1412760 \
    --hash=sha256:efbecf43d321f7b9281441f1f213f7c21c66988b0e06c2730ba13ed47a46bb08 \
    --hash=sha256:f2c634642694e6a0dad2ab1d375589fa671fd442edd5caf7d9737b8f6ca22906 \
    --hash=sha256:f3377c8c2b3ce898423c5e5dd94c7982e30aa7717a7e6ab2470b9de364963709 \
    --hash=sha256:f5187624823423e1d1b82b1072ac41dc837389e18d3d0572cc19bbee46cd550a \
    --hash=sha256:f77ac30b19221cd9bd3fcfa3d4614eff93140d0572ab730cded17b64adca05f3 \
    --hash=sha256:fe90228920fd8ff2be62622b6bb8a2b11acd65046d50c6b130614b5879605a20
typing-extensions==4.16.0 \
    --hash=sha256:481caa481374e813c1b176ada14e97f1f67a4539ce9cfeb3f350d78d6370c2e8 \
    --hash=sha256:dc983d19a509c94dba722ee6abd33940f7c05a89e243c47e907eb4db6f1a43e5
typing-inspection==0.4.4 \
    --hash=sha256:547274fa6b0a561ccf549cc9524b999a578e737d015d8709d021f9d0d13bea47 \
    --hash=sha256:65b8397ba37ccbce054456aaccddfc91e6e3083c92824df348d96ca832f3f147
```

**Archivo:** `requirements-dev.in`

```text
# Herramientas de desarrollo y CI; respetan las versiones de requirements.txt
#   pip-compile --strip-extras --no-annotate -o requirements-dev.txt requirements-dev.in
-c requirements.txt
actionlint-py
boto3
build
mypy
pip-audit
pip-tools
pytest
pytest-cov
ruff
shellcheck-py
twine
# cryptography (la usa keyring vía twine): <49 para que haya ruedas en macOS Intel
cryptography<49
```

**Archivo:** `requirements-dev.txt`

```text
# Versiones fijadas para la Práctica 6 (hay ruedas para Windows, macOS 10.15+ y Linux; Python 3.12-3.14).
# Generado con:
#   pip-compile --strip-extras --no-annotate -o requirements-dev.txt requirements-dev.in
actionlint-py==1.7.12.25
ast-serialize==0.13.1
boolean-py==5.0
boto3==1.43.120
botocore==1.43.120
build==1.6.1
cachecontrol==0.14.4
certifi==2026.10.23
cffi==2.1.1
charset-normalizer==3.5.2
click==8.5.0
coverage==7.17.0
cryptography==48.0.1
cyclonedx-python-lib==11.12.0
defusedxml==0.7.1
docutils==0.23
filelock==4.2.4
id==1.6.1
idna==3.20
iniconfig==2.3.1
jaraco-classes==3.4.0
jaraco-context==6.1.2
jaraco-functools==4.6.0
jeepney==0.9.0
jmespath==1.1.0
keyring==25.7.0
librt==0.16.0
license-expression==30.4.4
markdown-it-py==4.2.0
mdurl==0.1.2
more-itertools==11.2.1
msgpack==1.2.4
mypy==2.4.0
mypy-extensions==1.1.0
nh3==0.3.7
packageurl-python==0.17.6
packaging==26.3
pathspec==1.1.1
pip-api==0.0.35
pip-audit==2.10.1
pip-requirements-parser==32.0.1
pip-tools==7.6.2
platformdirs==4.13.6
pluggy==1.7.0
py-serializable==2.1.0
pycparser==3.12
pygments==2.21.0
pyparsing==3.3.3
pyproject-hooks==1.3.3
pytest==9.2.0
pytest-cov==7.1.0
python-dateutil==2.9.0.post0
readme-renderer==46.0
requests==2.34.2
requests-toolbelt==1.0.0
rfc3986==2.0.0
rich==15.0.0
ruff==0.17.3
s3transfer==0.19.3
secretstorage==3.5.0
shellcheck-py==0.11.0.1
six==1.17.0
sortedcontainers==2.4.0
tomli==2.5.0
tomli-w==1.2.0
twine==7.0.0
typing-extensions==4.16.0
urllib3==2.8.0
wheel==0.48.0

# The following packages are considered to be unsafe in a requirements file:
# pip
# setuptools
```

**0.5** Crea los archivos de control de versiones, la configuración no sensible y la plantilla de secretos (6.17, 6.22):

**Archivo:** `.gitignore`

```text
.venv/
__pycache__/
.pytest_cache/
.mypy_cache/
.ruff_cache/
.coverage
# Artefactos: los produce el pipeline, no se versionan
build/
dist/
*.egg-info/
# Datos, resultados y Airflow local
datos/
salida/
airflow_home/
# Secretos locales
.env
*.pem
```

**Archivo:** `.gitattributes`

```text
# Los scripts de bash conservan LF aunque se editen en Windows (CRLF rompe bash)
*.sh text eol=lf
Dockerfile text eol=lf
```

**Archivo:** `README.md`

````markdown
# cargaventas

Carga diaria de ventas: valida cada fila, minimiza los datos personales y carga
de forma idempotente (todo o nada). Se ejecuta como CLI, en un contenedor o desde
el DAG `carga_ventas` de Airflow 3 / Amazon MWAA.

```bash
cargaventas --version
cargaventas generar-ejemplo --fecha 2026-10-26
CLAVE_PSEUDONIMOS=... cargaventas cargar --fecha 2026-10-26 --dry-run
```

Códigos de salida (contrato con el orquestador): 0 éxito, 1 error inesperado,
2 uso, 3 configuración, 10 validación (no reintentar), 20 destino (reintentar).
````

**Archivo:** `config.toml`

```toml
# Configuración NO sensible: se versiona. Cambia por entorno con variables:
#   APP_ENV, CARGAVENTAS_DATOS y CARGAVENTAS_BD sobrescriben estos valores.
# Los secretos (CLAVE_PSEUDONIMOS) NUNCA van aquí: solo en el entorno.
entorno = "dev"

[entrada]
carpeta = "datos"          # solo se leen archivos dentro de esta carpeta
max_filas = 100000

[salida]
base_datos = "salida/ventas.db"

[calidad]
max_pct_invalidas = 20.0   # por encima, el lote se rechaza completo (código 10)
```

**Archivo:** `.env.example`

```bash
# Copia este archivo como .env. Se versiona .env.example, NUNCA .env.
# En producción el valor viene de AWS Secrets Manager.
CLAVE_PSEUDONIMOS=reemplaza-con-una-clave-de-al-menos-16-caracteres
```

**0.6** Crea el entorno virtual e instala. Como `requirements.txt` tiene hashes, pip entra en **modo de verificación de hashes**: si un paquete descargado no coincide, la instalación se detiene.

**PowerShell**
```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
```

**bash**
```bash
python -m venv .venv
source .venv/bin/activate
```

Después, en cualquier sistema operativo:

```bash
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
python -m pip install -r requirements-dev.txt
```

> El paquete se instala en la Parte 1, cuando exista su código.

**0.7** Inicia el repositorio. actionlint (Parte 6) necesita un repositorio de Git y cada entrega se identificará por una etiqueta y un commit:

```bash
git init -b main
git add .
git commit -m "Práctica 6: estructura, dependencias y configuración"
```

> Si Git pide tu identidad: `git config --global user.name "Tu Nombre"` y `git config --global user.email "tu@correo.com"`.

---

## Parte 1. Paquete interno y CLI con contrato (30 min)

Cron, Docker, Airflow o un pipeline de CI no importan funciones: ejecutan **comandos**. La CLI es la frontera y su contrato tiene cuatro reglas: parámetros por CLI o entorno, **código de salida honesto**, logs a **stderr** y resultado JSON en la **última línea de stdout** (6.4, 6.13).

### 1.1 Versión única y punto de entrada

La versión se escribe solo en `pyproject.toml`; el código la lee de los metadatos instalados:

**Archivo:** `src/cargaventas/__init__.py`

```python
"""cargaventas: carga diaria de ventas lista para contenedor y orquestador."""

from importlib.metadata import PackageNotFoundError, version

try:
    __version__ = version("cargaventas")  # la versión sale de pyproject.toml
except PackageNotFoundError:  # código sin instalar (no debería pasar en el lab)
    __version__ = "0.0.0+local"
```

**Archivo:** `src/cargaventas/__main__.py`

```python
"""Permite ejecutar el paquete con: python -m cargaventas ..."""

from cargaventas.cli import main

main()
```

### 1.2 El contrato de códigos de salida

**Archivo:** `src/cargaventas/contrato.py`

```python
"""Contrato con el orquestador: códigos de salida y qué hacer con cada uno.

La CLI, la imagen de Docker y el DAG usan los mismos números.
No se usa 99: BashOperator de Airflow lo interpreta como "skipped".
"""

from enum import IntEnum
from typing import Literal


class Codigo(IntEnum):
    EXITO = 0
    ERROR_INESPERADO = 1  # defecto del código: reintentar no lo arregla
    ERROR_USO = 2  # argumentos inválidos (lo produce click)
    ERROR_CONFIGURACION = 3  # falta un secreto o la configuración es inválida
    ERROR_VALIDACION = 10  # datos de entrada rechazados
    ERROR_DESTINO = 20  # el destino falló: transitorio, conviene reintentar


Decision = Literal["exito", "reintentar", "fallar_sin_reintento"]

REINTENTABLES = frozenset({Codigo.ERROR_DESTINO})


def interpretar(codigo: int) -> Decision:
    """Traduce un código de salida a la decisión que debe tomar el orquestador."""
    if codigo == Codigo.EXITO:
        return "exito"
    if codigo in REINTENTABLES:
        return "reintentar"
    return "fallar_sin_reintento"
```

| Código | Significado | ¿Reintentar? |
| :---: | :--- | :---: |
| 0 | Éxito | — |
| 1 | Error inesperado (defecto del código) | No |
| 2 | Error de uso: argumento inválido o ruta no permitida | No |
| 3 | Configuración inválida o falta un secreto | No |
| 10 | Datos de entrada rechazados | No |
| 20 | El destino falló (transitorio) | **Sí** |

### 1.3 Configuración y secretos

Tres capas (6.17): el archivo versionado, las variables de entorno que lo sobrescriben y el secreto, que **solo** existe en el entorno. Pydantic con `extra="forbid"` convierte un error de dedo en `config.toml` en un error al arrancar, no en un valor ignorado. El mensaje de error dice qué campo falló, nunca el valor recibido.

**Archivo:** `src/cargaventas/config.py`

```python
"""Configuración: TOML versionado + variables de entorno + secretos del entorno."""

import os
import tomllib
from collections.abc import Mapping
from pathlib import Path
from typing import Any, Literal

from pydantic import BaseModel, ConfigDict, Field, SecretStr, ValidationError

# Variables de entorno que sobrescriben valores NO sensibles del archivo
SOBRESCRITURAS = {
    "APP_ENV": ("entorno",),
    "CARGAVENTAS_DATOS": ("entrada", "carpeta"),
    "CARGAVENTAS_BD": ("salida", "base_datos"),
}


class ErrorConfiguracion(Exception):
    """Configuración ausente o inválida: el proceso no debe arrancar."""


class Estricto(BaseModel):
    # extra="forbid": una clave desconocida (p. ej. un error de dedo) es un error
    model_config = ConfigDict(extra="forbid", frozen=True)


class Entrada(Estricto):
    carpeta: Path
    max_filas: int = Field(gt=0)


class Salida(Estricto):
    base_datos: Path


class Calidad(Estricto):
    max_pct_invalidas: float = Field(ge=0, le=100)


class Config(Estricto):
    entorno: Literal["dev", "staging", "prod"]
    entrada: Entrada
    salida: Salida
    calidad: Calidad
    clave_pseudonimos: SecretStr = Field(min_length=16)  # nunca se imprime


def cargar_config(entorno: Mapping[str, str] | None = None) -> Config:
    """Lee config.toml (o CARGAVENTAS_CONFIG), aplica el entorno y valida todo."""
    env = os.environ if entorno is None else entorno
    ruta = Path(env.get("CARGAVENTAS_CONFIG", "config.toml"))
    try:
        datos: dict[str, Any] = tomllib.loads(ruta.read_text(encoding="utf-8"))
    except FileNotFoundError:
        raise ErrorConfiguracion(f"no existe el archivo {ruta}") from None
    except tomllib.TOMLDecodeError as exc:
        raise ErrorConfiguracion(f"{ruta} no es TOML válido: {exc}") from None

    for variable, (*seccion, clave) in SOBRESCRITURAS.items():
        if variable in env:
            destino = datos
            for parte in seccion:
                destino = destino.setdefault(parte, {})
            destino[clave] = env[variable]

    secreto = env.get("CLAVE_PSEUDONIMOS", "")
    if not secreto:
        raise ErrorConfiguracion("falta la variable de entorno CLAVE_PSEUDONIMOS")
    datos["clave_pseudonimos"] = secreto

    try:
        return Config.model_validate(datos)
    except ValidationError as exc:  # solo ubicación y motivo: nunca el valor recibido
        detalle = "; ".join(f"{'.'.join(map(str, e['loc']))}: {e['msg']}" for e in exc.errors())
        raise ErrorConfiguracion(f"configuración inválida: {detalle}") from None
```

### 1.4 Validación de entradas

Cada fila del CSV debe cumplir este contrato (6.34). `extra="forbid"` impide que lleguen campos no esperados al modelo:

**Archivo:** `src/cargaventas/modelos.py`

```python
"""Contrato de datos de entrada: cada fila del CSV debe cumplirlo."""

from datetime import date
from decimal import Decimal

from pydantic import BaseModel, ConfigDict, Field


class VentaEntrada(BaseModel):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True, frozen=True)

    venta_id: str = Field(pattern=r"^V-\d{6}$")
    fecha: date
    sku: str = Field(pattern=r"^[A-Z]{3}-\d{3}$")
    cantidad: int = Field(ge=1, le=1000)
    precio_unitario: Decimal = Field(gt=0, max_digits=10, decimal_places=2)
    email_cliente: str = Field(max_length=254, pattern=r"^[^@\s]+@[^@\s]+\.[^@\s]+$")


# Columnas que el proceso necesita; cualquier otra se descarta (minimización)
COLUMNAS = tuple(VentaEntrada.model_fields)
```

### 1.5 Logs con versión, commit y redacción

El filtro de contexto agrega `v=` y `commit=` a cada línea (6.14). El redactor va **en el handler**, así cubre cualquier logger hijo y reemplaza tokens, contraseñas, llaves de AWS, credenciales en URL y el valor real del secreto (6.38):

**Archivo:** `src/cargaventas/registro.py`

```python
"""Logs a stderr con versión y commit, y redacción de secretos en el handler."""

import logging
import os
import re
import sys
from collections.abc import Iterable
from typing import TextIO

PATRONES = [
    (re.compile(r"(?i)\bBearer\s+[A-Za-z0-9._~+/=-]+"), "Bearer ***"),
    (re.compile(r"(?i)\b(password|clave|token|secret)=\S+"), r"\1=***"),
    (re.compile(r"\bAKIA[0-9A-Z]{16}\b"), "AKIA***"),
    (re.compile(r"(\w+://)[^/\s:@]+:[^/\s@]+@"), r"\1***:***@"),  # usuario:clave@
]


class ContextoDespliegue(logging.Filter):
    """Agrega versión y commit a cada registro: qué código produjo esta línea."""

    def __init__(self, version: str, commit: str) -> None:
        super().__init__()
        self.version, self.commit = version, commit

    def filter(self, record: logging.LogRecord) -> bool:
        record.version, record.commit = self.version, self.commit
        return True


class RedactorSecretos(logging.Filter):
    """Reemplaza secretos conocidos y patrones típicos antes de escribir."""

    def __init__(self, secretos: Iterable[str] = ()) -> None:
        super().__init__()
        self.secretos = [s for s in secretos if s]

    def filter(self, record: logging.LogRecord) -> bool:
        mensaje = record.getMessage()
        for patron, reemplazo in PATRONES:
            mensaje = patron.sub(reemplazo, mensaje)
        for secreto in self.secretos:
            mensaje = mensaje.replace(secreto, "***")
        record.msg, record.args = mensaje, None
        return True


def configurar_registro(
    version: str,
    secretos: Iterable[str] = (),
    flujo: TextIO | None = None,
) -> logging.Logger:
    commit = (os.environ.get("GIT_SHA") or "sin-sha")[:7]  # SHA corto
    manejador = logging.StreamHandler(flujo or sys.stderr)  # logs a stderr
    manejador.setFormatter(
        logging.Formatter(
            "%(asctime)s %(levelname)s v=%(version)s commit=%(commit)s %(name)s: %(message)s"
        )
    )
    manejador.addFilter(ContextoDespliegue(version, commit))
    manejador.addFilter(RedactorSecretos(secretos))  # en el handler: cubre todo
    raiz = logging.getLogger("cargaventas")
    raiz.handlers.clear()
    raiz.addHandler(manejador)
    raiz.setLevel(logging.INFO)
    raiz.propagate = False
    return raiz
```

### 1.6 El proceso

Observa cuatro decisiones:

- **Ruta segura:** `resolve()` + `is_relative_to()` impide leer fuera de `datos/` (por ejemplo `../config.toml`).
- **Minimización (6.39):** solo se leen las columnas del modelo; `nombre_cliente` y `rfc` se descartan y el correo se guarda como seudónimo HMAC-SHA256 con la clave secreta.
- **Todo o nada:** el UPSERT de todas las filas va en **una** transacción; si algo falla, `with conexion` hace rollback y el código es 20.
- **Idempotencia:** `ON CONFLICT(venta_id) DO UPDATE`: repetir el mismo día no duplica.

**Archivo:** `src/cargaventas/proceso.py`

```python
"""Proceso: leer el CSV, validar, minimizar y cargar en una sola transacción."""

import csv
import hashlib
import hmac
import logging
import sqlite3
from dataclasses import asdict, dataclass, field
from datetime import date
from decimal import Decimal
from pathlib import Path
from typing import Any

from pydantic import ValidationError

from cargaventas.config import Config
from cargaventas.modelos import COLUMNAS, VentaEntrada

log = logging.getLogger(__name__)

CREAR_TABLA = """
CREATE TABLE IF NOT EXISTS ventas (
    venta_id        TEXT PRIMARY KEY,
    fecha           TEXT NOT NULL,
    sku             TEXT NOT NULL,
    cantidad        INTEGER NOT NULL,
    precio_unitario TEXT NOT NULL,
    total           TEXT NOT NULL,
    cliente_id      TEXT NOT NULL,      -- seudónimo HMAC: nunca el correo
    version_carga   TEXT NOT NULL
)"""

UPSERT = """
INSERT INTO ventas VALUES (?, ?, ?, ?, ?, ?, ?, ?)
ON CONFLICT(venta_id) DO UPDATE SET
    fecha = excluded.fecha, sku = excluded.sku, cantidad = excluded.cantidad,
    precio_unitario = excluded.precio_unitario, total = excluded.total,
    cliente_id = excluded.cliente_id, version_carga = excluded.version_carga"""


class RutaNoPermitida(ValueError):
    """La ruta pedida sale de la carpeta de entrada autorizada."""


class ErrorValidacion(Exception):
    """Los datos de entrada no son aceptables: reintentar no ayuda."""


class ErrorDestino(Exception):
    """El destino falló; la transacción se revirtió y se puede reintentar."""


@dataclass
class Metricas:
    fecha: str
    leidas: int = 0
    validas: int = 0
    invalidas: int = 0
    columnas_descartadas: list[str] = field(default_factory=list)
    cargadas: int = 0
    dry_run: bool = False
    version: str = ""

    def como_dict(self) -> dict[str, Any]:
        return asdict(self)


def ruta_segura(carpeta: Path, ruta: Path) -> Path:
    """Resuelve la ruta y exige que quede dentro de la carpeta autorizada."""
    base, destino = carpeta.resolve(), ruta.resolve()
    if not destino.is_relative_to(base):
        raise RutaNoPermitida(f"solo se permiten archivos dentro de {carpeta}")
    return destino


def seudonimo(email: str, clave: bytes) -> str:
    """Identificador estable del cliente sin guardar el correo (HMAC-SHA256)."""
    return hmac.new(clave, email.lower().encode(), hashlib.sha256).hexdigest()[:16]


def leer_y_validar(
    ruta: Path, fecha: date, config: Config, metricas: Metricas
) -> list[VentaEntrada]:
    validas: list[VentaEntrada] = []
    with ruta.open(newline="", encoding="utf-8") as archivo:
        lector = csv.DictReader(archivo)
        encabezado = list(lector.fieldnames or [])
        faltantes = [c for c in COLUMNAS if c not in encabezado]
        if faltantes:
            raise ErrorValidacion(f"faltan columnas obligatorias: {', '.join(faltantes)}")
        metricas.columnas_descartadas = [c for c in encabezado if c not in COLUMNAS]
        if metricas.columnas_descartadas:
            log.info(
                "minimización: se descartan columnas no necesarias: %s",
                ", ".join(metricas.columnas_descartadas),
            )
        for linea, fila in enumerate(lector, start=2):  # línea 1 = encabezado
            metricas.leidas += 1
            if metricas.leidas > config.entrada.max_filas:
                raise ErrorValidacion(f"más de {config.entrada.max_filas} filas")
            try:
                venta = VentaEntrada.model_validate({c: fila[c] for c in COLUMNAS})
            except ValidationError as exc:  # se registran campos, nunca valores
                campos = sorted({str(e["loc"][0]) for e in exc.errors()})
                log.warning("línea %d rechazada: %s", linea, ", ".join(campos))
                continue
            if venta.fecha != fecha:
                log.warning("línea %d rechazada: fecha distinta de %s", linea, fecha)
                continue
            validas.append(venta)

    metricas.validas = len(validas)
    metricas.invalidas = metricas.leidas - metricas.validas
    if metricas.leidas == 0:
        raise ErrorValidacion("el archivo no tiene filas de datos")
    pct = 100 * metricas.invalidas / metricas.leidas
    if pct > config.calidad.max_pct_invalidas:
        raise ErrorValidacion(
            f"{pct:.1f}% de filas inválidas supera el máximo de "
            f"{config.calidad.max_pct_invalidas}%: se rechaza el lote completo"
        )
    return validas


def cargar_bd(ruta_bd: Path, ventas: list[VentaEntrada], clave: bytes, version: str) -> int:
    """UPSERT de todas las filas en UNA transacción: todo o nada."""
    filas = [
        (
            v.venta_id,
            v.fecha.isoformat(),
            v.sku,
            v.cantidad,
            str(v.precio_unitario),
            str((v.precio_unitario * v.cantidad).quantize(Decimal("0.01"))),
            seudonimo(v.email_cliente, clave),
            version,
        )
        for v in ventas
    ]
    try:
        ruta_bd.parent.mkdir(parents=True, exist_ok=True)
        conexion = sqlite3.connect(ruta_bd, timeout=5, autocommit=False)
        try:
            with conexion:  # commit si todo sale bien; rollback si algo falla
                conexion.execute(CREAR_TABLA)
                conexion.executemany(UPSERT, filas)
        finally:
            conexion.close()
    except (sqlite3.Error, OSError) as exc:
        raise ErrorDestino(f"no se pudo cargar en {ruta_bd}: {exc}") from exc
    return len(filas)


def ejecutar(
    fecha: date, entrada: Path | None, *, dry_run: bool, config: Config, version: str
) -> Metricas:
    carpeta = config.entrada.carpeta
    ruta = ruta_segura(carpeta, entrada or carpeta / f"ventas_{fecha.isoformat()}.csv")
    if not ruta.is_file():
        raise ErrorValidacion(f"no existe el archivo de entrada {ruta.name}")
    metricas = Metricas(fecha=fecha.isoformat(), dry_run=dry_run, version=version)
    log.info("inicio: archivo=%s entorno=%s dry_run=%s", ruta.name, config.entorno, dry_run)
    ventas = leer_y_validar(ruta, fecha, config, metricas)
    if dry_run:
        log.info("dry-run: %d filas válidas; no se escribe en el destino", len(ventas))
    else:
        clave = config.clave_pseudonimos.get_secret_value().encode()
        metricas.cargadas = cargar_bd(config.salida.base_datos, ventas, clave, version)
        log.info(
            "carga confirmada: %d filas en %s",
            metricas.cargadas,
            config.salida.base_datos,
        )
    return metricas


def resumen(ruta_bd: Path, fecha: date) -> dict[str, Any]:
    """Lee el destino en modo solo lectura: filas y total de la fecha."""
    try:
        uri = f"{ruta_bd.resolve().as_uri()}?mode=ro"  # mode=ro: nunca crea ni escribe
        conexion = sqlite3.connect(uri, uri=True)
        try:
            totales = [
                Decimal(t)
                for (t,) in conexion.execute(
                    "SELECT total FROM ventas WHERE fecha = ?", (fecha.isoformat(),)
                )
            ]
        finally:
            conexion.close()
    except sqlite3.Error as exc:
        raise ErrorDestino(f"no se pudo leer {ruta_bd}: {exc}") from exc
    return {
        "fecha": fecha.isoformat(),
        "filas": len(totales),
        "total": str(sum(totales)),
    }
```

### 1.7 Datos sintéticos y la CLI

**Archivo:** `src/cargaventas/ejemplo.py`

```python
"""Generador de datos sintéticos para el laboratorio (sin datos reales)."""

import csv
import random
from datetime import date, timedelta
from pathlib import Path

# Columnas de más que trae el sistema de origen y el proceso NO necesita
EXTRA = ("nombre_cliente", "rfc")
ENCABEZADO = (
    "venta_id",
    "fecha",
    "sku",
    "cantidad",
    "precio_unitario",
    "email_cliente",
    *EXTRA,
)
DEFECTOS = ("sku", "cantidad", "email", "fecha")
FAMILIAS = ("LAP", "MON", "TEC", "RAT")


def generar(fecha: date, filas: int, pct_invalidas: float, semilla: int) -> list[dict[str, str]]:
    rng = random.Random(semilla)  # noqa: S311 (datos de prueba, no criptografía)
    n_invalidas = round(filas * pct_invalidas / 100)
    # fila mala -> defecto; se reparten los cuatro tipos de defecto
    malas = {
        n: DEFECTOS[i % 4] for i, n in enumerate(sorted(rng.sample(range(filas), n_invalidas)))
    }
    registros = []
    for n in range(filas):
        registro = {
            "venta_id": f"V-{n + 1:06d}",
            "fecha": fecha.isoformat(),
            "sku": f"{rng.choice(FAMILIAS)}-{rng.randint(1, 120):03d}",
            "cantidad": str(rng.randint(1, 20)),
            "precio_unitario": f"{rng.randint(100, 25000) / 10:.2f}",
            "email_cliente": f"cliente{n % 37}@ejemplo.com",
            "nombre_cliente": f"Cliente Sintético {n % 37}",
            "rfc": f"XAXX0101{n % 37:02d}000",
        }
        defecto = malas.get(n)
        if defecto:
            if defecto == "sku":
                registro["sku"] = registro["sku"].lower()
            elif defecto == "cantidad":
                registro["cantidad"] = "0"
            elif defecto == "email":
                registro["email_cliente"] = "sin-arroba.ejemplo.com"
            else:
                registro["fecha"] = (fecha - timedelta(days=1)).isoformat()
        registros.append(registro)
    return registros


def escribir_csv(ruta: Path, registros: list[dict[str, str]]) -> None:
    ruta.parent.mkdir(parents=True, exist_ok=True)
    with ruta.open("w", newline="", encoding="utf-8") as archivo:
        escritor = csv.DictWriter(archivo, fieldnames=ENCABEZADO)
        escritor.writeheader()
        escritor.writerows(registros)
```

**Archivo:** `src/cargaventas/cli.py`

```python
"""CLI: la frontera entre el orquestador (cron, Docker, Airflow) y el paquete."""

import json
import os
from datetime import date
from pathlib import Path

import click

from cargaventas import __version__
from cargaventas.config import ErrorConfiguracion, cargar_config
from cargaventas.contrato import Codigo
from cargaventas.ejemplo import escribir_csv, generar
from cargaventas.proceso import (
    ErrorDestino,
    ErrorValidacion,
    RutaNoPermitida,
    ejecutar,
    resumen,
)
from cargaventas.registro import configurar_registro


def _fecha(ctx: click.Context, param: click.Parameter, valor: str) -> date:
    try:
        return date.fromisoformat(valor)
    except ValueError:
        raise click.BadParameter("use el formato AAAA-MM-DD, p. ej. 2026-10-26") from None


@click.group()
@click.version_option(__version__, prog_name="cargaventas")
def cli() -> None:
    """Carga diaria de ventas: valida, minimiza y carga de forma idempotente."""


@cli.command()
@click.option("--fecha", required=True, callback=_fecha, help="Fecha lógica AAAA-MM-DD.")
@click.option(
    "--entrada",
    type=click.Path(path_type=Path, dir_okay=False),
    help="CSV a procesar (dentro de la carpeta de entrada).",
)
@click.option("--dry-run", is_flag=True, help="Valida sin escribir en el destino.")
def cargar(fecha: date, entrada: Path | None, dry_run: bool) -> None:
    """Valida el CSV de la fecha y lo carga en una sola transacción."""
    log = configurar_registro(__version__, secretos=[os.environ.get("CLAVE_PSEUDONIMOS", "")])
    try:
        config = cargar_config()
        metricas = ejecutar(fecha, entrada, dry_run=dry_run, config=config, version=__version__)
    except RutaNoPermitida as exc:
        raise click.BadParameter(str(exc), param_hint="--entrada") from None  # exit 2
    except ErrorConfiguracion as exc:
        log.error("configuración: %s", exc)
        raise SystemExit(Codigo.ERROR_CONFIGURACION) from None
    except ErrorValidacion as exc:
        log.error("validación: %s", exc)
        raise SystemExit(Codigo.ERROR_VALIDACION) from None
    except ErrorDestino as exc:
        log.error("destino: %s", exc)
        raise SystemExit(Codigo.ERROR_DESTINO) from None
    except Exception:
        log.exception("error inesperado")
        raise SystemExit(Codigo.ERROR_INESPERADO) from None

    resultado = json.dumps(metricas.como_dict(), ensure_ascii=False)
    if ruta := os.environ.get("XCOM_FILE"):  # p. ej. /airflow/xcom/return.json
        Path(ruta).parent.mkdir(parents=True, exist_ok=True)
        Path(ruta).write_text(resultado, encoding="utf-8")
    click.echo(resultado)  # última línea de stdout = resultado para el orquestador


@cli.command("resumen")
@click.option("--fecha", required=True, callback=_fecha, help="Fecha lógica AAAA-MM-DD.")
def resumen_cmd(fecha: date) -> None:
    """Muestra filas y total cargados para una fecha (solo lectura)."""
    log = configurar_registro(__version__, secretos=[os.environ.get("CLAVE_PSEUDONIMOS", "")])
    try:
        datos = resumen(cargar_config().salida.base_datos, fecha)
    except ErrorConfiguracion as exc:
        log.error("configuración: %s", exc)
        raise SystemExit(Codigo.ERROR_CONFIGURACION) from None
    except ErrorDestino as exc:
        log.error("destino: %s", exc)
        raise SystemExit(Codigo.ERROR_DESTINO) from None
    click.echo(json.dumps(datos))


@cli.command("generar-ejemplo")
@click.option("--fecha", required=True, callback=_fecha, help="Fecha lógica AAAA-MM-DD.")
@click.option("--filas", default=50, show_default=True, type=click.IntRange(1, 100_000))
@click.option("--pct-invalidas", default=10.0, show_default=True, type=click.FloatRange(0, 100))
@click.option("--semilla", default=7, show_default=True, type=int)
@click.option("--carpeta", default="datos", show_default=True, type=click.Path(path_type=Path))
def generar_ejemplo(
    fecha: date, filas: int, pct_invalidas: float, semilla: int, carpeta: Path
) -> None:
    """Genera un CSV sintético con filas inválidas y columnas de más."""
    ruta = carpeta / f"ventas_{fecha.isoformat()}.csv"
    escribir_csv(ruta, generar(fecha, filas, pct_invalidas, semilla))
    click.echo(f"{ruta} ({filas} filas, {pct_invalidas:g}% inválidas)")


def main() -> None:
    cli(prog_name="cargaventas")
```

Instala el paquete en modo editable. `--no-deps` porque las dependencias ya están instaladas desde el lockfile verificado:

```bash
python -m pip install --no-deps -e .
cargaventas --version
cargaventas --help
```

Resultado esperado: `cargaventas, version 1.0.0` y la ayuda con los comandos `cargar`, `generar-ejemplo` y `resumen`.

### 1.8 Probar el contrato

**1.8.1** Genera dos días de datos: uno normal y uno con 40% de filas inválidas:

```bash
cargaventas generar-ejemplo --fecha 2026-10-26
cargaventas generar-ejemplo --fecha 2026-10-27 --pct-invalidas 40
```

Resultado esperado:

```text
datos/ventas_2026-10-26.csv (50 filas, 10% inválidas)
datos/ventas_2026-10-27.csv (50 filas, 40% inválidas)
```

**1.8.2** Sin el secreto, el proceso no arranca:

```bash
cargaventas cargar --fecha 2026-10-26
```

Resultado esperado (código **3**):

```text
... ERROR v=1.0.0 commit=sin-sha cargaventas: configuración: falta la variable de entorno CLAVE_PSEUDONIMOS
```

**1.8.3** Define el secreto **solo en esta terminal** (en la Parte 2 vendrá de un archivo `.env` y en producción de AWS Secrets Manager) y valida sin escribir:

**PowerShell**
```powershell
$env:CLAVE_PSEUDONIMOS = "clave-de-laboratorio-0123456789"
```

**bash**
```bash
export CLAVE_PSEUDONIMOS="clave-de-laboratorio-0123456789"
```

```bash
cargaventas cargar --fecha 2026-10-26 --dry-run
```

Resultado esperado (código **0**; los logs van a stderr y la última línea, a stdout):

```text
... INFO v=1.0.0 commit=sin-sha cargaventas.proceso: inicio: archivo=ventas_2026-10-26.csv entorno=dev dry_run=True
... INFO ... minimización: se descartan columnas no necesarias: nombre_cliente, rfc
... WARNING ... línea 5 rechazada: sku
... WARNING ... línea 11 rechazada: cantidad
... WARNING ... línea 22 rechazada: email_cliente
... WARNING ... línea 27 rechazada: fecha distinta de 2026-10-26
... WARNING ... línea 43 rechazada: sku
... INFO ... dry-run: 45 filas válidas; no se escribe en el destino
{"fecha": "2026-10-26", "leidas": 50, "validas": 45, "invalidas": 5, "columnas_descartadas": ["nombre_cliente", "rfc"], "cargadas": 0, "dry_run": true, "version": "1.0.0"}
```

Observa que los rechazos dicen **qué campo** falló, nunca el valor (el correo inválido no aparece en el log). Comprueba que no se creó la carpeta `salida/`.

**1.8.4** Carga dos veces y consulta el destino:

```bash
cargaventas cargar --fecha 2026-10-26
cargaventas cargar --fecha 2026-10-26
cargaventas resumen --fecha 2026-10-26
```

Resultado esperado: las dos cargas terminan con `"cargadas": 45` y el resumen muestra **45**, no 90:

```text
{"fecha": "2026-10-26", "filas": 45, "total": "604946.00"}
```

**1.8.5** Comprueba qué quedó en la base de datos:

```bash
python -c "import sqlite3; c = sqlite3.connect('salida/ventas.db'); print([r[1] for r in c.execute('PRAGMA table_info(ventas)')]); print(c.execute('SELECT venta_id, sku, total, cliente_id FROM ventas LIMIT 2').fetchall()); c.close()"
```

Resultado esperado:

```text
['venta_id', 'fecha', 'sku', 'cantidad', 'precio_unitario', 'total', 'cliente_id', 'version_carga']
[('V-000001', 'LAP-106', '5731.20', 'bb9f0721c5f8775b'), ('V-000002', 'TEC-075', '3345.40', 'aa54536d601db91e')]
```

No hay correo, nombre ni RFC: solo el seudónimo. El valor del seudónimo depende de la clave; con la clave de este laboratorio obtendrás los mismos.

**1.8.6** Recorre los códigos de error. Ejecuta cada caso y anota su código de salida:

| # | Caso | PowerShell | bash | Código | Línea clave |
| :---: | :--- | :--- | :--- | :---: | :--- |
| a | Fecha mal escrita | `cargaventas cargar --fecha 26/10/2026` | igual | 2 | `Invalid value for '--fecha': use el formato AAAA-MM-DD` |
| b | Ruta fuera de `datos/` | `cargaventas cargar --fecha 2026-10-26 --entrada config.toml` | igual | 2 | `solo se permiten archivos dentro de datos` |
| c | Lote con 40% inválidas | `cargaventas cargar --fecha 2026-10-27` | igual | 10 | `40.0% de filas inválidas supera el máximo de 20.0%: se rechaza el lote completo` |
| d | Día sin archivo | `cargaventas cargar --fecha 2026-10-25` | igual | 10 | `no existe el archivo de entrada ventas_2026-10-25.csv` |
| e | Destino caído | `$env:CARGAVENTAS_BD = "datos"; cargaventas cargar --fecha 2026-10-26; Remove-Item Env:CARGAVENTAS_BD` | `CARGAVENTAS_BD=datos cargaventas cargar --fecha 2026-10-26` | 20 | `destino: no se pudo cargar en datos: unable to open database file` |
| f | Entorno inválido | `$env:APP_ENV = "qa"; cargaventas cargar --fecha 2026-10-26; Remove-Item Env:APP_ENV` | `APP_ENV=qa cargaventas cargar --fecha 2026-10-26` | 3 | `entorno: Input should be 'dev', 'staging' or 'prod'` |

> En el caso **e** el "destino" es una carpeta, no un archivo de base de datos: SQLite no puede abrirlo. Es la forma más simple de simular un destino no disponible. En la Parte 3 verás uno más realista: la base de datos bloqueada por otro proceso.

**1.8.7** Comprueba que el secreto nunca aparece en la salida:

**PowerShell**
```powershell
cargaventas cargar --fecha 2026-10-26 2>&1 | Select-String $env:CLAVE_PSEUDONIMOS
```

**bash**
```bash
cargaventas cargar --fecha 2026-10-26 2>&1 | grep -c "$CLAVE_PSEUDONIMOS"
```

Resultado esperado: ninguna línea (en bash, `0`).

> **Checkpoint 1.** `cargaventas --version` responde `1.0.0`; anotaste los seis códigos de la tabla; `resumen` muestra 45 filas. Haz commit: `git add . && git commit -m "Paquete y CLI con contrato"` (en PowerShell 5.1 usa `;` en lugar de `&&`).

---

## Parte 2. El contenedor con Docker (35 min)

El entorno virtual aísla paquetes de Python, pero no el intérprete ni el sistema operativo. La imagen empaqueta todo eso y es **inmutable**: la que pasó las pruebas es la que corre en producción (6.16).

### 2.1 Qué entra a la imagen y cómo se construye

`.dockerignore` deja fuera del contexto de construcción lo que nunca debe llegar a la imagen: `.git`, el entorno, los datos y, sobre todo, `.env`:

**Archivo:** `.dockerignore`

```text
# Lo que nunca entra al contexto de construcción de la imagen
.git
.github
.venv/
**/__pycache__/
.pytest_cache/
.mypy_cache/
.ruff_cache/
.coverage
build/
dist/
*.egg-info/
datos/
salida/
dags/
tests/
scripts/
airflow_home/
.env
.env.*
*.pem
```

El `Dockerfile` tiene dos etapas:

- **builder:** descarga las dependencias **verificando los hashes** del lockfile (`--require-hashes`) y construye el wheel del paquete.
- **final:** instala solo esos wheels sin acceso a PyPI (`--no-index`), sin el código fuente, con un usuario sin privilegios (uid 10001, sin shell) y con la versión y el commit como etiquetas OCI.

**Archivo:** `Dockerfile`

```dockerfile
# syntax=docker/dockerfile:1
# ---------- Etapa 1: dependencias verificadas por hash + wheel del paquete ----------
FROM python:3.14-slim AS builder
ENV PIP_DISABLE_PIP_VERSION_CHECK=1 PIP_ROOT_USER_ACTION=ignore
WORKDIR /src
COPY requirements.txt ./
RUN pip wheel --no-cache-dir --require-hashes --wheel-dir /wheels -r requirements.txt
COPY pyproject.toml README.md ./
COPY src/ src/
RUN pip wheel --no-cache-dir --no-deps --wheel-dir /wheels .

# ---------- Etapa 2: imagen final, sin código fuente ni herramientas de build ----------
FROM python:3.14-slim
ARG VERSION=dev
ARG GIT_SHA=sin-sha
LABEL org.opencontainers.image.title="cargaventas" \
      org.opencontainers.image.version=$VERSION \
      org.opencontainers.image.revision=$GIT_SHA
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1 GIT_SHA=$GIT_SHA \
    PIP_DISABLE_PIP_VERSION_CHECK=1 PIP_ROOT_USER_ACTION=ignore
RUN groupadd --gid 10001 app && \
    useradd --uid 10001 --gid app --no-create-home --shell /usr/sbin/nologin app
COPY --from=builder /wheels /wheels
RUN pip install --no-cache-dir --no-index --find-links=/wheels cargaventas \
    && rm -rf /wheels
WORKDIR /app
COPY config.toml ./
# salida/ pertenece a app: un volumen nuevo montado ahí hereda ese dueño
RUN mkdir -p datos salida && chown app:app salida
USER app
ENTRYPOINT ["cargaventas"]
CMD ["--help"]
```

### 2.2 Construir

**PowerShell**
```powershell
$sha = git rev-parse --short=7 HEAD
docker build --build-arg VERSION=1.0.0 --build-arg GIT_SHA=$sha -t cargaventas:1.0.0 .
```

**bash**
```bash
docker build --build-arg VERSION=1.0.0 --build-arg GIT_SHA="$(git rev-parse --short=7 HEAD)" -t cargaventas:1.0.0 .
```

Resultado esperado: termina con `naming to docker.io/library/cargaventas:1.0.0`.

### 2.3 Inspeccionar la imagen

```bash
docker run --rm cargaventas:1.0.0 --version
docker run --rm --entrypoint id cargaventas:1.0.0
docker inspect cargaventas:1.0.0 --format "{{json .Config.Labels}}"
docker run --rm --entrypoint ls cargaventas:1.0.0 -a /app /
```

Resultado esperado:

```text
cargaventas, version 1.0.0
uid=10001(app) gid=10001(app) groups=10001(app)
{"org.opencontainers.image.revision":"<tu commit>","org.opencontainers.image.title":"cargaventas","org.opencontainers.image.version":"1.0.0"}
```

y en el listado: `/app` contiene solo `config.toml`, `datos` y `salida`; en `/` no existen `src` ni `wheels`. El contenedor **es** la CLI (`ENTRYPOINT`): lo que escribes después del nombre de la imagen son sus argumentos.

### 2.4 El secreto llega desde fuera

Crea `.env` a partir de la plantilla. `.env` está en `.gitignore` y en `.dockerignore`: nunca llega al repositorio ni a la imagen.

**PowerShell**
```powershell
(Get-Content .env.example) -replace '^CLAVE_PSEUDONIMOS=.*', 'CLAVE_PSEUDONIMOS=clave-de-laboratorio-0123456789' | Set-Content -Encoding ascii .env
```

**bash**
```bash
sed 's/^CLAVE_PSEUDONIMOS=.*/CLAVE_PSEUDONIMOS=clave-de-laboratorio-0123456789/' .env.example > .env
```

> En PowerShell, `-Encoding ascii` es importante: con la codificación por defecto de Windows PowerShell 5.1 (UTF-16), `docker --env-file` no reconoce la variable.

### 2.5 Ejecutar la carga en el contenedor

Los datos se montan **de solo lectura** y la base de datos vive en un **volumen con nombre** (`cargaventas-salida`): Docker lo crea con el dueño de `/app/salida` en la imagen (uid 10001), así funciona igual en Windows, macOS y Linux. Además se endurece la ejecución: sistema de archivos de solo lectura, sin capacidades del kernel y sin escalar privilegios.

**PowerShell**
```powershell
docker run --rm --env-file .env --read-only --tmpfs /tmp --cap-drop ALL `
  --security-opt no-new-privileges `
  -v "${PWD}\datos:/app/datos:ro" -v cargaventas-salida:/app/salida `
  cargaventas:1.0.0 cargar --fecha 2026-10-26
```

**bash**
```bash
docker run --rm --env-file .env --read-only --tmpfs /tmp --cap-drop ALL \
  --security-opt no-new-privileges \
  -v "$PWD/datos:/app/datos:ro" -v cargaventas-salida:/app/salida \
  cargaventas:1.0.0 cargar --fecha 2026-10-26
```

Resultado esperado (código **0**): las mismas líneas de la Parte 1, ahora con `commit=<tu commit>` en lugar de `sin-sha`, y la última línea con `"cargadas": 45`. Consulta el volumen:

```bash
docker run --rm --env-file .env -v cargaventas-salida:/app/salida cargaventas:1.0.0 resumen --fecha 2026-10-26
```

Resultado esperado: `{"fecha": "2026-10-26", "filas": 45, "total": "604946.00"}`, el mismo total que en tu equipo.

### 2.6 El código de salida atraviesa el contenedor

Ejecuta cada caso y anota el código (`$LASTEXITCODE` o `echo $?`). En PowerShell cambia `$PWD/datos` por `${PWD}\datos`:

| Caso | Comando | Código | Línea clave |
| :--- | :--- | :---: | :--- |
| Sin `--env-file` | `docker run --rm -v "$PWD/datos:/app/datos:ro" cargaventas:1.0.0 cargar --fecha 2026-10-26` | 3 | `falta la variable de entorno CLAVE_PSEUDONIMOS` |
| Lote rechazado | `docker run --rm --env-file .env -v "$PWD/datos:/app/datos:ro" -v cargaventas-salida:/app/salida cargaventas:1.0.0 cargar --fecha 2026-10-27` | 10 | `se rechaza el lote completo` |
| Destino de solo lectura | `docker run --rm --env-file .env -e CARGAVENTAS_BD=/app/datos/ventas.db -v "$PWD/datos:/app/datos:ro" cargaventas:1.0.0 cargar --fecha 2026-10-26` | 20 | `no se pudo cargar en /app/datos/ventas.db: unable to open database file` |
| Escribir en la imagen | `docker run --rm --entrypoint python cargaventas:1.0.0 -c "open('/app/x', 'w')"` | 1 | `PermissionError: [Errno 13] Permission denied: '/app/x'` |

El orquestador no necesita saber Python: lee el código de salida del contenedor. Eso es lo que harán el DAG y ECS.

### 2.7 La prueba de humo de la imagen (bash)

Este script repite las verificaciones de 2.3 y 2.6 y es el que usa CI antes de publicar la imagen:

**Archivo:** `scripts/probar_imagen.sh`

```bash
#!/usr/bin/env bash
# Prueba de humo de la imagen: versión, usuario, etiquetas y contrato de salida.
# Uso: bash scripts/probar_imagen.sh cargaventas:1.0.0 1.0.0
set -euo pipefail
IMAGEN="${1:?indique la imagen, p. ej. cargaventas:1.0.0}"
VERSION="${2:?indique la versión esperada, p. ej. 1.0.0}"
TMP="$(mktemp -d)"
VOLUMEN="cargaventas-prueba-$$"
trap 'rm -rf "$TMP"; docker volume rm -f "$VOLUMEN" > /dev/null' EXIT

fallar() { echo "  ✗ $*"; exit 1; }
codigo() { set +e; "$@" > /dev/null 2>&1; local c=$?; set -e; echo "$c"; }

echo "== versión y usuario"
[[ "$(docker run --rm "$IMAGEN" --version)" == "cargaventas, version $VERSION" ]] \
  || fallar "la imagen no reporta la versión $VERSION"
[[ "$(docker run --rm --entrypoint id "$IMAGEN" -u)" == "10001" ]] || fallar "corre como root"
[[ "$(docker inspect "$IMAGEN" --format '{{ index .Config.Labels "org.opencontainers.image.version" }}')" == "$VERSION" ]] \
  || fallar "la etiqueta OCI de versión no coincide"

echo "== contrato de salida"
mkdir -p "$TMP/datos"
chmod 755 "$TMP" "$TMP/datos"
docker run --rm -v "$TMP/datos:/datos" --user "$(id -u)" "$IMAGEN" \
  generar-ejemplo --fecha 2026-10-26 --carpeta /datos > /dev/null
echo "CLAVE_PSEUDONIMOS=clave-de-prueba-de-imagen-123" > "$TMP/.env"
CORRER=(docker run --rm --read-only --tmpfs /tmp --cap-drop ALL
        -v "$TMP/datos:/app/datos:ro" -v "$VOLUMEN:/app/salida")
[[ "$(codigo "${CORRER[@]}" "$IMAGEN" cargar --fecha 2026-10-26)" == 3 ]] || fallar "sin secreto != 3"
[[ "$(codigo "${CORRER[@]}" --env-file "$TMP/.env" "$IMAGEN" cargar --fecha 26/10/2026)" == 2 ]] \
  || fallar "fecha inválida != 2"
[[ "$(codigo "${CORRER[@]}" --env-file "$TMP/.env" "$IMAGEN" cargar --fecha 2026-10-25)" == 10 ]] \
  || fallar "sin archivo != 10"
[[ "$(codigo "${CORRER[@]}" --env-file "$TMP/.env" -e CARGAVENTAS_BD=/app/datos/x.db \
      "$IMAGEN" cargar --fecha 2026-10-26)" == 20 ]] || fallar "destino caído != 20"
[[ "$(codigo "${CORRER[@]}" --env-file "$TMP/.env" "$IMAGEN" cargar --fecha 2026-10-26)" == 0 ]] \
  || fallar "carga válida != 0"
echo "  ✓ imagen $IMAGEN lista (versión $VERSION, uid 10001, códigos 0/2/3/10/20)"
```

**bash** (Linux, macOS o Git Bash en Windows):

```bash
bash scripts/probar_imagen.sh cargaventas:1.0.0 1.0.0
```

Resultado esperado:

```text
== versión y usuario
== contrato de salida
  ✓ imagen cargaventas:1.0.0 lista (versión 1.0.0, uid 10001, códigos 0/2/3/10/20)
```

> En Windows sin Git Bash, las secciones 2.3 y 2.6 ya hicieron las mismas comprobaciones a mano. En CI este script corre en Linux.

> **Checkpoint 2.** La imagen reporta `1.0.0`, corre como uid 10001, el volumen tiene 45 filas y anotaste los códigos 3, 10, 20 y 1. Commit: `git add . && git commit -m "Imagen multi-stage sin root"`.

---

## Parte 3. El DAG para Amazon MWAA (25 min)

En MWAA el trabajo pesado no corre en el worker de Airflow: el DAG **lanza el contenedor** (ECS Fargate) y espera su código de salida (6.8, 6.18). En tu equipo, el mismo DAG lanza el contenedor con Docker. El DAG no instala `cargaventas`: solo conoce la imagen y el contrato.

### 3.1 Los ejecutores (sin Airflow)

Este módulo vive junto al DAG, en `dags/`, y no importa Airflow: por eso se puede probar con pytest. Tiene dos ejecutores con el mismo contrato:

- **docker:** `docker run` endurecido, con el secreto por `--env-file` (no pasa por Airflow), los datos de solo lectura y un tiempo máximo.
- **ecs:** `run_task` en Fargate con el rol de ejecución de MWAA (sin llaves), espera con el waiter `tasks_stopped` y lee el `exitCode` del contenedor. El secreto lo inyecta la task definition desde Secrets Manager.

**Archivo:** `dags/cargaventas_ejecutores.py`

```python
"""Ejecutores del DAG carga_ventas: corren la imagen cargaventas y aplican el contrato.

No importa Airflow, así que se prueba con pytest sin instalarlo. El DAG lo
importa desde la misma carpeta dags/ (Airflow y MWAA la agregan al sys.path).
"""

import json
import logging
import re
import shutil
import subprocess
from collections.abc import Mapping, Sequence
from typing import Any

log = logging.getLogger(__name__)

# Copia del contrato de cargaventas.contrato: el entorno de Airflow no instala
# el paquete, solo lanza la imagen. tests/test_dag.py comprueba que coincidan.
EXITO = 0
REINTENTABLES = frozenset({20})


class FalloReintentable(RuntimeError):
    """Fallo transitorio: el orquestador debe reintentar."""


class FalloDefinitivo(RuntimeError):
    """Reintentar no ayuda: la tarea falla sin consumir reintentos."""


def decidir(codigo: int) -> str:
    if codigo == EXITO:
        return "exito"
    if codigo in REINTENTABLES:
        return "reintentar"
    return "fallar_sin_reintento"


def aplicar_contrato(codigo: int, detalle: str) -> None:
    decision = decidir(codigo)
    if decision == "reintentar":
        raise FalloReintentable(f"código {codigo}: {detalle}")
    if decision == "fallar_sin_reintento":
        raise FalloDefinitivo(f"código {codigo}: {detalle}")


def ultima_linea_json(texto: str) -> dict[str, Any]:
    """La última línea de stdout es el resultado (como en BashOperator)."""
    lineas = [linea for linea in texto.splitlines() if linea.strip()]
    if not lineas:
        return {}
    try:
        valor = json.loads(lineas[-1])
    except json.JSONDecodeError:
        return {}
    return valor if isinstance(valor, dict) else {}


def nombre_seguro(texto: str) -> str:
    """Nombre válido para un contenedor de Docker o el startedBy de ECS."""
    return re.sub(r"[^a-zA-Z0-9_.-]", "-", texto)[:63]


def comando_docker(cfg: Mapping[str, str], argumentos: Sequence[str], nombre: str) -> list[str]:
    docker = shutil.which("docker") or "docker"
    return [
        docker, "run", "--rm", "--name", nombre,
        "--env-file", cfg["env_file"],               # el secreto no pasa por Airflow
        "--read-only", "--tmpfs", "/tmp",            # noqa: S108 (tmpfs propio)
        "--cap-drop", "ALL", "--security-opt", "no-new-privileges",
        "-v", f"{cfg['datos']}:/app/datos:ro",       # la entrada, solo lectura
        "-v", f"{cfg['salida']}:/app/salida",        # volumen con la base de datos
        cfg["imagen"], *argumentos,
    ]  # fmt: skip


def ejecutar_docker(
    cfg: Mapping[str, str],
    argumentos: Sequence[str],
    nombre: str,
    timeout_s: int = 1800,
) -> dict[str, Any]:
    comando = comando_docker(cfg, argumentos, nombre)
    try:
        proceso = subprocess.run(  # noqa: S603 (lista fija, sin shell)
            comando, capture_output=True, text=True, timeout=timeout_s, check=False
        )
    except FileNotFoundError as exc:
        raise FalloDefinitivo("Docker no está instalado en este worker") from exc
    except subprocess.TimeoutExpired as exc:
        limpieza = [comando[0], "rm", "-f", nombre]  # no dejar el contenedor colgado
        subprocess.run(limpieza, capture_output=True, check=False)  # noqa: S603
        raise FalloReintentable(f"el contenedor superó {timeout_s} s") from exc
    # Los logs del contenedor (stderr) pasan al log de la tarea de Airflow
    for linea in proceso.stderr.splitlines():
        log.info("[contenedor] %s", linea)
    ultima = proceso.stderr.strip().splitlines()[-1:] or ["sin detalle"]
    aplicar_contrato(proceso.returncode, ultima[0])
    return ultima_linea_json(proceso.stdout)


def ejecutar_ecs(
    cfg: Mapping[str, str],
    argumentos: Sequence[str],
    nombre: str,
    cliente: Any = None,
    espera_s: int = 15,
) -> dict[str, Any]:
    """Lanza la tarea en ECS Fargate, espera su fin y lee el código del contenedor."""
    if cliente is None:
        import boto3  # disponible en MWAA; usa el rol de ejecución, sin llaves

        cliente = boto3.client("ecs")
    respuesta = cliente.run_task(
        cluster=cfg["cluster"],
        taskDefinition=cfg["task_definition"],  # inyecta el secreto (Secrets Manager)
        launchType="FARGATE",
        count=1,
        startedBy=nombre[:36],
        networkConfiguration={
            "awsvpcConfiguration": {
                "subnets": cfg["subredes"].split(","),
                "securityGroups": cfg["grupos_seguridad"].split(","),
                "assignPublicIp": "DISABLED",
            }
        },
        overrides={
            "containerOverrides": [{"name": cfg["contenedor"], "command": list(argumentos)}]
        },
    )
    if respuesta.get("failures") or not respuesta.get("tasks"):
        raise FalloReintentable(f"ECS no inició la tarea: {respuesta.get('failures')}")
    arn = respuesta["tasks"][0]["taskArn"]
    cliente.get_waiter("tasks_stopped").wait(
        cluster=cfg["cluster"],
        tasks=[arn],
        WaiterConfig={"Delay": espera_s, "MaxAttempts": 240},
    )
    tarea = cliente.describe_tasks(cluster=cfg["cluster"], tasks=[arn])["tasks"][0]
    contenedor: dict[str, Any] = next(
        (c for c in tarea["containers"] if c["name"] == cfg["contenedor"]), {}
    )
    codigo = contenedor.get("exitCode")
    if codigo is None:  # no llegó a ejecutar (imagen, red, capacidad): transitorio
        raise FalloReintentable(f"la tarea terminó sin código: {tarea.get('stoppedReason')}")
    aplicar_contrato(codigo, f"tarea {arn}")
    return {"ejecutor": "ecs", "tarea": arn}  # las métricas quedan en CloudWatch


def ejecutar(cfg: Mapping[str, str], argumentos: Sequence[str], nombre: str) -> dict[str, Any]:
    ejecutor = cfg.get("ejecutor", "docker")
    if ejecutor == "docker":
        return ejecutar_docker(cfg, argumentos, nombre_seguro(nombre))
    if ejecutor == "ecs":
        return ejecutar_ecs(cfg, argumentos, nombre_seguro(nombre))
    raise FalloDefinitivo(f"ejecutor desconocido: {ejecutor}")
```

> El contrato aparece aquí **copiado** (`REINTENTABLES = {20}`) porque el entorno de Airflow no instala el paquete. En la Parte 4 una **prueba de contrato** comprueba que la copia y `cargaventas.contrato` deciden lo mismo para cada código: si alguien cambia uno y olvida el otro, CI falla.

### 3.2 El DAG

**Archivo:** `dags/carga_ventas.py`

```python
"""DAG carga_ventas (Airflow 3.x / Amazon MWAA): validar, cargar, resumir y avisar.

Las tareas pesadas corren la imagen cargaventas (Docker en local, ECS Fargate en
MWAA). El código de salida decide: 0 éxito, 20 reintento, cualquier otro falla
sin reintentar. La configuración vive en la Variable "cargaventas" (JSON).
"""

import logging
import os
from datetime import UTC, datetime, timedelta
from typing import Any

from airflow.sdk import Variable, dag, task, teardown

from cargaventas_ejecutores import FalloDefinitivo, ejecutar

try:
    from airflow.sdk.exceptions import AirflowFailException  # Airflow 3.2+
except ImportError:  # Airflow 3.0 y 3.1
    from airflow.exceptions import AirflowFailException

log = logging.getLogger(__name__)

# Espera base entre reintentos; el laboratorio la acorta con esta variable de entorno
ESPERA = timedelta(seconds=int(os.environ.get("CARGAVENTAS_REINTENTO_S", "120")))


def correr(argumentos: list[str], ti: Any) -> dict[str, Any]:
    # La Variable se lee al ejecutar la tarea, no al parsear el archivo
    cfg = Variable.get("cargaventas", deserialize_json=True)
    nombre = f"cv-{ti.task_id}-{ti.run_id}-{ti.try_number}"
    try:
        return ejecutar(cfg, argumentos, nombre)
    except FalloDefinitivo as exc:  # 10, 3, 2, 1...: no se gastan reintentos
        raise AirflowFailException(str(exc)) from exc
    # FalloReintentable (20) se propaga: Airflow reintenta según "retries"


@dag(
    schedule="30 2 * * 1-5",  # lunes a viernes 02:30 (zona de start_date: UTC)
    start_date=datetime(2026, 10, 1, tzinfo=UTC),
    catchup=False,
    max_active_runs=1,  # nunca dos cargas encimadas
    tags=["ventas", "cargaventas"],
    default_args={
        "retries": 3,
        "retry_delay": ESPERA,
        "retry_exponential_backoff": True,  # ≈2, 4 y 8 min (con variación)
        "max_retry_delay": timedelta(minutes=20),
        "execution_timeout": timedelta(minutes=45),
    },
)
def carga_ventas() -> None:
    @task(retries=0)  # validar es determinista: si falla, reintentar no cambia nada
    def validar(ds: str | None = None, ti: Any = None) -> dict[str, Any]:
        return correr(["cargar", "--fecha", str(ds), "--dry-run"], ti)

    @task
    def cargar(ds: str | None = None, ti: Any = None) -> dict[str, Any]:
        return correr(["cargar", "--fecha", str(ds)], ti)

    @task
    def publicar_resumen(validacion: dict[str, Any], carga: dict[str, Any]) -> str:
        mensaje = (
            f"ventas {carga.get('fecha')}: {carga.get('cargadas')} filas cargadas, "
            f"{validacion.get('invalidas')} rechazadas, versión {carga.get('version')}"
        )
        log.info(mensaje)
        return mensaje

    @task(trigger_rule="one_failed")  # solo corre si validar o cargar fallan
    def alertar(ds: str | None = None) -> None:
        # En producción: SNS, Teams o Slack. Aquí, un log de nivel ERROR.
        log.error("ALERTA: falló la carga de ventas del %s; revise los logs", ds)

    @teardown  # corre siempre, como all_done, pero NO oculta el fallo del DagRun
    def cerrar(ds: str | None = None) -> None:
        log.info("ejecución del %s terminada", ds)

    validacion = validar()
    carga = cargar()
    validacion >> carga
    resumen = publicar_resumen(validacion, carga)
    aviso = alertar()
    [validacion, carga] >> aviso
    [resumen, aviso] >> cerrar()


carga_ventas()
```

Puntos clave:

| Elemento | Por qué |
| :--- | :--- |
| `Variable.get` **dentro** de `correr` | El scheduler parsea el archivo cada pocos segundos; leer Variables al parsear sería una consulta a la base de metadatos en cada pasada |
| `validar` con `retries=0` | Validar es determinista: si los datos están mal, reintentar no los arregla |
| `AirflowFailException` para 1, 2, 3 y 10 | La tarea falla **sin consumir** los reintentos |
| `FalloReintentable` (20) se propaga | Airflow la marca `up_for_retry` y espera ≈2, 4 y 8 min (en el laboratorio, segundos) |
| `alertar` con `trigger_rule="one_failed"` | Solo corre si `validar` o `cargar` fallan; si todo sale bien queda `skipped` |
| `cerrar` como `@teardown` | Corre siempre, como `all_done`, pero **no oculta el fallo**: con `all_done` en la última tarea, el DagRun quedaría en `success` aunque la carga fallara |
| `max_active_runs=1`, `catchup=False` | Nunca dos cargas encimadas ni reprocesos masivos al activar el DAG |

Crea también el ejemplo de la Variable que usará la parte opcional:

**Archivo:** `variable_cargaventas.example.json`

```json
{
  "ejecutor": "docker",
  "imagen": "cargaventas:1.0.0",
  "env_file": "/RUTA/ABSOLUTA/practica6_despliegue/.env",
  "datos": "/RUTA/ABSOLUTA/practica6_despliegue/datos",
  "salida": "cargaventas-salida"
}
```

### 3.3 Lo que hará cada tarea, sin instalar Airflow

Este script llama al **mismo ejecutor** que usa el DAG y traduce el resultado a lo que haría Airflow:

**Archivo:** `scripts/probar_ejecutor.py`

```python
"""Corre la imagen con el MISMO ejecutor que usa el DAG, sin instalar Airflow.

Uso: python scripts/probar_ejecutor.py 2026-10-26 [--bloquear]
  --bloquear  otro contenedor bloquea la base de datos 15 s: simula un destino caído
"""

import shutil
import subprocess
import sys
import time
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parents[1] / "dags"))
from cargaventas_ejecutores import FalloDefinitivo, FalloReintentable, ejecutar  # noqa: E402

IMAGEN = "cargaventas:1.0.0"
BLOQUEO = (
    "import sqlite3, time; c = sqlite3.connect('/app/salida/ventas.db'); "
    "c.execute('BEGIN EXCLUSIVE'); time.sleep(15)"
)


def main(fecha: str, bloquear: bool) -> int:
    docker = shutil.which("docker") or "docker"
    cfg = {  # lo mismo que la Variable "cargaventas" del DAG
        "ejecutor": "docker",
        "imagen": IMAGEN,
        "env_file": str(Path(".env").resolve()),
        "datos": str(Path("datos").resolve()),
        "salida": "cargaventas-salida",
    }
    if bloquear:
        subprocess.run(  # noqa: S603 (lista fija, sin shell)
            [docker, "run", "-d", "--rm", "--name", "cv-bloqueo", "-v",
             "cargaventas-salida:/app/salida", "--entrypoint", "python", IMAGEN, "-c", BLOQUEO],
            check=True, capture_output=True,
        )  # fmt: skip
        time.sleep(2)  # da tiempo a que tome el bloqueo
    try:
        metricas = ejecutar(cfg, ["cargar", "--fecha", fecha], f"cv-prueba-{fecha}")
        print(f"ÉXITO -> la tarea queda en success; XCom: {metricas}")
        return 0
    except FalloReintentable as exc:
        print(f"REINTENTAR -> Airflow marca up_for_retry y lo vuelve a intentar\n  {exc}")
        return 1
    except FalloDefinitivo as exc:
        print(f"FALLA SIN REINTENTO -> AirflowFailException: failed\n  {exc}")
        return 1
    finally:
        if bloquear:
            subprocess.run([docker, "rm", "-f", "cv-bloqueo"], capture_output=True, check=False)  # noqa: S603


if __name__ == "__main__":
    sys.stdout.reconfigure(errors="replace")
    sys.exit(main(sys.argv[1], "--bloquear" in sys.argv[2:]))
```

Ejecuta los tres escenarios:

```bash
python scripts/probar_ejecutor.py 2026-10-26
python scripts/probar_ejecutor.py 2026-10-27
python scripts/probar_ejecutor.py 2026-10-26 --bloquear
```

Resultado esperado:

```text
ÉXITO -> la tarea queda en success; XCom: {'fecha': '2026-10-26', 'leidas': 50, 'validas': 45, 'invalidas': 5, 'columnas_descartadas': ['nombre_cliente', 'rfc'], 'cargadas': 45, 'dry_run': False, 'version': '1.0.0'}

FALLA SIN REINTENTO -> AirflowFailException: failed
  código 10: ... validación: 40.0% de filas inválidas supera el máximo de 20.0%: se rechaza el lote completo

REINTENTAR -> Airflow marca up_for_retry y lo vuelve a intentar
  código 20: ... destino: no se pudo cargar en salida/ventas.db: database is locked
```

En el tercer caso, otro contenedor tiene la base de datos bloqueada durante 15 segundos: un fallo **transitorio**. Gracias a la transacción, no quedó nada a medias; si vuelves a ejecutar el primer comando cuando termine el bloqueo, carga sin problema. Eso es exactamente lo que hace un reintento.

### 3.4 Del laboratorio a MWAA

| Pieza | En tu equipo | En Amazon MWAA |
| :--- | :--- | :--- |
| DAG y ejecutores | `dags/` local | `s3://<bucket>/dags/` (los sube `entrega.yml`) |
| Variable `cargaventas` | `AIRFLOW_VAR_CARGAVENTAS` (parte opcional) | Variable de Airflow o Secrets Manager como backend |
| Ejecutor | `"ejecutor": "docker"` | `"ejecutor": "ecs"` con `cluster`, `task_definition` (la familia: usa la última revisión activa), `contenedor`, `subredes` y `grupos_seguridad` |
| Secreto | `.env` en el equipo | `secrets` de la task definition desde Secrets Manager |
| Permisos | Tu usuario | Rol de ejecución de MWAA: `ecs:RunTask`, `ecs:DescribeTasks` y `iam:PassRole` solo sobre los roles de la tarea |
| Logs | Terminal | CloudWatch (los del contenedor llegan con versión y commit) |

Ejemplo de la Variable en MWAA (no se usa en el laboratorio):

```json
{"ejecutor": "ecs", "cluster": "lotes", "task_definition": "cargaventas",
 "contenedor": "cargaventas", "subredes": "subnet-0a1,subnet-0b2", "grupos_seguridad": "sg-0c3"}
```

> **Checkpoint 3.** Los tres escenarios de 3.3 dan éxito, falla sin reintento y reintentar. Commit: `git add . && git commit -m "DAG y ejecutores"`.

---

## Parte 4. Pruebas y calidad (25 min)

### 4.1 La suite

`conftest.py` crea para cada prueba una carpeta temporal con su `config.toml`, su secreto y su CSV: las pruebas no dependen de tu carpeta `datos/` ni del orden.

**Archivo:** `tests/conftest.py`

```python
"""Cada prueba corre en una carpeta temporal con su config.toml y su secreto."""

import shutil
import sqlite3
from collections.abc import Iterator
from contextlib import contextmanager
from datetime import date
from pathlib import Path

import pytest

from cargaventas.ejemplo import escribir_csv, generar

RAIZ = Path(__file__).resolve().parents[1]
FECHA = date(2026, 10, 26)
CLAVE = "clave-de-prueba-0123456789"


@contextmanager
def abrir_bd(ruta: Path) -> Iterator[sqlite3.Connection]:
    """Conexión con commit al salir y cierre garantizado."""
    conexion = sqlite3.connect(ruta)
    try:
        with conexion:
            yield conexion
    finally:
        conexion.close()


@pytest.fixture
def proyecto(tmp_path: Path, monkeypatch: pytest.MonkeyPatch) -> Path:
    shutil.copy(RAIZ / "config.toml", tmp_path / "config.toml")
    monkeypatch.chdir(tmp_path)
    for variable in (
        "APP_ENV",
        "CARGAVENTAS_DATOS",
        "CARGAVENTAS_BD",
        "CARGAVENTAS_CONFIG",
    ):
        monkeypatch.delenv(variable, raising=False)
    monkeypatch.setenv("CLAVE_PSEUDONIMOS", CLAVE)
    escribir_csv(tmp_path / "datos" / "ventas_2026-10-26.csv", generar(FECHA, 50, 10, 7))
    return tmp_path
```

Pruebas de la CLI con `CliRunner`: códigos, stdout/stderr, idempotencia, datos protegidos y secreto ausente de los logs:

**Archivo:** `tests/test_cli.py`

```python
"""La CLI respeta el contrato: código de salida, logs en stderr y JSON en stdout."""

import json
from pathlib import Path

import pytest
from click.testing import CliRunner

from cargaventas import __version__
from cargaventas.cli import cli
from cargaventas.contrato import Codigo
from tests.conftest import CLAVE, abrir_bd


def correr(*argumentos: str) -> tuple[int, str, str]:
    resultado = CliRunner().invoke(cli, list(argumentos))
    return resultado.exit_code, resultado.stdout, resultado.stderr


def test_version_sale_de_los_metadatos() -> None:
    codigo, salida, _ = correr("--version")
    assert codigo == 0
    assert salida.strip() == f"cargaventas, version {__version__}"


def test_carga_exitosa_ultima_linea_json(proyecto: Path) -> None:
    codigo, salida, errores = correr("cargar", "--fecha", "2026-10-26")
    assert codigo == Codigo.EXITO
    metricas = json.loads(salida.strip().splitlines()[-1])
    assert metricas["leidas"] == 50
    assert metricas["validas"] == 45
    assert metricas["cargadas"] == 45
    assert metricas["columnas_descartadas"] == ["nombre_cliente", "rfc"]
    assert "carga confirmada" in errores  # logs en stderr, no en stdout


def test_carga_es_idempotente(proyecto: Path) -> None:
    for _ in range(2):
        assert correr("cargar", "--fecha", "2026-10-26")[0] == 0
    with abrir_bd(proyecto / "salida" / "ventas.db") as conexion:
        assert conexion.execute("SELECT COUNT(*) FROM ventas").fetchone() == (45,)


def test_datos_protegidos_no_llegan_al_destino(proyecto: Path) -> None:
    correr("cargar", "--fecha", "2026-10-26")
    with abrir_bd(proyecto / "salida" / "ventas.db") as conexion:
        columnas = [c[1] for c in conexion.execute("PRAGMA table_info(ventas)")]
        volcado = "\n".join(conexion.iterdump())
    assert "email_cliente" not in columnas and "rfc" not in columnas
    assert "@ejemplo.com" not in volcado and "XAXX" not in volcado


def test_dry_run_no_escribe(proyecto: Path) -> None:
    codigo, salida, _ = correr("cargar", "--fecha", "2026-10-26", "--dry-run")
    assert codigo == 0
    assert json.loads(salida)["dry_run"] is True
    assert not (proyecto / "salida").exists()


def test_xcom_file(proyecto: Path, monkeypatch: pytest.MonkeyPatch) -> None:
    destino = proyecto / "xcom" / "return.json"
    monkeypatch.setenv("XCOM_FILE", str(destino))
    correr("cargar", "--fecha", "2026-10-26")
    assert json.loads(destino.read_text(encoding="utf-8"))["cargadas"] == 45


@pytest.mark.parametrize(
    ("argumentos", "esperado"),
    [
        (["cargar", "--fecha", "26/10/2026"], Codigo.ERROR_USO),
        (["cargar"], Codigo.ERROR_USO),
        (
            ["cargar", "--fecha", "2026-10-26", "--entrada", "config.toml"],
            Codigo.ERROR_USO,
        ),
        (
            ["cargar", "--fecha", "2026-10-25"],
            Codigo.ERROR_VALIDACION,
        ),  # no hay archivo
    ],
)
def test_codigos_de_uso_y_validacion(proyecto: Path, argumentos: list[str], esperado: int) -> None:
    assert correr(*argumentos)[0] == esperado


def test_sin_secreto_es_error_de_configuracion(
    proyecto: Path, monkeypatch: pytest.MonkeyPatch
) -> None:
    monkeypatch.delenv("CLAVE_PSEUDONIMOS")
    codigo, _, errores = correr("cargar", "--fecha", "2026-10-26")
    assert codigo == Codigo.ERROR_CONFIGURACION
    assert "CLAVE_PSEUDONIMOS" in errores


def test_destino_caido_es_reintentable(proyecto: Path, monkeypatch: pytest.MonkeyPatch) -> None:
    monkeypatch.setenv("CARGAVENTAS_BD", str(proyecto / "datos"))  # una carpeta, no una BD
    codigo, salida, errores = correr("cargar", "--fecha", "2026-10-26")
    assert codigo == Codigo.ERROR_DESTINO
    assert salida == ""  # sin resultado: el orquestador no recibe métricas falsas
    assert "unable to open database file" in errores


def test_lote_con_demasiadas_invalidas(proyecto: Path) -> None:
    codigo, salida, _ = correr("generar-ejemplo", "--fecha", "2026-10-27", "--pct-invalidas", "40")
    assert codigo == 0 and "40% inválidas" in salida
    codigo, _, errores = correr("cargar", "--fecha", "2026-10-27")
    assert codigo == Codigo.ERROR_VALIDACION
    assert "se rechaza el lote completo" in errores


def test_error_inesperado_devuelve_1(proyecto: Path, monkeypatch: pytest.MonkeyPatch) -> None:
    def explotar(*args: object, **kwargs: object) -> None:
        raise RuntimeError("defecto")

    monkeypatch.setattr("cargaventas.cli.ejecutar", explotar)
    codigo, _, errores = correr("cargar", "--fecha", "2026-10-26")
    assert codigo == Codigo.ERROR_INESPERADO
    assert "error inesperado" in errores


def test_el_secreto_nunca_aparece_en_los_logs(proyecto: Path) -> None:
    _, salida, errores = correr("cargar", "--fecha", "2026-10-26")
    assert CLAVE not in salida + errores


def test_resumen(proyecto: Path, monkeypatch: pytest.MonkeyPatch) -> None:
    assert correr("resumen", "--fecha", "2026-10-26")[0] == Codigo.ERROR_DESTINO  # sin BD
    correr("cargar", "--fecha", "2026-10-26")
    codigo, salida, _ = correr("resumen", "--fecha", "2026-10-26")
    assert codigo == 0
    assert json.loads(salida)["filas"] == 45
    monkeypatch.delenv("CLAVE_PSEUDONIMOS")
    assert correr("resumen", "--fecha", "2026-10-26")[0] == Codigo.ERROR_CONFIGURACION
```

Configuración estricta, ruta segura, seudónimo y **rollback a mitad del lote**: un trigger de SQLite hace fallar la fila `V-000040` y la prueba comprueba que ninguna fila quedó con la versión nueva:

**Archivo:** `tests/test_config_y_proceso.py`

```python
"""Configuración estricta, ruta segura, minimización y transacción todo o nada."""

from pathlib import Path

import pytest

from cargaventas.config import ErrorConfiguracion, cargar_config
from cargaventas.proceso import (
    ErrorDestino,
    ErrorValidacion,
    RutaNoPermitida,
    cargar_bd,
    ejecutar,
    ruta_segura,
    seudonimo,
)
from tests.conftest import CLAVE, FECHA, abrir_bd


def test_entorno_sobrescribe_valores_no_sensibles(proyecto: Path) -> None:
    config = cargar_config(
        {"CLAVE_PSEUDONIMOS": CLAVE, "APP_ENV": "prod", "CARGAVENTAS_BD": "otra/bd.db"}
    )
    assert config.entorno == "prod"
    assert config.salida.base_datos == Path("otra/bd.db")
    assert CLAVE not in repr(config)  # SecretStr: no se imprime


@pytest.mark.parametrize(
    ("entorno", "mensaje"),
    [
        ({"CLAVE_PSEUDONIMOS": "corta"}, "clave_pseudonimos"),
        ({"CLAVE_PSEUDONIMOS": CLAVE, "APP_ENV": "qa"}, "entorno"),
        (
            {"CLAVE_PSEUDONIMOS": CLAVE, "CARGAVENTAS_CONFIG": "no-existe.toml"},
            "no existe",
        ),
    ],
)
def test_configuracion_invalida_falla_rapido(
    proyecto: Path, entorno: dict[str, str], mensaje: str
) -> None:
    with pytest.raises(ErrorConfiguracion, match=mensaje) as error:
        cargar_config(entorno)
    assert "corta" not in str(error.value)  # nunca se muestra el valor recibido


def test_claves_desconocidas_y_toml_roto(proyecto: Path) -> None:
    config = proyecto / "config.toml"
    config.write_text(config.read_text() + "\ndebug = true\n", encoding="utf-8")
    with pytest.raises(ErrorConfiguracion, match="debug"):
        cargar_config({"CLAVE_PSEUDONIMOS": CLAVE})
    config.write_text("entorno = ", encoding="utf-8")
    with pytest.raises(ErrorConfiguracion, match="TOML"):
        cargar_config({"CLAVE_PSEUDONIMOS": CLAVE})


def test_ruta_segura_bloquea_salir_de_la_carpeta(proyecto: Path) -> None:
    datos = proyecto / "datos"
    assert ruta_segura(datos, datos / "ventas_2026-10-26.csv").name == "ventas_2026-10-26.csv"
    with pytest.raises(RutaNoPermitida):
        ruta_segura(datos, datos / ".." / "config.toml")


def test_seudonimo_estable_y_sin_correo() -> None:
    a = seudonimo("Cliente1@Ejemplo.com", b"k" * 16)
    assert a == seudonimo("cliente1@ejemplo.com", b"k" * 16)
    assert a != seudonimo("cliente1@ejemplo.com", b"x" * 16)  # depende de la clave
    assert "@" not in a and len(a) == 16


def test_rollback_si_falla_a_mitad_del_lote(proyecto: Path) -> None:
    config = cargar_config()
    ejecutar(FECHA, None, dry_run=False, config=config, version="1.0.0")
    bd = config.salida.base_datos
    with abrir_bd(bd) as conexion:
        conexion.execute("UPDATE ventas SET version_carga = '0.9.0'")  # estado previo
        conexion.execute(  # simula que el destino falla en la fila V-000040
            "CREATE TRIGGER falla BEFORE UPDATE ON ventas "
            "WHEN NEW.venta_id = 'V-000040' "
            "BEGIN SELECT RAISE(ABORT, 'fallo simulado del destino'); END"
        )
    with pytest.raises(ErrorDestino, match="fallo simulado"):
        ejecutar(FECHA, None, dry_run=False, config=config, version="1.1.0")
    with abrir_bd(bd) as conexion:  # ninguna fila quedó con la versión nueva
        versiones = conexion.execute("SELECT DISTINCT version_carga FROM ventas").fetchall()
    assert versiones == [("0.9.0",)]


def test_cargar_bd_error_de_sistema_de_archivos(proyecto: Path) -> None:
    (proyecto / "archivo").write_text("x", encoding="utf-8")
    with pytest.raises(ErrorDestino):
        cargar_bd(proyecto / "archivo" / "ventas.db", [], b"k" * 16, "1.0.0")


@pytest.mark.parametrize(
    ("contenido", "mensaje"),
    [
        ("venta_id,fecha\nV-000001,2026-10-26\n", "faltan columnas obligatorias: sku"),
        ("venta_id,fecha,sku,cantidad,precio_unitario,email_cliente\n", "no tiene filas"),
    ],
)
def test_archivos_que_se_rechazan(proyecto: Path, contenido: str, mensaje: str) -> None:
    (proyecto / "datos" / "ventas_2026-10-26.csv").write_text(contenido, encoding="utf-8")
    with pytest.raises(ErrorValidacion, match=mensaje):
        ejecutar(FECHA, None, dry_run=True, config=cargar_config(), version="1.0.0")


def test_limite_de_filas(proyecto: Path) -> None:
    config = cargar_config()
    limitada = config.model_copy(
        update={"entrada": config.entrada.model_copy(update={"max_filas": 10})}
    )
    with pytest.raises(ErrorValidacion, match="más de 10 filas"):
        ejecutar(FECHA, None, dry_run=True, config=limitada, version="1.0.0")
```

Logs, redacción de secretos y el contrato:

**Archivo:** `tests/test_registro_y_contrato.py`

```python
"""Logs con versión y commit, redacción de secretos y contrato de códigos."""

import io
import logging

import pytest

from cargaventas.contrato import Codigo, interpretar
from cargaventas.registro import configurar_registro


def test_logs_llevan_version_y_commit(monkeypatch: pytest.MonkeyPatch) -> None:
    monkeypatch.setenv("GIT_SHA", "9c3f5a1e0b2d")
    flujo = io.StringIO()
    configurar_registro("1.2.3", flujo=flujo)
    logging.getLogger("cargaventas.prueba").info("hola")
    assert "v=1.2.3 commit=9c3f5a1 cargaventas.prueba: hola" in flujo.getvalue()


@pytest.mark.parametrize(
    "mensaje",
    [
        "Authorization: Bearer eyJhbGciOi.abc.def",
        "conexión con password=Sup3rS3creta",
        "llave AKIAIOSFODNN7EXAMPLE en el código",
        "url postgresql://etl:Sup3rS3creta@db:5432/ventas",
        "valor clave-real-de-pseudonimos-123",
    ],
)
def test_redaccion_de_secretos(mensaje: str) -> None:
    flujo = io.StringIO()
    configurar_registro("1.0.0", secretos=["clave-real-de-pseudonimos-123"], flujo=flujo)
    logging.getLogger("cargaventas").warning("%s", mensaje)
    escrito = flujo.getvalue()
    for secreto in ("eyJhbGciOi", "Sup3rS3creta", "IOSFODNN7", "clave-real"):
        assert secreto not in escrito
    assert "***" in escrito


@pytest.mark.parametrize(
    ("codigo", "decision"),
    [
        (Codigo.EXITO, "exito"),
        (Codigo.ERROR_DESTINO, "reintentar"),
        (Codigo.ERROR_VALIDACION, "fallar_sin_reintento"),
        (Codigo.ERROR_CONFIGURACION, "fallar_sin_reintento"),
        (Codigo.ERROR_USO, "fallar_sin_reintento"),
        (Codigo.ERROR_INESPERADO, "fallar_sin_reintento"),
        (137, "fallar_sin_reintento"),  # p. ej. el contenedor murió por memoria
    ],
)
def test_interpretar(codigo: int, decision: str) -> None:
    assert interpretar(codigo) == decision


def test_ningun_codigo_es_99() -> None:
    assert 99 not in {c.value for c in Codigo}  # 99 = "skipped" en BashOperator
```

El DAG **sin Airflow**: la prueba de contrato, la estructura del DAG por análisis estático (`ast`), el ejecutor de Docker con `subprocess` simulado y el de ECS con `botocore.Stubber` (sin AWS real):

**Archivo:** `tests/test_dag.py`

```python
"""El DAG y sus ejecutores, probados SIN instalar Airflow."""

import ast
import json
import subprocess
from pathlib import Path
from typing import Any

import boto3
import pytest
from botocore.stub import Stubber

import cargaventas_ejecutores as ej
from cargaventas.contrato import Codigo, interpretar

RAIZ = Path(__file__).resolve().parents[1]
DAG = RAIZ / "dags" / "carga_ventas.py"
CFG_DOCKER = {
    "ejecutor": "docker",
    "imagen": "cargaventas:1.0.0",
    "env_file": "/srv/cargaventas/.env",
    "datos": "/srv/cargaventas/datos",
    "salida": "cargaventas-salida",
}
CFG_ECS = {
    "ejecutor": "ecs",
    "cluster": "lotes",
    "task_definition": "cargaventas:7",
    "contenedor": "cargaventas",
    "subredes": "subnet-a,subnet-b",
    "grupos_seguridad": "sg-1",
}


# ---------- prueba de contrato: el DAG y el paquete deciden lo mismo ----------
@pytest.mark.parametrize("codigo", [*Codigo, 125, 137])
def test_el_dag_interpreta_igual_que_el_paquete(codigo: int) -> None:
    assert ej.decidir(codigo) == interpretar(codigo)


# ---------- estructura del DAG (análisis estático, sin importar Airflow) ----------
def _llamadas(arbol: ast.AST, nombre: str) -> list[ast.Call]:
    return [
        n
        for n in ast.walk(arbol)
        if isinstance(n, ast.Call)
        and (getattr(n.func, "id", None) == nombre or getattr(n.func, "attr", None) == nombre)
    ]


def test_estructura_del_dag() -> None:
    arbol = ast.parse(DAG.read_text(encoding="utf-8"))
    (decorador,) = _llamadas(arbol, "dag")
    args = {k.arg: k.value for k in decorador.keywords}
    assert ast.literal_eval(args["schedule"]) == "30 2 * * 1-5"
    assert ast.literal_eval(args["catchup"]) is False
    assert ast.literal_eval(args["max_active_runs"]) == 1
    reglas = {
        ast.literal_eval(k.value)
        for c in _llamadas(arbol, "task")
        for k in c.keywords
        if k.arg == "trigger_rule"
    }
    assert reglas == {"one_failed"}
    assert "teardown" in DAG.read_text(encoding="utf-8")


def test_variables_solo_dentro_de_funciones() -> None:
    arbol = ast.parse(DAG.read_text(encoding="utf-8"))
    nivel_modulo = [n for n in arbol.body if not isinstance(n, ast.FunctionDef)]
    for nodo in nivel_modulo:  # leer Variables al parsear = una consulta cada 30 s
        for llamada in _llamadas(nodo, "get"):
            assert ast.unparse(llamada.func) != "Variable.get", ast.unparse(nodo)


# ---------- ejecutor Docker ----------
def test_comando_docker_endurecido() -> None:
    comando = ej.comando_docker(CFG_DOCKER, ["cargar", "--fecha", "2026-10-26"], "cv-x")
    texto = " ".join(comando)
    for opcion in ("--rm", "--read-only", "--cap-drop ALL", "no-new-privileges"):
        assert opcion in texto
    assert "/srv/cargaventas/datos:/app/datos:ro" in comando
    assert comando[-4:] == ["cargaventas:1.0.0", "cargar", "--fecha", "2026-10-26"]


def _simular(monkeypatch: pytest.MonkeyPatch, codigo: int, stdout: str = "") -> None:
    def falso(comando: list[str], **kwargs: Any) -> subprocess.CompletedProcess[str]:
        return subprocess.CompletedProcess(comando, codigo, stdout, "INFO línea\nERROR motivo")

    monkeypatch.setattr(ej.subprocess, "run", falso)


def test_docker_exito_devuelve_metricas(monkeypatch: pytest.MonkeyPatch) -> None:
    _simular(monkeypatch, 0, 'algo\n{"cargadas": 45}\n')
    assert ej.ejecutar(CFG_DOCKER, ["cargar"], "cv:x/1") == {"cargadas": 45}


@pytest.mark.parametrize(
    ("codigo", "excepcion"), [(20, ej.FalloReintentable), (10, ej.FalloDefinitivo)]
)
def test_docker_aplica_el_contrato(
    monkeypatch: pytest.MonkeyPatch, codigo: int, excepcion: type[Exception]
) -> None:
    _simular(monkeypatch, codigo)
    with pytest.raises(excepcion, match="ERROR motivo"):
        ej.ejecutar(CFG_DOCKER, ["cargar"], "cv-x")


def test_docker_tiempo_agotado_es_reintentable(monkeypatch: pytest.MonkeyPatch) -> None:
    llamadas: list[list[str]] = []

    def falso(comando: list[str], **kwargs: Any) -> subprocess.CompletedProcess[str]:
        llamadas.append(comando)
        if "run" in comando:
            raise subprocess.TimeoutExpired(comando, 1)
        return subprocess.CompletedProcess(comando, 0, "", "")

    monkeypatch.setattr(ej.subprocess, "run", falso)
    with pytest.raises(ej.FalloReintentable):
        ej.ejecutar(CFG_DOCKER, ["cargar"], "cv-x")
    assert llamadas[-1][1:] == ["rm", "-f", "cv-x"]  # no deja el contenedor colgado


def test_sin_docker_es_definitivo(monkeypatch: pytest.MonkeyPatch) -> None:
    def falso(comando: list[str], **kwargs: Any) -> None:
        raise FileNotFoundError(comando[0])

    monkeypatch.setattr(ej.subprocess, "run", falso)
    with pytest.raises(ej.FalloDefinitivo, match="Docker"):
        ej.ejecutar(CFG_DOCKER, ["cargar"], "cv-x")


def test_ejecutor_desconocido() -> None:
    with pytest.raises(ej.FalloDefinitivo, match="desconocido"):
        ej.ejecutar({"ejecutor": "k8s"}, ["cargar"], "cv-x")


@pytest.mark.parametrize(
    ("texto", "esperado"),
    [("", {}), ("no es json", {}), ("[1, 2]", {}), ('x\n{"a": 1}', {"a": 1})],
)
def test_ultima_linea_json(texto: str, esperado: dict[str, Any]) -> None:
    assert ej.ultima_linea_json(texto) == esperado


# ---------- ejecutor ECS (botocore Stubber: sin AWS real) ----------
ARN = "arn:aws:ecs:us-east-1:123456789012:task/lotes/abc"


def _ecs(contenedores: list[dict[str, Any]], failures: list[Any] | None = None) -> Any:
    cliente = boto3.client(
        "ecs", region_name="us-east-1", aws_access_key_id="x", aws_secret_access_key="x"
    )
    stub = Stubber(cliente)
    tareas = [] if failures else [{"taskArn": ARN}]
    stub.add_response("run_task", {"tasks": tareas, "failures": failures or []})
    if not failures:
        tarea = {
            "taskArn": ARN,
            "lastStatus": "STOPPED",
            "containers": contenedores,
            "stoppedReason": "Essential container exited",
        }
        detenida = {"tasks": [tarea]}
        stub.add_response("describe_tasks", detenida)  # la usa el waiter
        stub.add_response("describe_tasks", detenida)  # la usa ejecutar_ecs
    stub.activate()
    return cliente


def test_ecs_exito() -> None:
    cliente = _ecs([{"name": "cargaventas", "exitCode": 0}])
    resultado = ej.ejecutar_ecs(CFG_ECS, ["cargar"], "cv-x", cliente=cliente, espera_s=0)
    assert resultado == {"ejecutor": "ecs", "tarea": ARN}


@pytest.mark.parametrize(
    ("contenedores", "excepcion"),
    [
        ([{"name": "cargaventas", "exitCode": 20}], ej.FalloReintentable),
        ([{"name": "cargaventas", "exitCode": 10}], ej.FalloDefinitivo),
        ([{"name": "cargaventas"}], ej.FalloReintentable),  # nunca arrancó
    ],
)
def test_ecs_aplica_el_contrato(
    contenedores: list[dict[str, Any]], excepcion: type[Exception]
) -> None:
    cliente = _ecs(contenedores)
    with pytest.raises(excepcion):
        ej.ejecutar_ecs(CFG_ECS, ["cargar"], "cv-x", cliente=cliente, espera_s=0)


def test_ecs_sin_capacidad_es_reintentable() -> None:
    cliente = _ecs([], failures=[{"arn": "x", "reason": "RESOURCE:MEMORY"}])
    with pytest.raises(ej.FalloReintentable, match="RESOURCE:MEMORY"):
        ej.ejecutar_ecs(CFG_ECS, ["cargar"], "cv-x", cliente=cliente, espera_s=0)


def test_variable_de_ejemplo_es_json_valido() -> None:
    ejemplo = json.loads((RAIZ / "variable_cargaventas.example.json").read_text())
    assert set(CFG_DOCKER) <= set(ejemplo)
```

### 4.2 Ejecutar la misma secuencia que CI

```bash
ruff check .
ruff format --check .
mypy
python -m pytest --cov --cov-report=term-missing
```

Resultado esperado:

```text
All checks passed!
19 files already formatted
Success: no issues found in 10 source files
...
TOTAL                              401      8     70      2    98%
Required test coverage of 85.0% reached. Total coverage: 97.88%
============================== 69 passed in 0.58s ==============================
```

Con Python 3.12 y 3.13 el total cambia un poco (`409` sentencias y `97.91%`) porque el intérprete cuenta distinto algunas líneas; lo que importa es que supere el umbral de 85%.

> Si `ruff format --check .` reporta archivos, ejecuta `ruff format .` y vuelve a comprobar: el formato es parte del contrato del equipo.

### 4.3 Romper el contrato a propósito

Abre `dags/cargaventas_ejecutores.py` y cambia `REINTENTABLES = frozenset({20})` por `REINTENTABLES = frozenset({10, 20})` (alguien decidió "reintentar también los datos inválidos"). Ejecuta:

```bash
python -m pytest tests/test_dag.py -q
```

Resultado esperado: `3 failed, 24 passed`:

```text
FAILED tests/test_dag.py::test_el_dag_interpreta_igual_que_el_paquete[10] - A...
FAILED tests/test_dag.py::test_docker_aplica_el_contrato[10-FalloDefinitivo]
FAILED tests/test_dag.py::test_ecs_aplica_el_contrato[contenedores1-FalloDefinitivo]
```

La primera es la prueba de contrato: el DAG ya no decide lo mismo que el paquete para el código 10. Las otras dos muestran la consecuencia: un lote inválido se reintentaría tres veces, con la misma respuesta cada vez. **Revierte el cambio** y confirma `27 passed`.

> **Checkpoint 4.** `69 passed`, cobertura ≥ 85% (≈98%), Ruff y mypy sin hallazgos. Commit: `git add . && git commit -m "Suite de pruebas y calidad"`.

---

## Parte 5. El artefacto: wheel verificado y dependencias auditadas (15 min)

### 5.1 El verificador y el script de construcción

**Archivo:** `scripts/verificar_wheel.py`

```python
"""Revisa un wheel antes de publicarlo: contenido prohibido, versión y entry points.

Uso: python scripts/verificar_wheel.py dist/cargaventas-1.0.0-py3-none-any.whl 1.0.0
"""

import fnmatch
import sys
import zipfile
from email.parser import Parser
from pathlib import Path

PROHIBIDOS = ["*.env", "*.env.*", ".env*", "*.pem", "*.key", "tests/*", "*__pycache__*", "*.db"]


def verificar_wheel(ruta: Path, version_esperada: str) -> list[str]:
    problemas = []
    with zipfile.ZipFile(ruta) as wheel:
        nombres = wheel.namelist()
        for nombre in nombres:
            if any(fnmatch.fnmatch(nombre, patron) for patron in PROHIBIDOS):
                problemas.append(f"archivo prohibido dentro del wheel: {nombre}")
        meta_path = next(n for n in nombres if n.endswith(".dist-info/METADATA"))
        meta = Parser().parsestr(wheel.read(meta_path).decode())
        puntos = next((n for n in nombres if n.endswith(".dist-info/entry_points.txt")), None)
        comandos = wheel.read(puntos).decode() if puntos else ""
    if meta["Version"] != version_esperada:
        problemas.append(f"versión {meta['Version']} != esperada {version_esperada}")
    if "cargaventas = cargaventas.cli:main" not in comandos:
        problemas.append("no declara el comando cargaventas (¿falta [project.scripts]?)")
    print(f"{ruta.name}: {len(nombres)} archivos, versión {meta['Version']}")
    return problemas


if __name__ == "__main__":
    sys.stdout.reconfigure(errors="replace")  # ✓ y ✗ no rompen consolas sin UTF-8
    ruta, version = Path(sys.argv[1]), sys.argv[2]
    problemas = verificar_wheel(ruta, version)
    for problema in problemas:
        print("  ✗", problema)
    print("  ✓ listo para publicar" if not problemas else "  bloqueado")
    sys.exit(1 if problemas else 0)  # en CI, exit 1 detiene la publicación
```

**Archivo:** `scripts/build_package.sh`

```bash
#!/usr/bin/env bash
# Construye y valida los artefactos. Uso: bash scripts/build_package.sh 1.0.0
set -euo pipefail                      # primero: cualquier fallo detiene el script
VERSION_ESPERADA="${1:?indique la versión esperada, p. ej. 1.0.0}"

echo "=== [1/4] Limpiando artefactos previos ==="
rm -rf build/ dist/ src/*.egg-info

echo "=== [2/4] Construyendo sdist y wheel en un entorno aislado ==="
python -m build

echo "=== [3/4] Validando metadatos con twine ==="
twine check --strict dist/*

echo "=== [4/4] Revisando el contenido del wheel ==="
python scripts/verificar_wheel.py dist/*.whl "$VERSION_ESPERADA"
sha256sum dist/*                       # huellas para el registro de la entrega
```

### 5.2 Construir

**bash**
```bash
bash scripts/build_package.sh 1.0.0
```

**PowerShell** (los mismos cuatro pasos):
```powershell
Remove-Item -Recurse -Force build, dist -ErrorAction SilentlyContinue
python -m build
twine check --strict dist/*
python scripts/verificar_wheel.py (Get-Item dist/*.whl).FullName 1.0.0
Get-FileHash dist/* -Algorithm SHA256 | Format-Table Hash, Path
```

Resultado esperado (las huellas cambian en cada construcción porque el wheel guarda fechas):

```text
=== [1/4] Limpiando artefactos previos ===
=== [2/4] Construyendo sdist y wheel en un entorno aislado ===
Successfully built cargaventas-1.0.0.tar.gz and cargaventas-1.0.0-py3-none-any.whl
=== [3/4] Validando metadatos con twine ===
Checking dist/cargaventas-1.0.0-py3-none-any.whl: PASSED
Checking dist/cargaventas-1.0.0.tar.gz: PASSED
=== [4/4] Revisando el contenido del wheel ===
cargaventas-1.0.0-py3-none-any.whl: 14 archivos, versión 1.0.0
  ✓ listo para publicar
<sha256>  dist/cargaventas-1.0.0-py3-none-any.whl
<sha256>  dist/cargaventas-1.0.0.tar.gz
```

### 5.3 El verificador bloquea lo que no debe salir

**5.3.1** Versión equivocada (como si la etiqueta fuera `v1.0.1` y nadie actualizó `pyproject.toml`):

```bash
python scripts/verificar_wheel.py dist/cargaventas-1.0.0-py3-none-any.whl 1.0.1
```

Resultado esperado (código **1**): `✗ versión 1.0.0 != esperada 1.0.1` y `bloqueado`.

**5.3.2** Un `.env` que se coló en el wheel:

```bash
python -c "import os, shutil, zipfile; os.makedirs('build', exist_ok=True); shutil.copy('dist/cargaventas-1.0.0-py3-none-any.whl', 'build/manipulado.whl'); z = zipfile.ZipFile('build/manipulado.whl', 'a'); z.writestr('cargaventas/.env', 'CLAVE_PSEUDONIMOS=filtrada'); z.close()"
python scripts/verificar_wheel.py build/manipulado.whl 1.0.0
```

Resultado esperado (código **1**):

```text
manipulado.whl: 15 archivos, versión 1.0.0
  ✗ archivo prohibido dentro del wheel: cargaventas/.env
  bloqueado
```

### 5.4 El wheel funciona fuera del proyecto

Instálalo en un entorno nuevo, como lo haría un servidor o un registro privado (6.12):

**PowerShell**
```powershell
python -m venv $env:TEMP\vwheel
& "$env:TEMP\vwheel\Scripts\python.exe" -m pip install -q dist/cargaventas-1.0.0-py3-none-any.whl
& "$env:TEMP\vwheel\Scripts\cargaventas.exe" --version
```

**bash**
```bash
python -m venv /tmp/vwheel
/tmp/vwheel/bin/python -m pip install -q dist/cargaventas-1.0.0-py3-none-any.whl
/tmp/vwheel/bin/cargaventas --version
```

Resultado esperado: `cargaventas, version 1.0.0`.

### 5.5 Auditar las dependencias

```bash
pip-audit -r requirements.txt --require-hashes --strict
```

Resultado esperado: `No known vulnerabilities found` (código 0). Con `--strict`, cualquier paquete que no se pueda auditar también detiene el proceso (6.26, 6.35).

> **Checkpoint 5.** El wheel pasó la verificación, los dos casos manipulados se bloquearon y pip-audit no reportó vulnerabilidades. Commit: `git add . && git commit -m "Wheel verificado"` (`dist/` y `build/` no se versionan).

---

## Parte 6. CI/CD con GitHub Actions (25 min)

### 6.1 Integración continua: `ci.yml`

Cada push a `main` y cada pull request pasan por cuatro jobs: **calidad** (Ruff, mypy, pytest con el umbral de cobertura, actionlint y shellcheck, en Python 3.12, 3.13 y 3.14), **dependencias** (pip-audit sobre el lockfile con hashes), **paquete** (el mismo `build_package.sh`) e **imagen** (construcción y prueba de humo). El token solo puede leer (`contents: read`).

**Archivo:** `.github/workflows/ci.yml`

```yaml
name: CI - calidad, seguridad e imagen

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

permissions:
  contents: read                      # mínimo privilegio para el GITHUB_TOKEN

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true            # un push nuevo cancela la corrida anterior

jobs:
  calidad:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.12", "3.13", "3.14"]
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-python@v6
        with:
          python-version: ${{ matrix.python-version }}
          cache: pip
          cache-dependency-path: requirements*.txt
      - name: Instalar dependencias fijadas y el paquete
        run: |
          python -m pip install -r requirements.txt
          python -m pip install -r requirements-dev.txt
          python -m pip install --no-deps -e .
      - name: Lint y formato (Ruff)
        run: |
          ruff check .
          ruff format --check .
      - name: Tipos (mypy)
        run: mypy
      - name: Pruebas y cobertura (umbral en pyproject.toml)
        run: pytest --cov --cov-report=term-missing
      - name: Workflows y scripts (actionlint y shellcheck)
        run: |
          actionlint
          shellcheck scripts/*.sh

  dependencias:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: pypa/gh-action-pip-audit@v1.1.0
        with:
          inputs: requirements.txt    # el lockfile de ejecución
          require-hashes: true

  paquete:
    needs: calidad
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-python@v6
        with:
          python-version: "3.14"
      - name: Herramientas de construcción
        run: python -m pip install -r requirements-dev.txt
      - name: Construir y verificar el wheel
        run: |
          VERSION=$(python -c 'import tomllib; print(tomllib.load(open("pyproject.toml", "rb"))["project"]["version"])')
          bash scripts/build_package.sh "$VERSION"

  imagen:
    needs: [calidad, dependencias]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - name: Construir la imagen
        run: |
          VERSION=$(python3 -c 'import tomllib; print(tomllib.load(open("pyproject.toml", "rb"))["project"]["version"])')
          echo "VERSION=$VERSION" >> "$GITHUB_ENV"
          docker build --build-arg VERSION="$VERSION" --build-arg GIT_SHA="${GITHUB_SHA::7}" \
            -t "cargaventas:$VERSION" .
      - name: Prueba de humo de la imagen
        run: bash scripts/probar_imagen.sh "cargaventas:$VERSION" "$VERSION"
```

### 6.2 Entrega controlada: `entrega.yml`

Se dispara con una etiqueta `vX.Y.Z`, o a mano para **reentregar** una versión anterior con un motivo (reversión manual, 6.30).

**Archivo:** `.github/workflows/entrega.yml`

```yaml
name: Entrega - versión etiquetada a ECR, ECS y MWAA

on:
  push:
    tags: ["v*.*.*"]                  # git tag -a v1.0.0 && git push origin v1.0.0
  workflow_dispatch:                  # reentregar o revertir a una versión ya etiquetada
    inputs:
      etiqueta_git:
        description: "Etiqueta de Git a entregar, p. ej. v1.0.0"
        required: true
        type: string
      motivo:
        description: "Motivo (queda en el resumen de la ejecución)"
        required: true
        type: string

permissions:
  contents: read

concurrency:
  group: entrega-produccion
  cancel-in-progress: false           # una entrega nunca se corta a la mitad

env:
  ETIQUETA_GIT: ${{ inputs.etiqueta_git || github.ref_name }}
  AWS_REGION: us-east-1
  REPOSITORIO: cargaventas

jobs:
  construir:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.meta.outputs.version }}
      etiqueta: ${{ steps.meta.outputs.etiqueta }}
    steps:
      - uses: actions/checkout@v5
        with:
          ref: ${{ env.ETIQUETA_GIT }}
      - uses: actions/setup-python@v6
        with:
          python-version: "3.14"
      - id: meta
        name: Versión desde la etiqueta (y que coincida con pyproject.toml)
        run: |
          if [[ ! "$ETIQUETA_GIT" =~ ^v[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
            echo "::error::Etiqueta inválida: $ETIQUETA_GIT (se espera vMAYOR.MENOR.PARCHE)"
            exit 1
          fi
          VERSION="${ETIQUETA_GIT#v}"
          DECLARADA=$(python -c 'import tomllib; print(tomllib.load(open("pyproject.toml", "rb"))["project"]["version"])')
          if [[ "$VERSION" != "$DECLARADA" ]]; then
            echo "::error::La etiqueta $ETIQUETA_GIT no coincide con pyproject.toml ($DECLARADA)"
            exit 1
          fi
          echo "version=$VERSION" >> "$GITHUB_OUTPUT"
          echo "etiqueta=$VERSION-$(git rev-parse --short=7 HEAD)" >> "$GITHUB_OUTPUT"
      - name: Herramientas de construcción
        run: python -m pip install -r requirements-dev.txt
      - name: Construir y verificar el wheel
        run: bash scripts/build_package.sh "${{ steps.meta.outputs.version }}"
      - uses: actions/upload-artifact@v6
        with:
          name: dist-${{ steps.meta.outputs.etiqueta }}
          path: dist/
          retention-days: 30

  imagen:
    needs: construir
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write                 # OIDC: credenciales temporales, ninguna llave guardada
    outputs:
      imagen: ${{ steps.publicar.outputs.imagen }}
    steps:
      - uses: actions/checkout@v5
        with:
          ref: ${{ env.ETIQUETA_GIT }}
      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: ${{ vars.AWS_ROLE_PUBLICAR }}
          aws-region: ${{ env.AWS_REGION }}
      - id: ecr
        uses: aws-actions/amazon-ecr-login@v2
      - id: publicar
        name: Construir, probar y publicar la imagen (una sola vez por versión)
        env:
          REGISTRO: ${{ steps.ecr.outputs.registry }}
          VERSION: ${{ needs.construir.outputs.version }}
          ETIQUETA: ${{ needs.construir.outputs.etiqueta }}
        run: |
          IMAGEN="$REGISTRO/$REPOSITORIO:$ETIQUETA"
          echo "imagen=$IMAGEN" >> "$GITHUB_OUTPUT"
          if aws ecr describe-images --repository-name "$REPOSITORIO" \
               --image-ids imageTag="$ETIQUETA" > /dev/null 2>&1; then
            echo "La imagen $ETIQUETA ya está en ECR: se reutiliza (etiquetas inmutables)"
            exit 0
          fi
          docker build --build-arg VERSION="$VERSION" --build-arg GIT_SHA="${ETIQUETA##*-}" \
            -t "$IMAGEN" .
          bash scripts/probar_imagen.sh "$IMAGEN" "$VERSION"
          docker push "$IMAGEN"

  desplegar:
    needs: [construir, imagen]
    runs-on: ubuntu-latest
    environment: produccion           # revisores del entorno aprueban antes de seguir
    permissions:
      contents: read
      id-token: write
    env:
      CLUSTER: ${{ vars.ECS_CLUSTER }}
      FAMILIA: ${{ vars.ECS_TASK_FAMILY }}
      BUCKET_MWAA: ${{ vars.MWAA_BUCKET }}
      IMAGEN: ${{ needs.imagen.outputs.imagen }}
      VERSION: ${{ needs.construir.outputs.version }}
    steps:
      - uses: actions/checkout@v5
        with:
          ref: ${{ env.ETIQUETA_GIT }}
      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: ${{ vars.AWS_ROLE_DESPLEGAR }}
          aws-region: ${{ env.AWS_REGION }}
      - id: revision
        name: Registrar una revisión nueva de la task definition (solo cambia la imagen)
        run: |
          ANTERIOR=$(aws ecs describe-task-definition --task-definition "$FAMILIA" \
            --query 'taskDefinition.taskDefinitionArn' --output text)
          echo "anterior=$ANTERIOR" >> "$GITHUB_OUTPUT"
          aws ecs describe-task-definition --task-definition "$ANTERIOR" --query taskDefinition \
            | jq --arg img "$IMAGEN" '.containerDefinitions[0].image = $img
                | del(.taskDefinitionArn, .revision, .status, .requiresAttributes,
                      .compatibilities, .registeredAt, .registeredBy)' > nueva.json
          NUEVA=$(aws ecs register-task-definition --cli-input-json file://nueva.json \
            --query 'taskDefinition.taskDefinitionArn' --output text)
          echo "nueva=$NUEVA" >> "$GITHUB_OUTPUT"
      - name: Prueba de humo en ECS (la imagen arranca y reporta su versión)
        env:
          NUEVA: ${{ steps.revision.outputs.nueva }}
          SUBREDES: ${{ vars.ECS_SUBNETS }}
          GRUPOS: ${{ vars.ECS_SECURITY_GROUPS }}
        run: |
          RED="awsvpcConfiguration={subnets=[$SUBREDES],securityGroups=[$GRUPOS],assignPublicIp=DISABLED}"
          TAREA=$(aws ecs run-task --cluster "$CLUSTER" --task-definition "$NUEVA" \
            --launch-type FARGATE --network-configuration "$RED" \
            --overrides '{"containerOverrides":[{"name":"cargaventas","command":["--version"]}]}' \
            --query 'tasks[0].taskArn' --output text)
          aws ecs wait tasks-stopped --cluster "$CLUSTER" --tasks "$TAREA"
          CODIGO=$(aws ecs describe-tasks --cluster "$CLUSTER" --tasks "$TAREA" \
            --query 'tasks[0].containers[0].exitCode' --output text)
          if [[ "$CODIGO" != "0" ]]; then
            echo "::error::La tarea de prueba terminó con código $CODIGO"
            exit 1
          fi
      - name: Respaldar y publicar el DAG en el bucket de MWAA
        run: |
          mkdir -p respaldo
          aws s3 sync "s3://$BUCKET_MWAA/dags/" respaldo/ --exclude "*" \
            --include "carga_ventas.py" --include "cargaventas_ejecutores.py"
          aws s3 cp dags/cargaventas_ejecutores.py "s3://$BUCKET_MWAA/dags/"
          aws s3 cp dags/carga_ventas.py "s3://$BUCKET_MWAA/dags/"
      - name: Revertir si algo falló
        if: failure() && steps.revision.outputs.nueva != ''
        env:
          NUEVA: ${{ steps.revision.outputs.nueva }}
          ANTERIOR: ${{ steps.revision.outputs.anterior }}
        run: |
          # Al desactivar la revisión nueva, la última ACTIVA vuelve a ser la anterior
          aws ecs deregister-task-definition --task-definition "$NUEVA" > /dev/null
          if compgen -G "respaldo/*.py" > /dev/null; then
            aws s3 cp respaldo/ "s3://$BUCKET_MWAA/dags/" --recursive
          fi
          echo "::error::Entrega revertida: ECS vuelve a $ANTERIOR y el DAG al respaldo"
      - name: Registrar la entrega
        if: always()                  # se registra el intento, salga bien o mal
        env:
          MOTIVO: ${{ inputs.motivo || 'etiqueta publicada' }}   # vía env: sin inyección
          RESULTADO: ${{ job.status }}
        run: |
          echo "entrega version=$VERSION imagen=$IMAGEN resultado=$RESULTADO actor=$GITHUB_ACTOR motivo=$MOTIVO" \
            | tee -a "$GITHUB_STEP_SUMMARY"
```

| Paso | Control | Lección |
| :--- | :--- | :---: |
| La etiqueta debe coincidir con `pyproject.toml` | Una versión, un artefacto | 6.14, 6.27 |
| `id-token: write` + `configure-aws-credentials` | OIDC: credenciales temporales, ninguna llave en GitHub | 6.33 |
| Imagen `versión-SHA`; si ya existe en ECR, se reutiliza | Construir una vez, promover el mismo artefacto (etiquetas inmutables) | 6.27 |
| `environment: produccion` | Los revisores del entorno aprueban antes de tocar producción | 6.28 |
| Nueva revisión de la task definition, solo cambia la imagen | Despliegue reproducible y con estado anterior conocido | 6.29 |
| Prueba de humo en ECS (`--version` debe salir con 0) | Verificar antes de dar por buena la entrega | 6.29 |
| Respaldo del DAG antes de subir el nuevo | Camino de vuelta para el DAG | 6.30 |
| `if: failure()`: se desactiva la revisión nueva y se restaura el DAG | Reversión automática | 6.30 |
| `if: always()` + `GITHUB_STEP_SUMMARY`; el motivo entra por `env` | Auditoría sin inyección de comandos | 6.30, 6.33 |

> Las acciones usan versiones vigentes con Node 24 (`upload-artifact@v6`, `configure-aws-credentials@v6`). En las diapositivas verás `@v4`, que corre en Node 20.

### 6.3 Validar los workflows y los scripts

```bash
actionlint
shellcheck scripts/*.sh
```

Resultado esperado: ninguna salida y código 0 en ambos. actionlint revisa la sintaxis, las expresiones `${{ }}`, los contextos y, con shellcheck instalado, cada bloque `run:`.

### 6.4 Lo que actionlint detecta

Crea el archivo `.github/workflows/roto.yml` con tres errores comunes (no es parte del proyecto: lo borrarás enseguida):

```yaml
name: Ejemplo inseguro
on: pull_request
jobs:
  saludo:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: echo "Revisando ${{ github.event.pull_request.title }}"
      - run: echo "listo" >> $GITHUB_OUTPUT
```

```bash
actionlint
```

Resultado esperado (código **1**):

```text
.github/workflows/roto.yml:7:15: the runner of "actions/checkout@v3" action is too old to run on GitHub Actions. update the action's version to fix this issue [action]
.github/workflows/roto.yml:8:34: "github.event.pull_request.title" is potentially untrusted. avoid using it directly in inline scripts. instead, pass it through an environment variable. ... [expression]
.github/workflows/roto.yml:9:9: shellcheck reported issue in this script: SC2086:info:1:17: Double quote to prevent globbing and word splitting [shellcheck]
```

El segundo es el más grave: el título de un PR lo escribe cualquiera, y `${{ }}` se sustituye **antes** de que bash lo lea; un título como `x"; curl ... | sh; echo "` se ejecutaría en el runner. Por eso `entrega.yml` pasa `motivo` por `env`. **Borra** `.github/workflows/roto.yml` y confirma que `actionlint` vuelve a terminar sin salida.

### 6.5 Mínimo privilegio para la entrega

Cada identidad tiene solo lo que su paso necesita (6.36, 6.37). Así quedaría en AWS:

| Identidad | Quién la asume | Permisos (y sobre qué recurso) |
| :--- | :--- | :--- |
| `AWS_ROLE_PUBLICAR` | GitHub Actions por OIDC, solo desde `refs/tags/v*` del repositorio | `ecr:GetAuthorizationToken`; `ecr:DescribeImages`, `BatchCheckLayerAvailability`, `InitiateLayerUpload`, `UploadLayerPart`, `CompleteLayerUpload`, `PutImage` solo en el repositorio `cargaventas` |
| `AWS_ROLE_DESPLEGAR` | GitHub Actions por OIDC, solo desde el entorno `produccion` | `ecs:DescribeTaskDefinition`, `RegisterTaskDefinition`, `DeregisterTaskDefinition`, `RunTask`, `DescribeTasks`; `iam:PassRole` solo sobre los roles de la tarea; `s3:ListBucket`, `GetObject`, `PutObject` solo en `dags/` del bucket de MWAA |
| Rol de ejecución de MWAA | El entorno de MWAA | `ecs:RunTask` sobre la familia `cargaventas`, `ecs:DescribeTasks`, `iam:PassRole` sobre los roles de la tarea, logs de CloudWatch |
| Rol de la tarea de ECS | El contenedor | `secretsmanager:GetSecretValue` solo sobre el secreto de `CLAVE_PSEUDONIMOS` (rol de ejecución de la tarea) y escritura en su destino |

Y la configuración que `entrega.yml` espera en GitHub (**Settings → Secrets and variables → Actions → Variables** y **Environments**): `AWS_ROLE_PUBLICAR`, `AWS_ROLE_DESPLEGAR`, `ECS_CLUSTER`, `ECS_TASK_FAMILY`, `ECS_SUBNETS`, `ECS_SECURITY_GROUPS`, `MWAA_BUCKET` y el entorno `produccion` con revisores. Ninguna es un secreto: los permisos vienen de los roles.

### 6.6 (Opcional) Ver CI en GitHub

Si tienes cuenta de GitHub: crea un repositorio **privado** vacío, conéctalo y sube `main`:

```bash
git add . && git commit -m "Workflows de CI y entrega"
git remote add origin https://github.com/<tu-usuario>/practica6-despliegue.git
git push -u origin main
```

En la pestaña **Actions** verás `CI - calidad, seguridad e imagen` con sus cuatro jobs. No crees la etiqueta `v1.0.0`: sin las variables y los roles de AWS, `entrega.yml` fallaría en el paso de credenciales (y eso también es correcto: no entrega nada).

> **Checkpoint 6.** `actionlint` y `shellcheck` sin hallazgos, el ejercicio 6.4 comprobado y borrado, y el último commit hecho. Etiqueta la versión entregable: `git tag -a v1.0.0 -m "Primera versión desplegable"` y `git describe --tags` (responde `v1.0.0`).

---

## Preguntas de cierre (10 min)

1. **Contrato.** ¿Por qué `validar` usa `retries=0` y `cargar` `retries=3`? ¿Qué pasaría con un lote inválido si todos los errores se reintentaran?
2. **Estados.** En el día con 40% inválidas, ¿en qué estado queda cada tarea (`validar`, `cargar`, `publicar_resumen`, `alertar`, `cerrar`) y el DagRun? ¿Por qué `cerrar` es `teardown` y no `all_done`?
3. **Reversión.** `entrega.yml` revierte la task definition y el DAG. ¿Qué **no** revierte y cómo lo manejarías si una versión cambiara el esquema de la tabla `ventas` (expandir y contraer, 6.30)?
4. **Seguridad.** Enumera los lugares por donde podría filtrarse `CLAVE_PSEUDONIMOS` (repositorio, imagen, logs, Airflow, GitHub) y el control de esta práctica que cierra cada uno.

---

## Entregables

Comprime `practica6_despliegue` **sin** `.venv/`, `.env`, `datos/`, `salida/`, `build/`, `dist/` ni `airflow_home/` (o comparte el repositorio de GitHub). Antes de entregar, revisa:

- [ ] Paquete `src/cargaventas/` con 9 archivos `.py` e instalado: `cargaventas --version` responde `1.0.0`
- [ ] Tabla de códigos de 1.8.6 y 2.6 completada
- [ ] Imagen `cargaventas:1.0.0`: uid 10001, etiquetas OCI con tu commit, `probar_imagen.sh` en verde (o 2.3 y 2.6 a mano)
- [ ] `dags/carga_ventas.py` y `dags/cargaventas_ejecutores.py`; salida de los tres escenarios de 3.3
- [ ] `python -m pytest --cov`: `69 passed`, cobertura ≥ 85%; Ruff y mypy sin hallazgos; ejercicio 4.3 revertido
- [ ] Salida de `build_package.sh` (o los pasos de PowerShell) y de los dos casos bloqueados de 5.3
- [ ] `pip-audit` sin vulnerabilidades
- [ ] `.github/workflows/ci.yml` y `entrega.yml`; `actionlint` sin hallazgos
- [ ] Etiqueta `v1.0.0` (`git tag`) y respuestas a las preguntas de cierre

---

## Parte 7 (opcional). Airflow 3 en tu equipo (+30 min)

Ejecuta el DAG real con `airflow dags test`, que corre una ejecución completa en un solo proceso, sin scheduler ni servidor web. Airflow no corre de forma nativa en Windows: usa **Linux, macOS o WSL2** (con Docker Desktop integrado a WSL2). Usa una **terminal nueva** con su propio entorno, separado del proyecto.

**7.1** Instala Airflow 3.3.2 con su archivo de restricciones en un entorno **fuera del proyecto** (así Ruff y pytest no lo revisan). Cambia `3.13` por tu versión de Python (probado con 3.13 y 3.14):

```bash
python3 -m venv ~/venv-airflow
source ~/venv-airflow/bin/activate
python -m pip install --upgrade pip
python -m pip install "apache-airflow==3.3.2" --constraint https://raw.githubusercontent.com/apache/airflow/constraints-3.3.2/constraints-3.13.txt
airflow version
```

Resultado esperado: `3.3.2` (la instalación tarda unos minutos).

**7.2** Desde la carpeta del proyecto, configura Airflow **solo para esta terminal**: la carpeta de trabajo, la carpeta de DAG, la espera corta entre reintentos y la Variable `cargaventas` (las rutas deben ser absolutas):

```bash
export AIRFLOW_HOME="$PWD/airflow_home"
export AIRFLOW__CORE__DAGS_FOLDER="$PWD/dags"
export AIRFLOW__CORE__LOAD_EXAMPLES=False
export CARGAVENTAS_REINTENTO_S=2
export AIRFLOW_VAR_CARGAVENTAS="{\"ejecutor\": \"docker\", \"imagen\": \"cargaventas:1.0.0\", \"env_file\": \"$PWD/.env\", \"datos\": \"$PWD/datos\", \"salida\": \"cargaventas-salida\"}"
airflow db migrate
airflow dags reserialize
airflow dags list
airflow tasks list carga_ventas
```

Resultado esperado: `carga_ventas` en la lista (pausado, que es lo normal) y sus cinco tareas: `alertar`, `cargar`, `cerrar`, `publicar_resumen`, `validar`.

**7.3 Escenario A: día normal.**

```bash
airflow dags test carga_ventas 2026-10-26
```

Resultado esperado, entre muchas líneas de log: `[contenedor] ... carga confirmada: 45 filas en salida/ventas.db`, `ventas 2026-10-26: 45 filas cargadas, 5 rechazadas, versión 1.0.0` y al final `DagRun Finished: ... state=success`.

**7.4 Escenario B: lote rechazado.**

```bash
airflow dags test carga_ventas 2026-10-27
airflow dags list-runs carga_ventas
```

`airflow dags test` termina con código 1 y la lista muestra la nueva ejecución en `failed`. Copia su `run_id` (empieza con `manual__`) y consulta el estado de cada tarea:

```bash
airflow tasks states-for-dag-run carga_ventas "<run_id>"
```

Resultado esperado:

| Tarea | Estado | Por qué |
| :--- | :--- | :--- |
| `validar` | `failed` | Código 10 → `AirflowFailException`: sin reintentos |
| `cargar` | `upstream_failed` | Su dependencia falló: nunca tocó el destino |
| `publicar_resumen` | `upstream_failed` | Ídem |
| `alertar` | `success` | `one_failed`: una de sus anteriores falló |
| `cerrar` | `success` | Teardown: corre siempre |
| **DagRun** | **`failed`** | El teardown no oculta el fallo |

**7.5 Escenario C: el reintento resuelve un fallo transitorio.** En **otra terminal**, en la carpeta del proyecto, bloquea la base de datos 20 segundos:

```bash
docker run --rm -v cargaventas-salida:/app/salida --entrypoint python cargaventas:1.0.0 -c "import sqlite3, time; c = sqlite3.connect('/app/salida/ventas.db'); c.execute('BEGIN EXCLUSIVE'); print('bloqueada', flush=True); time.sleep(20)"
```

En cuanto veas `bloqueada`, vuelve a la terminal de Airflow:

```bash
airflow dags test carga_ventas 2026-10-26
```

Resultado esperado: `validar` en `success` (el dry-run no toca el destino); `cargar` registra `código 20: ... database is locked` y `new_state=up_for_retry`; en el siguiente intento, ya sin bloqueo, `carga confirmada: 45 filas` y el DagRun termina en `success`. Según el momento, puede necesitar uno o dos reintentos. Repite `resumen` (Parte 2.5): siguen siendo 45 filas.

---

## Solución de problemas

| Síntoma | Causa probable | Solución |
| :--- | :--- | :--- |
| `ERROR: Hashes are required in --require-hashes mode` | Instalaste `requirements.txt` y `requirements-dev.txt` en el mismo comando, o editaste el lockfile a mano | Instálalos en comandos separados (0.6); no edites el lockfile |
| `THESE PACKAGES DO NOT MATCH THE HASHES` | Un proxy o espejo entregó otro archivo | No lo ignores: revisa con tu equipo de redes; el control funcionó |
| `cargaventas: command not found` / `no se reconoce` | El entorno no está activo o falta `pip install --no-deps -e .` | Activa `.venv` e instala el paquete (1.7) |
| `falta la variable de entorno CLAVE_PSEUDONIMOS` en una terminal nueva | La variable solo vive en la terminal donde la definiste | Vuelve a definirla (1.8.3) |
| `Cannot connect to the Docker daemon` / `error during connect` | Docker Desktop no está abierto | Ábrelo y espera a que diga *Engine running* |
| `docker run` no ve la clave aunque `.env` existe | `.env` se guardó en UTF-16 (PowerShell 5.1) | Créalo con el comando de 2.4 (`-Encoding ascii`) |
| `invalid mount config` o la carpeta montada aparece vacía | Ruta relativa o sintaxis de bash en PowerShell | En PowerShell usa `"${PWD}\datos:/app/datos:ro"` |
| `database is locked` sin haber lanzado el bloqueo | Quedó vivo el contenedor `cv-bloqueo` o el de 7.5 | `docker rm -f cv-bloqueo` o espera a que termine |
| `bash\r: No such file or directory` en CI | El script se guardó con fin de línea CRLF | `.gitattributes` lo evita; si ya ocurrió: `git add --renormalize .` y commit |
| `actionlint`: `no project was found` | La carpeta no es un repositorio de Git | `git init` (0.7) |
| La instalación de `actionlint-py` tarda o falla | Descarga el binario de actionlint desde GitHub al instalarse | Revisa el acceso a github.com o instala actionlint con tu gestor de paquetes |
| `airflow: command not found` o errores de Airflow en Windows | Airflow no corre de forma nativa en Windows | Usa WSL2 (Parte 7) |
| `Variable cargaventas does not exist` en Airflow | La variable `AIRFLOW_VAR_CARGAVENTAS` no está definida en esa terminal | Repite 7.2 en la misma terminal |
| PowerShell no deja activar `.venv` | Política de ejecución de Windows | `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` |

---

## Limpieza

**PowerShell**
```powershell
docker volume rm cargaventas-salida
docker image rm cargaventas:1.0.0
Remove-Item -Recurse -Force datos, salida, build, dist, .env, "$env:TEMP\vwheel" -ErrorAction SilentlyContinue
deactivate
```

**bash**
```bash
docker volume rm cargaventas-salida
docker image rm cargaventas:1.0.0
rm -rf datos salida build dist .env /tmp/vwheel airflow_home ~/venv-airflow
deactivate
```

---

## Resumen

| Lo que hiciste | Lección |
| :--- | :--- |
| Programaste el DAG de lunes a viernes, sin ejecuciones encimadas ni reprocesos masivos | 6.1 |
| Definiste dependencias y un contrato de códigos que entienden la CLI, Docker, el DAG y ECS | 6.3, 6.4 |
| Provocaste `success`, `failed`, `up_for_retry`, `upstream_failed` y `skipped`, y explicaste cada uno | 6.5, 6.7 |
| Separaste fallos transitorios (reintentar) de definitivos (`AirflowFailException`), con alerta y teardown | 6.6, 6.7 |
| Dejaste el DAG listo para MWAA: Variables al ejecutar, ECS Fargate con el rol de ejecución, DAG en S3 | 6.8, 6.18, 6.20 |
| Creaste un entorno reproducible desde un lockfile con hashes | 6.10, 6.11 |
| Convertiste el script en un paquete interno con CLI, versión única y versión y commit en los logs | 6.12, 6.13, 6.14 |
| Verificaste el wheel y bloqueaste uno manipulado | 6.15, 6.25 |
| Construiste una imagen multi-stage sin root, de solo lectura y etiquetada | 6.16, 6.27 |
| Separaste código, configuración y secretos | 6.17, 6.33 |
| Versionaste con Git, etiqueta anotada y fin de línea controlado | 6.22 |
| Integraste pruebas, calidad, auditoría y validación de workflows en CI | 6.23, 6.24, 6.26, 6.31, 6.32, 6.35 |
| Diseñaste una entrega por etiqueta con aprobación, prueba de humo, reversión y auditoría | 6.28, 6.29, 6.30 |
| Validaste entradas, aplicaste mínimo privilegio, redactaste logs y minimizaste datos personales | 6.34, 6.36, 6.37, 6.38, 6.39 |
