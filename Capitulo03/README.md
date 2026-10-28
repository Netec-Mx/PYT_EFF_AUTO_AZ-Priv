# Práctica 3. Automatización de integración de datos con Snowflake

| Dato | Detalle |
| :--- | :--- |
| **Curso** | Automatización de procesos con Python |
| **Capítulo** | 3. Integración con APIs, servicios, bases de datos y warehouses |
| **Duración** | 120 minutos |
| **Modalidad** | Individual |
| **Nivel de Bloom** | Aplicar / Analizar |

---

## Objetivo

Automatizar la integración de datos de una **API ficticia** con **Snowflake o un entorno simulado**. La automatización cubre estos puntos:

- **Extracción:** autenticación, paginación, límites de tasa, timeouts y reintentos.
- **Validación:** contrato de la respuesta con Pydantic.
- **Carga:** control de esquema, reconciliación e idempotencia.
- **Credenciales:** en variables de entorno, nunca en el código.

**El destino es un emulador local de Snowflake.** Así nadie necesita una cuenta, una tarjeta ni conexión a la nube durante la práctica. El código usa el **conector oficial** `snowflake-connector-python`, de modo que el mismo programa funciona contra una cuenta real cambiando solo la configuración (sección opcional al final).

## Lo que vas a construir

Un integrador llamado **integracion-ventas**. Lo ejecutas con un comando y recorre estas fases. Cada fase deja un evento JSON con el mismo `trace_id`:

```text
 Terminal 1: api_simulada.py  (el sistema externo: OAuth 2.0, cursor, fallas a propósito)
        │
        │  HTTPS/JSON
        ▼
 Terminal 2: python -m integracion_ventas
        │
 1. extraer_validar ─────────────────────────────────────────────────────────────
    OAuth2Client ───── token Client Credentials en caché
    ExtractorAPI ───── cursor + timeouts (3.05, 2) + Retry de urllib3 (timeout, 429, 503)
    validar_lote() ─── contrato Pydantic ──► inválidos ──► cuarentena/*.json (con motivos)
    aplanar() ──────── JSON anidado → fila relacional
    persistir_staging() ─ staging/<ejecucion>.csv.gz  (escritura atómica .tmp → renombre)
        │
        ├── % en cuarentena > 25 % ──► se detiene, no se carga nada (código 5)
        ▼
 2. cargar ──────────────────────────────────────────────────────────────────────
    validar_esquema_destino() ── INFORMATION_SCHEMA.COLUMNS ──► incompatible (código 6)
    TRUNCATE staging → llenar staging → MERGE por ID_TRANSACCION   (idempotente)
        │
        ▼
 3. reconciliar ─────────────────────────────────────────────────────────────────
    conteo + suma de TOTAL + SHA-256 por fila ──► no cuadra (código 4)
        │
        ▼
    CONTROL_EJECUCIONES (metadatos de la ejecución) + evento "resumen"
```

### Relación con el capítulo

| Componente del laboratorio | Lección del capítulo |
| :--- | :--- |
| `requests.Session`, `Accept: application/json`, códigos HTTP | 3.1 APIs REST |
| `OAuth2Client`: Client Credentials, token en caché y renovado 30 s antes de vencer | 3.2 Autenticación |
| `ExtractorAPI.lotes()`: generador con cursor `starting_after` y doble salida `has_more` / `next_cursor` | 3.3 Paginación |
| `Retry` de urllib3: solo errores transitorios y solo `GET`; un 401 no se reintenta | 3.4 Reintentos |
| 429 con `Retry-After` (urllib3) y 429 sin cabecera (manejo propio en `lotes()`) | 3.5 Limitación de tasa |
| `backoff_factor` + `backoff_jitter` | 3.6 Espera exponencial |
| `timeout=(3.05, 2)`: conexión y lectura por separado | 3.7 Tiempos de espera |
| Reintentar solo `GET`; `MERGE` por clave; carpeta de stage por ejecución | 3.8 Idempotencia |
| `validar_respuesta()`: `Content-Type`, JSON, estructura `data` | 3.9 Validación de respuestas |
| Cuarentena con todos los motivos de cada registro; el lote continúa | 3.10 Respuestas inválidas o incompletas |
| `TransaccionAPI`: `alias`, `pattern`, `Decimal`, `AwareDatetime`, `field_validator`, `model_validator` | 3.11 Pydantic |
| `warehouse.conectar()`: context manager que cierra siempre la conexión | 3.12 Gestión de conexiones |
| `executemany`, `fetchall`, `fetchone` con el conector DB-API | 3.13 Ejecución de consultas |
| `MERGE` como sentencia única y atómica con `autocommit` | 3.14 Transacciones |
| `PUT` + `COPY INTO` (Snowflake real) | 3.15 Carga masiva, 3.20 Snowflake |
| `INFORMATION_SCHEMA.COLUMNS` | 3.17 Operaciones sobre metadatos |
| Valores con `%s`; tablas en lista blanca; el nombre `O'Brien'); DROP TABLE…` se guarda como dato | 3.18 Prácticas seguras de SQL |
| `pydantic-settings` + `SecretStr` + filtro `OcultarSecretos`; par de claves para Snowflake | 3.19 Credenciales |
| API → staging `.csv.gz` → warehouse | 3.21 Integración, 3.22 Ingesta |
| `ExtractorAPI` con `HTTPAdapter(max_retries=Retry)` | 3.23 Extracción mediante APIs |
| Contrato Pydantic (origen) y `validar_esquema_destino()` (destino) | 3.24 Comparación de esquemas, 3.27 Metadatos |
| `reconciliar()`: conteo, suma de control y hash SHA-256 por fila | 3.25 Reconciliación |
| `JsonFormatter` + `monitor()` con `trace_id`, duración y estado por fase | 3.26 Monitoreo de interfaces |

## Criterios de evaluación

| Criterio | Evidencia en tu entrega |
| :--- | :--- |
| **Corrección** | 25 recibidos, 20 cargados y 5 en cuarentena con su motivo; el cambio de esquema de la API (v2) y el del destino se detectan antes de cargar |
| **Resiliencia** | El timeout, el 429 y el 503 se superan solos con reintentos; un registro inválido no detiene el lote; un error permanente termina con código 3 sin reintentar |
| **Mantenibilidad** | Integración (`api.py`), validación (`modelos.py`) y persistencia (`warehouse.py`) están separadas; el emulador y Snowflake real comparten el código |
| **Seguridad** | Secretos solo en `.env` o en el entorno, nunca en el código ni en los logs; SQL parametrizado; autenticación por par de claves en modo real |
| **Valor operativo** | La segunda ejecución no duplica (0 insertadas, 20 actualizadas); la reconciliación cuadra; cada ejecución queda en `CONTROL_EJECUCIONES` y en eventos JSON |

---

## Requisitos

| Elemento | Requisito |
| :--- | :--- |
| Python | **3.11 o superior** (probado con 3.11, 3.13 y 3.14) |
| Sistema operativo | Windows 10/11, Linux, o **macOS 15 o superior** (el conector de Snowflake 4.8 no publica paquetes para macOS 14 o anterior) |
| Editor | Visual Studio Code con la extensión de Python |
| Terminal | **Dos** terminales integradas de VS Code (PowerShell en Windows; bash/zsh en macOS o Linux) |
| Red | Acceso a PyPI para instalar paquetes. La práctica no se conecta a Internet: la API y el warehouse son locales |
| Cuenta de Snowflake | **No se necesita** |

> **Convenciones**
> - Todos los comandos se ejecutan desde la **carpeta raíz del proyecto** (`practica3_integracion`) y con el entorno virtual activo **en las dos terminales**.
> - **Terminal 1** ejecuta la API simulada todo el tiempo. **Terminal 2** ejecuta la automatización.
> - Cuando un comando cambia según el sistema operativo, verás la versión **PowerShell** y la versión **bash**.
> - Se mantienen los estándares de las prácticas 1 y 2: src-layout, `pyproject.toml`, versiones fijadas, configuración validada con Pydantic y códigos de salida.

### Distribución del tiempo

| Parte | Contenido | Minutos |
| :--- | :--- | :---: |
| 0. Preparación | Estructura, entorno y dependencias | 10 |
| 1. La API de origen | Servidor simulado con OAuth 2.0, cursor y fallas | 8 |
| 2. Configuración, secretos y monitoreo | Excepciones, `.env`, `pydantic-settings`, logs JSON | 12 |
| 3. Extracción resiliente | OAuth, paginación, timeouts, reintentos, límite de tasa | 20 |
| 4. Contrato y staging | Pydantic, cuarentena, aplanado y `.csv.gz` | 12 |
| 5. Warehouse y reconciliación | Conexión, esquema, carga idempotente, SQL seguro | 20 |
| 6. Orquestación y ejecución | Pipeline, CLI, primera y segunda ejecución | 15 |
| 7. Escenarios de falla | Credencial, caída, cambio de esquema, destino alterado | 15 |
| 8. Pruebas | pytest, ruff y mypy | 8 |

> **Checkpoints.** Al final de las partes 2, 3, 4 y 5 hay un punto de control. Si te atrasas, el instructor puede darte la carpeta del checkpoint.

---

## Parte 0. Preparación (10 min)

**0.1** Crea la carpeta `practica3_integracion`, ábrela en VS Code (**File → Open Folder**) y abre una terminal.

**0.2** Crea la estructura de carpetas:

**PowerShell**
```powershell
New-Item -ItemType Directory -Force -Path src/integracion_ventas, tests, scripts
```

**bash**
```bash
mkdir -p src/integracion_ventas tests scripts
```

**0.3** Crea los archivos base del proyecto. Observa en `pyproject.toml` que `fakesnow` es el emulador y que el conector se limita a la serie 4.x:

**Archivo:** `pyproject.toml`

```toml
[build-system]
requires = ["setuptools>=69"]
build-backend = "setuptools.build_meta"

[project]
name = "integracion-ventas"
version = "0.1.0"
description = "Integración API REST → staging → Snowflake (o emulador local)"
requires-python = ">=3.11"
dependencies = [
    "fakesnow==0.12.1",                         # emulador local de Snowflake
    "pydantic>=2.7,<3",
    "pydantic-settings>=2.3,<3",
    "requests>=2.32,<3",
    "snowflake-connector-python>=4.8,<5",       # fakesnow 0.12 no funciona con 5.x
]

[tool.setuptools.packages.find]
where = ["src"]

[tool.ruff]
line-length = 88
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP"]

[tool.mypy]
python_version = "3.11"
disallow_untyped_defs = true
ignore_missing_imports = true

[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["."]  # para importar api_simulada en las pruebas
filterwarnings = ["ignore:Arrow output on a DB-API result"]
```

**0.4** Crea los archivos de versiones. En las prácticas anteriores los generaste con `pip-compile`; **aquí se entregan ya resueltos** porque el emulador depende de versiones exactas y estas son las que se probaron en Windows, macOS y Linux. La primera línea de cada archivo indica el comando con el que se generaron.

**Archivo:** `requirements.txt`

```text
# Versiones probadas para la Práctica 3 (Windows, macOS 15+ y Linux; Python 3.11-3.14).
# Generado con:
#   pip-compile --strip-extras --no-annotate -P "cryptography<49" -P "pyopenssl<26.4" pyproject.toml -o requirements.txt
# cryptography<49 y pyopenssl<26.4: así también hay ruedas para macOS Intel.
annotated-types==0.8.0
asn1crypto==1.5.1
boto3==1.43.118
botocore==1.43.118
certifi==2026.10.23
cffi==2.1.1
charset-normalizer==3.5.2
cryptography==48.0.1
duckdb==2.0.0
fakesnow==0.12.1
filelock==3.32.7
idna==3.20
jmespath==1.1.0
packaging==26.3
platformdirs==4.13.5
pyarrow==26.0.0
pycparser==3.12
pydantic==2.14.1
pydantic-core==2.50.1
pydantic-settings==2.16.1
pyjwt==2.15.1
pyopenssl==26.2.0
python-dateutil==2.9.0.post0
python-dotenv==1.2.4
pytz==2026.5
requests==2.34.2
s3transfer==0.19.3
six==1.17.0
snowflake-connector-python==4.8.0
sortedcontainers==2.4.0
sqlglot==30.22.0
tomlkit==0.15.2
typing-extensions==4.16.0
typing-inspection==0.4.4
urllib3==2.8.0
```

**Archivo:** `requirements-dev.in`

```text
# Herramientas de desarrollo; respetan las versiones de requirements.txt
-c requirements.txt
mypy
pip-tools
pytest
ruff
types-requests
```

**Archivo:** `requirements-dev.txt`

```text
# Versiones probadas para la Práctica 3 (Windows, macOS 15+ y Linux; Python 3.11-3.14).
# Generado con:
#   pip-compile --strip-extras --no-annotate requirements-dev.in -o requirements-dev.txt
ast-serialize==0.13.1
build==1.6.1
click==8.5.0
iniconfig==2.3.1
librt==0.16.0
mypy==2.4.0
mypy-extensions==1.1.0
packaging==26.3
pathspec==1.1.1
pip-tools==7.6.2
pluggy==1.7.0
pygments==2.21.0
pyproject-hooks==1.3.3
pytest==9.1.1
ruff==0.17.2
types-requests==2.33.0.20261006
typing-extensions==4.16.0
urllib3==2.8.0
wheel==0.48.0
```

**Archivo:** `.gitignore`

```text
.env
.venv/
__pycache__/
*.egg-info/
.mypy_cache/
.ruff_cache/
.pytest_cache/

# Datos y estado de cada ejecución
staging/
cuarentena/
emulador_snowflake/
```

**Archivo:** `src/integracion_ventas/__init__.py`

```python
"""Integración de ventas: API REST → staging → Snowflake."""

__version__ = "0.1.0"
```

**0.5** Crea el entorno virtual e instala:

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
python -m pip install -r requirements.txt -r requirements-dev.txt
python -m pip install -e .
```

**Verificación:**

```bash
python -c "import integracion_ventas, fakesnow, snowflake.connector, pydantic, requests; print('ok', integracion_ventas.__version__)"
```

Resultado esperado: `ok 0.1.0`

---

## Parte 1. La API de origen (8 min)

### 1.1 El sistema externo

`api_simulada.py` representa la API del proveedor. **No es parte de tu automatización:** cópiala tal cual. Solo usa la biblioteca estándar y se comporta de forma determinista:

| Qué hace | Para qué sirve en la práctica |
| :--- | :--- |
| `POST /oauth/token` con Client Credentials; token de 300 s | 3.2 Autenticación |
| `GET /v1/transacciones?limit=10&starting_after=…` con `has_more` y `next_cursor` | 3.3 Paginación |
| Primera vez por token: página 1 tarda 3 s, página 2 responde 429 con `Retry-After: 1`, página 3 responde 503 | 3.4 a 3.7 |
| 25 transacciones; 5 no cumplen el contrato | 3.10, 3.11 |
| El cliente 1005 se llama `O'Brien'); DROP TABLE VENTAS_API;--` | 3.18 SQL seguro |
| `--esquema v2` renombra `total` a `importe` | 3.24 Cambios de esquema |

**Archivo:** `api_simulada.py`

```python
"""API REST simulada para la Práctica 3. Solo usa la biblioteca estándar.

Representa el sistema externo: no forma parte de la automatización.

Endpoints:
    POST /oauth/token        OAuth 2.0 Client Credentials (form-urlencoded)
    GET  /v1/transacciones   Paginación por cursor: ?limit=10&starting_after=TXN-00010

Fallas deterministas, una vez por cada token (es decir, por ejecución del pipeline):
    página 1 → responde después de 3 s (provoca un timeout de lectura)
    página 2 → 429 Too Many Requests con Retry-After: 1
    página 3 → 503 Service Unavailable

Uso:
    python api_simulada.py                 # puerto 8000, esquema v1
    python api_simulada.py --esquema v2    # el campo "total" cambia a "importe"
    python api_simulada.py --puerto 8010 --sin-fallas
"""

import argparse
import json
import secrets
import threading
import time
from decimal import Decimal
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from typing import Any
from urllib.parse import parse_qs, urlparse

CLIENT_ID = "lab-integracion"
CLIENT_SECRET = "s3cr3to-lab-2026"
TOKEN_TTL_S = 300
TOTAL_TRANSACCIONES = 25

CLIENTES = [
    (1001, "Comercial Peña", "empresa"),
    (1002, "Ana López", "premium"),
    (1003, "Distribuidora Núñez", "empresa"),
    (1004, "Luis Gómez", None),
    (1005, "O'Brien'); DROP TABLE VENTAS_API;--", "basico"),
]


def _transaccion(n: int) -> dict[str, Any]:
    cliente_id, nombre, segmento = CLIENTES[n % len(CLIENTES)]
    lineas = [
        (f"SKU-{1000 + n}", 1 + n % 3, "125.50"),
        (f"SKU-{2000 + n}", 2, f"{10 + n}.25"),
    ]
    items = [{"sku": s, "cantidad": c, "precioUnitario": p} for s, c, p in lineas]
    total = sum((Decimal(p) * c for _, c, p in lineas), Decimal("0"))
    registro: dict[str, Any] = {
        "id": f"TXN-{n:05d}",
        "fechaOperacion": f"2026-10-24T{8 + n % 10:02d}:{(n * 7) % 60:02d}:00Z",
        "clienteId": str(cliente_id),  # la API manda números como texto
        "cliente": {"nombre": nombre, "segmento": segmento},
        "items": items,
        "total": str(total),
        "moneda": "MXN",
    }
    # Registros que no cumplen el contrato (5 de 25)
    if n == 5:
        registro["total"] = str(total + 1)  # no cuadra con los items
    elif n == 9:
        registro["id"] = "TXN-9"  # no cumple el patrón
    elif n == 13:
        registro["items"][0]["cantidad"] = 0  # cantidad no positiva
    elif n == 17:
        registro["fechaOperacion"] = "fecha-invalida"
    elif n == 21:
        registro["clienteId"] = "no-es-un-entero"
    return registro


DATOS = [_transaccion(n) for n in range(1, TOTAL_TRANSACCIONES + 1)]


class Estado:
    def __init__(self, esquema: str, con_fallas: bool) -> None:
        self.esquema = esquema
        self.con_fallas = con_fallas
        self.tokens: dict[str, float] = {}
        self.fallas_usadas: set[tuple[str, int]] = set()
        self.candado = threading.Lock()

    def falla_pendiente(self, token: str, pagina: int) -> bool:
        """True solo la primera vez que este token pide esta página."""
        if not self.con_fallas or pagina > 3:
            return False
        with self.candado:
            if (token, pagina) in self.fallas_usadas:
                return False
            self.fallas_usadas.add((token, pagina))
            return True


class Manejador(BaseHTTPRequestHandler):
    estado: Estado

    def log_message(self, format: str, *args: Any) -> None:
        print(f"[api] {self.command} {self.path} -> {args[1] if len(args) > 1 else ''}")

    def _json(
        self, codigo: int, cuerpo: Any, cabeceras: dict[str, str] | None = None
    ) -> None:
        datos = json.dumps(cuerpo, ensure_ascii=False).encode("utf-8")
        try:
            self.send_response(codigo)
            self.send_header("Content-Type", "application/json; charset=utf-8")
            self.send_header("Content-Length", str(len(datos)))
            for clave, valor in (cabeceras or {}).items():
                self.send_header(clave, valor)
            self.end_headers()
            self.wfile.write(datos)
        except (BrokenPipeError, ConnectionResetError, ConnectionAbortedError):
            pass  # el cliente ya cerró la conexión (p. ej., por su timeout)

    def do_POST(self) -> None:
        if urlparse(self.path).path != "/oauth/token":
            self._json(404, {"error": "no encontrado"})
            return
        largo = int(self.headers.get("Content-Length", "0"))
        form = parse_qs(self.rfile.read(largo).decode("utf-8"))
        valido = (
            form.get("grant_type") == ["client_credentials"]
            and form.get("client_id") == [CLIENT_ID]
            and form.get("client_secret") == [CLIENT_SECRET]
        )
        if not valido:
            self._json(401, {"error": "invalid_client"})
            return
        token = secrets.token_hex(16)
        self.estado.tokens[token] = time.monotonic() + TOKEN_TTL_S
        self._json(
            200,
            {"access_token": token, "token_type": "Bearer", "expires_in": TOKEN_TTL_S},
        )

    def do_GET(self) -> None:
        url = urlparse(self.path)
        if url.path != "/v1/transacciones":
            self._json(404, {"error": "no encontrado"})
            return
        token = self.headers.get("Authorization", "").removeprefix("Bearer ")
        if self.estado.tokens.get(token, 0) < time.monotonic():
            self._json(401, {"error": "token inválido o vencido"})
            return

        params = parse_qs(url.query)
        limite = max(1, min(50, int(params.get("limit", ["10"])[0])))
        despues = params.get("starting_after", [None])[0]
        inicio = 0
        if despues is not None:
            ids = [r["id"] for r in DATOS]
            if despues not in ids:
                self._json(400, {"error": f"cursor desconocido: {despues}"})
                return
            inicio = ids.index(despues) + 1
        pagina = inicio // limite + 1

        if self.estado.falla_pendiente(token, pagina):
            if pagina == 1:
                time.sleep(3)  # más que el timeout de lectura del cliente
            elif pagina == 2:
                self._json(
                    429, {"error": "demasiadas solicitudes"}, {"Retry-After": "1"}
                )
                return
            else:
                self._json(503, {"error": "servicio no disponible"})
                return

        lote = [dict(r) for r in DATOS[inicio : inicio + limite]]
        if self.estado.esquema == "v2":
            for registro in lote:
                registro["importe"] = registro.pop("total")
        hay_mas = inicio + limite < len(DATOS)
        self._json(
            200,
            {
                "data": lote,
                "has_more": hay_mas,
                "next_cursor": lote[-1]["id"] if hay_mas else None,
            },
        )


def crear_servidor(
    puerto: int, esquema: str = "v1", con_fallas: bool = True
) -> ThreadingHTTPServer:
    # Subclase propia: cada servidor tiene su estado (útil en las pruebas).
    manejador = type(
        "ManejadorAPI", (Manejador,), {"estado": Estado(esquema, con_fallas)}
    )
    servidor = ThreadingHTTPServer(("127.0.0.1", puerto), manejador)
    servidor.daemon_threads = True
    return servidor


def main() -> None:
    parser = argparse.ArgumentParser(description="API REST simulada (Práctica 3)")
    parser.add_argument("--puerto", type=int, default=8000)
    parser.add_argument("--esquema", choices=["v1", "v2"], default="v1")
    parser.add_argument("--sin-fallas", action="store_true")
    args = parser.parse_args()
    servidor = crear_servidor(args.puerto, args.esquema, not args.sin_fallas)
    print(
        f"[api] Escuchando en http://127.0.0.1:{args.puerto} "
        f"(esquema {args.esquema}, fallas {'no' if args.sin_fallas else 'sí'}). "
        "Ctrl+C para detener."
    )
    try:
        servidor.serve_forever()
    except KeyboardInterrupt:
        print("\n[api] Detenida.")
    finally:
        servidor.server_close()


if __name__ == "__main__":
    main()
```

### 1.2 Levantar la API (Terminal 1)

En la **Terminal 1**, con el entorno activo:

```bash
python api_simulada.py
```

Resultado esperado (la terminal queda ocupada; déjala abierta):

```text
[api] Escuchando en http://127.0.0.1:8000 (esquema v1, fallas sí). Ctrl+C para detener.
```

### 1.3 Comprobar que exige autenticación (Terminal 2)

Abre una **segunda terminal** (**+** en el panel de terminales), activa el entorno (paso 0.5) y ejecuta:

```bash
python -c "import requests; r = requests.get('http://127.0.0.1:8000/v1/transacciones'); print(r.status_code, r.json())"
```

Resultado esperado:

```text
401 {'error': 'token inválido o vencido'}
```

En la Terminal 1 aparece `[api] GET /v1/transacciones -> 401`. Sin token no hay datos: eso es lo que resuelve `OAuth2Client` en la Parte 3.

---

## Parte 2. Configuración, secretos y monitoreo (12 min)

### 2.1 Excepciones con código de salida

Cada error controlado sabe con qué código debe terminar el proceso. Un orquestador (cron, Airflow, Task Scheduler) usa ese código para decidir si reintenta o alerta:

**Archivo:** `src/integracion_ventas/errores.py`

```python
"""Excepciones controladas y sus códigos de salida."""


class IntegracionError(Exception):
    codigo_salida: int = 1


class ConfiguracionInvalidaError(IntegracionError):
    """Falta una variable de entorno o tiene un valor inválido."""

    codigo_salida = 1


class ApiError(IntegracionError):
    """Error permanente de la API o reintentos agotados."""

    codigo_salida = 3


class ReconciliacionError(IntegracionError):
    """El destino no coincide con lo que se validó."""

    codigo_salida = 4


class LoteNoAprobadoError(IntegracionError):
    """Demasiados registros en cuarentena: no se carga nada."""

    codigo_salida = 5


class EsquemaDestinoError(IntegracionError):
    """La tabla destino no tiene la estructura esperada."""

    codigo_salida = 6
```

### 2.2 Variables de entorno y `.env`

**2.2.1** Crea la plantilla. Esta **sí** se versiona: documenta qué variables existen, sin valores secretos.

**Archivo:** `.env.example`

```bash
# Copia este archivo como .env. .env.example SÍ se versiona; .env NUNCA.
# Un .env es texto plano: sirve para desarrollo local, no cifra nada.

# --- API de origen (OAuth 2.0 Client Credentials) ---
API_BASE_URL=http://127.0.0.1:8000
API_CLIENT_ID=lab-integracion
API_CLIENT_SECRET=reemplaza-con-el-secreto
API_TIMEOUT_CONEXION=3.05
# Lectura de 2 s para que veas el reintento por timeout; en producción, más alto.
API_TIMEOUT_LECTURA=2
API_TAMANO_PAGINA=10

# --- Destino: "emulador" (fakesnow, local) o "real" (cuenta de Snowflake) ---
SF_MODO=emulador
SF_DATABASE=ANALITICA
SF_SCHEMA=VENTAS

# Solo para SF_MODO=real (autenticación por par de claves; sin contraseña)
# SF_ACCOUNT=miorg-micuenta
# SF_USER=SVC_INTEGRACION
# SF_ROLE=ROL_ETL
# SF_WAREHOUSE=WH_ETL_XS
# SF_PRIVATE_KEY_PATH=C:/claves/svc_integracion.p8
# SF_PRIVATE_KEY_PASSPHRASE=
```

**2.2.2** Copia la plantilla como `.env`:

**PowerShell**
```powershell
Copy-Item .env.example .env
```

**bash**
```bash
cp .env.example .env
```

**2.2.3** Abre `.env` en VS Code y reemplaza el secreto. El proveedor de la API te lo entregaría por un canal seguro; en esta práctica es el que define `api_simulada.py`:

```bash
API_CLIENT_SECRET=s3cr3to-lab-2026
```

> **Importante.** `.env` está en `.gitignore`. Un `.env` es **texto plano**: protege contra subir secretos al repositorio, no contra quien tenga acceso al equipo. En producción el secreto vendría de un gestor de secretos (AWS Secrets Manager, Azure Key Vault) o de las variables del orquestador.

### 2.3 Configuración validada con `pydantic-settings`

`SecretStr` evita que el secreto aparezca al imprimir la configuración. El validador `requisitos_modo_real` exige las credenciales de Snowflake solo cuando `SF_MODO=real`. Si algo falta, el proceso se detiene **antes** de llamar a la API (fail-fast). El mensaje de error nunca incluye los valores:

**Archivo:** `src/integracion_ventas/configuracion.py`

```python
"""Configuración y secretos con pydantic-settings (variables de entorno o .env)."""

from pathlib import Path
from typing import Literal

from pydantic import Field, SecretStr, ValidationError, model_validator
from pydantic_settings import BaseSettings, SettingsConfigDict

from integracion_ventas.errores import ConfiguracionInvalidaError


class Configuracion(BaseSettings):
    """Lee API_*, SF_* y demás del entorno o de .env (que va en .gitignore)."""

    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    # API de origen
    api_base_url: str = "http://127.0.0.1:8000"
    api_client_id: str
    api_client_secret: SecretStr  # no se imprime por accidente
    api_timeout_conexion: float = Field(default=3.05, gt=0)
    api_timeout_lectura: float = Field(default=15.0, gt=0)
    api_tamano_pagina: int = Field(default=10, ge=1, le=50)

    # Calidad del lote
    max_pct_cuarentena: float = Field(default=25.0, ge=0, le=100)

    # Destino
    sf_modo: Literal["emulador", "real"] = "emulador"
    sf_database: str = "ANALITICA"
    sf_schema: str = "VENTAS"
    sf_account: str | None = None
    sf_user: str | None = None
    sf_role: str | None = None
    sf_warehouse: str | None = None
    sf_private_key_path: Path | None = None
    sf_private_key_passphrase: SecretStr | None = None
    emulador_dir: Path = Path("emulador_snowflake")

    @model_validator(mode="after")
    def requisitos_modo_real(self) -> "Configuracion":
        if self.sf_modo == "real":
            faltan = [
                nombre
                for nombre in (
                    "sf_account",
                    "sf_user",
                    "sf_role",
                    "sf_warehouse",
                    "sf_private_key_path",
                    "sf_private_key_passphrase",
                )
                if getattr(self, nombre) is None
            ]
            if faltan:
                raise ValueError(f"SF_MODO=real requiere: {', '.join(faltan).upper()}")
        return self


def cargar_configuracion() -> Configuracion:
    """Fail-fast: valida todo antes de llamar a la API o al warehouse."""
    try:
        return Configuracion()  # type: ignore[call-arg]
    except ValidationError as error:
        # Sin input_value: el mensaje nunca incluye los valores (ni los secretos).
        motivos = "; ".join(
            f"{'.'.join(map(str, e['loc'])) or 'configuración'}: {e['msg']}"
            for e in error.errors(include_input=False)
        )
        raise ConfiguracionInvalidaError(
            f"Configuración inválida: {motivos}"
        ) from error
```

### 2.4 Logs JSON, ocultamiento de secretos y medición por fase

`monitor()` es un context manager: mide la duración, recibe métricas y registra `SUCCESS` o `FAILURE` sin ensuciar la lógica de cada fase. `OcultarSecretos` reemplaza cualquier secreto conocido por `[REDACTED]` antes de escribir el log, incluidos los mensajes de bibliotecas como urllib3:

**Archivo:** `src/integracion_ventas/monitoreo.py`

```python
"""Telemetría: logs JSON, ocultamiento de secretos y medición por fase."""

import json
import logging
import time
from collections.abc import Iterator
from contextlib import contextmanager
from datetime import UTC, datetime
from typing import Any

logger = logging.getLogger("integracion_ventas")


class JsonFormatter(logging.Formatter):
    """Una línea JSON por evento: una plataforma de observabilidad la indexa."""

    def format(self, record: logging.LogRecord) -> str:
        evento = {
            "ts": datetime.fromtimestamp(record.created, UTC).isoformat(
                timespec="milliseconds"
            ),
            "nivel": record.levelname,
            "logger": record.name,
            "mensaje": record.getMessage(),
        }
        evento |= getattr(record, "telemetria", {})
        if record.exc_info:
            evento["excepcion"] = self.formatException(record.exc_info)
        return json.dumps(evento, ensure_ascii=False, default=str)


class OcultarSecretos(logging.Filter):
    """Reemplaza cualquier secreto conocido por [REDACTED] antes de escribirlo."""

    def __init__(self, secretos: list[str]) -> None:
        super().__init__()
        self.secretos = [s for s in secretos if s]

    def filter(self, record: logging.LogRecord) -> bool:
        mensaje = record.getMessage()
        for secreto in self.secretos:
            mensaje = mensaje.replace(secreto, "[REDACTED]")
        record.msg, record.args = mensaje, None
        return True


def configurar_logging(secretos: list[str]) -> None:
    """Un handler en la raíz: también formatea y filtra los logs de urllib3."""
    handler = logging.StreamHandler()
    handler.setFormatter(JsonFormatter())
    handler.addFilter(OcultarSecretos(secretos))
    raiz = logging.getLogger()
    raiz.handlers[:] = [handler]
    raiz.setLevel(logging.INFO)
    for ruidoso in ("snowflake", "fakesnow", "botocore"):
        logging.getLogger(ruidoso).setLevel(logging.WARNING)
    # RetryConAviso ya registra cada reintento; se evita el mensaje duplicado.
    logging.getLogger("urllib3").setLevel(logging.ERROR)


@contextmanager
def monitor(fase: str, trace_id: str) -> Iterator[dict[str, Any]]:
    """Mide duración, volumen y errores de una fase sin ensuciar la lógica."""
    metricas: dict[str, Any] = {}
    inicio = time.perf_counter()
    estado, error = "SUCCESS", None
    try:
        yield metricas
    except Exception as exc:
        estado, error = "FAILURE", f"{type(exc).__name__}: {exc}"
        raise
    finally:
        nivel = logging.INFO if estado == "SUCCESS" else logging.ERROR
        telemetria = {
            "trace_id": trace_id,
            "fase": fase,
            "estado": estado,
            "duracion_ms": round((time.perf_counter() - inicio) * 1000, 1),
            **metricas,
            "error": error,
        }
        logger.log(nivel, "fase.%s", fase, extra={"telemetria": telemetria})
```

> **Checkpoint 2.** Ejecuta:
>
> ```bash
> python -c "from integracion_ventas.configuracion import cargar_configuracion; c = cargar_configuracion(); print(c.api_client_secret, c.sf_modo, c.api_timeout_lectura)"
> ```
>
> Resultado esperado: `********** emulador 2.0`. El secreto se leyó de `.env`, pero no se imprime.

---

## Parte 3. Extracción resiliente (20 min)

### 3.1 El extractor

Crea `api.py`. Lee los comentarios: cada decisión corresponde a una lección.

**Archivo:** `src/integracion_ventas/api.py`

```python
"""Extracción desde la API: OAuth 2.0, reintentos, límite de tasa, timeouts y cursor."""

import logging
import time
from collections.abc import Iterator
from typing import Any

import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

from integracion_ventas.errores import ApiError

logger = logging.getLogger(__name__)


class RetryConAviso(Retry):
    """Retry de urllib3 que deja un WARNING en cada reintento (observabilidad)."""

    def increment(  # type: ignore[override]
        self,
        method: str | None = None,
        url: str | None = None,
        response: Any = None,
        error: Exception | None = None,
        **kwargs: Any,
    ) -> Retry:
        nuevo = super().increment(method, url, response, error, **kwargs)
        motivo = type(error).__name__ if error else f"HTTP {response.status}"
        logger.warning("Reintento automático por %s (quedan %s)", motivo, nuevo.total)
        return nuevo


class OAuth2Client:
    """Client Credentials (M2M) con caché del token hasta 30 s antes de vencer."""

    def __init__(
        self,
        token_url: str,
        client_id: str,
        client_secret: str,
        timeout: tuple[float, float],
    ) -> None:
        self._token_url = token_url
        self._client_id = client_id
        self._client_secret = client_secret
        self._timeout = timeout
        self._token: str | None = None
        self._vence_en = 0.0
        self.tokens_solicitados = 0
        # Solo se reintenta lo transitorio (no hay conexión); un 401 no se repite.
        self._sesion = requests.Session()
        self._sesion.mount(
            "http://",
            HTTPAdapter(
                max_retries=RetryConAviso(
                    total=3, connect=3, read=0, status=0, backoff_factor=0.5
                )
            ),
        )
        self._sesion.mount("https://", self._sesion.get_adapter("http://"))

    def token_valido(self) -> str:
        if self._token is None or time.monotonic() >= self._vence_en:
            self._obtener_nuevo_token()
        assert self._token is not None
        return self._token

    def _obtener_nuevo_token(self) -> None:
        try:
            r = self._sesion.post(
                self._token_url,
                timeout=self._timeout,
                data={  # form-urlencoded, como pide RFC 6749
                    "grant_type": "client_credentials",
                    "client_id": self._client_id,
                    "client_secret": self._client_secret,
                },
            )
        except requests.RequestException as error:
            raise ApiError(
                f"No se pudo contactar el servidor de tokens {self._token_url} "
                f"({type(error).__name__})"
            ) from error
        if r.status_code in (400, 401, 403):  # permanente: reintentar no lo arregla
            raise ApiError(f"Autenticación rechazada: HTTP {r.status_code}")
        r.raise_for_status()
        cuerpo = r.json()
        self._token = cuerpo["access_token"]
        self._vence_en = time.monotonic() + cuerpo["expires_in"] - 30  # margen
        self.tokens_solicitados += 1


def validar_respuesta(r: requests.Response) -> dict[str, Any]:
    """Un 200 OK no garantiza datos válidos: se valida en capas."""
    tipo = r.headers.get("Content-Type", "")
    if not tipo.startswith("application/json"):
        raise ApiError(f"Content-Type inesperado: {tipo!r}")
    try:
        datos = r.json()
    except requests.JSONDecodeError as error:
        raise ApiError(f"JSON inválido o truncado: {error}") from error
    if not isinstance(datos, dict) or not isinstance(datos.get("data"), list):
        raise ApiError("La página no trae una lista 'data'")
    return datos


class ExtractorAPI:
    """Sesión persistente, reintentos de 5xx y timeouts, 429 con Retry-After."""

    def __init__(
        self,
        url_base: str,
        oauth: OAuth2Client,
        timeout: tuple[float, float],
        max_esperas_429: int = 5,
    ) -> None:
        self._url_base = url_base.rstrip("/")
        self._oauth = oauth
        self._timeout = timeout
        self._max_esperas_429 = max_esperas_429
        self.sesion = requests.Session()
        self.sesion.headers["Accept"] = "application/json"
        reintentos = RetryConAviso(
            total=4,
            connect=3,
            read=2,  # timeouts de lectura: GET es idempotente
            status=3,
            backoff_factor=0.5,
            backoff_jitter=0.3,  # azar para no llegar todos juntos
            # 429 y 503 que traen Retry-After: urllib3 ya espera lo indicado.
            # El 429 que no lo trae (o que persiste) se maneja en lotes().
            status_forcelist=[500, 502, 503, 504],
            allowed_methods=["GET"],  # solo métodos idempotentes
            raise_on_status=False,
        )
        adaptador = HTTPAdapter(max_retries=reintentos)
        self.sesion.mount("http://", adaptador)
        self.sesion.mount("https://", adaptador)

    def lotes(self, endpoint: str, limite: int) -> Iterator[list[dict[str, Any]]]:
        """Generador: entrega página por página; en memoria solo vive una."""
        url = f"{self._url_base}/{endpoint.lstrip('/')}"
        params: dict[str, Any] = {"limit": limite}
        esperas_429 = 0
        while True:
            r = self._get(url, params)
            if r.status_code == 429:
                esperas_429 += 1
                if esperas_429 > self._max_esperas_429:
                    raise ApiError(
                        "Límite de tasa persistente: se agotaron las esperas"
                    )
                valor = r.headers.get("Retry-After", "")
                espera = int(valor) if valor.isdigit() else 2
                logger.warning("HTTP 429: espero %d s y repito la misma página", espera)
                time.sleep(espera)
                continue
            if r.status_code >= 400:
                raise ApiError(f"HTTP {r.status_code} en {r.url}")
            pagina = validar_respuesta(r)
            yield pagina["data"]
            cursor = pagina.get("next_cursor")
            if not pagina.get("has_more") or not cursor:  # evita un bucle infinito
                return
            params["starting_after"] = cursor

    def _get(self, url: str, params: dict[str, Any]) -> requests.Response:
        cabeceras = {"Authorization": f"Bearer {self._oauth.token_valido()}"}
        try:
            return self.sesion.get(
                url, params=params, headers=cabeceras, timeout=self._timeout
            )
        except requests.RequestException as error:
            raise ApiError(
                f"API no disponible tras reintentos: {url} ({type(error).__name__})"
            ) from error
```

Puntos clave del código:

| Decisión | Por qué |
| :--- | :--- |
| El `POST` del token solo se reintenta si **no hubo conexión** (`connect=3, read=0, status=0`) | Si la petición nunca llegó, repetirla es seguro. Un 401 es **permanente**: reintentarlo no lo arregla y además puede bloquear la cuenta |
| `allowed_methods=["GET"]` | Solo se reintentan métodos idempotentes |
| `status_forcelist=[500, 502, 503, 504]` | Errores transitorios del servidor |
| 429 **con** `Retry-After` | urllib3 lo reintenta solo y **espera lo que indica la cabecera** (`respect_retry_after_header=True` por defecto) |
| 429 **sin** `Retry-After`, o que persiste | Lo maneja `lotes()`: espera, repite **la misma página** y se rinde tras 5 esperas |
| `timeout=(3.05, 2)` | La conexión falla rápido; la lectura tiene su propio límite. Sin `timeout`, `requests` puede esperar para siempre |
| `backoff_jitter=0.3` | Agrega azar a la espera para que muchos clientes no reintenten al mismo tiempo |
| Generador `lotes()` | En memoria vive una sola página, sin importar cuántas tenga la API |
| `if not pagina.get("has_more") or not cursor` | Doble condición de salida: una API defectuosa no provoca un bucle infinito |

### 3.2 Probar la extracción sola

Crea este script de apoyo. Usa el extractor sin validar ni cargar nada:

**Archivo:** `scripts/probar_extraccion.py`

```python
"""Checkpoint de la Parte 3: recorre todas las páginas sin validar ni cargar."""

from integracion_ventas.api import ExtractorAPI, OAuth2Client
from integracion_ventas.configuracion import cargar_configuracion
from integracion_ventas.monitoreo import configurar_logging


def main() -> None:
    config = cargar_configuracion()
    configurar_logging([config.api_client_secret.get_secret_value()])
    timeout = (config.api_timeout_conexion, config.api_timeout_lectura)
    oauth = OAuth2Client(
        f"{config.api_base_url.rstrip('/')}/oauth/token",
        config.api_client_id,
        config.api_client_secret.get_secret_value(),
        timeout,
    )
    extractor = ExtractorAPI(config.api_base_url, oauth, timeout)
    total = 0
    paginas = extractor.lotes("/v1/transacciones", config.api_tamano_pagina)
    for numero, lote in enumerate(paginas, start=1):
        total += len(lote)
        primero, ultimo = lote[0]["id"], lote[-1]["id"]
        print(f"Página {numero}: {len(lote)} registros ({primero} a {ultimo})")
    print(f"Total: {total} registros con {oauth.tokens_solicitados} token OAuth")


if __name__ == "__main__":
    main()
```

En la **Terminal 2**:

```bash
python scripts/probar_extraccion.py
```

Resultado esperado (tarda unos 4 s; las líneas JSON se muestran abreviadas):

```text
{"ts": "…", "nivel": "WARNING", "logger": "integracion_ventas.api", "mensaje": "Reintento automático por ReadTimeoutError (quedan 3)"}
Página 1: 10 registros (TXN-00001 a TXN-00010)
{"ts": "…", "nivel": "WARNING", "logger": "integracion_ventas.api", "mensaje": "Reintento automático por HTTP 429 (quedan 3)"}
Página 2: 10 registros (TXN-00011 a TXN-00020)
{"ts": "…", "nivel": "WARNING", "logger": "integracion_ventas.api", "mensaje": "Reintento automático por HTTP 503 (quedan 3)"}
Página 3: 5 registros (TXN-00021 a TXN-00025)
Total: 25 registros con 1 token OAuth
```

Interpreta la salida:

- **Página 1:** el servidor tardó 3 s y la lectura tenía un límite de 2 s, así que hubo un `ReadTimeoutError`. El `GET` se repitió y respondió a tiempo.
- **Página 2:** llegó un 429. urllib3 esperó 1 s (`Retry-After`) y repitió la petición.
- **Página 3:** llegó un 503 y se reintentó con espera exponencial.
- **Un solo token** sirvió para las tres páginas gracias a la caché.

En la Terminal 1 verás cada petición, incluidas las fallidas (`-> 429`, `-> 503`).

> **Checkpoint 3.** 25 registros en 3 páginas, 1 token y tres reintentos registrados.

---

## Parte 4. Contrato y staging (12 min)

### 4.1 Contrato de la respuesta con Pydantic

El contrato define qué es un registro válido. Observa:

- **Alias:** la API usa `camelCase` (`fechaOperacion`) y el modelo usa `snake_case`.
- **`Decimal`:** el dinero nunca se maneja con `float`.
- **`AwareDatetime`:** exige zona horaria.
- **`model_validator`:** comprueba que el total cuadre con los productos.

`validar_lote()` no se detiene en el primer error. Cada registro inválido va a cuarentena con **todos** sus motivos, y el resto del lote continúa:

**Archivo:** `src/integracion_ventas/modelos.py`

```python
"""Contrato de la respuesta con Pydantic y cuarentena de registros inválidos."""

import json
from datetime import UTC, datetime
from decimal import Decimal
from pathlib import Path
from typing import Any, Literal

from pydantic import (
    AwareDatetime,
    BaseModel,
    Field,
    ValidationError,
    field_validator,
    model_validator,
)

LIMITE_OPERATIVO = Decimal("50000.00")


class ItemTransaccion(BaseModel):
    sku: str = Field(min_length=5, max_length=15)
    cantidad: int = Field(gt=0)
    precio_unitario: Decimal = Field(alias="precioUnitario", gt=0, decimal_places=2)


class Cliente(BaseModel):
    nombre: str = Field(min_length=1)
    segmento: str | None = None


class TransaccionAPI(BaseModel):
    id: str = Field(pattern=r"^TXN-\d{5}$")
    fecha_operacion: AwareDatetime = Field(alias="fechaOperacion")  # exige zona
    cliente_id: int = Field(alias="clienteId")  # "1001" -> 1001
    cliente: Cliente
    items: list[ItemTransaccion] = Field(min_length=1)
    total: Decimal = Field(decimal_places=2)
    moneda: Literal["MXN", "USD"] = "MXN"

    @field_validator("total")
    @classmethod
    def limite_operativo(cls, valor: Decimal) -> Decimal:
        if valor > LIMITE_OPERATIVO:
            raise ValueError("supera el límite operativo de 50,000")
        return valor

    @model_validator(mode="after")
    def total_cuadra(self) -> "TransaccionAPI":
        calculado = sum(i.cantidad * i.precio_unitario for i in self.items)
        if calculado != self.total:
            raise ValueError(f"total {self.total} no coincide con items ({calculado})")
        return self


def validar_lote(
    registros: list[dict[str, Any]],
) -> tuple[list[TransaccionAPI], list[dict[str, Any]]]:
    """Un registro inválido se aísla con TODOS sus errores; el resto continúa."""
    validas: list[TransaccionAPI] = []
    cuarentena: list[dict[str, Any]] = []
    for registro in registros:
        try:
            validas.append(TransaccionAPI.model_validate(registro))
        except ValidationError as error:
            motivos = [
                f"{'.'.join(map(str, e['loc'])) or 'registro'}: {e['msg']}"
                for e in error.errors()
            ]
            cuarentena.append(
                {"id": registro.get("id"), "motivos": motivos, "registro": registro}
            )
    return validas, cuarentena


def guardar_cuarentena(
    cuarentena: list[dict[str, Any]], carpeta: Path, id_ejecucion: str
) -> Path | None:
    """Archivo auditable para revisión y reproceso (dead-letter queue)."""
    if not cuarentena:
        return None
    carpeta.mkdir(parents=True, exist_ok=True)
    marca = datetime.now(UTC).strftime("%Y%m%dT%H%M%SZ")
    destino = carpeta / f"{marca}_{id_ejecucion}.json"
    destino.write_text(
        json.dumps(cuarentena, ensure_ascii=False, indent=2), encoding="utf-8"
    )
    return destino
```

### 4.2 Aplanar y escribir el archivo de staging

El warehouse recibe filas, no JSON anidado. `aplanar()` normaliza la fecha a UTC y convierte el segmento vacío en `""` (que el warehouse guardará como `NULL`). `persistir_staging()` recibe un **iterable**: escribe fila por fila mientras llegan las páginas. Primero escribe un `.tmp` y al final lo renombra, así nunca queda un archivo a medias. `leer_staging()` lo usarán la carga y la reconciliación:

**Archivo:** `src/integracion_ventas/staging.py`

```python
"""Transformación (aplanar al esquema destino) y archivo de staging .csv.gz."""

import csv
import gzip
from collections.abc import Iterable
from datetime import UTC
from pathlib import Path

from integracion_ventas.modelos import TransaccionAPI

# Mismo orden que la tabla de staging: COPY INTO carga CSV por posición.
COLUMNAS = [
    "ID_TRANSACCION",
    "FECHA_OPERACION",
    "CLIENTE_ID",
    "CLIENTE_NOMBRE",
    "SEGMENTO",
    "NUM_ITEMS",
    "TOTAL",
    "MONEDA",
]


def aplanar(tx: TransaccionAPI) -> dict[str, str]:
    """JSON anidado -> fila relacional. Las fechas se normalizan a UTC."""
    return {
        "ID_TRANSACCION": tx.id,
        "FECHA_OPERACION": tx.fecha_operacion.astimezone(UTC).strftime(
            "%Y-%m-%d %H:%M:%S"
        ),
        "CLIENTE_ID": str(tx.cliente_id),
        "CLIENTE_NOMBRE": tx.cliente.nombre,
        "SEGMENTO": tx.cliente.segmento or "",  # vacío -> NULL en el warehouse
        "NUM_ITEMS": str(len(tx.items)),
        "TOTAL": f"{tx.total:.2f}",
        "MONEDA": tx.moneda,
    }


def persistir_staging(filas: Iterable[dict[str, str]], ruta: Path) -> int:
    """Escribe .csv.gz; primero a .tmp y luego renombra (sin archivos a medias)."""
    ruta.parent.mkdir(parents=True, exist_ok=True)
    temporal = ruta.with_name(ruta.name + ".tmp")
    total = 0
    try:
        with gzip.open(temporal, "wt", encoding="utf-8", newline="") as archivo:
            escritor = csv.DictWriter(archivo, fieldnames=COLUMNAS)
            escritor.writeheader()
            for fila in filas:
                escritor.writerow(fila)
                total += 1
    except BaseException:
        temporal.unlink(missing_ok=True)  # si la extracción falla, no queda nada
        raise
    temporal.replace(ruta)  # renombre atómico en el mismo disco
    return total


def leer_staging(ruta: Path) -> list[dict[str, str]]:
    with gzip.open(ruta, "rt", encoding="utf-8", newline="") as archivo:
        return list(csv.DictReader(archivo))
```

> **Checkpoint 4.** Valida los 25 registros de la API simulada sin pasar por la red:
>
> ```bash
> python -c "from api_simulada import DATOS; from integracion_ventas.modelos import validar_lote; v, c = validar_lote(DATOS); print(len(v), 'válidos', len(c), 'en cuarentena'); [print(x['id'], x['motivos']) for x in c]"
> ```
>
> Resultado esperado:
>
> ```text
> 20 válidos 5 en cuarentena
> TXN-00005 ['registro: Value error, total 408.00 no coincide con items (407.00)']
> TXN-9 ["id: String should match pattern '^TXN-\\d{5}$'"]
> TXN-00013 ['items.0.cantidad: Input should be greater than 0']
> TXN-00017 ['fechaOperacion: Input should be a valid datetime or date, invalid character in year']
> TXN-00021 ['clienteId: Input should be a valid integer, unable to parse string as an integer']
> ```
>
> `registro:` indica que el error es del registro completo (el `model_validator`), no de un campo.

---

## Parte 5. Warehouse y reconciliación (20 min)

### 5.1 El destino: Snowflake o emulador con el mismo código

`warehouse.conectar()` devuelve una conexión del conector **oficial** de Snowflake:

- **`SF_MODO=emulador`:** `fakesnow.patch()` intercepta `snowflake.connector.connect` y ejecuta el SQL de Snowflake sobre DuckDB. Los datos persisten en la carpeta `emulador_snowflake/`.
- **`SF_MODO=real`:** se conecta a tu cuenta con autenticación por **par de claves**, sin contraseña.

La carga se hace en tres pasos:

1. `TRUNCATE` de la tabla de staging.
2. Llenar el staging:
   - **Real:** `PUT` sube el `.csv.gz` a una carpeta del stage **exclusiva de esta ejecución** y `COPY INTO` carga solo esa carpeta.
   - **Emulador:** las mismas filas con `INSERT` parametrizado por lotes (`executemany`), porque fakesnow no resuelve de forma confiable sus stages internos en Windows.
3. `MERGE` por `ID_TRANSACCION`: inserta las nuevas y actualiza las existentes. Es la clave de la **idempotencia**: ejecutar dos veces no duplica.

**Archivo:** `src/integracion_ventas/warehouse.py`

```python
"""Destino Snowflake: el mismo código sirve para la cuenta real y para el emulador."""

import logging
import warnings
from collections.abc import Iterator
from contextlib import contextmanager
from pathlib import Path
from typing import Any

import snowflake.connector

from integracion_ventas.configuracion import Configuracion
from integracion_ventas.errores import EsquemaDestinoError
from integracion_ventas.staging import COLUMNAS, leer_staging

# DuckDB (motor del emulador) emite este aviso de rendimiento en cada consulta.
warnings.filterwarnings("ignore", message="Arrow output on a DB-API result")

logger = logging.getLogger(__name__)

TABLA_FINAL = "VENTAS_API"
TABLA_STAGING = "STG_VENTAS_API"
TABLA_CONTROL = "CONTROL_EJECUCIONES"
STAGE = "STG_INTEGRACION"
# Lista blanca: los identificadores no se pueden parametrizar con %s.
TABLAS_PERMITIDAS = {TABLA_FINAL, TABLA_STAGING, TABLA_CONTROL}

COLUMNAS_DDL = """
    ID_TRANSACCION  VARCHAR(20) NOT NULL,
    FECHA_OPERACION TIMESTAMP_NTZ,
    CLIENTE_ID      NUMBER(10,0),
    CLIENTE_NOMBRE  VARCHAR(200),
    SEGMENTO        VARCHAR(30),
    NUM_ITEMS       NUMBER(5,0),
    TOTAL           NUMBER(12,2),
    MONEDA          VARCHAR(3)"""

# Tipo lógico que devuelve INFORMATION_SCHEMA.COLUMNS.DATA_TYPE en Snowflake
ESQUEMA_ESPERADO = {
    "ID_TRANSACCION": "TEXT",
    "FECHA_OPERACION": "TIMESTAMP_NTZ",
    "CLIENTE_ID": "NUMBER",
    "CLIENTE_NOMBRE": "TEXT",
    "SEGMENTO": "TEXT",
    "NUM_ITEMS": "NUMBER",
    "TOTAL": "NUMBER",
    "MONEDA": "TEXT",
    "ID_EJECUCION": "TEXT",
}

SQL_INSERT_STAGING = (
    f"INSERT INTO {TABLA_STAGING} ({', '.join(COLUMNAS)}) "
    f"VALUES ({', '.join(['%s'] * len(COLUMNAS))})"
)

SQL_COPY = f"""
COPY INTO {TABLA_STAGING} FROM @{STAGE}/{{carpeta}}/
  FILE_FORMAT = (TYPE = CSV SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"'
                 COMPRESSION = GZIP)
  ON_ERROR = ABORT_STATEMENT
  PURGE = TRUE"""

_SET = ", ".join(f"d.{c} = s.{c}" for c in COLUMNAS[1:])
SQL_MERGE = f"""
MERGE INTO {TABLA_FINAL} AS d USING {TABLA_STAGING} AS s
  ON d.ID_TRANSACCION = s.ID_TRANSACCION
WHEN MATCHED THEN UPDATE SET {_SET}, d.ID_EJECUCION = %s
WHEN NOT MATCHED THEN INSERT ({", ".join(COLUMNAS)}, ID_EJECUCION)
  VALUES ({", ".join(f"s.{c}" for c in COLUMNAS)}, %s)"""


def cargar_clave_privada(ruta_pem: Path, passphrase: str) -> bytes:
    """Descifra la clave PKCS#8 en memoria y la entrega en DER (modo real)."""
    from cryptography.hazmat.primitives import serialization

    clave = serialization.load_pem_private_key(
        ruta_pem.read_bytes(), password=passphrase.encode()
    )
    return clave.private_bytes(
        encoding=serialization.Encoding.DER,
        format=serialization.PrivateFormat.PKCS8,
        encryption_algorithm=serialization.NoEncryption(),
    )


@contextmanager
def conectar(config: Configuracion) -> Iterator[Any]:
    """Devuelve una conexión del conector oficial; se cierra SIEMPRE."""
    if config.sf_modo == "emulador":
        import fakesnow  # intercepta snowflake.connector.connect y usa DuckDB

        config.emulador_dir.mkdir(parents=True, exist_ok=True)
        with fakesnow.patch(db_path=str(config.emulador_dir)):
            conn = snowflake.connector.connect(
                account="emulador",
                user="SVC_INTEGRACION",
                database=config.sf_database,
                schema=config.sf_schema,
                autocommit=True,
            )
            try:
                cur = conn.cursor()
                esquema = f"{config.sf_database}.{config.sf_schema}"
                # En la cuenta real, base y esquema los crea un administrador.
                cur.execute(f"CREATE DATABASE IF NOT EXISTS {config.sf_database}")
                cur.execute(f"CREATE SCHEMA IF NOT EXISTS {esquema}")
                cur.execute(f"USE SCHEMA {esquema}")
                yield conn
            finally:
                conn.close()
        return

    assert config.sf_private_key_path and config.sf_private_key_passphrase
    conn = snowflake.connector.connect(
        account=config.sf_account,
        user=config.sf_user,  # usuario de servicio, no personal
        private_key=cargar_clave_privada(
            config.sf_private_key_path,
            config.sf_private_key_passphrase.get_secret_value(),
        ),
        role=config.sf_role,
        warehouse=config.sf_warehouse,
        database=config.sf_database,
        schema=config.sf_schema,
        autocommit=True,
        login_timeout=30,
        network_timeout=120,
    )
    try:
        yield conn
    finally:
        conn.close()


def preparar_destino(cur: Any) -> None:
    """DDL idempotente: si ya existe, no cambia nada."""
    cur.execute(
        f"CREATE TABLE IF NOT EXISTS {TABLA_FINAL} ({COLUMNAS_DDL},\n"
        "    ID_EJECUCION VARCHAR(40))"
    )
    cur.execute(f"CREATE TABLE IF NOT EXISTS {TABLA_STAGING} ({COLUMNAS_DDL})")
    cur.execute(
        f"CREATE TABLE IF NOT EXISTS {TABLA_CONTROL} ("
        "ID_EJECUCION VARCHAR(40), FIN TIMESTAMP_NTZ, RECIBIDOS NUMBER(10,0), "
        "VALIDOS NUMBER(10,0), CUARENTENA NUMBER(10,0), INSERTADOS NUMBER(10,0), "
        "ACTUALIZADOS NUMBER(10,0), ESTADO VARCHAR(20))"
    )
    cur.execute(f"CREATE STAGE IF NOT EXISTS {STAGE}")


def esquema_actual(cur: Any, esquema: str, tabla: str) -> dict[str, str]:
    """Metadatos técnicos del catálogo; los nombres viajan como PARÁMETROS."""
    if tabla not in TABLAS_PERMITIDAS:
        raise ValueError(f"Tabla no permitida: {tabla}")
    filas = cur.execute(
        "SELECT column_name, data_type FROM information_schema.columns "
        "WHERE table_schema = %s AND table_name = %s",
        (esquema.upper(), tabla),
    ).fetchall()
    return {str(nombre).upper(): str(tipo).upper() for nombre, tipo in filas}


def validar_esquema_destino(cur: Any, esquema: str) -> list[str]:
    """Faltantes o tipos distintos rompen; columnas nuevas solo se avisan.

    Raises:
        EsquemaDestinoError: Si el destino no es compatible; no se carga nada.
    """
    actual = esquema_actual(cur, esquema, TABLA_FINAL)
    faltan = sorted(set(ESQUEMA_ESPERADO) - set(actual))
    distintos = {
        c: (t, actual[c])
        for c, t in ESQUEMA_ESPERADO.items()
        if c in actual and actual[c] != t
    }
    if faltan or distintos:
        raise EsquemaDestinoError(
            f"{TABLA_FINAL} incompatible: faltan={faltan} tipos_distintos={distintos}"
        )
    nuevas = sorted(set(actual) - set(ESQUEMA_ESPERADO))
    for columna in nuevas:
        logger.warning("Columna nueva en %s: %s (se ignora)", TABLA_FINAL, columna)
    return nuevas


def llenar_staging_copy(cur: Any, archivo: Path, carpeta: str) -> int:
    """Snowflake real: PUT sube el .csv.gz a un stage interno y COPY INTO lo carga.

    Cada ejecución usa su propia carpeta del stage: COPY solo lee ese archivo.
    """
    ruta = archivo.resolve().as_posix()
    if "'" in ruta:
        raise ValueError("La ruta del archivo no debe contener comillas simples")
    # La ruta la genera el propio script: no es entrada externa.
    cur.execute(
        f"PUT 'file://{ruta}' @{STAGE}/{carpeta}/ "
        "AUTO_COMPRESS = FALSE OVERWRITE = TRUE"
    )
    resultado_copy = cur.execute(SQL_COPY.format(carpeta=carpeta)).fetchall()
    return sum(int(fila[3]) for fila in resultado_copy)  # columna rows_loaded


def llenar_staging_insert(cur: Any, archivo: Path) -> int:
    """Emulador: las mismas filas con INSERT parametrizado por lotes (executemany).

    fakesnow no resuelve las rutas de sus stages internos en Windows, así que en
    modo emulador no se usa PUT/COPY. El resto del flujo es idéntico.
    """
    filas = [
        [valor if valor != "" else None for valor in fila.values()]  # "" -> NULL
        for fila in leer_staging(archivo)
    ]
    cur.executemany(SQL_INSERT_STAGING, filas)
    return len(filas)


def cargar(cur: Any, archivo: Path, carpeta: str, carga_masiva: bool) -> dict[str, int]:
    """TRUNCATE staging → llenar staging → MERGE por clave (idempotente)."""
    cur.execute(f"TRUNCATE TABLE {TABLA_STAGING}")
    if carga_masiva:
        en_staging = llenar_staging_copy(cur, archivo, carpeta)
    else:
        en_staging = llenar_staging_insert(cur, archivo)
    insertadas, actualizadas = cur.execute(SQL_MERGE, (carpeta, carpeta)).fetchone()
    return {
        "en_staging": en_staging,
        "insertadas": int(insertadas),
        "actualizadas": int(actualizadas),
    }


def leer_destino(cur: Any, ids: list[str]) -> list[dict[str, Any]]:
    """Filas del destino para las claves del lote (valores parametrizados)."""
    if not ids:
        return []
    marcadores = ", ".join(["%s"] * len(ids))
    cur.execute(
        f"SELECT {', '.join(COLUMNAS)} FROM {TABLA_FINAL} "
        f"WHERE ID_TRANSACCION IN ({marcadores})",
        ids,
    )
    return [dict(zip(COLUMNAS, fila, strict=True)) for fila in cur.fetchall()]


def registrar_ejecucion(cur: Any, datos: dict[str, Any]) -> None:
    """Metadatos operacionales de la ejecución, con SQL parametrizado."""
    cur.execute(
        f"INSERT INTO {TABLA_CONTROL} (ID_EJECUCION, FIN, RECIBIDOS, VALIDOS, "
        "CUARENTENA, INSERTADOS, ACTUALIZADOS, ESTADO) "
        "VALUES (%s, CURRENT_TIMESTAMP(), %s, %s, %s, %s, %s, %s)",
        (
            datos["id_ejecucion"],
            datos["recibidos"],
            datos["validos"],
            datos["cuarentena"],
            datos["insertadas"],
            datos["actualizadas"],
            datos["estado"],
        ),
    )
```

Observa las prácticas de SQL seguro (lección 3.18):

| Dónde | Qué protege |
| :--- | :--- |
| `leer_destino()`, `registrar_ejecucion()`, `MERGE`, `INSERT`: valores con `%s` | El conector envía los valores por separado del SQL; un nombre como `O'Brien'); DROP TABLE…` es solo texto |
| `esquema_actual()`: nombres de esquema y tabla como **parámetros** de `INFORMATION_SCHEMA` | Consultas de metadatos sin concatenar texto |
| `TABLAS_PERMITIDAS` | Los identificadores (nombres de tabla) **no** se pueden parametrizar: se validan contra una lista blanca |
| `PUT 'file://…'` | La ruta la genera el propio programa, entre comillas, y se rechaza si contiene una comilla simple |

### 5.2 Reconciliación origen-destino

Después de cargar, se compara el archivo de staging (lo que se validó) contra lo que quedó en el warehouse, en tres niveles:

- **Conteo** de filas.
- **Suma de control** de `TOTAL`.
- **Hash SHA-256 por fila:** detecta un valor alterado aunque el conteo y la suma coincidan.

`normalizar()` da la misma representación de texto a los tipos de cada lado. Por ejemplo, `datetime` y el texto de la fecha, o `Decimal('251.0')` y `"251.00"`:

**Archivo:** `src/integracion_ventas/reconciliacion.py`

```python
"""Reconciliación origen (staging validado) contra destino en tres niveles."""

import hashlib
import json
from datetime import datetime
from decimal import Decimal
from typing import Any

from integracion_ventas.staging import COLUMNAS


def normalizar(columna: str, valor: Any) -> str:
    """Misma representación de texto en origen y destino."""
    if valor is None or valor == "":
        return ""
    if isinstance(valor, datetime):
        return valor.strftime("%Y-%m-%d %H:%M:%S")
    if columna == "TOTAL":
        return f"{Decimal(str(valor)):.2f}"
    if columna in ("CLIENTE_ID", "NUM_ITEMS"):
        return str(int(Decimal(str(valor))))
    return str(valor)


def hash_fila(fila: dict[str, Any]) -> str:
    """SHA-256 sin ambigüedad: se serializa la lista antes de calcularlo."""
    valores = [normalizar(c, fila.get(c)) for c in COLUMNAS]
    texto = json.dumps(valores, ensure_ascii=False)
    return hashlib.sha256(texto.encode("utf-8")).hexdigest()


def reconciliar(
    origen: list[dict[str, Any]], destino: list[dict[str, Any]]
) -> dict[str, Any]:
    """Nivel 1 conteo, nivel 2 suma de TOTAL, nivel 3 hash por fila."""
    o = {f["ID_TRANSACCION"]: hash_fila(f) for f in origen}
    d = {f["ID_TRANSACCION"]: hash_fila(f) for f in destino}
    suma_o = sum((Decimal(normalizar("TOTAL", f["TOTAL"])) for f in origen), Decimal(0))
    suma_d = sum(
        (Decimal(normalizar("TOTAL", f["TOTAL"])) for f in destino), Decimal(0)
    )
    resultado = {
        "conteo": [len(origen), len(destino)],
        "suma_total": [str(suma_o), str(suma_d)],
        "faltan_en_destino": sorted(set(o) - set(d)),
        "sobran_en_destino": sorted(set(d) - set(o)),
        "valores_distintos": sorted(k for k in set(o) & set(d) if o[k] != d[k]),
    }
    resultado["exito"] = (
        len(origen) == len(destino)
        and suma_o == suma_d
        and not resultado["faltan_en_destino"]
        and not resultado["sobran_en_destino"]
        and not resultado["valores_distintos"]
    )
    return resultado
```

> **Checkpoint 5.**
>
> ```bash
> python -c "import integracion_ventas.warehouse, integracion_ventas.reconciliacion; print('ok')"
> ```
>
> Resultado esperado: `ok`

---

## Parte 6. Orquestación y ejecución (15 min)

### 6.1 El pipeline

`ejecutar()` une las fases. La extracción, la validación y el staging ocurren **en flujo**: el generador interno `filas_validas()` toma una página, la valida y entrega las filas válidas a `persistir_staging()`. Después de la extracción se aplica la **compuerta de calidad**: si más del 25 % del lote quedó en cuarentena, no se carga nada.

**Archivo:** `src/integracion_ventas/pipeline.py`

```python
"""Orquestación: API → validación → staging → warehouse → reconciliación."""

import logging
import uuid
from collections.abc import Iterator
from pathlib import Path
from typing import Any

from integracion_ventas import warehouse
from integracion_ventas.api import ExtractorAPI, OAuth2Client
from integracion_ventas.configuracion import Configuracion
from integracion_ventas.errores import LoteNoAprobadoError, ReconciliacionError
from integracion_ventas.modelos import guardar_cuarentena, validar_lote
from integracion_ventas.monitoreo import monitor
from integracion_ventas.reconciliacion import reconciliar
from integracion_ventas.staging import aplanar, leer_staging, persistir_staging

logger = logging.getLogger(__name__)

CARPETA_STAGING = Path("staging")
CARPETA_CUARENTENA = Path("cuarentena")


def ejecutar(config: Configuracion) -> dict[str, Any]:
    """Ejecuta el flujo completo y devuelve el resumen.

    Raises:
        ApiError, LoteNoAprobadoError, EsquemaDestinoError, ReconciliacionError
    """
    id_ejecucion = f"ejecucion_{uuid.uuid4().hex[:12]}"  # trace_id común
    timeout = (config.api_timeout_conexion, config.api_timeout_lectura)
    oauth = OAuth2Client(
        f"{config.api_base_url.rstrip('/')}/oauth/token",
        config.api_client_id,
        config.api_client_secret.get_secret_value(),
        timeout,
    )
    extractor = ExtractorAPI(config.api_base_url, oauth, timeout)
    archivo = CARPETA_STAGING / f"{id_ejecucion}.csv.gz"
    cuarentena: list[dict[str, Any]] = []

    # 1. Extracción, validación y staging en flujo (una página en memoria)
    with monitor("extraer_validar", id_ejecucion) as m:
        m |= {"paginas": 0, "recibidos": 0}

        def filas_validas() -> Iterator[dict[str, str]]:
            for lote in extractor.lotes("/v1/transacciones", config.api_tamano_pagina):
                m["paginas"] += 1
                m["recibidos"] += len(lote)
                validas, invalidas = validar_lote(lote)
                cuarentena.extend(invalidas)
                for tx in validas:
                    yield aplanar(tx)

        m["validos"] = persistir_staging(filas_validas(), archivo)
        m["cuarentena"] = len(cuarentena)
        m["tokens_oauth"] = oauth.tokens_solicitados
        destino_dlq = guardar_cuarentena(cuarentena, CARPETA_CUARENTENA, id_ejecucion)
        m["archivo_cuarentena"] = str(destino_dlq) if destino_dlq else None
    for registro in cuarentena:
        logger.warning(
            "En cuarentena %s: %s", registro["id"], "; ".join(registro["motivos"])
        )

    recibidos, validos = m["recibidos"], m["validos"]
    pct = round(100 * len(cuarentena) / recibidos, 2) if recibidos else 0.0
    if pct > config.max_pct_cuarentena:
        raise LoteNoAprobadoError(
            f"{pct}% en cuarentena supera el máximo {config.max_pct_cuarentena}%; "
            "no se carga nada"
        )

    with warehouse.conectar(config) as conn:
        cur = conn.cursor()
        # 2. Carga idempotente
        with monitor("cargar", id_ejecucion) as m:
            warehouse.preparar_destino(cur)
            warehouse.validar_esquema_destino(cur, config.sf_schema)
            m |= warehouse.cargar(
                cur, archivo, id_ejecucion, carga_masiva=config.sf_modo == "real"
            )
        carga = dict(m)

        # 3. Reconciliación origen (staging) contra destino
        with monitor("reconciliar", id_ejecucion) as m:
            origen = leer_staging(archivo)
            destino = warehouse.leer_destino(cur, [f["ID_TRANSACCION"] for f in origen])
            m |= reconciliar(origen, destino)
        reconciliacion = dict(m)

        estado = "SUCCESS" if reconciliacion["exito"] else "RECONCILIACION_FALLIDA"
        warehouse.registrar_ejecucion(
            cur,
            {
                "id_ejecucion": id_ejecucion,
                "recibidos": recibidos,
                "validos": validos,
                "cuarentena": len(cuarentena),
                "insertadas": carga["insertadas"],
                "actualizadas": carga["actualizadas"],
                "estado": estado,
            },
        )
    if not reconciliacion["exito"]:
        raise ReconciliacionError(f"Origen y destino no coinciden: {reconciliacion}")

    return {
        "id_ejecucion": id_ejecucion,
        "recibidos": recibidos,
        "validos": validos,
        "cuarentena": len(cuarentena),
        "insertadas": carga["insertadas"],
        "actualizadas": carga["actualizadas"],
    }
```

### 6.2 Punto de entrada

`cli.py` configura los logs **dos veces**. La primera, antes de leer la configuración, para poder reportar sus errores. La segunda, ya con los secretos, para que el filtro los oculte. Cada `IntegracionError` se convierte en su código de salida:

**Archivo:** `src/integracion_ventas/cli.py`

```python
"""Punto de entrada: python -m integracion_ventas."""

import argparse
import logging

from integracion_ventas import pipeline
from integracion_ventas.configuracion import cargar_configuracion
from integracion_ventas.errores import IntegracionError
from integracion_ventas.monitoreo import configurar_logging

logger = logging.getLogger("integracion_ventas")


def main(argv: list[str] | None = None) -> int:
    argparse.ArgumentParser(
        prog="python -m integracion_ventas",
        description="Extrae transacciones de la API, las valida y las carga en "
        "Snowflake (o en el emulador local). La configuración se lee de .env.",
    ).parse_args(argv)
    configurar_logging(secretos=[])
    try:
        config = cargar_configuracion()
        secretos = [config.api_client_secret.get_secret_value()]
        if config.sf_private_key_passphrase:
            secretos.append(config.sf_private_key_passphrase.get_secret_value())
        configurar_logging(secretos)  # desde aquí, ningún secreto llega a los logs
        resumen = pipeline.ejecutar(config)
    except IntegracionError as error:
        logger.error(
            "%s", error, extra={"telemetria": {"codigo_salida": error.codigo_salida}}
        )
        return error.codigo_salida
    except Exception:
        logger.exception("Error inesperado")
        return 1
    logger.info("resumen", extra={"telemetria": resumen})
    return 0
```

**Archivo:** `src/integracion_ventas/__main__.py`

```python
"""Permite ejecutar el paquete con: python -m integracion_ventas."""

from integracion_ventas.cli import main

raise SystemExit(main())
```

### 6.3 Script de consulta del destino

Usa la misma configuración que el pipeline, así que funciona igual contra el emulador o contra Snowflake real:

**Archivo:** `scripts/consultar_destino.py`

```python
"""Consulta el destino (emulador o Snowflake real) con la misma configuración."""

from integracion_ventas import warehouse
from integracion_ventas.configuracion import cargar_configuracion

NOMBRE_SOSPECHOSO = "O'Brien'); DROP TABLE VENTAS_API;--"


def main() -> None:
    config = cargar_configuracion()
    with warehouse.conectar(config) as conn:
        cur = conn.cursor()
        warehouse.preparar_destino(cur)  # idempotente: si no existen, quedan vacías
        filas, suma = cur.execute(
            f"SELECT COUNT(*), SUM(TOTAL) FROM {warehouse.TABLA_FINAL}"
        ).fetchone()
        print(f"{warehouse.TABLA_FINAL}: {filas} filas | suma TOTAL = {suma or 0:.2f}")
        # El valor viaja como parámetro: se compara como texto literal.
        literal = cur.execute(
            f"SELECT COUNT(*) FROM {warehouse.TABLA_FINAL} WHERE CLIENTE_NOMBRE = %s",
            (NOMBRE_SOSPECHOSO,),
        ).fetchone()[0]
        print(f"Filas con el nombre {NOMBRE_SOSPECHOSO!r}: {literal}")
        print(f"Ejecuciones registradas en {warehouse.TABLA_CONTROL}:")
        for ejecucion, rec, val, cua, ins, act, estado in cur.execute(
            f"SELECT ID_EJECUCION, RECIBIDOS, VALIDOS, CUARENTENA, INSERTADOS, "
            f"ACTUALIZADOS, ESTADO FROM {warehouse.TABLA_CONTROL} ORDER BY FIN"
        ).fetchall():
            print(
                f"  {ejecucion}  recibidos={rec} válidos={val} cuarentena={cua} "
                f"insertadas={ins} actualizadas={act}  {estado}"
            )


if __name__ == "__main__":
    main()
```

### Códigos de salida

| Código | Significado |
| :---: | :--- |
| 0 | Extracción, carga y reconciliación correctas |
| 1 | Configuración inválida o error inesperado |
| 2 | Argumentos de línea de comandos inválidos |
| 3 | Error de la API: autenticación rechazada o reintentos agotados |
| 4 | La reconciliación no cuadra |
| 5 | Demasiados registros en cuarentena; no se carga nada |
| 6 | La tabla destino no tiene el esquema esperado; no se carga nada |

Para ver el código de salida del último comando: **PowerShell** `$LASTEXITCODE` | **bash** `echo $?`

### 6.4 Primera ejecución

Con la API corriendo en la Terminal 1, en la **Terminal 2**:

```bash
python -m integracion_ventas --help
python -m integracion_ventas
```

Resultado esperado: el código de salida es `0` y aparece una línea JSON por evento. Aquí se muestran abreviadas (`…`), con los campos clave:

```text
{"nivel": "WARNING", "logger": "integracion_ventas.api", "mensaje": "Reintento automático por ReadTimeoutError (quedan 3)"}
{"nivel": "WARNING", "logger": "integracion_ventas.api", "mensaje": "Reintento automático por HTTP 429 (quedan 3)"}
{"nivel": "WARNING", "logger": "integracion_ventas.api", "mensaje": "Reintento automático por HTTP 503 (quedan 3)"}
{"nivel": "INFO", "mensaje": "fase.extraer_validar", "trace_id": "ejecucion_…", "estado": "SUCCESS", "duracion_ms": 3018.9, "paginas": 3, "recibidos": 25, "validos": 20, "cuarentena": 5, "tokens_oauth": 1, "archivo_cuarentena": "cuarentena/…_ejecucion_….json", …}
{"nivel": "WARNING", "mensaje": "En cuarentena TXN-00005: registro: Value error, total 408.00 no coincide con items (407.00)"}
… (una línea por cada uno de los 5 registros en cuarentena)
{"nivel": "INFO", "mensaje": "fase.cargar", "trace_id": "ejecucion_…", "estado": "SUCCESS", "en_staging": 20, "insertadas": 20, "actualizadas": 0, …}
{"nivel": "INFO", "mensaje": "fase.reconciliar", "trace_id": "ejecucion_…", "estado": "SUCCESS", "conteo": [20, 20], "suma_total": ["5950.00", "5950.00"], "faltan_en_destino": [], "sobran_en_destino": [], "valores_distintos": [], "exito": true, …}
{"nivel": "INFO", "mensaje": "resumen", "id_ejecucion": "ejecucion_…", "recibidos": 25, "validos": 20, "cuarentena": 5, "insertadas": 20, "actualizadas": 0}
```

Revisa también los archivos generados:

- `staging/ejecucion_….csv.gz`: las 20 filas válidas que se cargaron.
- `cuarentena/…_ejecucion_….json`: los 5 registros originales con sus motivos, listos para revisión y reproceso.
- `emulador_snowflake/ANALITICA.db`: la base de datos del emulador.

### 6.5 Segunda ejecución: idempotencia

Ejecuta **otra vez** exactamente el mismo comando:

```bash
python -m integracion_ventas
```

Resultado esperado en `fase.cargar` y `resumen`: `"insertadas": 0, "actualizadas": 20`. Los mismos datos no se duplican.

### 6.6 Consultar el destino

```bash
python scripts/consultar_destino.py
```

Resultado esperado (los identificadores de ejecución cambian):

```text
VENTAS_API: 20 filas | suma TOTAL = 5950.00
Filas con el nombre "O'Brien'); DROP TABLE VENTAS_API;--": 4
Ejecuciones registradas en CONTROL_EJECUCIONES:
  ejecucion_…  recibidos=25 válidos=20 cuarentena=5 insertadas=20 actualizadas=0  SUCCESS
  ejecucion_…  recibidos=25 válidos=20 cuarentena=5 insertadas=0 actualizadas=20  SUCCESS
```

- Después de dos ejecuciones hay **20 filas, no 40**.
- El nombre con `'); DROP TABLE` está guardado **como dato** en 4 filas, y la tabla sigue ahí.

### 6.7 Cambiar el tamaño de página sin tocar el código

Ejecuta con páginas de 5 registros. La variable de entorno tiene prioridad sobre `.env`.

**PowerShell**
```powershell
$env:API_TAMANO_PAGINA = "5"; python -m integracion_ventas; Remove-Item Env:API_TAMANO_PAGINA
```

**bash**
```bash
API_TAMANO_PAGINA=5 python -m integracion_ventas
```

Resultado esperado: `"paginas": 5` en `fase.extraer_validar` y otra vez `"insertadas": 0, "actualizadas": 20`. El resultado no depende de cómo pagine la API.

---

## Parte 7. Escenarios de falla (15 min)

En cada escenario anota el **código de salida** y qué evento lo explica.

### 7.1 Credencial incorrecta: error permanente

**PowerShell**
```powershell
$env:API_CLIENT_SECRET = "incorrecto"; python -m integracion_ventas; $LASTEXITCODE; Remove-Item Env:API_CLIENT_SECRET
```

**bash**
```bash
API_CLIENT_SECRET=incorrecto python -m integracion_ventas; echo $?
```

Resultado esperado:

```text
{"nivel": "ERROR", "mensaje": "fase.extraer_validar", "estado": "FAILURE", "duracion_ms": 4.0, "paginas": 0, "recibidos": 0, "error": "ApiError: Autenticación rechazada: HTTP 401", …}
{"nivel": "ERROR", "mensaje": "Autenticación rechazada: HTTP 401", "codigo_salida": 3}
3
```

Termina en milisegundos y **sin reintentos**: un 401 no se arregla repitiendo. El secreto no aparece en ningún log.

### 7.2 API caída: error transitorio que se agota

**7.2.1** En la **Terminal 1** presiona **Ctrl+C** para detener la API.

**7.2.2** En la **Terminal 2**:

```bash
python -m integracion_ventas
```

Resultado esperado (código de salida `3`; en Windows puede tardar unos segundos más):

```text
{"nivel": "WARNING", "mensaje": "Reintento automático por NewConnectionError (quedan 2)"}
{"nivel": "WARNING", "mensaje": "Reintento automático por NewConnectionError (quedan 1)"}
{"nivel": "WARNING", "mensaje": "Reintento automático por NewConnectionError (quedan 0)"}
{"nivel": "ERROR", "mensaje": "fase.extraer_validar", "estado": "FAILURE", …}
{"nivel": "ERROR", "mensaje": "No se pudo contactar el servidor de tokens http://127.0.0.1:8000/oauth/token (ConnectionError)", "codigo_salida": 3}
```

Compara con 7.1: aquí **sí** hubo reintentos, porque la falla era de conexión (transitoria).

### 7.3 La API cambia su esquema sin avisar

**7.3.1** En la **Terminal 1**, levanta la versión 2 de la API (renombra `total` a `importe`):

```bash
python api_simulada.py --esquema v2
```

**7.3.2** En la **Terminal 2**:

```bash
python -m integracion_ventas
```

Resultado esperado (código de salida `5`):

```text
{"nivel": "INFO", "mensaje": "fase.extraer_validar", "estado": "SUCCESS", "paginas": 3, "recibidos": 25, "validos": 0, "cuarentena": 25, …}
{"nivel": "WARNING", "mensaje": "En cuarentena TXN-00001: total: Field required"}
… (25 registros en cuarentena)
{"nivel": "ERROR", "mensaje": "100.0% en cuarentena supera el máximo 25.0%; no se carga nada", "codigo_salida": 5}
```

El contrato Pydantic detectó el cambio de esquema del **origen** y la compuerta de calidad evitó cargar el lote. Ejecuta `python scripts/consultar_destino.py`: siguen las mismas 20 filas.

**7.3.3** Detén la API v2 con **Ctrl+C** y vuelve a levantar la normal en la Terminal 1:

```bash
python api_simulada.py
```

### 7.4 Alguien altera la tabla destino

**7.4.1** Crea este script. Simula un cambio no coordinado en el destino y solo funciona contra el emulador:

**Archivo:** `scripts/alterar_destino.py`

```python
"""Simula un cambio no coordinado en la tabla destino (para el ejercicio de esquema).

python scripts/alterar_destino.py --romper      # elimina la columna SEGMENTO
python scripts/alterar_destino.py --restaurar   # borra la tabla; la siguiente
                                                # ejecución la recrea y recarga
"""

import argparse

from integracion_ventas import warehouse
from integracion_ventas.configuracion import cargar_configuracion


def main() -> None:
    parser = argparse.ArgumentParser()
    accion = parser.add_mutually_exclusive_group(required=True)
    accion.add_argument("--romper", action="store_true")
    accion.add_argument("--restaurar", action="store_true")
    args = parser.parse_args()
    config = cargar_configuracion()
    if config.sf_modo != "emulador":
        raise SystemExit("Este script solo modifica el emulador local.")
    with warehouse.conectar(config) as conn:
        cur = conn.cursor()
        if args.romper:
            cur.execute(f"ALTER TABLE {warehouse.TABLA_FINAL} DROP COLUMN SEGMENTO")
            print(f"Columna SEGMENTO eliminada de {warehouse.TABLA_FINAL}")
        else:
            cur.execute(f"DROP TABLE IF EXISTS {warehouse.TABLA_FINAL}")
            print(f"Tabla {warehouse.TABLA_FINAL} eliminada; se recreará al ejecutar")


if __name__ == "__main__":
    main()
```

**7.4.2** Elimina una columna del destino y ejecuta:

```bash
python scripts/alterar_destino.py --romper
python -m integracion_ventas
```

Resultado esperado (código de salida `6`):

```text
Columna SEGMENTO eliminada de VENTAS_API
…
{"nivel": "ERROR", "mensaje": "fase.cargar", "estado": "FAILURE", "error": "EsquemaDestinoError: VENTAS_API incompatible: faltan=['SEGMENTO'] tipos_distintos={}", …}
{"nivel": "ERROR", "mensaje": "VENTAS_API incompatible: faltan=['SEGMENTO'] tipos_distintos={}", "codigo_salida": 6}
```

El catálogo (`INFORMATION_SCHEMA.COLUMNS`) reveló el cambio **antes** de cargar.

**7.4.3** Restaura el destino y vuelve a cargar:

```bash
python scripts/alterar_destino.py --restaurar
python -m integracion_ventas
```

Resultado esperado: código de salida `0` con `"insertadas": 20, "actualizadas": 0`. La tabla se recreó y se volvió a llenar desde la API.

### 7.5 Modo real sin credenciales: fail-fast

**PowerShell**
```powershell
$env:SF_MODO = "real"; python -m integracion_ventas; $LASTEXITCODE; Remove-Item Env:SF_MODO
```

**bash**
```bash
SF_MODO=real python -m integracion_ventas; echo $?
```

Resultado esperado:

```text
{"nivel": "ERROR", "mensaje": "Configuración inválida: configuración: Value error, SF_MODO=real requiere: SF_ACCOUNT, SF_USER, SF_ROLE, SF_WAREHOUSE, SF_PRIVATE_KEY_PATH, SF_PRIVATE_KEY_PASSPHRASE", "codigo_salida": 1}
1
```

No se llamó a la API ni al warehouse: la configuración se valida primero.

### 7.6 Preguntas de cierre

1. **Seguridad.** Busca `s3cr3to-lab-2026` en todo lo que imprimió la Terminal 2. ¿Aparece en algún evento? ¿Qué dos mecanismos lo impiden?
2. **Idempotencia.** Si el proceso se cayera justo después del `MERGE` y lo volvieras a ejecutar, ¿cuántas filas tendría `VENTAS_API`? ¿Por qué?
3. **Integración (3.28).** En una plataforma como Fivetran o SnapLogic, ¿qué partes de este proyecto te daría ya resueltas el conector, y cuáles seguirían siendo tu responsabilidad (contrato, reconciliación, compuerta de calidad)?

---

## Parte 8. Pruebas y calidad del código (8 min)

Las pruebas levantan la API simulada en un hilo, en un puerto libre, y trabajan en una carpeta temporal. Tus carpetas `staging/`, `cuarentena/` y `emulador_snowflake/` no se tocan, y la API de la Terminal 1 puede seguir corriendo.

**8.1** Crea `tests/__init__.py` y `tests/conftest.py`:

**Archivo:** `tests/__init__.py`

```python
"""Pruebas de la Práctica 3."""
```

**Archivo:** `tests/conftest.py`

```python
"""Fixtures: API simulada en un hilo y configuración aislada en una carpeta temporal."""

import threading
from collections.abc import Iterator
from pathlib import Path

import pytest

from api_simulada import CLIENT_SECRET, crear_servidor
from integracion_ventas.configuracion import Configuracion


def _levantar(esquema: str, con_fallas: bool) -> Iterator[str]:
    servidor = crear_servidor(0, esquema, con_fallas)  # puerto 0: uno libre
    hilo = threading.Thread(target=servidor.serve_forever, daemon=True)
    hilo.start()
    yield f"http://127.0.0.1:{servidor.server_address[1]}"
    servidor.shutdown()
    servidor.server_close()


@pytest.fixture
def api_url() -> Iterator[str]:
    yield from _levantar("v1", con_fallas=False)


@pytest.fixture
def api_url_con_fallas() -> Iterator[str]:
    yield from _levantar("v1", con_fallas=True)


@pytest.fixture
def api_url_v2() -> Iterator[str]:
    yield from _levantar("v2", con_fallas=False)


@pytest.fixture
def carpeta(tmp_path: Path, monkeypatch: pytest.MonkeyPatch) -> Path:
    """Cada prueba trabaja en su propia carpeta: sin .env, staging ni emulador."""
    monkeypatch.chdir(tmp_path)
    for variable in ("API_CLIENT_SECRET", "SF_MODO", "API_BASE_URL"):
        monkeypatch.delenv(variable, raising=False)
    return tmp_path


def hacer_config(url: str, **cambios: object) -> Configuracion:
    valores: dict[str, object] = {
        "api_base_url": url,
        "api_client_id": "lab-integracion",
        "api_client_secret": CLIENT_SECRET,
        "api_timeout_lectura": 2,
    }
    valores |= cambios
    return Configuracion(**valores)  # type: ignore[arg-type]
```

**8.2** Crea `tests/test_modelos.py`:

**Archivo:** `tests/test_modelos.py`

```python
from decimal import Decimal
from typing import Any

from integracion_ventas.modelos import TransaccionAPI, validar_lote


def registro(**cambios: Any) -> dict[str, Any]:
    base: dict[str, Any] = {
        "id": "TXN-00001",
        "fechaOperacion": "2026-10-24T09:07:00-06:00",
        "clienteId": "1002",
        "cliente": {"nombre": "Ana López", "segmento": None},
        "items": [{"sku": "SKU-1001", "cantidad": 2, "precioUnitario": "125.50"}],
        "total": "251.00",
        "moneda": "MXN",
    }
    return base | cambios


def test_registro_valido_convierte_tipos() -> None:
    tx = TransaccionAPI.model_validate(registro())
    assert tx.cliente_id == 1002  # texto "1002" -> int
    assert tx.total == Decimal("251.00")
    assert tx.items[0].precio_unitario == Decimal("125.50")


def test_total_que_no_cuadra_va_a_cuarentena_con_motivo() -> None:
    validas, cuarentena = validar_lote([registro(total="999.00")])
    assert validas == []
    assert cuarentena[0]["id"] == "TXN-00001"
    assert cuarentena[0]["motivos"][0].startswith("registro: Value error, total")


def test_fecha_sin_zona_horaria_se_rechaza() -> None:
    _, cuarentena = validar_lote([registro(fechaOperacion="2026-10-24T09:07:00")])
    assert cuarentena[0]["motivos"][0].startswith("fechaOperacion:")


def test_limite_operativo() -> None:
    caro = registro(
        items=[{"sku": "SKU-1001", "cantidad": 1, "precioUnitario": "60000.00"}],
        total="60000.00",
    )
    _, cuarentena = validar_lote([caro])
    assert "límite operativo" in cuarentena[0]["motivos"][0]


def test_un_registro_malo_no_detiene_el_lote() -> None:
    validas, cuarentena = validar_lote([registro(), registro(id="TXN-9"), registro()])
    assert (len(validas), len(cuarentena)) == (2, 1)
```

**8.3** Crea `tests/test_api.py`:

**Archivo:** `tests/test_api.py`

```python
import logging
import time

import pytest

from integracion_ventas.api import ExtractorAPI, OAuth2Client
from integracion_ventas.errores import ApiError

TIMEOUT = (3.05, 2.0)


def extractor(url: str, secreto: str = "s3cr3to-lab-2026") -> ExtractorAPI:
    oauth = OAuth2Client(f"{url}/oauth/token", "lab-integracion", secreto, TIMEOUT)
    return ExtractorAPI(url, oauth, TIMEOUT)


def test_paginacion_por_cursor_trae_todo(api_url: str) -> None:
    paginas = list(extractor(api_url).lotes("/v1/transacciones", 10))
    assert [len(p) for p in paginas] == [10, 10, 5]
    ids = [r["id"] for p in paginas for r in p]
    assert len(ids) == len(set(ids)) == 25


def test_un_solo_token_para_todas_las_paginas(api_url: str) -> None:
    ext = extractor(api_url)
    list(ext.lotes("/v1/transacciones", 5))
    assert ext._oauth.tokens_solicitados == 1  # caché del token


def test_resiste_timeout_429_y_503(
    api_url_con_fallas: str, caplog: pytest.LogCaptureFixture
) -> None:
    caplog.set_level(logging.WARNING)
    paginas = list(extractor(api_url_con_fallas).lotes("/v1/transacciones", 10))
    assert sum(len(p) for p in paginas) == 25
    avisos = " ".join(r.getMessage() for r in caplog.records)
    for motivo in ("ReadTimeoutError", "HTTP 429", "HTTP 503"):
        assert motivo in avisos


def test_credencial_incorrecta_falla_sin_reintentar(api_url: str) -> None:
    inicio = time.monotonic()
    with pytest.raises(ApiError, match="Autenticación rechazada: HTTP 401"):
        list(extractor(api_url, secreto="incorrecto").lotes("/v1/transacciones", 10))
    assert time.monotonic() - inicio < 1  # error permanente: sin esperas


def test_servidor_caido_reintenta_y_termina_en_api_error(
    caplog: pytest.LogCaptureFixture,
) -> None:
    caplog.set_level(logging.WARNING)
    with pytest.raises(ApiError, match="servidor de tokens"):
        list(extractor("http://127.0.0.1:9").lotes("/v1/transacciones", 10))
    nuestros = [r for r in caplog.records if r.name == "integracion_ventas.api"]
    assert sum("NewConnectionError" in r.getMessage() for r in nuestros) == 3
```

**8.4** Crea `tests/test_warehouse.py`. La segunda prueba ejecuta el camino de Snowflake real (`PUT` + `COPY INTO`) contra el emulador. En Windows se omite, por la misma limitación de fakesnow descrita en la Parte 5:

**Archivo:** `tests/test_warehouse.py`

```python
import sys
from pathlib import Path
from typing import Any

import pytest

from integracion_ventas import warehouse
from integracion_ventas.errores import EsquemaDestinoError
from integracion_ventas.staging import persistir_staging
from tests.conftest import hacer_config

NOMBRE = "O'Brien'); DROP TABLE VENTAS_API;--"


def fila(n: int, total: str = "100.00") -> dict[str, str]:
    return {
        "ID_TRANSACCION": f"TXN-{n:05d}",
        "FECHA_OPERACION": "2026-10-24 15:07:00",
        "CLIENTE_ID": "1005",
        "CLIENTE_NOMBRE": NOMBRE,
        "SEGMENTO": "",
        "NUM_ITEMS": "1",
        "TOTAL": total,
        "MONEDA": "MXN",
    }


def cargar_lote(
    cur: Any, filas: list[dict[str, str]], id_ejec: str, masiva: bool = False
) -> dict[str, int]:
    archivo = Path("staging") / f"{id_ejec}.csv.gz"
    persistir_staging(filas, archivo)
    return warehouse.cargar(cur, archivo, id_ejec, carga_masiva=masiva)


def test_carga_idempotente_y_literal_seguro(carpeta: Path) -> None:
    config = hacer_config("http://no-se-usa")
    with warehouse.conectar(config) as conn:
        cur = conn.cursor()
        warehouse.preparar_destino(cur)
        warehouse.validar_esquema_destino(cur, config.sf_schema)
        primera = cargar_lote(cur, [fila(1), fila(2)], "ejecucion_a")
        segunda = cargar_lote(cur, [fila(1, "150.00"), fila(2)], "ejecucion_b")
        filas = warehouse.leer_destino(cur, ["TXN-00001", "TXN-00002"])
        conteo = cur.execute("SELECT COUNT(*) FROM VENTAS_API").fetchone()[0]
    assert primera == {"en_staging": 2, "insertadas": 2, "actualizadas": 0}
    assert segunda == {"en_staging": 2, "insertadas": 0, "actualizadas": 2}
    assert conteo == 2  # repetir la carga no duplica
    assert {str(f["TOTAL"]) for f in filas} == {"150.00", "100.00"}
    assert all(f["CLIENTE_NOMBRE"] == NOMBRE for f in filas)  # dato, no SQL
    assert all(f["SEGMENTO"] is None for f in filas)  # vacío -> NULL


@pytest.mark.skipif(
    sys.platform == "win32", reason="fakesnow no resuelve stages internos en Windows"
)
def test_put_y_copy_into_del_modo_real(carpeta: Path) -> None:
    """El camino de Snowflake real (PUT + COPY INTO) contra el emulador."""
    with warehouse.conectar(hacer_config("http://no-se-usa")) as conn:
        cur = conn.cursor()
        warehouse.preparar_destino(cur)
        primera = cargar_lote(cur, [fila(1), fila(2)], "ejecucion_a", masiva=True)
        segunda = cargar_lote(cur, [fila(2), fila(3)], "ejecucion_b", masiva=True)
        filas = warehouse.leer_destino(cur, ["TXN-00001", "TXN-00002", "TXN-00003"])
    assert primera == {"en_staging": 2, "insertadas": 2, "actualizadas": 0}
    # COPY solo leyó la carpeta de ejecucion_b, no el archivo anterior
    assert segunda == {"en_staging": 2, "insertadas": 1, "actualizadas": 1}
    assert len(filas) == 3 and all(f["SEGMENTO"] is None for f in filas)


def test_esquema_destino_incompatible_detiene_la_carga(carpeta: Path) -> None:
    config = hacer_config("http://no-se-usa")
    with warehouse.conectar(config) as conn:
        cur = conn.cursor()
        warehouse.preparar_destino(cur)
        cur.execute("ALTER TABLE VENTAS_API DROP COLUMN SEGMENTO")
        with pytest.raises(EsquemaDestinoError, match="SEGMENTO"):
            warehouse.validar_esquema_destino(cur, config.sf_schema)


def test_lista_blanca_de_tablas(carpeta: Path) -> None:
    with warehouse.conectar(hacer_config("http://no-se-usa")) as conn:
        with pytest.raises(ValueError, match="no permitida"):
            warehouse.esquema_actual(conn.cursor(), "VENTAS", "USUARIOS; DROP")
```

**8.5** Crea `tests/test_reconciliacion.py`:

**Archivo:** `tests/test_reconciliacion.py`

```python
from datetime import datetime
from decimal import Decimal

from integracion_ventas.reconciliacion import reconciliar

ORIGEN = [
    {
        "ID_TRANSACCION": "TXN-00001",
        "FECHA_OPERACION": "2026-10-24 15:07:00",
        "CLIENTE_ID": "1002",
        "CLIENTE_NOMBRE": "Ana López",
        "SEGMENTO": "",
        "NUM_ITEMS": "2",
        "TOTAL": "251.00",
        "MONEDA": "MXN",
    }
]


def destino(**cambios: object) -> list[dict[str, object]]:
    """Así llegan los tipos desde el warehouse: datetime, Decimal, None."""
    fila: dict[str, object] = {
        "ID_TRANSACCION": "TXN-00001",
        "FECHA_OPERACION": datetime(2026, 10, 24, 15, 7),
        "CLIENTE_ID": Decimal("1002"),
        "CLIENTE_NOMBRE": "Ana López",
        "SEGMENTO": None,
        "NUM_ITEMS": Decimal("2"),
        "TOTAL": Decimal("251.0"),
        "MONEDA": "MXN",
    }
    return [fila | cambios]


def test_tipos_distintos_mismo_valor_cuadran() -> None:
    assert reconciliar(ORIGEN, destino())["exito"] is True


def test_un_valor_distinto_se_detecta_aunque_conteo_y_suma_coincidan() -> None:
    resultado = reconciliar(ORIGEN, destino(CLIENTE_NOMBRE="Ana Lopez"))
    assert resultado["conteo"] == [1, 1]
    assert resultado["suma_total"] == ["251.00", "251.00"]
    assert resultado["valores_distintos"] == ["TXN-00001"]
    assert resultado["exito"] is False


def test_fila_faltante() -> None:
    resultado = reconciliar(ORIGEN, [])
    assert resultado["faltan_en_destino"] == ["TXN-00001"]
    assert resultado["exito"] is False
```

**8.6** Crea `tests/test_pipeline.py`:

**Archivo:** `tests/test_pipeline.py`

```python
import json
import logging
from pathlib import Path

import pytest

from integracion_ventas import cli, pipeline
from integracion_ventas.monitoreo import OcultarSecretos
from tests.conftest import hacer_config


def test_flujo_completo_y_segunda_ejecucion_idempotente(
    carpeta: Path, api_url: str
) -> None:
    config = hacer_config(api_url)
    primera = pipeline.ejecutar(config)
    segunda = pipeline.ejecutar(config)
    assert (primera["recibidos"], primera["validos"], primera["cuarentena"]) == (
        25,
        20,
        5,
    )
    assert (primera["insertadas"], primera["actualizadas"]) == (20, 0)
    assert (segunda["insertadas"], segunda["actualizadas"]) == (0, 20)
    dlq = json.loads(next(Path("cuarentena").glob("*.json")).read_text("utf-8"))
    assert {r["id"] for r in dlq} == {
        "TXN-00005",
        "TXN-9",
        "TXN-00013",
        "TXN-00017",
        "TXN-00021",
    }


def test_codigos_de_salida(
    carpeta: Path, api_url: str, api_url_v2: str, monkeypatch: pytest.MonkeyPatch
) -> None:
    monkeypatch.setenv("API_CLIENT_ID", "lab-integracion")
    monkeypatch.setenv("API_TIMEOUT_LECTURA", "2")
    monkeypatch.setenv("API_BASE_URL", api_url)
    monkeypatch.setenv("API_CLIENT_SECRET", "s3cr3to-lab-2026")
    assert cli.main([]) == 0
    monkeypatch.setenv("API_CLIENT_SECRET", "incorrecto")
    assert cli.main([]) == 3
    monkeypatch.setenv("API_CLIENT_SECRET", "s3cr3to-lab-2026")
    monkeypatch.setenv("SF_MODO", "real")
    assert cli.main([]) == 1
    monkeypatch.setenv("SF_MODO", "emulador")
    monkeypatch.setenv("API_BASE_URL", api_url_v2)
    assert cli.main([]) == 5


def test_los_secretos_no_llegan_al_log() -> None:
    registro = logging.LogRecord(
        "x", logging.INFO, __file__, 1, "token=%s", ("s3cr3to-lab-2026",), None
    )
    OcultarSecretos(["s3cr3to-lab-2026"]).filter(registro)
    assert registro.getMessage() == "token=[REDACTED]"
```

**8.7** Ejecuta:

```bash
python -m pytest -q
ruff check src tests scripts api_simulada.py
python -m mypy src tests scripts api_simulada.py
```

Resultado esperado:

- **pytest:** `20 passed` en macOS y Linux; `19 passed, 1 skipped` en Windows. Tarda unos 12 s, porque una prueba recorre las fallas de la API.
- **ruff:** `All checks passed!`
- **mypy:** `Success: no issues found in 23 source files`

---

## Entregables

Comprime `practica3_integracion` **sin** `.venv/`, `.env`, `staging/`, `cuarentena/` ni `emulador_snowflake/`. Antes de entregar, revisa:

- [ ] Paquete `src/integracion_ventas/` con 12 archivos `.py`
- [ ] `.env.example` versionado; `.env` **no** incluido
- [ ] Primera ejecución: 25 recibidos, 20 cargados, 5 en cuarentena, reconciliación correcta
- [ ] Segunda ejecución: 0 insertadas, 20 actualizadas
- [ ] `consultar_destino.py`: 20 filas, suma 5950.00, 4 filas con el nombre sospechoso guardado como dato
- [ ] Escenarios 7.1 a 7.5 con sus códigos de salida 3, 3, 5, 6 y 1
- [ ] Respuestas a las preguntas de cierre
- [ ] `pytest`, `ruff` y `mypy` sin errores

---

## Opcional: ejecutar contra una cuenta real de Snowflake

No es necesario para la práctica. Si quieres probar el mismo código contra la nube, Snowflake ofrece una **cuenta de prueba de 30 días** sin tarjeta de crédito. Al terminar la prueba, la cuenta se suspende.

> Esta sección no se ejecutó durante la preparación del laboratorio, porque requiere una cuenta. Revísala con tu instructor antes de usarla.

**1. Genera el par de claves** (requiere OpenSSL; en Windows viene con Git for Windows). La clave privada queda cifrada con una frase de paso:

```bash
openssl genrsa 2048 | openssl pkcs8 -topk8 -v2 aes256 -inform PEM -out svc_integracion.p8
openssl rsa -in svc_integracion.p8 -pubout -out svc_integracion.pub
```

Guarda `svc_integracion.p8` **fuera** de la carpeta del proyecto.

**2. En Snowflake**, con un rol administrador, crea los objetos y el usuario de servicio. En `RSA_PUBLIC_KEY` pega el contenido de `svc_integracion.pub` sin las líneas `-----BEGIN/END PUBLIC KEY-----`:

```sql
CREATE ROLE IF NOT EXISTS ROL_ETL;
CREATE WAREHOUSE IF NOT EXISTS WH_ETL_XS WAREHOUSE_SIZE = XSMALL AUTO_SUSPEND = 60 AUTO_RESUME = TRUE;
CREATE DATABASE IF NOT EXISTS ANALITICA;
CREATE SCHEMA IF NOT EXISTS ANALITICA.VENTAS;
GRANT USAGE ON WAREHOUSE WH_ETL_XS TO ROLE ROL_ETL;
GRANT USAGE ON DATABASE ANALITICA TO ROLE ROL_ETL;
GRANT USAGE, CREATE TABLE, CREATE STAGE ON SCHEMA ANALITICA.VENTAS TO ROLE ROL_ETL;
CREATE USER IF NOT EXISTS SVC_INTEGRACION TYPE = SERVICE DEFAULT_ROLE = ROL_ETL
  RSA_PUBLIC_KEY = 'MIIBIjANBgkqh...';
GRANT ROLE ROL_ETL TO USER SVC_INTEGRACION;
```

**3. En `.env`**, cambia `SF_MODO=real` y descomenta las variables `SF_*`:

- **`SF_ACCOUNT`:** identificador de la cuenta en formato `organizacion-cuenta`.
- **`SF_PRIVATE_KEY_PATH`:** ruta del archivo `.p8`.
- **`SF_PRIVATE_KEY_PASSPHRASE`:** la frase de paso de la clave.

**4.** Ejecuta `python -m integracion_ventas` y `python scripts/consultar_destino.py`. En modo real, la carga usa `PUT` + `COPY INTO … PURGE = TRUE`. El script `alterar_destino.py` se niega a ejecutarse contra una cuenta real.

### Qué cambia entre el emulador y Snowflake real

| Aspecto | Emulador (fakesnow) | Snowflake real |
| :--- | :--- | :--- |
| Motor | DuckDB local que traduce el SQL de Snowflake | Servicio en la nube |
| Llenado del staging | `INSERT` parametrizado (`executemany`) | `PUT` a un stage interno + `COPY INTO` |
| Base y esquema | Los crea `conectar()` | Los crea un administrador (paso 2) |
| Autenticación | No aplica | Par de claves del usuario de servicio |
| Versión del conector | Serie 4.x (fakesnow 0.12 no funciona con 5.x) | 4.x o superior |
| Rigor del SQL | Más permisivo: una consulta que pasa aquí puede fallar en la nube | Referencia |

`MERGE`, `INFORMATION_SCHEMA`, el SQL parametrizado, la reconciliación y el monitoreo son **el mismo código** en los dos modos.

---

## Solución de problemas

| Síntoma | Causa probable | Solución |
| :--- | :--- | :--- |
| `ModuleNotFoundError: No module named 'integracion_ventas'` | El paquete no está instalado en el entorno activo | Activa `.venv` y ejecuta `python -m pip install -e .` desde la raíz |
| `No se pudo contactar el servidor de tokens … (ConnectionError)` fuera del escenario 7.2 | La API no está corriendo en la Terminal 1 | Ejecuta `python api_simulada.py` en la Terminal 1 |
| `OSError: [Errno 98] Address already in use` (o `WinError 10048`) al levantar la API | Ya hay otra API simulada abierta en el puerto 8000 | Ciérrala con **Ctrl+C**, o usa `python api_simulada.py --puerto 8010` y `API_BASE_URL=http://127.0.0.1:8010` en `.env` |
| `No module named 'snowflake.connector.converter'` | Se instaló el conector 5.x | Ejecuta `python -m pip install -r requirements.txt` |
| `No matching distribution found for snowflake-connector-python==4.8.0` en macOS | macOS 14 o anterior | Actualiza a macOS 15 o usa un equipo con Windows o Linux |
| Código 6 en una ejecución normal | Quedó roto el destino del ejercicio 7.4 | `python scripts/alterar_destino.py --restaurar` |
| Código 5 en una ejecución normal | La Terminal 1 sigue con `--esquema v2` | Reinicia la API sin `--esquema v2` |
| `Could not set lock on file …ANALITICA.db` | Otro proceso tiene abierto el emulador (otra ejecución, o una extensión de VS Code que abrió el `.db`) | Espera a que termine la otra ejecución o cierra el archivo; el emulador admite un proceso a la vez |
| `Configuración inválida: api_client_secret: Field required` | No existe `.env` o no estás en la raíz del proyecto | Revisa el paso 2.2 y ejecuta desde `practica3_integracion` |
| PowerShell no deja activar `.venv` | Política de ejecución de Windows | `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` |
| Acentos raros al redirigir la salida a un archivo en Windows | La consola usa otra codificación | Antes de ejecutar: `$env:PYTHONUTF8 = "1"` |

---

## Limpieza

Detén la API en la Terminal 1 con **Ctrl+C**. Después, en la Terminal 2:

**PowerShell**
```powershell
Remove-Item -Recurse -Force staging, cuarentena, emulador_snowflake -ErrorAction SilentlyContinue
deactivate
```

**bash**
```bash
rm -rf staging cuarentena emulador_snowflake
deactivate
```

---

## Resumen

| Lo que hiciste | Lección |
| :--- | :--- |
| Obtuviste un token Client Credentials y lo reutilizaste en todas las páginas | 3.2 Autenticación |
| Recorriste la API con un generador por cursor con doble condición de salida | 3.3 Paginación |
| Superaste un timeout, un 429 y un 503 sin intervención y sin reintentar el 401 | 3.4 a 3.7 |
| Reintentaste solo operaciones idempotentes y cargaste con `MERGE` por clave | 3.8 Idempotencia |
| Validaste el `Content-Type`, el JSON y la estructura de cada página | 3.9 Validación de respuestas |
| Aislaste 5 registros con todos sus motivos sin detener el lote | 3.10 Respuestas inválidas |
| Definiste el contrato con alias, `Decimal`, zona horaria y validadores de negocio | 3.11 Pydantic |
| Abriste y cerraste la conexión con un context manager | 3.12 Conexiones |
| Usaste `executemany`, `fetchone` y `fetchall` del conector DB-API | 3.13 Consultas |
| Aplicaste la carga en una sola sentencia `MERGE` atómica | 3.14 Transacciones |
| Preparaste la carga masiva con `PUT` + `COPY INTO` por carpeta de ejecución | 3.15 Carga masiva, 3.20 Snowflake |
| Leíste el catálogo `INFORMATION_SCHEMA` para validar el destino | 3.17, 3.27 Metadatos |
| Guardaste `'); DROP TABLE` como dato gracias a los parámetros y la lista blanca | 3.18 SQL seguro |
| Mantuviste los secretos fuera del código y de los logs | 3.19 Credenciales |
| Integraste API, archivo de staging y warehouse | 3.21 a 3.23 |
| Detectaste el cambio de esquema del origen (v2) y del destino (columna eliminada) | 3.24 Comparación de esquemas |
| Reconciliaste con conteo, suma de control y SHA-256 por fila | 3.25 Reconciliación |
| Emitiste eventos JSON con `trace_id`, estado, duración y métricas por fase | 3.26 Monitoreo |

