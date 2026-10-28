# Práctica 4. Monitoreo de pipelines y remediación controlada de pipelines

| Dato | Detalle |
| :--- | :--- |
| **Curso** | Automatización de procesos con Python |
| **Capítulo** | 4. Ingeniería de confiabilidad, observabilidad y remediación |
| **Duración** | 90 minutos |
| **Modalidad** | Individual |
| **Nivel de Bloom** | Aplicar / Analizar |

---

## Objetivo

Implementar el monitoreo de un pipeline de datos como se haría en **Amazon MWAA con CloudWatch**:

- **Detectar fallos** con las señales de código de salida, tiempo máximo, volumen y latido.
- **Recolectar información diagnóstica** sin exponer datos sensibles.
- **Decidir si cada fallo es recuperable**: transitorio, idempotente y con intentos disponibles.
- **Aplicar reintentos, alertas o remediación controlada** según el caso.
- **Registrar cada decisión operativa** en una auditoría verificable.

**No necesitas cuenta de AWS.** La tarea se ejecuta en un proceso aparte, como en MWAA, y un vigilante local aplica los mismos controles que Airflow: reintentos de la tarea, `execution_timeout` y callback de fallo. Las llamadas a CloudWatch (`PutMetricData`) y a la API de MWAA (`InvokeRestApi`, para pausar un DAG) usan `boto3` real y se validan con **botocore Stubber** contra el modelo oficial de cada API, igual que en las demos del capítulo. El DAG de referencia para MWAA queda en el proyecto y se comprueba que compila.

## Lo que vas a construir

Un monitor llamado **monitoreo-pipeline** que cierra el ciclo detectar → diagnosticar → decidir → actuar → registrar (tema 4.30):

```text
 python -m monitoreo_pipeline ejecutar --escenario <escenario>
        │
        ▼
 Vigilante (papel de MWAA) ── ejecuta la tarea en un subproceso con timeout
        │                       └─ log JSON de la tarea ──► salida/logs/  (papel de CloudWatch Logs)
        │
        │   Tarea: API simulada ─► reintentos (tenacity, Full Jitter, Retry-After, 401)
        │                       ─► validación ─► inválidos ──► salida/dlq_ventas.jsonl
        │                       ─► umbral de fallos parciales ─► carga UPSERT (dw.db)
        │                       └─ si falla: @recolectar_diagnostico (sanitizado)
        ▼
 Señal: SUCCESS | HTTP 404 | HTTP 503 | TIMEOUT | SUSPECT (0 filas) | DATOS_INVALIDOS
        │
        ▼
 decidir() ── transitorio + idempotente + intentos ──► REINTENTAR | DETENER | ESCALAR
        │
        ├─ config/politica.toml: señal → acción del catálogo
        │     limpiar_temporales (bajo)  ─► dry-run ─► ejecutar ─► verificar
        │     pausar_dag_consumidor (bajo) ─► MWAA invoke_rest_api (Stubber)
        │     reprocesar_lote (alto) ─► solicitud de aprobación con token y caducidad
        │
        ├─ alerta enriquecida, deduplicada y con límite de tasa ──► salida/alertas.jsonl
        ├─ métricas ──► CloudWatch PutMetricData (Stubber)
        └─ cada decisión ──► salida/auditoria.jsonl  (HMAC encadenado)
```

### Relación con el capítulo

| Componente del laboratorio | Lección del capítulo |
| :--- | :--- |
| `AutomationError` → `InfrastructureError` / `DataQualityError`, `to_dict()`, `raise … from` | 4.1 Diseño de excepciones |
| `clasificar()` con `match` y el `Enum` `Ruta` | 4.2 Errores recuperables y no recuperables |
| `construir_politica()`: `retry_if_exception`, dos límites con `\|`, `Retry-After` o Full Jitter, `reraise=True` | 4.3 Políticas de reintento, 4.25 Reintento seguro |
| `enviar_a_dlq()`: payload intacto en JSONL | 4.4 Mensajes no procesados (DLQ) |
| `ResultadoLote` y umbral de tolerancia | 4.5 Fallos parciales |
| Alertas al canal (archivo que simula el webhook) | 4.7 Notificaciones |
| Carga con `ON CONFLICT … DO UPDATE` | 4.9 Idempotencia |
| `JsonFormatter` | 4.10 Registro estructurado |
| `publicar_metricas()` con `PutMetricData` | 4.11 Métricas |
| `AlertManager`: deduplicación y límite por minuto | 4.14 Generación de alertas |
| `Almacen`: ejecuciones, alertas, remediaciones y aprobaciones en SQLite | 4.15 Estado de ejecución |
| `Auditoria`: canal propio, solo se añade | 4.16 Registros de auditoría |
| Comando `panel` | 4.17 Paneles operativos |
| `FiltroSensible` en el handler | 4.18 Registro seguro |
| `recolectar_diagnostico`: marco más interno, `dict(f_locals)`, lista blanca de variables | 4.19, 4.23 Información diagnóstica |
| `FiltroContexto`: `execution_id` en todos los registros | 4.21 Estado, métricas, logs y alertas |
| `ejecutar_vigilado()`: código de salida, timeout, 0 filas y latido; `dags/ventas_diarias.py` | 4.22 Detección de trabajos fallidos |
| `decidir()`: transitorio, idempotente e intentos | 4.24 Operaciones recuperables |
| `enriquecer()`: impacto, ubicación, versión, runbook y enlace a los logs | 4.26 Enriquecimiento de alertas |
| `MotorRemediacion`: catálogo, riesgo, límite por hora, precondición, dry-run y verificación | 4.8, 4.27 Remediación controlada |
| `PuertaAprobacion`: token de un solo uso, caducidad, sin autoaprobación | 4.28 Aprobación humana |
| HMAC encadenado y `verificar()` | 4.29 Auditabilidad |
| `Controlador.ejecutar()` | 4.30 Observabilidad, orquestación y remediación |
| `pausar_dag()` con `InvokeRestApi` de MWAA | 4.31 MWAA y Acceldata |
| `config/politica.toml`: señal → acción | 4.32 Señales operativas |

## Criterios de evaluación

| Criterio | Evidencia en tu entrega |
| :--- | :--- |
| **Corrección** | Cada uno de los 7 escenarios termina con la señal, la decisión y el código de salida esperados |
| **Resiliencia** | El 503, el 429 y el 401 se superan solos; el transitorio persistente y la tarea colgada se reintentan con límite y escalan; los registros inválidos van a la DLQ sin detener el lote |
| **Mantenibilidad** | Intentos, esperas, timeout, umbral, deduplicación, límite de remediaciones y mapa señal → acción están solo en `config/politica.toml` |
| **Seguridad** | Ni el token, ni la clave de auditoría, ni los correos aparecen en logs o alertas; reprocesar requiere aprobación de otra persona; la auditoría detecta una edición manual |
| **Valor operativo** | Las alertas traen impacto, ubicación, runbook y enlace a los logs; no se repiten; cada decisión queda en la auditoría con su `execution_id` |

---

## Requisitos

| Elemento | Requisito |
| :--- | :--- |
| Python | **3.11 o superior** (probado con 3.11, 3.13 y 3.14) |
| Sistema operativo | Windows 10/11, macOS o Linux |
| Editor | Visual Studio Code con la extensión de Python |
| Terminal | Terminal integrada de VS Code (PowerShell en Windows; bash/zsh en macOS o Linux) |
| Red | Acceso a PyPI para instalar paquetes. La práctica no se conecta a AWS |
| Cuenta de AWS | **No se necesita** |

> **Convenciones**
> - Todos los comandos se ejecutan desde la **carpeta raíz del proyecto** (`practica4_monitoreo`) con el entorno virtual activo.
> - Cuando un comando cambia según el sistema operativo, verás la versión **PowerShell** y la versión **bash**.
> - Se mantienen los estándares de las prácticas anteriores: src-layout, `pyproject.toml`, versiones fijadas, configuración validada con Pydantic y códigos de salida.
> - En las salidas de ejemplo, los identificadores (`R-…`, tokens, firmas de alerta) y los tiempos cambian en cada ejecución.

### Distribución del tiempo

| Parte | Contenido | Minutos |
| :--- | :--- | :---: |
| 0. Preparación | Estructura, entorno y dependencias | 8 |
| 1. Política y configuración | `politica.toml`, `.env` y validación | 7 |
| 2. Errores con significado | Jerarquía, clasificación y decisión | 12 |
| 3. Registro seguro | Logs JSON, `execution_id` y sanitización | 8 |
| 4. La tarea del pipeline | Fuente simulada, reintentos, DLQ, UPSERT y diagnóstico | 15 |
| 5. Vigilar, alertar, remediar y auditar | Vigilante, alertas, AWS, remediación, aprobación y auditoría | 20 |
| 6. El ciclo completo | Controlador, CLI y siete escenarios | 15 |
| 7. Pruebas | pytest, ruff y mypy | 5 |

> **Checkpoints.** Al final de las partes 1 a 5 hay un punto de control. Si te atrasas, el instructor puede darte la carpeta del checkpoint.

---

## Parte 0. Preparación (8 min)

**0.1** Crea la carpeta `practica4_monitoreo`, ábrela en VS Code (**File → Open Folder**) y abre una terminal.

**0.2** Crea la estructura de carpetas:

**PowerShell**
```powershell
New-Item -ItemType Directory -Force -Path src/monitoreo_pipeline, tests, config, dags, scripts
```

**bash**
```bash
mkdir -p src/monitoreo_pipeline tests config dags scripts
```

**0.3** Crea los archivos base del proyecto:

**Archivo:** `pyproject.toml`

```toml
[build-system]
requires = ["setuptools>=69"]
build-backend = "setuptools.build_meta"

[project]
name = "monitoreo-pipeline"
version = "0.1.0"
description = "Monitoreo y remediación controlada de un pipeline (MWAA y CloudWatch simulados)"
requires-python = ">=3.11"
dependencies = [
    "boto3>=1.35",               # clientes de CloudWatch y MWAA; se validan con Stubber
    "pydantic>=2.7,<3",
    "pydantic-settings>=2.3,<3",
    "tenacity>=9,<10",
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
exclude = ["dags/"]

[tool.pytest.ini_options]
testpaths = ["tests"]
```

**0.4** Crea los archivos de versiones. Se entregan ya resueltos para que todos instalen exactamente lo que se probó; la primera línea indica el comando con el que se generaron:

**Archivo:** `requirements.txt`

```text
# Versiones probadas para la Práctica 4 (Windows, macOS y Linux; Python 3.11-3.14).
# Generado con:
#   pip-compile --strip-extras --no-annotate pyproject.toml -o requirements.txt
annotated-types==0.8.0
boto3==1.43.119
botocore==1.43.119
jmespath==1.1.0
pydantic==2.14.1
pydantic-core==2.50.1
pydantic-settings==2.16.1
python-dateutil==2.9.0.post0
python-dotenv==1.2.4
s3transfer==0.19.3
six==1.17.0
tenacity==9.2.1
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
```

**Archivo:** `requirements-dev.txt`

```text
# Versiones probadas para la Práctica 4 (Windows, macOS y Linux; Python 3.11-3.14).
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
ruff==0.17.3
typing-extensions==4.16.0
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

# Resultados de cada ejecución (logs, DLQ, alertas, auditoría, bases locales)
salida/
```

**Archivo:** `src/monitoreo_pipeline/__init__.py`

```python
"""Monitoreo de pipelines y remediación controlada (Práctica 4)."""

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
python -c "import monitoreo_pipeline, boto3, tenacity, pydantic; print('ok', monitoreo_pipeline.__version__)"
```

Resultado esperado: `ok 0.1.0`

---

## Parte 1. Política y configuración (7 min)

### 1.1 La política operativa en un único lugar

Todo lo que decide **cuánto** reintentar, **cuándo** cortar una tarea, **qué** acción corresponde a cada señal y **cuánto** ruido de alertas se tolera vive en un archivo. Cambiar un umbral no exige tocar el código:

**Archivo:** `config/politica.toml`

```toml
# Política operativa del pipeline: TODO lo que se puede ajustar está aquí.
# Cambiar un umbral o una acción no requiere tocar el código.

[reintento]            # capa 1: cada llamada a la API (tenacity)
max_intentos = 4
max_segundos = 20
multiplicador = 0.5    # Full Jitter: espera al azar entre 0 y multiplicador * 2^n
espera_maxima = 8

[tarea]                # capa 2: la tarea completa, como "retries" de Airflow
reintentos = 1
idempotente = true     # la carga usa UPSERT: repetirla deja el mismo estado
timeout_s = 4          # como execution_timeout: corta las tareas colgadas
umbral_fallos = 0.2    # más del 20 % de registros inválidos es un fallo sistémico

[alertas]
ventana_dedup_s = 300
max_por_minuto = 5

[remediacion]
max_por_hora = 3
ttl_aprobacion_s = 1800

# Señal detectada -> acción del catálogo de remediación
[acciones]
TIMEOUT = "limpiar_temporales"
SUSPECT = "pausar_dag_consumidor"
DATOS_INVALIDOS = "reprocesar_lote"

# Señal detectada -> runbook que se enlaza en la alerta
[runbooks]
TIMEOUT = "rb-001-tareas-colgadas"
"HTTP 404" = "rb-002-recurso-inexistente"
"HTTP 503" = "rb-003-api-no-disponible"
SUSPECT = "rb-004-ejecucion-sin-datos"
DATOS_INVALIDOS = "rb-005-calidad-de-datos"
```

Observa las dos capas de reintento del tema 4.25:

- **Llamada:** hasta 4 intentos con tenacity.
- **Tarea completa:** 1 reintento, como `retries` de Airflow.

El peor caso son 4 × 2 = **8 llamadas** a la API por ejecución. Así se mantiene acotado el presupuesto de reintentos.

### 1.2 Secretos en `.env`

**1.2.1** Crea la plantilla. Esta **sí** se versiona:

**Archivo:** `.env.example`

```bash
# Copia este archivo como .env. .env.example SÍ se versiona; .env NUNCA.
# Un .env es texto plano: en producción estos valores vienen de un gestor de secretos.

ENTORNO=desarrollo
APP_VERSION=1.0.0
PIPELINE=ventas_diarias
DAG_CONSUMIDOR=carga_ventas_snowflake
MWAA_ENTORNO=mwaa-lab

# Token con el que la tarea consulta la API de ventas (simulada)
API_TOKEN=reemplaza-con-el-token
# Clave de la firma HMAC de la auditoría (mínimo 16 caracteres)
AUDIT_HMAC_KEY=reemplaza-con-una-clave-larga
```

**1.2.2** Cópiala como `.env`:

**PowerShell**
```powershell
Copy-Item .env.example .env
```

**bash**
```bash
cp .env.example .env
```

**1.2.3** Abre `.env` en VS Code y reemplaza los dos valores secretos:

```bash
API_TOKEN=tok-lab-7f3a9c21
AUDIT_HMAC_KEY=clave-auditoria-lab-2026
```

> `.env` está en `.gitignore` y es texto plano. En MWAA, estos valores vendrían de AWS Secrets Manager o de las variables del entorno de Airflow.

### 1.3 Configuración validada

`Ajustes` lee `.env` con `pydantic-settings` y `Politica` valida `politica.toml` con modelos de Pydantic. Si falta un secreto o un valor está fuera de rango, el proceso se detiene **antes** de hacer nada (fail-fast). El mensaje de error nunca incluye los valores:

**Archivo:** `src/monitoreo_pipeline/configuracion.py`

```python
"""Ajustes (.env) y política (config/politica.toml), validados al inicio."""

import tomllib
from pathlib import Path

from pydantic import BaseModel, Field, SecretStr, ValidationError
from pydantic_settings import BaseSettings, SettingsConfigDict

from monitoreo_pipeline.errores import ConfiguracionError


class Ajustes(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    entorno: str = "desarrollo"
    app_version: str = "sin-version"
    pipeline: str = "ventas_diarias"
    dag_consumidor: str = Field(default="carga_ventas_snowflake", pattern=r"^[\w.-]+$")
    mwaa_entorno: str = "mwaa-lab"
    api_token: SecretStr
    audit_hmac_key: SecretStr = Field(min_length=16)
    wiki_url: str = "https://wiki.empresa.test/runbooks"
    logs_url: str = "https://logs.empresa.test/search"
    politica_path: Path = Path("config/politica.toml")
    salida: Path = Path("salida")


class PoliticaReintento(BaseModel):
    max_intentos: int = Field(ge=1, le=10)
    max_segundos: float = Field(gt=0)
    multiplicador: float = Field(ge=0)
    espera_maxima: float = Field(ge=0)


class PoliticaTarea(BaseModel):
    reintentos: int = Field(ge=0, le=3)
    idempotente: bool
    timeout_s: float = Field(gt=0)
    umbral_fallos: float = Field(ge=0, le=1)


class PoliticaAlertas(BaseModel):
    ventana_dedup_s: int = Field(ge=0)
    max_por_minuto: int = Field(ge=1)


class PoliticaRemediacion(BaseModel):
    max_por_hora: int = Field(ge=1)
    ttl_aprobacion_s: int = Field(ge=60)


class Politica(BaseModel):
    reintento: PoliticaReintento
    tarea: PoliticaTarea
    alertas: PoliticaAlertas
    remediacion: PoliticaRemediacion
    acciones: dict[str, str]
    runbooks: dict[str, str]


def cargar() -> tuple[Ajustes, Politica]:
    """Fail-fast: si algo falta o es inválido, nada se ejecuta."""
    try:
        ajustes = Ajustes()  # type: ignore[call-arg]
        with ajustes.politica_path.open("rb") as archivo:
            politica = Politica.model_validate(tomllib.load(archivo))
    except (ValidationError, OSError, tomllib.TOMLDecodeError) as error:
        motivos = (
            "; ".join(
                f"{'.'.join(map(str, e['loc']))}: {e['msg']}"
                for e in error.errors(include_input=False)
            )
            if isinstance(error, ValidationError)
            else str(error)
        )
        raise ConfiguracionError(f"Configuración inválida: {motivos}") from None
    return ajustes, politica
```

Este módulo usa `ConfiguracionError`, que crearás en la Parte 2. Por eso el checkpoint se ejecuta al terminar el paso 2.1.

> **Checkpoint 1** (después del paso 2.1).
>
> ```bash
> python -c "from monitoreo_pipeline.configuracion import cargar; a, p = cargar(); print(a.api_token, a.audit_hmac_key, p.tarea.timeout_s, p.acciones)"
> ```
>
> Resultado esperado:
>
> ```text
> ********** ********** 4.0 {'TIMEOUT': 'limpiar_temporales', 'SUSPECT': 'pausar_dag_consumidor', 'DATOS_INVALIDOS': 'reprocesar_lote'}
> ```

---

## Parte 2. Errores con significado (12 min)

### 2.1 Jerarquía de excepciones

Hay una raíz común, `AutomationError`, y dos categorías que ya dicen la acción:

- **`InfrastructureError`:** suele ser transitorio; se reintenta.
- **`DataQualityError`:** reintentar no lo arregla; se aísla.

Cada excepción sabe con qué **código de salida** debe terminar la tarea, que es lo que ve el orquestador:

**Archivo:** `src/monitoreo_pipeline/errores.py`

```python
"""Jerarquía de excepciones del proyecto (tema 4.1)."""

from typing import Any


class AutomationError(Exception):
    """Raíz de las excepciones del proyecto: hereda de Exception."""

    codigo_salida = 1

    def __init__(self, mensaje: str, *, contexto: dict[str, Any] | None = None) -> None:
        super().__init__(mensaje)
        self.contexto = contexto or {}

    def to_dict(self) -> dict[str, Any]:
        causa = self.__cause__
        return {
            "tipo": type(self).__name__,
            "mensaje": str(self),
            "contexto": self.contexto,
            "causa": repr(causa) if causa else None,
        }


class ConfiguracionError(AutomationError):
    """Falta una variable de entorno o la política tiene un valor inválido."""


class InfrastructureError(AutomationError):
    """Red, base de datos, disco: suele ser transitorio."""

    codigo_salida = 3


class DataQualityError(AutomationError):
    """Dato inválido: reintentar no lo arregla."""

    codigo_salida = 5


class HTTPStatusError(InfrastructureError):
    """Respuesta HTTP con error; guarda el código y Retry-After para decidir."""

    def __init__(self, status: int, retry_after: float | None = None) -> None:
        super().__init__(f"HTTP {status}", contexto={"status": status})
        self.status = status
        self.retry_after = retry_after


class LoteRechazadoError(DataQualityError):
    """La tasa de registros inválidos supera el umbral: el fallo es sistémico."""
```

### 2.2 De la excepción a la ruta, y del fallo a la decisión

Este módulo tiene dos niveles:

- **`clasificar()`** decide dentro de la tarea qué hacer con **un error** (tema 4.2).
- **`decidir()`** decide en el controlador qué hacer con **una tarea fallida** (tema 4.24). Solo se reintenta si el fallo es transitorio, la tarea es idempotente y quedan intentos. Si falta cualquiera de las tres condiciones, se detiene o se escala.

**Archivo:** `src/monitoreo_pipeline/clasificacion.py`

```python
"""Del error a la ruta (tema 4.2) y del fallo a la decisión (tema 4.24)."""

from dataclasses import dataclass
from enum import Enum

from monitoreo_pipeline.errores import DataQualityError, HTTPStatusError


class Ruta(Enum):
    REINTENTAR = "reintentar con espera"
    CORREGIR_Y_REINTENTAR = "acción correctiva y un reintento"
    CUARENTENA = "aislar el registro en la DLQ"
    DETENER = "detener y alertar"


CODIGOS_TRANSITORIOS = {408, 429, 500, 502, 503, 504}


def clasificar(error: BaseException) -> Ruta:
    """Ruta de UN error dentro de la tarea; el orden de los casos importa."""
    match error:
        case HTTPStatusError(status=401):
            return Ruta.CORREGIR_Y_REINTENTAR  # token caducado: renovarlo
        case HTTPStatusError(status=s) if s in CODIGOS_TRANSITORIOS:
            return Ruta.REINTENTAR
        case HTTPStatusError():
            return Ruta.DETENER  # 400, 403, 404, 422: no cambia al repetir
        case TimeoutError() | ConnectionError():
            return Ruta.REINTENTAR
        case ValueError() | KeyError() | DataQualityError():
            return Ruta.CUARENTENA  # el dato es el problema, no el entorno
        case _:
            return Ruta.DETENER  # lo desconocido no se reintenta a ciegas


class Decision(Enum):
    REINTENTAR = "reintentar la tarea"
    DETENER = "detener y alertar"
    ESCALAR = "escalar a revisión humana"


# Señales que el vigilante extrae de la ejecución o de su diagnóstico
TRANSITORIAS = {f"HTTP {c}" for c in CODIGOS_TRANSITORIOS} | {
    "TIMEOUT",
    "ConnectionError",
}
PERMANENTES = {"HTTP 400", "HTTP 403", "HTTP 404", "HTTP 422", "DATOS_INVALIDOS"}


@dataclass(frozen=True)
class Fallo:
    senal: str  # p. ej. "HTTP 503", "TIMEOUT", "SUSPECT"
    idempotente: bool  # ¿repetir la tarea deja el mismo estado?
    intentos: int
    max_intentos: int


def decidir(fallo: Fallo) -> tuple[Decision, str]:
    """Recuperable = transitorio Y idempotente Y con intentos disponibles."""
    if fallo.senal in PERMANENTES:
        return Decision.DETENER, "error permanente, requiere corrección"
    if fallo.senal == "SUSPECT":
        return Decision.ESCALAR, "terminó sin datos (éxito vacío), revisión humana"
    if fallo.senal not in TRANSITORIAS:
        return Decision.ESCALAR, "señal desconocida, revisión humana"
    if not fallo.idempotente:
        return Decision.ESCALAR, "transitorio pero NO idempotente (riesgo de duplicar)"
    if fallo.intentos >= fallo.max_intentos:
        return Decision.ESCALAR, "transitorio persistente, intentos agotados"
    return Decision.REINTENTAR, f"intento {fallo.intentos + 1} de {fallo.max_intentos}"
```

> **Checkpoint 2.**
>
> ```bash
> python -c "from monitoreo_pipeline.clasificacion import clasificar, decidir, Fallo; from monitoreo_pipeline.errores import HTTPStatusError; [print(f'{str(e):<16} -> {clasificar(e).value}') for e in (HTTPStatusError(503), HTTPStatusError(401), HTTPStatusError(404), TimeoutError('lectura'), ValueError('N/A'), TypeError('None'))]; [print(f'{f.senal:<16} idem={f.idempotente!s:<5} intento={f.intentos} -> {decidir(f)[0].name}: {decidir(f)[1]}') for f in (Fallo('HTTP 503', True, 1, 2), Fallo('HTTP 503', True, 2, 2), Fallo('TIMEOUT', False, 1, 2), Fallo('HTTP 404', True, 1, 2), Fallo('SUSPECT', True, 1, 2))]"
> ```
>
> Resultado esperado:
>
> ```text
> HTTP 503         -> reintentar con espera
> HTTP 401         -> acción correctiva y un reintento
> HTTP 404         -> detener y alertar
> lectura          -> reintentar con espera
> N/A              -> aislar el registro en la DLQ
> None             -> detener y alertar
> HTTP 503         idem=True  intento=1 -> REINTENTAR: intento 2 de 2
> HTTP 503         idem=True  intento=2 -> ESCALAR: transitorio persistente, intentos agotados
> TIMEOUT          idem=False intento=1 -> ESCALAR: transitorio pero NO idempotente (riesgo de duplicar)
> HTTP 404         idem=True  intento=1 -> DETENER: error permanente, requiere corrección
> SUSPECT          idem=True  intento=1 -> ESCALAR: terminó sin datos (éxito vacío), revisión humana
> ```
>
> Fíjate en el tercer caso de `decidir()`: un timeout es transitorio, pero si repetir la tarea no es seguro, no se reintenta.

---

## Parte 3. Registro seguro (8 min)

Este módulo reúne tres temas del capítulo:

- **`JsonFormatter` (4.10):** una línea JSON por evento.
- **`FiltroContexto` (4.21):** añade el `execution_id` a **todos** los registros sin tocar cada llamada a `log`.
- **`FiltroSensible` (4.18):** redacta secretos, correos, tokens y tarjetas. Va en el **handler**, para cubrir también los registros de bibliotecas como tenacity. También limpia los campos que llegan por `extra=`.

**Archivo:** `src/monitoreo_pipeline/registro.py`

```python
"""Logs JSON (4.10), sanitización en el handler (4.18) y execution_id común (4.21)."""

import json
import logging
import re
import sys
from contextvars import ContextVar
from datetime import UTC, datetime
from typing import Any, TextIO

# Contexto de la ejecución: se inyecta en TODOS los registros
contexto_ejecucion: ContextVar[dict[str, Any] | None] = ContextVar(
    "contexto_ejecucion", default=None
)

# Atributos propios de LogRecord: todo lo demás llegó por extra=
ESTANDAR = set(vars(logging.makeLogRecord({}))) | {"message", "asctime"}

CLAVES_SENSIBLES = re.compile(
    r"pass|secret|token|key|auth|jwt|cred|clave|contrase", re.IGNORECASE
)
PATRONES = [
    (re.compile(r"[\w.+-]+@[\w-]+\.[\w.-]+"), "[EMAIL]"),
    (re.compile(r"Bearer\s+[\w.~+/=-]+"), "Bearer [TOKEN]"),
    (re.compile(r"\b(?:\d[ -]?){12}(\d{4})\b"), r"****-****-****-\1"),
]


def limpiar(valor: Any) -> Any:
    """Redacta claves sensibles y patrones conocidos, también si están anidados."""
    if isinstance(valor, dict):
        return {
            k: "[REDACTADO]" if CLAVES_SENSIBLES.search(str(k)) else limpiar(v)
            for k, v in valor.items()
        }
    if isinstance(valor, (list, tuple)):
        return type(valor)(limpiar(v) for v in valor)
    if isinstance(valor, str):
        for patron, reemplazo in PATRONES:
            valor = patron.sub(reemplazo, valor)
    return valor


class FiltroSensible(logging.Filter):
    """En el HANDLER: cubre a todos los loggers, incluidos los de bibliotecas."""

    def filter(self, record: logging.LogRecord) -> bool:
        record.msg = limpiar(record.getMessage())
        record.args = None
        for clave in set(vars(record)) - ESTANDAR:  # campos que llegaron por extra=
            if CLAVES_SENSIBLES.search(clave):
                setattr(record, clave, "[REDACTADO]")
            else:
                setattr(record, clave, limpiar(getattr(record, clave)))
        return True  # nunca descarta: solo limpia


class FiltroContexto(logging.Filter):
    """Añade execution_id e intento a cada registro sin tocar las llamadas a log."""

    def filter(self, record: logging.LogRecord) -> bool:
        contexto = contexto_ejecucion.get() or {}
        record.execution_id = contexto.get("execution_id")
        record.intento = contexto.get("intento")
        return True


class JsonFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        evento = {
            "ts": datetime.fromtimestamp(record.created, UTC).isoformat(
                timespec="milliseconds"
            ),
            "nivel": record.levelname,
            "logger": record.name,
            "mensaje": record.getMessage(),
        }
        evento |= {k: v for k, v in vars(record).items() if k not in ESTANDAR}
        if record.exc_info and record.exc_info[0]:
            evento["error_tipo"] = record.exc_info[0].__name__
        return json.dumps(evento, ensure_ascii=False, default=str)


def configurar_logging(destino: TextIO = sys.stdout) -> None:
    manejador = logging.StreamHandler(destino)
    manejador.setFormatter(JsonFormatter())
    manejador.addFilter(FiltroContexto())  # primero el contexto...
    manejador.addFilter(FiltroSensible())  # ...y al final la limpieza
    logging.basicConfig(level=logging.INFO, handlers=[manejador], force=True)
```

> **Checkpoint 3.**
>
> ```bash
> python -c "import logging; from monitoreo_pipeline.registro import configurar_logging; configurar_logging(); log = logging.getLogger('prueba'); log.info('Usuario %s inició la carga', 'ana.perez@empresa.com'); log.info('Config', extra={'api_key': 'sk-123', 'db': {'password': 'x', 'host': 'dw.local'}})"
> ```
>
> Resultado esperado (las fechas cambian):
>
> ```text
> {"ts": "…", "nivel": "INFO", "logger": "prueba", "mensaje": "Usuario [EMAIL] inició la carga", "execution_id": null, "intento": null}
> {"ts": "…", "nivel": "INFO", "logger": "prueba", "mensaje": "Config", "api_key": "[REDACTADO]", "db": {"password": "[REDACTADO]", "host": "dw.local"}, "execution_id": null, "intento": null}
> ```

---

## Parte 4. La tarea del pipeline (15 min)

### 4.1 La fuente simulada

`fuente.py` representa la API de ventas del proveedor. **No es parte de tu automatización:** cópiala tal cual. Cada escenario define las fallas de las primeras llamadas y después responde con 20 ventas:

| Escenario | Qué hace la API |
| :--- | :--- |
| `normal` | 503, después 429 con `Retry-After: 1`, después responde (2 ventas inválidas) |
| `token_caducado` | El token inicial está vencido: 401 |
| `permanente` | Siempre 404 |
| `transitorio_persistente` | Siempre 503 |
| `colgado` | No responde nunca |
| `vacio` | Responde 200 OK, pero sin datos |
| `datos_corruptos` | Responde, pero la mitad de las ventas son inválidas |

**Archivo:** `src/monitoreo_pipeline/fuente.py`

```python
"""API de ventas SIMULADA: representa el sistema externo, no la automatización.

Cada escenario define las fallas de las primeras llamadas; después responde.
Todo es determinista para que el laboratorio dé siempre el mismo resultado.
"""

import secrets
import time
from typing import Any

from monitoreo_pipeline.errores import HTTPStatusError

COLGAR = "colgar"

ESCENARIOS: dict[str, list[Any]] = {
    "normal": [HTTPStatusError(503), HTTPStatusError(429, retry_after=1)],
    "token_caducado": [],  # el token inicial está vencido: 401
    "permanente": [HTTPStatusError(404)] * 99,
    "transitorio_persistente": [HTTPStatusError(503)] * 99,
    "colgado": [COLGAR],
    "vacio": [],
    "datos_corruptos": [],
}


def generar_ventas(escenario: str) -> list[dict[str, Any]]:
    if escenario == "vacio":
        return []  # responde 200 OK... sin datos
    ventas: list[dict[str, Any]] = [
        {
            "id": f"V-{n:04d}",
            "importe": f"{100 + n * 7.5:.2f}",
            "fecha": f"2026-10-{1 + n % 25:02d}",
            "cliente_email": f"cliente{n}@empresa.test",
        }
        for n in range(1, 21)
    ]
    malos = range(1, 21, 2) if escenario == "datos_corruptos" else (4, 12)
    for n in malos:
        ventas[n - 1]["importe"] = "N/A" if n % 3 else "-15.00"
    return ventas


class APISimulada:
    def __init__(self, escenario: str, token_inicial: str) -> None:
        if escenario not in ESCENARIOS:
            raise ValueError(f"Escenario desconocido: {escenario}")
        self.escenario = escenario
        self.pendientes = list(ESCENARIOS[escenario])
        caducado = escenario == "token_caducado"
        self._token_valido = secrets.token_hex(8) if caducado else token_inicial
        self.llamadas = 0

    def renovar_token(self) -> str:
        self._token_valido = secrets.token_hex(8)
        return self._token_valido

    def obtener_ventas(self, token: str) -> list[dict[str, Any]]:
        self.llamadas += 1
        if self.pendientes:
            falla = self.pendientes.pop(0)
            if falla == COLGAR:
                time.sleep(10)  # mucho más que el timeout de la tarea
                raise TimeoutError("la API no respondió")
            raise falla
        if token != self._token_valido:
            raise HTTPStatusError(401)
        return generar_ventas(self.escenario)
```

### 4.2 Diagnóstico automático

El decorador `@recolectar_diagnostico` hace esto cuando la función falla:

1. Recorre la traza hasta el **marco más interno**, donde ocurrió el error.
2. Copia sus variables locales con `dict()`; desde Python 3.13, `f_locals` es un proxy.
3. Sanitiza los datos y añade el sistema y una **lista blanca** de variables de entorno.
4. Registra todo como JSON y **relanza** el error para que el orquestador decida.

**Archivo:** `src/monitoreo_pipeline/diagnostico.py`

```python
"""Recolección automática de información diagnóstica (temas 4.19 y 4.23)."""

import functools
import logging
import os
import platform
import re
import shutil
import traceback
from collections.abc import Callable
from datetime import UTC, datetime
from typing import Any, ParamSpec, TypeVar

from monitoreo_pipeline.errores import AutomationError

logger = logging.getLogger("monitoreo.diagnostico")

SECRETO = re.compile(r"pass|secret|token|key|auth|jwt|cred|clave|contrase", re.I)
# Nunca se vuelca os.environ completo: solo esta lista blanca
ENV_PERMITIDAS = ("ENTORNO", "APP_VERSION", "PIPELINE", "AWS_REGION")

P = ParamSpec("P")
R = TypeVar("R")


def sanitizar(datos: Any) -> Any:
    if isinstance(datos, dict):
        return {
            k: "***" if SECRETO.search(str(k)) else sanitizar(v)
            for k, v in datos.items()
        }
    if isinstance(datos, (list, tuple)):
        return [sanitizar(v) for v in datos[:20]]  # un volcado no debe ser enorme
    if isinstance(datos, str):
        return datos[:200]
    if isinstance(datos, (int, float, bool, type(None))):
        return datos
    return repr(datos)[:200]


def recolectar_diagnostico(funcion: Callable[P, R]) -> Callable[P, R]:
    """Si la función falla, registra un informe JSON y relanza el error."""

    @functools.wraps(funcion)
    def envoltura(*args: P.args, **kwargs: P.kwargs) -> R:
        try:
            return funcion(*args, **kwargs)
        except Exception as error:
            tb = error.__traceback__
            assert tb is not None
            while tb.tb_next:
                tb = tb.tb_next  # marco más interno: donde ocurrió el error
            marco = traceback.extract_tb(tb)[0]
            locales = dict(tb.tb_frame.f_locals)  # 3.13+: f_locals es un proxy
            detalle = error.to_dict() if isinstance(error, AutomationError) else {}
            informe = {
                "ts": datetime.now(UTC).isoformat(timespec="seconds"),
                "funcion": funcion.__qualname__,
                "error": {
                    "tipo": type(error).__name__,
                    "mensaje": str(error)[:300],
                    "contexto": detalle.get("contexto", {}),
                    "archivo": os.path.basename(marco.filename),
                    "linea": marco.lineno,
                    "en": marco.name,
                },
                "datos": {"kwargs": sanitizar(kwargs), "locales": sanitizar(locales)},
                "sistema": {
                    "python": platform.python_version(),
                    "pid": os.getpid(),
                    "disco_libre_gb": round(shutil.disk_usage(".").free / 2**30, 1),
                    "entorno": {k: os.environ.get(k) for k in ENV_PERMITIDAS},
                },
            }
            logger.error("diagnostico", extra={"informe": informe})
            raise  # el orquestador decide qué hacer

    return envoltura


def resumen(informe: dict[str, Any]) -> str:
    error = informe["error"]
    return f"{error['tipo']} en {error['archivo']}:{error['linea']} en {error['en']}()"
```

### 4.3 La tarea

Es lo que en MWAA sería la tarea de un DAG. Observa:

- **`construir_politica()`:** reintenta solo lo que `clasificar()` marca como transitorio. Respeta `Retry-After` y, si no viene, usa Full Jitter (`wait_random_exponential`).
- **`extraer()`:** ante un 401 renueva el token y reintenta **una sola vez**.
- **`procesar_lote()`:** un registro malo va a la DLQ y el lote continúa, salvo que se supere el umbral.
- **`cargar()`:** UPSERT, así repetir la tarea no duplica filas.
- **El temporal de staging:** se borra en el `finally`. Si el proceso **muere** (timeout), queda en disco, y eso es lo que limpiará la remediación.

**Archivo:** `src/monitoreo_pipeline/tarea.py`

```python
"""La tarea del pipeline: lo que en MWAA sería la tarea de un DAG.

Extrae ventas de la API, aísla los registros inválidos en la DLQ y carga el
resto con UPSERT. El vigilante la ejecuta en un proceso aparte.
"""

import argparse
import json
import logging
import sqlite3
import traceback
import uuid
from contextlib import closing
from dataclasses import dataclass, field
from datetime import UTC, date, datetime
from decimal import Decimal, InvalidOperation
from pathlib import Path
from typing import Any

from tenacity import (
    RetryCallState,
    Retrying,
    before_sleep_log,
    retry_if_exception,
    stop_after_attempt,
    stop_after_delay,
    wait_random_exponential,
)

from monitoreo_pipeline import configuracion
from monitoreo_pipeline.clasificacion import Ruta, clasificar
from monitoreo_pipeline.configuracion import Ajustes, Politica, PoliticaReintento
from monitoreo_pipeline.diagnostico import recolectar_diagnostico
from monitoreo_pipeline.errores import AutomationError, LoteRechazadoError
from monitoreo_pipeline.fuente import APISimulada
from monitoreo_pipeline.registro import configurar_logging, contexto_ejecucion

logger = logging.getLogger("monitoreo.tarea")


# --- Política de reintento (temas 4.3 y 4.25) ---------------------------------
def construir_politica(p: PoliticaReintento) -> Retrying:
    full_jitter = wait_random_exponential(
        multiplier=p.multiplicador, max=p.espera_maxima
    )

    def esperar(estado: RetryCallState) -> float:
        """Respeta Retry-After si el servidor lo envía; si no, Full Jitter."""
        error = estado.outcome.exception() if estado.outcome else None
        retry_after = getattr(error, "retry_after", None)
        if retry_after is not None:
            return float(min(retry_after, 30))
        return float(full_jitter(estado))

    return Retrying(
        retry=retry_if_exception(lambda e: clasificar(e) is Ruta.REINTENTAR),
        stop=stop_after_attempt(p.max_intentos) | stop_after_delay(p.max_segundos),
        wait=esperar,
        before_sleep=before_sleep_log(logger, logging.WARNING),
        reraise=True,  # al agotar, sale el error original
    )


def extraer(api: APISimulada, politica: Retrying, token: str) -> list[dict[str, Any]]:
    try:
        return politica(api.obtener_ventas, token)
    except AutomationError as error:
        if clasificar(error) is not Ruta.CORREGIR_Y_REINTENTAR:
            raise
        logger.warning("HTTP 401: se renueva el token y se reintenta una sola vez")
        nuevo = api.renovar_token()
        logger.info("Cabecera renovada: Authorization: Bearer %s", nuevo)
        return politica(api.obtener_ventas, nuevo)


# --- Fallos parciales y DLQ (temas 4.4 y 4.5) ----------------------------------
@dataclass
class ResultadoLote:
    exitos: list[dict[str, Any]] = field(default_factory=list)
    fallos: list[dict[str, Any]] = field(default_factory=list)

    @property
    def tasa_fallos(self) -> float:
        total = len(self.exitos) + len(self.fallos)
        return len(self.fallos) / total if total else 0.0


def validar(venta: dict[str, Any]) -> dict[str, Any]:
    try:
        importe = Decimal(venta["importe"])
    except InvalidOperation:
        raise ValueError(f"importe inválido: {venta['importe']!r}") from None
    if importe <= 0:
        raise ValueError("importe no positivo")
    date.fromisoformat(venta["fecha"])  # ValueError si la fecha no existe
    return {"id": venta["id"], "importe": str(importe), "fecha": venta["fecha"]}


def enviar_a_dlq(dlq: Path, venta: dict[str, Any], error: Exception) -> None:
    """El payload se guarda INTACTO: es lo que se reprocesará."""
    registro = {
        "dlq_id": str(uuid.uuid4()),
        "fecha_utc": datetime.now(UTC).isoformat(timespec="seconds"),
        "componente": "tarea_ventas",
        "execution_id": (contexto_ejecucion.get() or {}).get("execution_id"),
        "error_tipo": type(error).__name__,
        "error_mensaje": str(error),
        "traza": traceback.format_exception(error)[-1].strip(),
        "payload": venta,
    }
    dlq.parent.mkdir(parents=True, exist_ok=True)
    with dlq.open("a", encoding="utf-8") as archivo:  # append: no se reescribe
        archivo.write(json.dumps(registro, ensure_ascii=False) + "\n")


def procesar_lote(
    ventas: list[dict[str, Any]], dlq: Path, umbral: float
) -> ResultadoLote:
    resultado = ResultadoLote()
    for venta in ventas:
        try:  # un registro no tumba el lote
            resultado.exitos.append(validar(venta))
        except (ValueError, KeyError) as error:
            resultado.fallos.append(venta)
            enviar_a_dlq(dlq, venta, error)
            logger.warning(  # el log se sanitiza; la DLQ conserva el payload intacto
                "Registro %s a la DLQ (cliente %s): %s",
                venta.get("id"),
                venta.get("cliente_email"),
                error,
            )
    logger.info(
        "Lote validado",
        extra={
            "exitos": len(resultado.exitos),
            "en_dlq": len(resultado.fallos),
            "tasa_fallos": round(resultado.tasa_fallos, 2),
        },
    )
    if resultado.tasa_fallos > umbral:  # fallo sistémico: se detiene
        raise LoteRechazadoError(
            f"tasa de fallos {resultado.tasa_fallos:.0%} > {umbral:.0%}",
            contexto={"en_dlq": len(resultado.fallos)},
        )
    return resultado


# --- Carga idempotente (tema 4.9) ---------------------------------------------
def cargar(ruta_bd: Path, filas: list[dict[str, Any]]) -> int:
    with closing(sqlite3.connect(ruta_bd)) as conn, conn:
        conn.execute(
            "CREATE TABLE IF NOT EXISTS ventas (id TEXT PRIMARY KEY, importe TEXT, "
            "fecha TEXT)"
        )
        conn.executemany(  # UPSERT: repetir la carga deja el mismo estado
            "INSERT INTO ventas (id, importe, fecha) VALUES (:id, :importe, :fecha) "
            "ON CONFLICT(id) DO UPDATE SET importe = excluded.importe, "
            "fecha = excluded.fecha",
            filas,
        )
    return len(filas)


@recolectar_diagnostico
def ejecutar_tarea(escenario: str, ajustes: Ajustes, politica: Politica) -> int:
    contexto = contexto_ejecucion.get() or {}
    staging = ajustes.salida / "staging"
    staging.mkdir(parents=True, exist_ok=True)
    temporal = staging / f"extract_{contexto.get('execution_id')}.tmp"
    temporal.write_text("descarga en curso", encoding="utf-8")
    try:
        return _extraer_validar_cargar(escenario, ajustes, politica)
    finally:  # se borra si la tarea termina, bien o con error; si la matan, queda
        temporal.unlink(missing_ok=True)


def _extraer_validar_cargar(
    escenario: str, ajustes: Ajustes, politica: Politica
) -> int:
    api = APISimulada(escenario, ajustes.api_token.get_secret_value())
    ventas = extraer(
        api,
        construir_politica(politica.reintento),
        ajustes.api_token.get_secret_value(),
    )
    logger.info(
        "Extracción completa",
        extra={"registros": len(ventas), "llamadas_api": api.llamadas},
    )
    lote = procesar_lote(
        ventas, ajustes.salida / "dlq_ventas.jsonl", politica.tarea.umbral_fallos
    )
    return cargar(ajustes.salida / "dw.db", lote.exitos)


def main(argv: list[str] | None = None) -> int:
    parser = argparse.ArgumentParser(prog="python -m monitoreo_pipeline.tarea")
    parser.add_argument("--escenario", required=True)
    parser.add_argument("--execution-id", required=True)
    parser.add_argument("--intento", type=int, default=1)
    args = parser.parse_args(argv)
    configurar_logging()
    contexto_ejecucion.set({"execution_id": args.execution_id, "intento": args.intento})
    try:
        ajustes, politica = configuracion.cargar()
        filas = ejecutar_tarea(args.escenario, ajustes, politica)
    except AutomationError as error:
        logger.error("Tarea fallida", extra={"error": error.to_dict()})
        return error.codigo_salida
    except Exception:
        logger.exception("Error inesperado")
        return 1
    logger.info("resultado", extra={"filas": filas})
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

> **Checkpoint 4.** Ejecuta la tarea **sola**, sin vigilante, primero con el escenario normal:
>
> ```bash
> python -m monitoreo_pipeline.tarea --escenario normal --execution-id R-prueba
> ```
>
> Resultado esperado (código de salida `0`; se muestran los campos clave):
>
> ```text
> {"nivel": "WARNING", "logger": "monitoreo.tarea", "mensaje": "Retrying monitoreo_pipeline.fuente.APISimulada.obtener_ventas in 0.0268 seconds as it raised HTTPStatusError: HTTP 503.", "execution_id": "R-prueba", "intento": 1}
> {"nivel": "WARNING", "logger": "monitoreo.tarea", "mensaje": "Retrying monitoreo_pipeline.fuente.APISimulada.obtener_ventas in 1 seconds as it raised HTTPStatusError: HTTP 429.", …}
> {"nivel": "INFO", "mensaje": "Extracción completa", "registros": 20, "llamadas_api": 3, …}
> {"nivel": "WARNING", "mensaje": "Registro V-0004 a la DLQ (cliente [EMAIL]): importe inválido: 'N/A'", …}
> {"nivel": "WARNING", "mensaje": "Registro V-0012 a la DLQ (cliente [EMAIL]): importe no positivo", …}
> {"nivel": "INFO", "mensaje": "Lote validado", "exitos": 18, "en_dlq": 2, "tasa_fallos": 0.1, …}
> {"nivel": "INFO", "mensaje": "resultado", "filas": 18, …}
> ```
>
> En la salida observa:
>
> - **La espera tras el 503 es aleatoria** (Full Jitter); tras el 429, es exactamente 1 s (`Retry-After`).
> - **El log oculta el correo**, pero `salida/dlq_ventas.jsonl` conserva el payload intacto para reprocesarlo.
>
> Después ejecuta el escenario permanente:
>
> ```bash
> python -m monitoreo_pipeline.tarea --escenario permanente --execution-id R-prueba2
> ```
>
> El código de salida es `3` y aparece el evento `diagnostico`. Observa estos campos:
>
> - **`"en": "obtener_ventas"`:** el marco donde ocurrió el error.
> - **`"token": "[REDACTADO]"`:** la variable local con el secreto aparece oculta.
> - **`"python"`:** la versión con la que corriste la tarea.
>
> Al terminar, borra la carpeta `salida` para empezar limpio la Parte 6:
>
> **PowerShell** `Remove-Item -Recurse -Force salida` | **bash** `rm -rf salida`

---

## Parte 5. Vigilar, alertar, remediar y auditar (20 min)

### 5.1 Estado operativo persistido

Cada comando de la Parte 6 es un **proceso nuevo**. Para que la deduplicación de alertas, el límite de remediaciones por hora y las aprobaciones sobrevivan entre comandos, se guardan en SQLite:

**Archivo:** `src/monitoreo_pipeline/almacen.py`

```python
"""Estado operativo persistido en SQLite (tema 4.15).

Guarda las ejecuciones, las alertas enviadas, las remediaciones y las
solicitudes de aprobación. Así cada comando, aunque sea un proceso nuevo,
conoce lo que ya ocurrió (deduplicación, límite por hora, aprobaciones).
"""

import json
import sqlite3
from contextlib import closing
from pathlib import Path
from typing import Any

ESQUEMA = """
CREATE TABLE IF NOT EXISTS ejecuciones (
    execution_id TEXT PRIMARY KEY, escenario TEXT, inicio REAL, fin REAL,
    estado TEXT, senal TEXT, decision TEXT, intentos INTEGER);
CREATE TABLE IF NOT EXISTS alertas (firma TEXT, enviada REAL);
CREATE TABLE IF NOT EXISTS remediaciones (accion TEXT, ts REAL);
CREATE TABLE IF NOT EXISTS solicitudes (
    token TEXT PRIMARY KEY, accion TEXT, parametros TEXT, solicitante TEXT,
    expira REAL, estado TEXT, historial TEXT);
"""


class Almacen:
    def __init__(self, ruta: Path) -> None:
        ruta.parent.mkdir(parents=True, exist_ok=True)
        self.ruta = ruta
        with self._conectar() as conn:
            conn.executescript(ESQUEMA)

    def _conectar(self) -> sqlite3.Connection:
        return sqlite3.connect(self.ruta, isolation_level=None)  # autocommit

    def ejecutar(
        self, sql: str, parametros: tuple[Any, ...] = ()
    ) -> list[tuple[Any, ...]]:
        with closing(self._conectar()) as conn:
            return conn.execute(sql, parametros).fetchall()

    # --- ejecuciones ---
    def iniciar(self, execution_id: str, escenario: str, ahora: float) -> None:
        self.ejecutar(
            "INSERT INTO ejecuciones (execution_id, escenario, inicio, estado, "
            "intentos) VALUES (?, ?, ?, 'RUNNING', 0)",
            (execution_id, escenario, ahora),
        )

    def cerrar(self, execution_id: str, **campos: Any) -> None:
        columnas = ", ".join(f"{c} = ?" for c in campos)  # nombres fijos del código
        self.ejecutar(
            f"UPDATE ejecuciones SET {columnas} WHERE execution_id = ?",
            (*campos.values(), execution_id),
        )

    # --- solicitudes de aprobación ---
    def guardar_solicitud(self, token: str, datos: dict[str, Any]) -> None:
        self.ejecutar(
            "INSERT OR REPLACE INTO solicitudes VALUES (?, ?, ?, ?, ?, ?, ?)",
            (
                token,
                datos["accion"],
                json.dumps(datos["parametros"]),
                datos["solicitante"],
                datos["expira"],
                datos["estado"],
                json.dumps(datos["historial"], ensure_ascii=False),
            ),
        )

    def leer_solicitud(self, token: str) -> dict[str, Any] | None:
        filas = self.ejecutar("SELECT * FROM solicitudes WHERE token = ?", (token,))
        if not filas:
            return None
        _, accion, parametros, solicitante, expira, estado, historial = filas[0]
        return {
            "accion": accion,
            "parametros": json.loads(parametros),
            "solicitante": solicitante,
            "expira": expira,
            "estado": estado,
            "historial": json.loads(historial),
        }
```

### 5.2 El vigilante: el papel de MWAA

`ejecutar_vigilado()` lanza la tarea con `subprocess.run(..., timeout=...)`, como Airflow con `execution_timeout`. Después convierte lo que observa en una **señal**:

| Lo que observa | Señal |
| :--- | :--- |
| Código de salida distinto de 0 | Se lee del diagnóstico: `HTTP 404`, `HTTP 503`, `DATOS_INVALIDOS`… |
| No terminó a tiempo | `TIMEOUT` |
| Terminó bien, pero con 0 filas | `SUSPECT` |
| Un `RUNNING` sin latido | `latido_vencido()` |

El log de cada intento se guarda en `salida/logs/`, que hace el papel de CloudWatch Logs:

**Archivo:** `src/monitoreo_pipeline/vigilante.py`

```python
"""Detección de trabajos fallidos con cuatro señales (tema 4.22).

Ejecuta la tarea en un proceso aparte, como lo hace MWAA, y vigila:
código de salida, tiempo máximo (execution_timeout), volumen de datos y
el diagnóstico que la tarea dejó en su log.
"""

import json
import os
import subprocess
import sys
import time
from dataclasses import dataclass
from pathlib import Path
from typing import Any


@dataclass
class ResultadoTarea:
    estado: str  # SUCCESS, FAILED o SUSPECT
    senal: str | None
    codigo_salida: int | None
    duracion_s: float
    filas: int | None
    diagnostico: dict[str, Any] | None
    log: Path


def senal_de(diagnostico: dict[str, Any] | None, codigo: int) -> str:
    """Traduce el diagnóstico a la señal que usa la política."""
    if diagnostico is None:
        return f"EXIT_{codigo}"
    error = diagnostico["error"]
    if "status" in error["contexto"]:
        return f"HTTP {error['contexto']['status']}"
    if error["tipo"] == "LoteRechazadoError":
        return "DATOS_INVALIDOS"
    return str(error["tipo"])


def ejecutar_vigilado(
    escenario: str, execution_id: str, intento: int, timeout_s: float, logs: Path
) -> ResultadoTarea:
    comando = [
        sys.executable,
        "-m",
        "monitoreo_pipeline.tarea",
        "--escenario",
        escenario,
        "--execution-id",
        execution_id,
        "--intento",
        str(intento),
    ]
    entorno = os.environ | {"PYTHONIOENCODING": "utf-8"}
    log = logs / f"{execution_id}_intento{intento}.jsonl"  # simula CloudWatch Logs
    logs.mkdir(parents=True, exist_ok=True)
    inicio = time.monotonic()
    try:
        proceso = subprocess.run(
            comando, capture_output=True, timeout=timeout_s, env=entorno
        )
    except subprocess.TimeoutExpired as colgado:  # señal 2: tiempo máximo
        log.write_bytes(colgado.stdout or b"")
        return ResultadoTarea(
            "FAILED",
            "TIMEOUT",
            None,
            round(time.monotonic() - inicio, 2),
            None,
            None,
            log,
        )
    duracion = round(time.monotonic() - inicio, 2)
    salida = proceso.stdout.decode("utf-8")
    log.write_text(salida, encoding="utf-8")
    eventos = [
        json.loads(linea) for linea in salida.splitlines() if linea.startswith("{")
    ]
    diagnostico = next(
        (e["informe"] for e in eventos if e["mensaje"] == "diagnostico"), None
    )
    if proceso.returncode != 0:  # señal 1: código de salida
        senal = senal_de(diagnostico, proceso.returncode)
        return ResultadoTarea(
            "FAILED", senal, proceso.returncode, duracion, None, diagnostico, log
        )
    filas = next(e["filas"] for e in eventos if e["mensaje"] == "resultado")
    if filas == 0:  # señal 3: terminó "bien" sin datos
        return ResultadoTarea("SUSPECT", "SUSPECT", 0, duracion, 0, None, log)
    return ResultadoTarea("SUCCESS", None, 0, duracion, filas, None, log)


def latido_vencido(
    ultimo_latido: float, ahora: float, max_silencio_s: float = 300
) -> bool:
    """Señal 4: un RUNNING que no actualiza su latido se considera caído."""
    return ahora - ultimo_latido > max_silencio_s
```

### 5.3 Alertas enriquecidas y sin ruido

`enriquecer()` arma la alerta que la guardia necesita: impacto, ubicación exacta, entorno, versión, runbook y un enlace a los logs filtrados por `execution_id`.

`AlertManager` evita el ruido:

- **Deduplicación:** la misma firma (servicio + señal) no se repite en 5 minutos.
- **Límite de tasa:** máximo 5 alertas por minuto. Una alerta `CRITICAL` pasa aunque se haya alcanzado el límite.

**Archivo:** `src/monitoreo_pipeline/alertas.py`

```python
"""Alertas enriquecidas (4.26) y sin ruido: deduplicación y límite de tasa (4.14)."""

import hashlib
import json
import socket
import time
import uuid
from collections.abc import Callable
from datetime import UTC, datetime, timedelta
from pathlib import Path
from typing import Any
from urllib.parse import quote

from monitoreo_pipeline.almacen import Almacen
from monitoreo_pipeline.configuracion import Ajustes


def enriquecer(
    ajustes: Ajustes,
    *,
    severidad: str,
    senal: str,
    impacto: str,
    execution_id: str,
    runbook: str,
    diagnostico: dict[str, Any] | None,
) -> dict[str, Any]:
    """Añade el contexto que la guardia necesita: impacto, dónde mirar y qué hacer."""
    error = (diagnostico or {}).get("error", {})
    desde = (datetime.now(UTC) - timedelta(minutes=15)).isoformat(timespec="seconds")
    return {
        "alerta_id": str(uuid.uuid4()),
        "servicio": ajustes.pipeline,
        "severidad": severidad,
        "senal": senal,
        "impacto": impacto,
        "error_tipo": error.get("tipo", senal),
        "error_mensaje": error.get("mensaje", senal),
        "ubicacion": (
            f"{error['archivo']}:{error['linea']} en {error['en']}()" if error else None
        ),
        "entorno": ajustes.entorno,
        "version": ajustes.app_version,
        "host": socket.gethostname(),
        "execution_id": execution_id,
        "enlaces": {
            "runbook": f"{ajustes.wiki_url}/{runbook}",
            "logs": f"{ajustes.logs_url}?execution_id={quote(execution_id)}"
            f"&desde={quote(desde)}",
        },
    }


class CanalArchivo:
    """Simula el webhook (Slack, Teams): cada alerta es una línea JSON."""

    def __init__(self, ruta: Path) -> None:
        self.ruta = ruta

    def enviar(self, alerta: dict[str, Any]) -> None:
        with self.ruta.open("a", encoding="utf-8") as archivo:
            archivo.write(json.dumps(alerta, ensure_ascii=False) + "\n")


class AlertManager:
    def __init__(
        self,
        almacen: Almacen,
        enviar: Callable[[dict[str, Any]], None],
        ventana_dedup_s: int = 300,
        max_por_minuto: int = 5,
        reloj: Callable[[], float] = time.time,
    ) -> None:
        self.almacen, self.enviar, self.reloj = almacen, enviar, reloj
        self.ventana, self.max_por_minuto = ventana_dedup_s, max_por_minuto

    def alertar(self, alerta: dict[str, Any]) -> str:
        firma = hashlib.sha256(
            f"{alerta['servicio']}|{alerta['senal']}".encode()
        ).hexdigest()[:12]
        ahora = self.reloj()
        duplicadas = self.almacen.ejecutar(
            "SELECT COUNT(*) FROM alertas WHERE firma = ? AND enviada > ?",
            (firma, ahora - self.ventana),
        )[0][0]
        if duplicadas:
            return f"suprimida (duplicada, firma {firma})"
        ultimo_minuto = self.almacen.ejecutar(
            "SELECT COUNT(*) FROM alertas WHERE enviada > ?", (ahora - 60,)
        )[0][0]
        if ultimo_minuto >= self.max_por_minuto and alerta["severidad"] != "CRITICAL":
            return "suprimida (límite de tasa)"
        self.almacen.ejecutar("INSERT INTO alertas VALUES (?, ?)", (firma, ahora))
        self.enviar(alerta | {"firma": firma})
        return f"enviada (firma {firma})"
```

### 5.4 CloudWatch y MWAA con botocore Stubber

Los clientes son los de `boto3`. `Stubber` intercepta la llamada **después** de validar los parámetros contra el modelo de la API, así que una métrica mal formada falla igual que fallaría en AWS. En producción se usa el mismo código sin el Stubber.

Observa en `publicar_metricas()` las dimensiones `Pipeline` y `Entorno`: el `execution_id` **no** es dimensión, porque tendría miles de valores (tema 4.11).

**Archivo:** `src/monitoreo_pipeline/aws.py`

```python
"""Clientes de AWS: CloudWatch (métricas, 4.11) y MWAA (pausar un DAG, 4.31).

En el laboratorio no hay cuenta de AWS: cada llamada pasa por botocore Stubber,
que valida la petición contra el modelo oficial de la API y devuelve una
respuesta preparada. En producción se usaría el mismo código sin el Stubber.
"""

import json
import re
from pathlib import Path
from typing import Any

import boto3
from botocore.stub import Stubber

REGION = "us-east-1"


def _cliente_simulado(servicio: str) -> tuple[Any, Stubber]:
    cliente = boto3.client(
        servicio,
        region_name=REGION,
        aws_access_key_id="laboratorio",  # nunca se envían: Stubber responde antes
        aws_secret_access_key="laboratorio",
    )
    stubber = Stubber(cliente)
    stubber.activate()
    return cliente, stubber


def publicar_metricas(
    contadores: dict[str, float],
    duracion_s: float,
    exito: bool,
    dimensiones: dict[str, str],
    registro: Path,
) -> dict[str, Any]:
    """Una sola llamada PutMetricData al final de la ejecución."""
    dims = [{"Name": n, "Value": v} for n, v in dimensiones.items()]
    datos = [
        {"MetricName": n, "Value": float(v), "Unit": "Count", "Dimensions": dims}
        for n, v in contadores.items()
    ]
    datos += [
        {
            "MetricName": "DuracionSegundos",
            "Value": duracion_s,
            "Unit": "Seconds",
            "Dimensions": dims,
        },
        {
            "MetricName": "EjecucionFallida",
            "Value": 0.0 if exito else 1.0,
            "Unit": "Count",
            "Dimensions": dims,
        },
    ]
    peticion = {"Namespace": "Automatizacion/Pipelines", "MetricData": datos}
    cloudwatch, stubber = _cliente_simulado("cloudwatch")
    stubber.add_response("put_metric_data", {}, peticion)
    cloudwatch.put_metric_data(**peticion)  # ParamValidationError si algo no cuadra
    stubber.assert_no_pending_responses()
    with registro.open("a", encoding="utf-8") as archivo:
        archivo.write(json.dumps(peticion, ensure_ascii=False) + "\n")
    return peticion


def pausar_dag(entorno: str, dag_id: str) -> dict[str, Any]:
    """PATCH /dags/{dag_id} con is_paused=true mediante la API REST de MWAA."""
    if not re.fullmatch(r"[A-Za-z0-9_.-]+", dag_id):  # se usa en una ruta: validar
        raise ValueError(f"dag_id inválido: {dag_id!r}")
    parametros = {
        "Name": entorno,
        "Method": "PATCH",
        "Path": f"/dags/{dag_id}",
        "Body": {"is_paused": True},
    }
    mwaa, stubber = _cliente_simulado("mwaa")
    stubber.add_response(
        "invoke_rest_api",
        {
            "RestApiStatusCode": 200,
            "RestApiResponse": {"dag_id": dag_id, "is_paused": True},
        },
        parametros,
    )
    respuesta = mwaa.invoke_rest_api(**parametros)
    if respuesta["RestApiStatusCode"] != 200:
        raise RuntimeError(f"MWAA respondió {respuesta['RestApiStatusCode']}")
    return dict(respuesta["RestApiResponse"])
```

### 5.5 Remediación controlada y aprobación humana

`MotorRemediacion` aplica cinco controles en orden:

1. **Catálogo:** solo se ejecutan las acciones de la lista.
2. **Riesgo:** lo que no es de riesgo bajo pasa a aprobación humana.
3. **Límite por hora:** evita entrar en bucles.
4. **Precondición:** se comprueba que la situación que justifica la acción sigue presente.
5. **Dry-run, ejecución y verificación:** primero se simula, después se ejecuta y al final se comprueba que el problema se resolvió.

`PuertaAprobacion` emite un **token de un solo uso** con caducidad. Nadie puede aprobar su propia solicitud, y sin respuesta a tiempo la acción **no** se ejecuta.

**Archivo:** `src/monitoreo_pipeline/remediacion.py`

```python
"""Remediación controlada (4.27) y aprobación humana en el ciclo (4.28)."""

import secrets
import time
from collections.abc import Callable
from dataclasses import dataclass
from pathlib import Path
from typing import Any

from monitoreo_pipeline import aws
from monitoreo_pipeline.almacen import Almacen
from monitoreo_pipeline.configuracion import Ajustes


@dataclass
class Accion:
    riesgo: str  # "bajo" se ejecuta sola; cualquier otro requiere aprobación
    precondicion: Callable[[], bool]  # ¿sigue dándose la situación que la justifica?
    ejecutar: Callable[[bool], list[str]]  # recibe dry_run y devuelve los cambios
    verificar: Callable[[], bool]  # ¿se resolvió el problema?


class MotorRemediacion:
    def __init__(
        self,
        catalogo: dict[str, Accion],
        almacen: Almacen,
        max_por_hora: int = 3,
        reloj: Callable[[], float] = time.time,
    ) -> None:
        self.catalogo, self.almacen = catalogo, almacen
        self.max_por_hora, self.reloj = max_por_hora, reloj

    def remediar(
        self, nombre: str, dry_run: bool = True, aprobada: bool = False
    ) -> str:
        accion = self.catalogo.get(nombre)
        if accion is None:
            return f"RECHAZADA: '{nombre}' no está en el catálogo"
        if accion.riesgo != "bajo" and not aprobada:
            return "PENDIENTE: requiere aprobación humana"
        recientes = self.almacen.ejecutar(
            "SELECT COUNT(*) FROM remediaciones WHERE accion = ? AND ts > ?",
            (nombre, self.reloj() - 3600),
        )[0][0]
        if recientes >= self.max_por_hora:
            return "BLOQUEADA: límite por hora alcanzado, se escala a guardia"
        if not accion.precondicion():
            return "OMITIDA: la condición que la justifica ya no se cumple"
        cambios = accion.ejecutar(dry_run)
        if dry_run:
            return f"DRY-RUN: {len(cambios)} cambio(s) previsto(s) {cambios}"
        self.almacen.ejecutar(
            "INSERT INTO remediaciones VALUES (?, ?)", (nombre, self.reloj())
        )
        return (
            "OK: verificada" if accion.verificar() else "FALLÓ la verificación: escalar"
        )


class PuertaAprobacion:
    def __init__(
        self,
        almacen: Almacen,
        ttl_s: int = 1800,
        reloj: Callable[[], float] = time.time,
    ) -> None:
        self.almacen, self.ttl, self.reloj = almacen, ttl_s, reloj

    def solicitar(
        self, accion: str, parametros: dict[str, Any], solicitante: str
    ) -> str:
        token = secrets.token_urlsafe(8)  # se envía en el mensaje a los aprobadores
        self.almacen.guardar_solicitud(
            token,
            {
                "accion": accion,
                "parametros": parametros,
                "solicitante": solicitante,
                "expira": self.reloj() + self.ttl,
                "estado": "PENDIENTE",
                "historial": [],
            },
        )
        return token

    def resolver(self, token: str, aprobador: str, aprobar: bool) -> str:
        s = self.almacen.leer_solicitud(token)
        if s is None:
            return "token desconocido"
        if s["estado"] != "PENDIENTE":
            return f"ya resuelta: {s['estado']}"
        if self.reloj() > s["expira"]:
            s["estado"] = "EXPIRADA"  # sin respuesta a tiempo = NO se ejecuta
        elif aprobador == s["solicitante"]:
            return "rechazado: nadie aprueba su propia solicitud"
        else:
            s["estado"] = "APROBADA" if aprobar else "DENEGADA"
        s["historial"].append(f"{s['estado']} por {aprobador}")
        self.almacen.guardar_solicitud(token, s)
        return str(s["estado"])

    def puede_ejecutar(self, token: str) -> bool:
        s = self.almacen.leer_solicitud(token)
        return bool(s and s["estado"] == "APROBADA" and self.reloj() <= s["expira"])

    def marcar_ejecutada(self, token: str) -> None:
        s = self.almacen.leer_solicitud(token)
        if s:
            s["estado"] = "EJECUTADA"  # un token sirve una sola vez
            self.almacen.guardar_solicitud(token, s)


def construir_catalogo(
    ajustes: Ajustes, reprocesar: Callable[[], bool]
) -> dict[str, Accion]:
    """Lo que no está en esta lista no se ejecuta, la pida quien la pida."""
    staging: Path = ajustes.salida / "staging"
    estado_dag: dict[str, Any] = {}

    def temporales() -> list[Path]:
        return sorted(staging.glob("*.tmp"))

    def limpiar(dry_run: bool) -> list[str]:
        archivos = temporales()
        if not dry_run:
            for archivo in archivos:
                archivo.unlink()
        return [a.name for a in archivos]

    def pausar(dry_run: bool) -> list[str]:
        cambio = f"PATCH /dags/{ajustes.dag_consumidor} is_paused=true"
        if not dry_run:
            estado_dag.update(
                aws.pausar_dag(ajustes.mwaa_entorno, ajustes.dag_consumidor)
            )
        return [cambio]

    resultado_reproceso: dict[str, bool] = {}

    def reprocesar_lote(dry_run: bool) -> list[str]:
        if not dry_run:
            resultado_reproceso["ok"] = reprocesar()
        return ["re-ejecutar la tarea con el lote corregido por el proveedor"]

    return {
        "limpiar_temporales": Accion(
            "bajo", lambda: bool(temporales()), limpiar, lambda: not temporales()
        ),
        "pausar_dag_consumidor": Accion(
            "bajo", lambda: True, pausar, lambda: estado_dag.get("is_paused") is True
        ),
        "reprocesar_lote": Accion(
            "alto",
            lambda: True,
            reprocesar_lote,
            lambda: resultado_reproceso.get("ok", False),
        ),
    }
```

### 5.6 Auditoría con HMAC encadenado

La auditoría es un canal propio, separado del log de depuración. Cada evento responde quién, qué, cuándo, por qué (`correlation_id`) y con qué resultado. Además firma el evento anterior, así que editar uno rompe la cadena desde ese punto:

**Archivo:** `src/monitoreo_pipeline/auditoria.py`

```python
"""Auditoría de decisiones con HMAC encadenado (temas 4.16 y 4.29)."""

import hashlib
import hmac
import json
import uuid
from datetime import UTC, datetime
from pathlib import Path
from typing import Any


class Auditoria:
    """Canal propio (no el log de depuración). Cada evento firma el anterior."""

    def __init__(self, ruta: Path, clave: bytes) -> None:
        self.ruta, self.clave = ruta, clave

    def _firma(self, evento: dict[str, Any]) -> str:
        canonico = json.dumps(evento, sort_keys=True, ensure_ascii=False).encode()
        return hmac.new(self.clave, canonico, hashlib.sha256).hexdigest()

    def registrar(
        self,
        actor: str,
        accion: str,
        recurso: str,
        resultado: str,
        correlation_id: str,
        **detalle: Any,
    ) -> None:
        lineas = (
            self.ruta.read_text(encoding="utf-8").splitlines()
            if self.ruta.exists()
            else []
        )
        anterior = json.loads(lineas[-1])["firma"] if lineas else "0" * 64
        evento = {
            "event_id": str(uuid.uuid4()),
            "correlation_id": correlation_id,
            "ts": datetime.now(UTC).isoformat(timespec="milliseconds"),
            "actor": actor,
            "accion": accion,
            "recurso": recurso,
            "resultado": resultado,
            "detalle": detalle,
            "firma_anterior": anterior,
        }
        evento["firma"] = self._firma(evento)
        with self.ruta.open("a", encoding="utf-8") as archivo:  # solo se añade
            archivo.write(json.dumps(evento, ensure_ascii=False) + "\n")

    def verificar(self) -> str:
        if not self.ruta.exists():
            return "sin eventos"
        anterior = "0" * 64
        lineas = self.ruta.read_text(encoding="utf-8").splitlines()
        for n, linea in enumerate(lineas, start=1):
            evento = json.loads(linea)
            firma = evento.pop("firma")
            if evento["firma_anterior"] != anterior or not hmac.compare_digest(
                firma, self._firma(evento)
            ):
                return f"ALTERADO en el evento {n}"
            anterior = firma
        return f"íntegro ({len(lineas)} eventos)"
```

> **Checkpoint 5.**
>
> ```bash
> python -c "import monitoreo_pipeline.vigilante, monitoreo_pipeline.alertas, monitoreo_pipeline.aws, monitoreo_pipeline.remediacion, monitoreo_pipeline.auditoria; print('ok')"
> ```
>
> Resultado esperado: `ok`

---

## Parte 6. El ciclo completo (15 min)

### 6.1 Controlador, punto de entrada y DAG de referencia

`Controlador.ejecutar()` cierra el bucle del tema 4.30. Por cada intento sigue estos pasos:

1. Vigila la tarea.
2. Muestra el diagnóstico.
3. Decide.
4. Audita la decisión.
5. Aplica la acción que la política asocia a la señal.

Si la decisión es `REINTENTAR`, repite la tarea. Si no, alerta. Al final publica las métricas y cierra el registro de la ejecución:

**Archivo:** `src/monitoreo_pipeline/controlador.py`

```python
"""Bucle cerrado: detectar → diagnosticar → decidir → actuar → registrar (4.30).

Cumple el papel que en producción reparten MWAA (ejecutar y reintentar la
tarea), CloudWatch (logs y métricas) y el controlador de remediación.
"""

import json
import time
import uuid
from collections.abc import Callable
from typing import Any

from monitoreo_pipeline import aws
from monitoreo_pipeline.alertas import AlertManager, CanalArchivo, enriquecer
from monitoreo_pipeline.almacen import Almacen
from monitoreo_pipeline.auditoria import Auditoria
from monitoreo_pipeline.clasificacion import Decision, Fallo, decidir
from monitoreo_pipeline.configuracion import Ajustes, Politica
from monitoreo_pipeline.diagnostico import resumen
from monitoreo_pipeline.remediacion import (
    MotorRemediacion,
    PuertaAprobacion,
    construir_catalogo,
)
from monitoreo_pipeline.vigilante import ResultadoTarea, ejecutar_vigilado

ACTOR = "controlador_remediacion"
CODIGO_POR_ESTADO = {"SUCCESS": 0, "FAILED": 3, "SUSPECT": 4}


class Controlador:
    def __init__(
        self,
        ajustes: Ajustes,
        politica: Politica,
        mostrar: Callable[[str], None] = print,
    ) -> None:
        self.ajustes, self.politica, self.mostrar = ajustes, politica, mostrar
        salida = ajustes.salida
        self.almacen = Almacen(salida / "operacion.db")
        self.auditoria = Auditoria(
            salida / "auditoria.jsonl",
            ajustes.audit_hmac_key.get_secret_value().encode(),
        )
        self.alertas = AlertManager(
            self.almacen,
            CanalArchivo(salida / "alertas.jsonl").enviar,
            politica.alertas.ventana_dedup_s,
            politica.alertas.max_por_minuto,
        )
        self.puerta = PuertaAprobacion(
            self.almacen, politica.remediacion.ttl_aprobacion_s
        )
        self.motor = MotorRemediacion(
            construir_catalogo(ajustes, lambda: self.ejecutar("normal") == "SUCCESS"),
            self.almacen,
            politica.remediacion.max_por_hora,
        )

    def _auditar(
        self, accion: str, resultado: str, execution_id: str, **detalle: Any
    ) -> None:
        self.auditoria.registrar(
            actor=detalle.pop("actor", ACTOR),
            accion=accion,
            recurso=f"dag:{self.ajustes.pipeline}",
            resultado=resultado,
            correlation_id=execution_id,
            **detalle,
        )

    # --- una ejecución vigilada completa -------------------------------------
    def ejecutar(self, escenario: str) -> str:
        execution_id = f"R-{uuid.uuid4().hex[:8]}"
        inicio = time.monotonic()
        self.almacen.iniciar(execution_id, escenario, time.time())
        self.mostrar(f"[{execution_id}] escenario={escenario}")
        tarea = self.politica.tarea
        max_intentos = 1 + tarea.reintentos
        intento, decision = 1, "SIN_ACCION"
        while True:
            r = ejecutar_vigilado(
                escenario,
                execution_id,
                intento,
                tarea.timeout_s,
                self.ajustes.salida / "logs",
            )
            detalle = f"filas={r.filas}" if r.filas is not None else f"señal={r.senal}"
            self.mostrar(
                f"  intento {intento}: {r.estado:<7} {detalle}  "
                f"exit={r.codigo_salida}  ({r.duracion_s} s)"
            )
            if r.estado == "SUCCESS":
                self._auditar("evaluar", "SIN ACCIÓN: ejecución correcta", execution_id)
                break
            if r.diagnostico:
                self.mostrar(f"  diagnóstico: {resumen(r.diagnostico)}")
            fallo = Fallo(
                r.senal or "DESCONOCIDA", tarea.idempotente, intento, max_intentos
            )
            ruta, motivo = decidir(fallo)
            decision = ruta.name
            self.mostrar(f"  decisión: {ruta.name} - {motivo}")
            self._auditar(
                f"decidir:{ruta.name}",
                motivo,
                execution_id,
                senal=r.senal,
                intento=intento,
            )
            self._remediar_segun_politica(r.senal or "", execution_id)
            if ruta is Decision.REINTENTAR:
                intento += 1
                continue
            self._alertar(r, execution_id)
            break
        self._publicar_metricas(r, intento, time.monotonic() - inicio, execution_id)
        self.almacen.cerrar(
            execution_id,
            fin=time.time(),
            estado=r.estado,
            senal=r.senal,
            decision=decision,
            intentos=intento,
        )
        self.mostrar(f"  estado final: {r.estado}  | log: {r.log.as_posix()}")
        return r.estado

    def _remediar_segun_politica(self, senal: str, execution_id: str) -> None:
        nombre = self.politica.acciones.get(senal)
        if nombre is None:
            return
        previo = self.motor.remediar(nombre, dry_run=True)  # primero, simular
        self.mostrar(f"  remediación {nombre}: {previo}")
        self._auditar(f"remediar:{nombre}", previo, execution_id, dry_run=True)
        if previo.startswith("DRY-RUN"):
            final = self.motor.remediar(nombre, dry_run=False)
            self.mostrar(f"  remediación {nombre}: {final}")
            self._auditar(f"remediar:{nombre}", final, execution_id, dry_run=False)
        elif previo.startswith("PENDIENTE"):
            token = self.puerta.solicitar(
                nombre, {"execution_id": execution_id, "senal": senal}, ACTOR
            )
            self.mostrar(
                f"  solicitud de aprobación: token={token} (caduca en "
                f"{self.politica.remediacion.ttl_aprobacion_s // 60} min)"
            )
            self._auditar(
                f"solicitar_aprobacion:{nombre}", "PENDIENTE", execution_id, token=token
            )

    def _alertar(self, r: ResultadoTarea, execution_id: str) -> None:
        sospechosa = r.estado == "SUSPECT"
        senal = r.senal or "DESCONOCIDA"
        alerta = enriquecer(
            self.ajustes,
            severidad="WARNING" if sospechosa else "CRITICAL",
            senal=senal,
            impacto=(
                "La carga terminó sin datos; el DAG consumidor quedó en pausa"
                if sospechosa
                else "El panel de ventas no tiene los datos de hoy"
            ),
            execution_id=execution_id,
            runbook=self.politica.runbooks.get(senal, "rb-000-general"),
            diagnostico=r.diagnostico,
        )
        estado = self.alertas.alertar(alerta)
        self.mostrar(f"  alerta {alerta['severidad']}: {estado}")
        self._auditar("alertar", estado, execution_id, severidad=alerta["severidad"])

    def _publicar_metricas(
        self, r: ResultadoTarea, intentos: int, duracion_s: float, execution_id: str
    ) -> None:
        dlq = self.ajustes.salida / "dlq_ventas.jsonl"
        en_dlq = 0
        if dlq.exists():
            for linea in dlq.read_text(encoding="utf-8").splitlines():
                en_dlq += json.loads(linea)["execution_id"] == execution_id
        contadores: dict[str, float] = {
            "FilasCargadas": r.filas or 0,
            "RegistrosEnDLQ": en_dlq,
            "IntentosTarea": intentos,
        }
        aws.publicar_metricas(
            contadores,
            round(duracion_s, 3),
            r.estado == "SUCCESS",
            {"Pipeline": self.ajustes.pipeline, "Entorno": self.ajustes.entorno},
            self.ajustes.salida / "cloudwatch_metricas.jsonl",
        )
        texto = " ".join(f"{k}={v}" for k, v in contadores.items())
        self.mostrar(
            f"  métricas CloudWatch: {texto} EjecucionFallida="
            f"{0 if r.estado == 'SUCCESS' else 1} (validadas con Stubber)"
        )

    # --- acciones de las personas --------------------------------------------
    def resolver(self, token: str, aprobador: str, aprobar: bool) -> str:
        resultado = self.puerta.resolver(token, aprobador, aprobar)
        solicitud = self.almacen.leer_solicitud(token) or {}
        execution_id = solicitud.get("parametros", {}).get("execution_id", "-")
        self.mostrar(f"Solicitud {token}: {resultado}")
        self._auditar(
            "resolver_aprobacion", resultado, execution_id, actor=aprobador, token=token
        )
        return resultado

    def ejecutar_aprobada(self, token: str) -> str:
        solicitud = self.almacen.leer_solicitud(token)
        if solicitud is None or not self.puerta.puede_ejecutar(token):
            estado = solicitud["estado"] if solicitud else "desconocida"
            self.mostrar(f"No se ejecuta: la solicitud está {estado}")
            return "NO_EJECUTADA"
        execution_id = solicitud["parametros"]["execution_id"]
        self.puerta.marcar_ejecutada(token)
        resultado = self.motor.remediar(
            solicitud["accion"], dry_run=False, aprobada=True
        )
        self.mostrar(f"remediación {solicitud['accion']}: {resultado}")
        self._auditar(
            f"remediar:{solicitud['accion']}",
            resultado,
            execution_id,
            aprobada_por=solicitud["historial"],
            token=token,
        )
        return resultado

    def remediar_manual(self, nombre: str, ejecutar: bool, actor: str) -> str:
        resultado = self.motor.remediar(nombre, dry_run=not ejecutar)
        self.mostrar(f"remediación {nombre}: {resultado}")
        self._auditar(
            f"remediar:{nombre}", resultado, "manual", actor=actor, dry_run=not ejecutar
        )
        return resultado

    def panel(self) -> None:
        self.mostrar("Ejecuciones (más recientes al final):")
        for fila in self.almacen.ejecutar(
            "SELECT execution_id, escenario, estado, senal, decision, intentos "
            "FROM ejecuciones ORDER BY inicio"
        ):
            execution_id, escenario, estado, senal, decision, intentos = fila
            self.mostrar(
                f"  {execution_id}  {escenario:<24} {estado or '-':<8} "
                f"señal={senal or '-':<16} decisión={decision or '-':<11} "
                f"intentos={intentos}"
            )
        pendientes = self.almacen.ejecutar(
            "SELECT token, accion, estado FROM solicitudes ORDER BY expira"
        )
        self.mostrar("Solicitudes de aprobación:")
        for token, accion, estado in pendientes:
            self.mostrar(f"  {token}  {accion:<20} {estado}")
        self.mostrar(f"Auditoría: {self.auditoria.verificar()}")
```

**Archivo:** `src/monitoreo_pipeline/cli.py`

```python
"""Punto de entrada: python -m monitoreo_pipeline <comando>."""

import argparse

from monitoreo_pipeline import configuracion
from monitoreo_pipeline.controlador import CODIGO_POR_ESTADO, Controlador
from monitoreo_pipeline.errores import ConfiguracionError
from monitoreo_pipeline.fuente import ESCENARIOS


def construir_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(
        prog="python -m monitoreo_pipeline",
        description="Ejecuta el pipeline vigilado y gestiona su remediación.",
    )
    sub = parser.add_subparsers(dest="comando", required=True)
    ejecutar = sub.add_parser("ejecutar", help="ejecuta la tarea vigilada")
    ejecutar.add_argument("--escenario", choices=sorted(ESCENARIOS), default="normal")
    aprobar = sub.add_parser("aprobar", help="resuelve una solicitud de aprobación")
    aprobar.add_argument("token")
    aprobar.add_argument("--aprobador", required=True)
    aprobar.add_argument("--denegar", action="store_true")
    sub.add_parser(
        "ejecutar-aprobada", help="ejecuta una acción ya aprobada"
    ).add_argument("token")
    remediar = sub.add_parser("remediar", help="pide una acción a mano (dry-run)")
    remediar.add_argument("accion")
    remediar.add_argument("--ejecutar", action="store_true", help="sin esto, dry-run")
    remediar.add_argument("--actor", default="operador")
    sub.add_parser("panel", help="estado de ejecuciones, aprobaciones y auditoría")
    return parser


def main(argv: list[str] | None = None) -> int:
    args = construir_parser().parse_args(argv)
    try:
        ajustes, politica = configuracion.cargar()
    except ConfiguracionError as error:
        print(error)
        return error.codigo_salida
    controlador = Controlador(ajustes, politica)
    if args.comando == "ejecutar":
        return CODIGO_POR_ESTADO[controlador.ejecutar(args.escenario)]
    if args.comando == "aprobar":
        controlador.resolver(args.token, args.aprobador, aprobar=not args.denegar)
    elif args.comando == "ejecutar-aprobada":
        controlador.ejecutar_aprobada(args.token)
    elif args.comando == "remediar":
        controlador.remediar_manual(args.accion, args.ejecutar, args.actor)
    else:
        controlador.panel()
    return 0
```

**Archivo:** `src/monitoreo_pipeline/__main__.py`

```python
"""Permite ejecutar el paquete con: python -m monitoreo_pipeline."""

from monitoreo_pipeline.cli import main

raise SystemExit(main())
```

El DAG de referencia configura en Airflow los mismos controles que el vigilante. No se ejecuta en el laboratorio, porque requiere Airflow:

**Archivo:** `dags/ventas_diarias.py`

```python
"""DAG de REFERENCIA para Amazon MWAA (no se ejecuta en el laboratorio).

Los mismos controles que el vigilante local, configurados en Airflow:
reintentos de la tarea, timeout de ejecución y callback de fallo. Solo se
comprueba que compila: python -m py_compile dags/ventas_diarias.py
"""

from datetime import datetime, timedelta

from airflow import DAG

try:  # Airflow 3
    from airflow.providers.standard.operators.python import PythonOperator
except ImportError:  # Airflow 2
    from airflow.operators.python import PythonOperator


def avisar_fallo(context):
    """Se ejecuta cuando la tarea falla definitivamente (tras agotar sus reintentos)."""
    ti = context["task_instance"]
    print(
        {
            "evento": "task_failed",
            "dag": ti.dag_id,
            "tarea": ti.task_id,
            "intento": ti.try_number,
            "run_id": context["run_id"],
            "error": repr(context.get("exception")),
        }
    )  # JSON en CloudWatch Logs


def ejecutar_tarea():
    from monitoreo_pipeline.tarea import main

    codigo = main(["--escenario", "normal", "--execution-id", "airflow"])
    if codigo != 0:
        raise RuntimeError(f"la tarea terminó con código {codigo}")


with DAG(
    dag_id="ventas_diarias",
    start_date=datetime(2026, 1, 1),
    schedule="0 6 * * *",
    catchup=False,
    default_args={
        "retries": 1,  # [tarea] reintentos
        "retry_delay": timedelta(minutes=2),
        "retry_exponential_backoff": True,
        "execution_timeout": timedelta(minutes=30),  # [tarea] timeout_s
        "on_failure_callback": avisar_fallo,
    },
) as dag:
    PythonOperator(task_id="cargar_ventas", python_callable=ejecutar_tarea)
```

Y el script para el ejercicio de auditoría del paso 6.10:

**Archivo:** `scripts/alterar_auditoria.py`

```python
"""Simula que alguien edita a mano la auditoría para ocultar una decisión (6.10)."""

import json
from pathlib import Path

RUTA = Path("salida/auditoria.jsonl")


def main() -> None:
    lineas = RUTA.read_text(encoding="utf-8").splitlines()
    for numero, linea in enumerate(lineas, start=1):
        evento = json.loads(linea)
        if evento["accion"].startswith("decidir:"):  # la primera decisión tomada
            anterior = evento["resultado"]
            evento["resultado"] = "SIN ACCIÓN: ejecución correcta"  # se maquilla
            lineas[numero - 1] = json.dumps(evento, ensure_ascii=False)
            RUTA.write_text("\n".join(lineas) + "\n", encoding="utf-8")
            print(
                f"Evento {numero} ({evento['accion']}) modificado: "
                f"'{anterior}' -> '{evento['resultado']}'"
            )
            return
    print("No hay decisiones que alterar: ejecuta antes un escenario con fallo")


if __name__ == "__main__":
    main()
```

### Códigos de salida de `ejecutar`

| Código | Significado |
| :---: | :--- |
| 0 | `SUCCESS`: la tarea cargó datos |
| 1 | Configuración inválida |
| 2 | Argumentos de línea de comandos inválidos |
| 3 | `FAILED`: la tarea falló y no se recuperó |
| 4 | `SUSPECT`: la tarea terminó sin datos |

Para ver el código de salida del último comando: **PowerShell** `$LASTEXITCODE` | **bash** `echo $?`

Ejecuta los escenarios **en este orden**. Anota en cada uno la señal, la decisión y el código de salida.

### 6.2 Escenario normal: se recupera solo

```bash
python -m monitoreo_pipeline ejecutar --escenario normal
```

```text
[R-88be689d] escenario=normal
  intento 1: SUCCESS filas=18  exit=0  (1.37 s)
  métricas CloudWatch: FilasCargadas=18 RegistrosEnDLQ=2 IntentosTarea=1 EjecucionFallida=0 (validadas con Stubber)
  estado final: SUCCESS  | log: salida/logs/R-88be689d_intento1.jsonl
```

El 503 y el 429 se resolvieron **dentro** de la tarea: el vigilante solo ve un éxito.

### 6.3 Token caducado: acción correctiva y un reintento

```bash
python -m monitoreo_pipeline ejecutar --escenario token_caducado
```

Resultado esperado: `SUCCESS` con `filas=18`. Abre el log que indica la última línea. Verás `HTTP 401: se renueva el token…` y después `Authorization: Bearer [TOKEN]`: el token nuevo nunca llega al log.

### 6.4 Error permanente: detener y alertar, sin ruido

```bash
python -m monitoreo_pipeline ejecutar --escenario permanente
```

```text
[R-02d58cac] escenario=permanente
  intento 1: FAILED  señal=HTTP 404  exit=3  (0.27 s)
  diagnóstico: HTTPStatusError en fuente.py:65 en obtener_ventas()
  decisión: DETENER - error permanente, requiere corrección
  alerta CRITICAL: enviada (firma 5a6b59f52892)
  métricas CloudWatch: FilasCargadas=0 RegistrosEnDLQ=0 IntentosTarea=1 EjecucionFallida=1 (validadas con Stubber)
  estado final: FAILED  | log: salida/logs/R-02d58cac_intento1.jsonl
```

Mira la alerta enriquecida que recibiría la guardia:

```bash
python -c "import json; print(json.dumps(json.loads(open('salida/alertas.jsonl', encoding='utf-8').readlines()[-1]), indent=2, ensure_ascii=False))"
```

```json
{
  "alerta_id": "f5743bc3-5cac-4b78-9b0e-27a7971ebc1b",
  "servicio": "ventas_diarias",
  "severidad": "CRITICAL",
  "senal": "HTTP 404",
  "impacto": "El panel de ventas no tiene los datos de hoy",
  "error_tipo": "HTTPStatusError",
  "error_mensaje": "HTTP 404",
  "ubicacion": "fuente.py:65 en obtener_ventas()",
  "entorno": "desarrollo",
  "version": "1.0.0",
  "host": "…",
  "execution_id": "R-02d58cac",
  "enlaces": {
    "runbook": "https://wiki.empresa.test/runbooks/rb-002-recurso-inexistente",
    "logs": "https://logs.empresa.test/search?execution_id=R-02d58cac&desde=…"
  },
  "firma": "5a6b59f52892"
}
```

Ahora **repite** el mismo comando:

```bash
python -m monitoreo_pipeline ejecutar --escenario permanente
```

La línea de la alerta cambia a `alerta CRITICAL: suprimida (duplicada, firma 5a6b59f52892)`. Es el mismo problema: la guardia ya está avisada.

### 6.5 Transitorio persistente: reintento por capas, con límite

```bash
python -m monitoreo_pipeline ejecutar --escenario transitorio_persistente
```

```text
  intento 1: FAILED  señal=HTTP 503  exit=3  (2.16 s)
  diagnóstico: HTTPStatusError en fuente.py:65 en obtener_ventas()
  decisión: REINTENTAR - intento 2 de 2
  intento 2: FAILED  señal=HTTP 503  exit=3  (3.15 s)
  diagnóstico: HTTPStatusError en fuente.py:65 en obtener_ventas()
  decisión: ESCALAR - transitorio persistente, intentos agotados
  alerta CRITICAL: enviada (firma 70fb701c55ac)
```

Cada intento de la tarea hizo 4 llamadas, así que fueron 8 llamadas en total: el presupuesto calculado en la Parte 1. Código de salida `3`.

### 6.6 Tarea colgada: timeout, limpieza y escalado

```bash
python -m monitoreo_pipeline ejecutar --escenario colgado
```

Tarda unos 8 s: dos intentos de 4 s.

```text
  intento 1: FAILED  señal=TIMEOUT  exit=None  (4.01 s)
  decisión: REINTENTAR - intento 2 de 2
  remediación limpiar_temporales: DRY-RUN: 1 cambio(s) previsto(s) ['extract_R-0b19a1e8.tmp']
  remediación limpiar_temporales: OK: verificada
  intento 2: FAILED  señal=TIMEOUT  exit=None  (4.0 s)
  decisión: ESCALAR - transitorio persistente, intentos agotados
  remediación limpiar_temporales: DRY-RUN: 1 cambio(s) previsto(s) ['extract_R-0b19a1e8.tmp']
  remediación limpiar_temporales: OK: verificada
  alerta CRITICAL: enviada (firma ebe09264a9b0)
```

`exit=None`: el proceso no terminó, lo cortó el vigilante, y por eso no hay diagnóstico. El temporal que dejó se limpia **antes** de reintentar: primero en dry-run y después de verdad, con verificación.

### 6.7 Éxito vacío: pausar el DAG consumidor

```bash
python -m monitoreo_pipeline ejecutar --escenario vacio
```

```text
  intento 1: SUSPECT filas=0  exit=0  (0.21 s)
  decisión: ESCALAR - terminó sin datos (éxito vacío), revisión humana
  remediación pausar_dag_consumidor: DRY-RUN: 1 cambio(s) previsto(s) ['PATCH /dags/carga_ventas_snowflake is_paused=true']
  remediación pausar_dag_consumidor: OK: verificada
  alerta WARNING: enviada (firma 555d865deb6b)
```

La tarea terminó con código `0`, pero el vigilante detectó la cuarta forma de fallar: terminar **bien** y sin datos. Para que el dato incompleto no siga aguas abajo, la remediación pausa el DAG consumidor con la API de MWAA. Código de salida `4`.

### 6.8 Datos corruptos: el reproceso requiere aprobación humana

```bash
python -m monitoreo_pipeline ejecutar --escenario datos_corruptos
```

```text
  intento 1: FAILED  señal=DATOS_INVALIDOS  exit=5  (0.22 s)
  diagnóstico: LoteRechazadoError en tarea.py:141 en procesar_lote()
  decisión: DETENER - error permanente, requiere corrección
  remediación reprocesar_lote: PENDIENTE: requiere aprobación humana
  solicitud de aprobación: token=zwohj9Eiqgs (caduca en 30 min)
  alerta CRITICAL: enviada (firma c0aecc505a88)
  métricas CloudWatch: FilasCargadas=0 RegistrosEnDLQ=10 IntentosTarea=1 EjecucionFallida=1 (validadas con Stubber)
```

Copia **tu** token y úsalo en los siguientes comandos (aquí se muestra como `TOKEN`):

```bash
python -m monitoreo_pipeline ejecutar-aprobada TOKEN
python -m monitoreo_pipeline aprobar TOKEN --aprobador controlador_remediacion
python -m monitoreo_pipeline aprobar TOKEN --aprobador guardia.ana
python -m monitoreo_pipeline ejecutar-aprobada TOKEN
python -m monitoreo_pipeline ejecutar-aprobada TOKEN
```

Resultado esperado:

```text
No se ejecuta: la solicitud está PENDIENTE
Solicitud TOKEN: rechazado: nadie aprueba su propia solicitud
Solicitud TOKEN: APROBADA
[R-87be9531] escenario=normal
  intento 1: SUCCESS filas=18  exit=0  (1.41 s)
  métricas CloudWatch: FilasCargadas=18 RegistrosEnDLQ=2 IntentosTarea=1 EjecucionFallida=0 (validadas con Stubber)
  estado final: SUCCESS  | log: salida/logs/R-87be9531_intento1.jsonl
remediación reprocesar_lote: OK: verificada
No se ejecuta: la solicitud está EJECUTADA
```

Así funciona la aprobación:

- **Sin aprobación** no se ejecuta nada.
- **La autoaprobación** se rechaza.
- **Después de aprobar**, el reproceso se ejecuta y se verifica.
- **El token** sirve una sola vez.

### 6.9 Pedir acciones a mano

```bash
python -m monitoreo_pipeline remediar reiniciar_servidor
python -m monitoreo_pipeline remediar limpiar_temporales
python -m monitoreo_pipeline remediar pausar_dag_consumidor
```

```text
remediación reiniciar_servidor: RECHAZADA: 'reiniciar_servidor' no está en el catálogo
remediación limpiar_temporales: OMITIDA: la condición que la justifica ya no se cumple
remediación pausar_dag_consumidor: DRY-RUN: 1 cambio(s) previsto(s) ['PATCH /dags/carga_ventas_snowflake is_paused=true']
```

Sin `--ejecutar`, `remediar` solo hace el dry-run. Una persona tampoco puede saltarse el catálogo.

### 6.10 Panel operativo y auditoría

```bash
python -m monitoreo_pipeline panel
```

```text
Ejecuciones (más recientes al final):
  R-88be689d  normal                   SUCCESS  señal=-                decisión=SIN_ACCION  intentos=1
  R-d8ff05c0  token_caducado           SUCCESS  señal=-                decisión=SIN_ACCION  intentos=1
  R-02d58cac  permanente               FAILED   señal=HTTP 404         decisión=DETENER     intentos=1
  R-947f2119  permanente               FAILED   señal=HTTP 404         decisión=DETENER     intentos=1
  R-e02bc231  transitorio_persistente  FAILED   señal=HTTP 503         decisión=ESCALAR     intentos=2
  R-0b19a1e8  colgado                  FAILED   señal=TIMEOUT          decisión=ESCALAR     intentos=2
  R-de74cf79  vacio                    SUSPECT  señal=SUSPECT          decisión=ESCALAR     intentos=1
  R-989a537f  datos_corruptos          FAILED   señal=DATOS_INVALIDOS  decisión=DETENER     intentos=1
  R-87be9531  normal                   SUCCESS  señal=-                decisión=SIN_ACCION  intentos=1
Solicitudes de aprobación:
  zwohj9Eiqgs  reprocesar_lote      EJECUTADA
Auditoría: íntegro (31 eventos)
```

Ahora alguien intenta ocultar una decisión editando la auditoría a mano:

```bash
python scripts/alterar_auditoria.py
python -m monitoreo_pipeline panel
```

```text
Evento 3 (decidir:DETENER) modificado: 'error permanente, requiere corrección' -> 'SIN ACCIÓN: ejecución correcta'
…
Auditoría: ALTERADO en el evento 3
```

La firma HMAC encadenada delata exactamente qué evento se tocó.

### 6.11 Preguntas de cierre

1. **Seguridad.** Busca `tok-lab-7f3a9c21`, `clave-auditoria-lab-2026` y `@empresa.test` en `salida/logs/` y en `salida/alertas.jsonl`. ¿Cuántas veces aparecen? ¿Y los correos en `salida/dlq_ventas.jsonl`? ¿Por qué esa diferencia es correcta?
2. **Mantenibilidad.** Sin tocar código, haz que la tarea colgada se reintente dos veces en lugar de una. ¿Qué línea de `config/politica.toml` cambiaste? ¿Cuántas llamadas a la API serían ahora el peor caso del escenario `transitorio_persistente`?
3. **Idempotencia.** Si en `politica.toml` pusieras `idempotente = false`, ¿qué decidiría el controlador en el escenario `transitorio_persistente`? Compruébalo y devuelve el valor a `true`.
4. **MWAA.** Compara `dags/ventas_diarias.py` con `config/politica.toml`. ¿Qué parámetro de Airflow cumple el papel de `timeout_s`? ¿Y el de `reintentos`?

---

## Parte 7. Pruebas y calidad del código (5 min)

Las pruebas trabajan en una carpeta temporal con una política sin esperas. Tu carpeta `salida/` no se toca.

**7.1** Crea `tests/__init__.py` y `tests/conftest.py`:

**Archivo:** `tests/__init__.py`

```python
"""Pruebas de la Práctica 4."""
```

**Archivo:** `tests/conftest.py`

```python
"""Cada prueba trabaja en una carpeta temporal con una política rápida."""

from pathlib import Path

import pytest

from monitoreo_pipeline import configuracion
from monitoreo_pipeline.configuracion import Ajustes, Politica

POLITICA = Path(__file__).parents[1] / "config" / "politica.toml"


@pytest.fixture
def proyecto(
    tmp_path: Path, monkeypatch: pytest.MonkeyPatch
) -> tuple[Ajustes, Politica]:
    rapida = (
        POLITICA.read_text(encoding="utf-8")
        .replace("multiplicador = 0.5", "multiplicador = 0")  # sin esperas
        .replace("timeout_s = 4", "timeout_s = 10")
    )
    (tmp_path / "config").mkdir()
    (tmp_path / "config" / "politica.toml").write_text(rapida, encoding="utf-8")
    monkeypatch.chdir(tmp_path)
    # Variables de entorno: también las heredan las tareas en subprocesos
    monkeypatch.setenv("API_TOKEN", "tok-pruebas-123")
    monkeypatch.setenv("AUDIT_HMAC_KEY", "clave-de-pruebas-larga")
    return configuracion.cargar()
```

**7.2** Crea las pruebas:

**Archivo:** `tests/test_clasificacion.py`

```python
import pytest

from monitoreo_pipeline.clasificacion import Decision, Fallo, Ruta, clasificar, decidir
from monitoreo_pipeline.errores import HTTPStatusError, LoteRechazadoError


@pytest.mark.parametrize(
    ("error", "ruta"),
    [
        (HTTPStatusError(503), Ruta.REINTENTAR),
        (HTTPStatusError(429, retry_after=1), Ruta.REINTENTAR),
        (HTTPStatusError(401), Ruta.CORREGIR_Y_REINTENTAR),
        (HTTPStatusError(404), Ruta.DETENER),
        (TimeoutError("lectura"), Ruta.REINTENTAR),
        (ValueError("fecha '31/02'"), Ruta.CUARENTENA),
        (LoteRechazadoError("50 % inválido"), Ruta.CUARENTENA),
        (TypeError("NoneType"), Ruta.DETENER),
    ],
)
def test_clasificar(error: Exception, ruta: Ruta) -> None:
    assert clasificar(error) is ruta


@pytest.mark.parametrize(
    ("fallo", "esperada"),
    [
        (Fallo("HTTP 503", True, 1, 2), Decision.REINTENTAR),
        (Fallo("HTTP 503", True, 2, 2), Decision.ESCALAR),  # intentos agotados
        (Fallo("TIMEOUT", False, 1, 2), Decision.ESCALAR),  # no idempotente
        (Fallo("HTTP 404", True, 1, 2), Decision.DETENER),
        (Fallo("DATOS_INVALIDOS", True, 1, 2), Decision.DETENER),
        (Fallo("SUSPECT", True, 1, 2), Decision.ESCALAR),
        (Fallo("MemoryError", True, 1, 2), Decision.ESCALAR),  # desconocida
    ],
)
def test_decidir(fallo: Fallo, esperada: Decision) -> None:
    assert decidir(fallo)[0] is esperada
```

**Archivo:** `tests/test_registro.py`

```python
import io
import json
import logging

from monitoreo_pipeline.registro import configurar_logging, contexto_ejecucion


def test_logs_json_sanitizados_y_con_execution_id() -> None:
    salida = io.StringIO()
    configurar_logging(salida)
    contexto_ejecucion.set({"execution_id": "R-prueba", "intento": 1})
    log = logging.getLogger("prueba")
    log.info("Usuario %s inició la carga", "ana.perez@empresa.com")
    log.warning("Cabecera: Authorization: Bearer eyJhbGciOi.abc123")
    log.info(
        "Config", extra={"api_key": "sk-123", "db": {"password": "x", "host": "h"}}
    )
    log.info("Pago con tarjeta 4111 1111 1111 1234 aprobado")
    eventos = [json.loads(linea) for linea in salida.getvalue().splitlines()]
    assert eventos[0]["mensaje"] == "Usuario [EMAIL] inició la carga"
    assert eventos[1]["mensaje"].endswith("Bearer [TOKEN]")
    assert eventos[2]["api_key"] == "[REDACTADO]"
    assert eventos[2]["db"] == {"password": "[REDACTADO]", "host": "h"}
    assert eventos[3]["mensaje"] == "Pago con tarjeta ****-****-****-1234 aprobado"
    assert all(e["execution_id"] == "R-prueba" for e in eventos)
    assert "sk-123" not in salida.getvalue()
```

**Archivo:** `tests/test_tarea.py`

```python
import json
import sqlite3
from pathlib import Path

import pytest

from monitoreo_pipeline.configuracion import PoliticaReintento
from monitoreo_pipeline.errores import HTTPStatusError, LoteRechazadoError
from monitoreo_pipeline.fuente import APISimulada, generar_ventas
from monitoreo_pipeline.tarea import cargar, construir_politica, extraer, procesar_lote

SIN_ESPERA = PoliticaReintento(
    max_intentos=4, max_segundos=20, multiplicador=0, espera_maxima=0
)


def test_transitorios_se_reintentan_y_respetan_retry_after() -> None:
    api = APISimulada("normal", "tok")
    ventas = extraer(api, construir_politica(SIN_ESPERA), "tok")  # 503, 429, OK
    assert len(ventas) == 20
    assert api.llamadas == 3


def test_401_renueva_el_token_y_reintenta_una_vez() -> None:
    api = APISimulada("token_caducado", "tok")
    assert len(extraer(api, construir_politica(SIN_ESPERA), "tok")) == 20
    assert api.llamadas == 2


def test_permanente_sale_al_primer_intento() -> None:
    api = APISimulada("permanente", "tok")
    with pytest.raises(HTTPStatusError, match="HTTP 404"):
        extraer(api, construir_politica(SIN_ESPERA), "tok")
    assert api.llamadas == 1


def test_transitorio_persistente_agota_los_intentos() -> None:
    api = APISimulada("transitorio_persistente", "tok")
    with pytest.raises(HTTPStatusError, match="HTTP 503"):  # error original
        extraer(api, construir_politica(SIN_ESPERA), "tok")
    assert api.llamadas == 4


def test_dlq_conserva_el_payload_y_el_umbral_detiene(tmp_path: Path) -> None:
    dlq = tmp_path / "dlq.jsonl"
    normal = generar_ventas("normal")
    lote = procesar_lote(normal, dlq, umbral=0.2)
    assert (len(lote.exitos), len(lote.fallos)) == (18, 2)
    registro = json.loads(dlq.read_text(encoding="utf-8").splitlines()[0])
    assert registro["payload"]["cliente_email"] == "cliente4@empresa.test"  # intacto
    corruptos = generar_ventas("datos_corruptos")
    with pytest.raises(LoteRechazadoError, match="50%"):
        procesar_lote(corruptos, dlq, umbral=0.2)


def test_carga_upsert_idempotente(tmp_path: Path) -> None:
    bd = tmp_path / "dw.db"
    filas = [{"id": "V-1", "importe": "10", "fecha": "2026-10-01"}]
    cargar(bd, filas)
    cargar(bd, filas)  # el reintento de la tarea no duplica
    with sqlite3.connect(bd) as conn:
        assert conn.execute("SELECT COUNT(*) FROM ventas").fetchone()[0] == 1
```

**Archivo:** `tests/test_remediacion.py`

```python
from pathlib import Path

from monitoreo_pipeline.almacen import Almacen
from monitoreo_pipeline.auditoria import Auditoria
from monitoreo_pipeline.remediacion import Accion, MotorRemediacion, PuertaAprobacion


class Reloj:
    def __init__(self) -> None:
        self.ahora = 1_000_000.0

    def __call__(self) -> float:
        return self.ahora


def test_controles_de_la_remediacion(tmp_path: Path) -> None:
    pendientes = ["a.tmp", "b.tmp"]

    def limpiar(dry_run: bool) -> list[str]:
        cambios = list(pendientes)
        if not dry_run:
            pendientes.clear()
        return cambios

    catalogo = {
        "limpiar": Accion(
            "bajo", lambda: bool(pendientes), limpiar, lambda: not pendientes
        ),
        "borrar_historia": Accion("alto", lambda: True, lambda d: [], lambda: True),
    }
    motor = MotorRemediacion(catalogo, Almacen(tmp_path / "op.db"), max_por_hora=1)
    assert motor.remediar("reiniciar_servidor").startswith("RECHAZADA")
    assert motor.remediar("borrar_historia").startswith("PENDIENTE")
    assert (
        motor.remediar("limpiar")
        == "DRY-RUN: 2 cambio(s) previsto(s) ['a.tmp', 'b.tmp']"
    )
    assert motor.remediar("limpiar", dry_run=False) == "OK: verificada"
    pendientes.append("c.tmp")
    assert motor.remediar("limpiar", dry_run=False).startswith("BLOQUEADA")  # 1/hora


def test_precondicion_que_ya_no_se_cumple(tmp_path: Path) -> None:
    catalogo = {"limpiar": Accion("bajo", lambda: False, lambda d: [], lambda: True)}
    motor = MotorRemediacion(catalogo, Almacen(tmp_path / "op.db"))
    assert motor.remediar("limpiar").startswith("OMITIDA")


def test_aprobacion_humana_con_caducidad(tmp_path: Path) -> None:
    reloj = Reloj()
    puerta = PuertaAprobacion(Almacen(tmp_path / "op.db"), ttl_s=1800, reloj=reloj)
    token = puerta.solicitar("reprocesar_lote", {"lote": "L-1"}, "controlador")
    assert puerta.resolver(token, "controlador", True).startswith("rechazado")
    assert not puerta.puede_ejecutar(token)
    assert puerta.resolver(token, "guardia.ana", True) == "APROBADA"
    assert puerta.puede_ejecutar(token)
    tardia = puerta.solicitar("reprocesar_lote", {}, "controlador")
    reloj.ahora += 1801  # nadie respondió a tiempo
    assert puerta.resolver(tardia, "guardia.ana", True) == "EXPIRADA"
    assert not puerta.puede_ejecutar(tardia)


def test_auditoria_detecta_alteraciones(tmp_path: Path) -> None:
    ruta = tmp_path / "auditoria.jsonl"
    auditoria = Auditoria(ruta, b"clave-de-pruebas-larga")
    for n in range(3):
        auditoria.registrar("sistema", f"accion_{n}", "dag:x", "OK", "R-1")
    assert auditoria.verificar() == "íntegro (3 eventos)"
    lineas = ruta.read_text(encoding="utf-8").splitlines()
    lineas[1] = lineas[1].replace('"OK"', '"FALLO"')
    ruta.write_text("\n".join(lineas) + "\n", encoding="utf-8")
    assert auditoria.verificar() == "ALTERADO en el evento 2"
```

**Archivo:** `tests/test_aws.py`

```python
from pathlib import Path

import pytest
from botocore.exceptions import ParamValidationError

from monitoreo_pipeline import aws


def test_put_metric_data_valida_contra_el_modelo(tmp_path: Path) -> None:
    registro = tmp_path / "metricas.jsonl"
    peticion = aws.publicar_metricas(
        {"FilasCargadas": 18}, 1.2, True, {"Pipeline": "p", "Entorno": "e"}, registro
    )
    nombres = [m["MetricName"] for m in peticion["MetricData"]]
    assert nombres == ["FilasCargadas", "DuracionSegundos", "EjecucionFallida"]
    with pytest.raises(
        ParamValidationError
    ):  # dimensión sin nombre: no cumple el modelo
        aws.publicar_metricas({"FilasCargadas": 18}, 1.2, True, {"": "e"}, registro)
    assert len(registro.read_text(encoding="utf-8").splitlines()) == 1  # no se registró


def test_pausar_dag_valida_el_identificador() -> None:
    assert aws.pausar_dag("mwaa-lab", "carga_ventas")["is_paused"] is True
    with pytest.raises(ValueError, match="dag_id inválido"):
        aws.pausar_dag("mwaa-lab", "../variables")
```

**Archivo:** `tests/test_controlador.py`

```python
"""Pruebas de extremo a extremo: la tarea corre en un subproceso, como en MWAA."""

import json
from pathlib import Path

from monitoreo_pipeline.configuracion import Ajustes, Politica
from monitoreo_pipeline.controlador import Controlador
from monitoreo_pipeline.vigilante import ejecutar_vigilado, latido_vencido


def test_fallo_permanente_alerta_una_sola_vez(
    proyecto: tuple[Ajustes, Politica],
) -> None:
    mensajes: list[str] = []
    controlador = Controlador(*proyecto, mostrar=mensajes.append)
    assert controlador.ejecutar("permanente") == "FAILED"
    assert controlador.ejecutar("permanente") == "FAILED"
    alertas = Path("salida/alertas.jsonl").read_text(encoding="utf-8").splitlines()
    assert len(alertas) == 1  # la segunda se suprime por duplicada
    alerta = json.loads(alertas[0])
    assert alerta["enlaces"]["runbook"].endswith("rb-002-recurso-inexistente")
    assert any("suprimida (duplicada" in m for m in mensajes)
    assert "tok-pruebas-123" not in Path("salida/logs").joinpath(
        next(p.name for p in Path("salida/logs").iterdir())
    ).read_text(encoding="utf-8")


def test_exito_vacio_pausa_el_dag_consumidor(
    proyecto: tuple[Ajustes, Politica],
) -> None:
    mensajes: list[str] = []
    assert (
        Controlador(*proyecto, mostrar=mensajes.append).ejecutar("vacio") == "SUSPECT"
    )
    assert "  remediación pausar_dag_consumidor: OK: verificada" in mensajes


def test_reproceso_solo_con_aprobacion(proyecto: tuple[Ajustes, Politica]) -> None:
    mensajes: list[str] = []
    controlador = Controlador(*proyecto, mostrar=mensajes.append)
    assert controlador.ejecutar("datos_corruptos") == "FAILED"
    token = next(m for m in mensajes if "token=" in m).split("token=")[1].split()[0]
    assert controlador.ejecutar_aprobada(token) == "NO_EJECUTADA"
    assert controlador.resolver(token, "guardia.ana", aprobar=True) == "APROBADA"
    assert controlador.ejecutar_aprobada(token) == "OK: verificada"
    assert controlador.ejecutar_aprobada(token) == "NO_EJECUTADA"  # un solo uso
    assert controlador.auditoria.verificar().startswith("íntegro")


def test_tarea_colgada_se_corta_por_timeout(proyecto: tuple[Ajustes, Politica]) -> None:
    resultado = ejecutar_vigilado("colgado", "R-colgado", 1, 2, Path("salida/logs"))
    assert (resultado.estado, resultado.senal) == ("FAILED", "TIMEOUT")
    assert list(Path("salida/staging").glob("*.tmp"))  # el temporal quedó


def test_latido_vencido() -> None:
    assert not latido_vencido(ultimo_latido=1000, ahora=1000 + 120)  # 2 min: normal
    assert latido_vencido(ultimo_latido=1000, ahora=1000 + 540)  # 9 min: caído
```

**7.3** Ejecuta:

```bash
python -m pytest -q
ruff check .
python -m mypy src tests scripts
```

Resultado esperado: `33 passed` (en unos 6 s), `All checks passed!` y `Success: no issues found in 26 source files`. Comprueba también que el DAG de referencia compila:

```bash
python -m py_compile dags/ventas_diarias.py
```

Si no aparece ningún mensaje, compila.

---

## Entregables

Comprime `practica4_monitoreo` **sin** `.venv/`, `.env` ni `salida/`. Antes de entregar, revisa:

- [ ] Paquete `src/monitoreo_pipeline/` con 17 archivos `.py`
- [ ] `config/politica.toml` y `.env.example` versionados; `.env` **no** incluido
- [ ] Los siete escenarios con su señal, decisión y código de salida anotados
- [ ] Alerta duplicada suprimida en 6.4
- [ ] Reproceso ejecutado solo después de la aprobación de otra persona (6.8)
- [ ] Auditoría `íntegro` y, tras el ejercicio 6.10, `ALTERADO en el evento 3`
- [ ] Respuestas a las preguntas de cierre
- [ ] `pytest`, `ruff` y `mypy` sin errores

---

## Del laboratorio a Amazon MWAA

| En el laboratorio | En producción con AWS |
| :--- | :--- |
| `ejecutar_vigilado()` con `subprocess.run(timeout=…)` | Airflow en MWAA: `retries`, `execution_timeout`, `on_failure_callback` (ver `dags/ventas_diarias.py`) |
| `salida/logs/*.jsonl` | CloudWatch Logs, consultado con Logs Insights filtrando por `execution_id` |
| `publicar_metricas()` con Stubber | El mismo `put_metric_data`, sin Stubber, con un rol de IAM que lo permita |
| `pausar_dag()` con Stubber | El mismo `invoke_rest_api` contra el entorno de MWAA |
| `salida/alertas.jsonl` | Webhook de Slack o Teams, o una alarma de CloudWatch hacia Amazon SNS |
| `operacion.db` (SQLite) | Una tabla en PostgreSQL o DynamoDB con escrituras condicionales |
| `auditoria.jsonl` | Almacenamiento con retención larga e inmutable, como S3 Object Lock |
| `.env` | AWS Secrets Manager o las variables del entorno de MWAA |

---

## Solución de problemas

| Síntoma | Causa probable | Solución |
| :--- | :--- | :--- |
| `ModuleNotFoundError: No module named 'monitoreo_pipeline'` | El paquete no está instalado en el entorno activo | Activa `.venv` y ejecuta `python -m pip install -e .` desde la raíz |
| `Configuración inválida: api_token: Field required` | No existe `.env` o no estás en la raíz del proyecto | Revisa el paso 1.2 y ejecuta desde `practica4_monitoreo` |
| `Configuración inválida: audit_hmac_key: String should have at least 16 characters` | La clave de auditoría es demasiado corta | Usa la del paso 1.2.3 |
| `No such file or directory: 'config/politica.toml'` | No estás en la raíz del proyecto | Cambia a la carpeta `practica4_monitoreo` |
| Una alerta sale `suprimida (duplicada…)` cuando esperabas `enviada` | Repetiste el escenario en menos de 5 minutos | Es el comportamiento esperado; para empezar de cero borra la carpeta `salida` |
| `remediar pausar_dag_consumidor --ejecutar` responde `BLOQUEADA` | Se alcanzó el límite de 3 ejecuciones por hora | Es el control del tema 4.27; espera o borra `salida/operacion.db` |
| `Auditoría: ALTERADO…` en una ejecución nueva | Hiciste el ejercicio 6.10 y la cadena sigue rota | Es lo esperado; para empezar de cero borra la carpeta `salida` |
| En el escenario `colgado` un intento tarda más de 4 s (hasta unos 10 s) | En algunos equipos el sistema tarda en liberar la salida del proceso cortado | No afecta el resultado: la señal sigue siendo `TIMEOUT` |
| PowerShell no deja activar `.venv` | Política de ejecución de Windows | `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` |

---

## Limpieza

**PowerShell**
```powershell
Remove-Item -Recurse -Force salida -ErrorAction SilentlyContinue
deactivate
```

**bash**
```bash
rm -rf salida
deactivate
```

---

## Resumen

| Lo que hiciste | Lección |
| :--- | :--- |
| Diseñaste una jerarquía de excepciones con categorías que dicen la acción | 4.1 |
| Clasificaste cada error en reintentar, corregir y reintentar, aislar o detener | 4.2 |
| Reintentaste solo lo transitorio, con Full Jitter, `Retry-After` y dos límites | 4.3, 4.25 |
| Aislaste registros inválidos en una DLQ con el payload intacto | 4.4 |
| Detuviste el lote cuando la tasa de fallos superó el umbral | 4.5 |
| Repetiste la tarea sin duplicar gracias al UPSERT | 4.9 |
| Emitiste logs JSON con `execution_id` y sin datos sensibles | 4.10, 4.18, 4.21 |
| Publicaste métricas validadas contra el modelo de CloudWatch | 4.11 |
| Evitaste alertas duplicadas y en ráfagas | 4.14 |
| Persististe el estado de ejecuciones, alertas, remediaciones y aprobaciones | 4.15 |
| Consultaste todo en un panel operativo | 4.17 |
| Recolectaste el diagnóstico en el punto exacto del fallo, sanitizado | 4.19, 4.23 |
| Detectaste fallos por código de salida, timeout, volumen y latido | 4.22 |
| Decidiste según transitorio, idempotente e intentos | 4.24 |
| Enviaste alertas con impacto, ubicación, runbook y enlace a los logs | 4.26 |
| Remediaste con catálogo, límite, precondición, dry-run y verificación | 4.8, 4.27 |
| Exigiste aprobación de otra persona, con caducidad, para reprocesar | 4.28 |
| Detectaste una edición manual de la auditoría | 4.16, 4.29 |
| Cerraste el ciclo detectar → decidir → actuar → registrar | 4.30, 4.32 |
| Pausaste un DAG con la API de MWAA | 4.31 |
