# Práctica 1. De un proceso manual a una automatización mantenible

| Dato | Detalle |
| :--- | :--- |
| **Curso** | Automatización de procesos con Python |
| **Capítulo** | 1. Diseño de automatizaciones y estándares de ingeniería Python |
| **Duración** | 90 minutos |
| **Modalidad** | Individual |
| **Nivel de Bloom** | Aplicar / Analizar |

---

## Objetivo

Analizar un proceso repetitivo relacionado con datos y transformarlo en la base de una automatización Python estructurada y reutilizable.

Primero vas a identificar qué conviene automatizar, definir el alcance, los modos de fallo y los puntos que requieren aprobación humana. Después construirás una solución modular con anotaciones de tipo, configuración externa, variables de entorno, gestión de dependencias y documentación.

## Lo que vas a construir

Una automatización llamada **compras-auto** que reemplaza una revisión manual diaria. La automatización:

1. Valida el archivo de entrada antes de procesarlo (precondiciones).
2. Lee el archivo registro por registro con un generador.
3. Aplica reglas de negocio y clasifica cada solicitud.
4. Envía a aprobación humana las compras que superan el límite.
5. Separa en cuarentena los registros inválidos, sin intentar corregirlos.
6. Comprueba al final que ningún registro se perdió (reconciliación).

```text
                    datos/solicitudes_AAAA-MM-DD.csv
                                │
           ┌────────────────────┘
           ▼
   ┌────────────────┐   ┌────────────────┐   ┌──────────────────────────┐
   │ Ingesta        │──►│ Dominio        │──►│ Salida y notificación    │
   │ lector.py      │   │ reglas.py      │   │ salida.py                │
   └────────────────┘   └────────────────┘   └──────────────────────────┘
           ▲                    ▲                         ▲
           └────────────────────┴─────────────────────────┘
                    Orquestación: orquestador.py + cli.py
                 (recibe la configuración y los componentes)

   Salidas: aprobadas_automaticamente.csv
            pendientes_aprobacion.csv   ← requieren aprobación humana (HITL)
            rechazadas.csv              ← cuarentena para revisión
```

## Criterios de evaluación

| Criterio | Qué se revisa en tu entrega |
| :--- | :--- |
| **Corrección** | La automatización hace lo que definiste en `docs/analisis.md`; 11 registros leídos = 2 + 2 + 7 |
| **Mantenibilidad** | Estructura `src/`, módulos con una sola responsabilidad, tipos, docstrings, `ruff` y `mypy` sin errores |
| **Seguridad** | El token no está en el código, no se imprime y `.env` está en `.gitignore` |
| **Valor operativo** | El proceso elimina la revisión manual y explica cada rechazo |

---

## Requisitos

| Elemento | Requisito |
| :--- | :--- |
| Python | **3.11 o superior** (se usa `tomllib`, que forma parte de la biblioteca estándar desde 3.11) |
| Editor | Visual Studio Code con la extensión de Python |
| Terminal | La terminal integrada de VS Code (PowerShell en Windows; bash/zsh en macOS o Linux) |
| Red | Acceso a PyPI para instalar paquetes con `pip` |

> **Convenciones de esta guía**
> - Todos los comandos se ejecutan desde la **carpeta raíz del proyecto** (`practica1_compras`).
> - Cuando un comando cambia según el sistema operativo, verás dos versiones: **PowerShell** (Windows) y **bash** (macOS/Linux).
> - Si en tu equipo el comando es `py` o `python3` en lugar de `python`, úsalo solo para **crear** el entorno virtual. Una vez activado, `python` apunta al Python del entorno.
> - Para crear un archivo: en el explorador de VS Code, haz clic derecho sobre la carpeta → **New File**, escribe el nombre y pega el contenido.

### Distribución del tiempo

| Parte | Tema del capítulo | Minutos |
| :--- | :--- | :---: |
| 0. Preparación | Estructura de proyectos | 5 |
| 1. Análisis del proceso | Principios de diseño de automatizaciones | 15 |
| 2. Proyecto y dependencias | Estructura src, pyproject, gestión de dependencias | 10 |
| 3. Configuración y secretos | Configuración, variables de entorno, separación de entornos y secretos | 15 |
| 4. Componentes | Diseño modular, SoC, inyección de dependencias, tipos, docstrings | 25 |
| 5. Ejecución y pruebas de fallo | Fronteras, fail-fast, HITL, parametrización | 12 |
| 6. Calidad y documentación | Calidad de código, mantenibilidad, README y runbook | 8 |

> **Checkpoints.** Al final de las partes 2, 3 y 4 hay un punto de control. Si te atrasas, el instructor puede darte la carpeta del checkpoint para continuar desde una versión que funciona.

---

## Parte 0. Preparar la carpeta del proyecto (5 min)

**0.1** Crea una carpeta llamada `practica1_compras` y ábrela en VS Code con **File → Open Folder**.

**0.2** Abre la terminal (**Terminal → New Terminal**) y comprueba tu versión de Python:

```bash
python --version
```

Debe mostrar `Python 3.11.x` o superior. Si muestra 3.10 o anterior, avisa al instructor antes de continuar.

**0.3** Crea la estructura de carpetas:

**PowerShell**
```powershell
New-Item -ItemType Directory -Force -Path src/compras_auto, tests, config, datos, docs, legado
```

**bash**
```bash
mkdir -p src/compras_auto tests config datos docs legado
```

Cada carpeta tiene una responsabilidad:

| Carpeta | Contenido |
| :--- | :--- |
| `src/compras_auto/` | Código de la automatización (src-layout) |
| `tests/` | Pruebas automatizadas |
| `config/` | Configuración por entorno, **sin secretos** |
| `datos/` | Archivos de entrada de ejemplo |
| `docs/` | Análisis y runbook |
| `legado/` | El script original que vamos a sustituir |

---

## Parte 1. Analizar el proceso antes de programar (15 min)

### 1.1 El escenario

Cada día hábil, Lucía, analista de compras, recibe el archivo `solicitudes_AAAA-MM-DD.csv` con unas 300 solicitudes. Su proceso manual es:

1. Abre el archivo en Excel y revisa a ojo que cada fila tenga id, solicitante, correo y monto válidos.
2. Separa las solicitudes de más de **$5,000**, porque esas necesitan la aprobación del gerente.
3. Corrige "como puede" los registros con errores, o los borra.
4. Envía las solicitudes listas a Tesorería y las de más de $5,000 al gerente.

Le toma **90 minutos al día**. Cerca del **4 %** de los registros tiene errores, y el mes pasado se pagó una compra de $7,200 sin aprobación.

Un compañero escribió un script para "automatizarlo". Ese script es tu punto de partida (fase lineal).

### 1.2 Crear los datos de entrada

**1.2.1** Crea el archivo de solicitudes del día. Contiene, a propósito, registros válidos e inválidos:

**Archivo:** `datos/solicitudes_2026-10-23.csv`

```text
id,solicitante,correo,monto
1,  Ana   López ,ANA.LOPEZ@EMPRESA.MX,1500.00
2,Bruno Díaz,bruno.diaz@empresa.mx,7200.50
3,Carla Ruiz,carla.ruiz@empresa.mx,MONTO_INVALIDO
4,,dario.mena@empresa.mx,300.00
5,Elena Soto,elena.soto-empresa.mx,950.00
6,Fernando Gil,fernando.gil@empresa.mx,5000.00
7,Gabriela Paz,gabriela.paz@empresa.mx,-200.00
X8,Hugo Vera,hugo.vera@empresa.mx,100.00
9,Irene Vela,irene.vela@empresa.mx,12000
10,Jorge Luna,jorge.luna@empresa.mx,NaN
11,Karla Ríos,karla.rios@empresa.mx,820.00,EXTRA
```

**1.2.2** Crea un archivo al que le falta una columna obligatoria. Lo usarás en la Parte 5:

**Archivo:** `datos/sin_columna_correo.csv`

```text
id,solicitante,monto
1,Ana López,1500.00
```

### 1.3 Ejecutar el script original

**1.3.1** Crea el script heredado. **No lo corrijas**: sirve para observar los problemas.

**Archivo:** `legado/procesar_compras_v0.py`

```python
# Script original (fase lineal). NO lo corrijas: es el punto de partida.
import csv, os, sys

TOKEN = "tok-prod-REAL-8f3a91"
ARCHIVO = "datos/solicitudes_2026-10-23.csv"

def proc(f):
    d = list(csv.DictReader(open(f)))
    for r in d:
        m = float(r["monto"])
        if m > 5000:
            print("REQUIERE APROBACION", r["id"], r["correo"], m)
        else:
            print("OK", r["id"], r["correo"], m)
    print("Notificando con token", TOKEN)

proc(ARCHIVO)
```

**1.3.2** Ejecútalo:

```bash
python legado/procesar_compras_v0.py
```

**Resultado esperado:**

```text
OK 1 ANA.LOPEZ@EMPRESA.MX 1500.0
REQUIERE APROBACION 2 bruno.diaz@empresa.mx 7200.5
Traceback (most recent call last):
  ...
ValueError: could not convert string to float: 'MONTO_INVALIDO'
```

**1.3.3** Responde en tu cuaderno o en el chat del curso:

- ¿Cuántos registros se procesaron antes del fallo y qué pasó con los demás?
- ¿Qué habría impreso el script si no hubiera fallado? Fíjate en la última línea de `proc()`.
- ¿Qué tendrías que modificar en el código para procesar el archivo de otro día?
- Si mañana el límite cambia a $8,000, ¿cuántos lugares tendrías que editar?

> En Windows es posible que veas caracteres extraños en algunos nombres. `open(f)` sin `encoding` usa la codificación del sistema operativo: es otro modo de fallo que debes anotar.

### 1.4 Documentar el análisis

**1.4.1** Crea `docs/analisis.md` con la siguiente plantilla y **completa cada `___`**. La fila 1 del AMFE ya está resuelta como ejemplo.

**Archivo:** `docs/analisis.md`

```markdown
# Análisis de la automatización: clasificación de solicitudes de compra

## 1. Selección y priorización
| Criterio | Valor |
|---|---|
| Tiempo manual por ejecución | 90 min |
| Frecuencia | 1 vez por día hábil (≈ 22 al mes) |
| Horas manuales al mes | ___ |
| Errores manuales observados | ≈ 4 % de los registros |
| ¿Reglas claras y documentadas? | ___ |
| ¿Datos estructurados? | ___ |
| Cuadrante de la matriz impacto/esfuerzo | ___ |
| Decisión (automatizar / estabilizar primero / descartar) | ___ |

## 2. Alcance
- **Entradas:** ___
- **Procesamiento (reglas):** ___
- **Salidas:** ___
- **Excepciones y su tratamiento:** ___
- **Fuera de alcance:** ___

## 3. Criterios de entrada, salida y éxito
- **Precondiciones:** ___
- **Postcondiciones:** ___
- **Criterio de éxito (reconciliación):** ___

## 4. Modos de fallo (AMFE)
| # | Modo de fallo | Condición que lo provoca | S | O | D | NPR | Mitigación en el código | D después | NPR después |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Un registro inválido llega a las salidas sin detectarse | Monto, correo o id con formato incorrecto | 8 | 4 | 7 | 224 | `validar_solicitud()` y archivo `rechazadas.csv` | 2 | 64 |
| 2 | El archivo llega sin una columna obligatoria | Cambio de formato en el sistema origen | 7 | 3 | 6 | ___ | ___ | 1 | ___ |
| 3 | Una compra mayor al límite se procesa sin aprobación | Falta de control de montos | 10 | 3 | 8 | ___ | ___ | 2 | ___ |
| 4 | Una credencial queda expuesta | Token escrito en el código o impreso en logs | 9 | 5 | 9 | ___ | ___ | 3 | ___ |
| 5 | El proceso agota la memoria | Archivo mucho mayor de lo esperado | 5 | 2 | 5 | ___ | ___ | 1 | ___ |

> Escalas de 1 a 10 y umbral de NPR = 100: valores didácticos; cada equipo define los suyos.

## 5. Automatización frente a aprobación humana
| Caso | Modo (Zero-Touch / HITL / revisión de excepciones) | Justificación |
|---|---|---|
| Solicitud válida con monto ≤ límite | ___ | ___ |
| Solicitud válida con monto > límite | ___ | ___ |
| Registro inválido | ___ | ___ |

## 6. Escalabilidad
- Volumen esperado hoy y en un año: ___
- Límite que protege los recursos: ___
- Técnica de lectura para no cargar todo en memoria: ___
```

**Pistas:**

- **Horas al mes:** 90 min × 22 días.
- **NPR = S × O × D.** Calcula el NPR antes y después de la mitigación. Los valores de "D después" ya vienen dados.
- **Alcance:** la automatización **no corrige** registros inválidos. Los aísla con su motivo para revisión humana. No detecta solicitudes duplicadas ni envía correos reales: anótalo en "Fuera de alcance".
- **Criterio de éxito:** registros leídos = aprobados automáticamente + pendientes + rechazados.

> **Checkpoint de análisis.** Antes de programar, comparte con el instructor tu cuadrante, tu tabla AMFE y tu decisión de HITL.

---

## Parte 2. Estructura del proyecto y dependencias (10 min)

### 2.1 Configurar el proyecto

**2.1.1** Crea `pyproject.toml` en la raíz. Declara el paquete, las dependencias **directas** con rangos y la configuración de las herramientas de calidad:

**Archivo:** `pyproject.toml`

```toml
[build-system]
requires = ["setuptools>=69"]
build-backend = "setuptools.build_meta"

[project]
name = "compras-auto"
version = "0.1.0"
description = "Clasificación diaria de solicitudes de compra"
requires-python = ">=3.11"
dependencies = [
    "pydantic>=2.7,<3",
    "python-dotenv>=1.0,<2",
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
strict = true

[tool.pytest.ini_options]
testpaths = ["tests"]
```

**2.1.2** Crea `src/compras_auto/__init__.py`. Su presencia convierte la carpeta en un paquete:

**Archivo:** `src/compras_auto/__init__.py`

```python
"""Automatización de clasificación de solicitudes de compra."""

__version__ = "0.1.0"
```

**2.1.3** Crea `.gitignore`. Desde el primer minuto, `.env` queda fuera del repositorio:

**Archivo:** `.gitignore`

```text
# Secretos locales
.env

# Entorno virtual y cachés
.venv/
__pycache__/
*.egg-info/
.mypy_cache/
.ruff_cache/
.pytest_cache/

# Resultados generados
salida/
```

### 2.2 Crear y activar el entorno virtual

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

Al activarlo verás `(.venv)` al inicio del prompt.

### 2.3 Fijar versiones con pip-tools

**2.3.1** Crea `requirements-dev.in` con las herramientas de desarrollo. La línea `-c requirements.txt` obliga a que respeten las versiones de producción:

**Archivo:** `requirements-dev.in`

```text
# Herramientas de desarrollo; respetan las versiones de requirements.txt
-c requirements.txt
mypy
pip-tools
pytest
radon
ruff
```

**2.3.2** Instala pip-tools y genera los archivos bloqueados:

```bash
python -m pip install --upgrade pip
python -m pip install pip-tools
pip-compile --strip-extras pyproject.toml -o requirements.txt
pip-compile --strip-extras requirements-dev.in -o requirements-dev.txt
```

En este proyecto, el archivo de **dependencias lógicas** de producción es `pyproject.toml`, que cumple la función de `requirements.in`. `pip-compile` lo lee y genera `requirements.txt` con **todas** las versiones resueltas, incluidas las transitivas.

**2.3.3** Abre `requirements.txt` y localiza una dependencia **transitiva**, es decir, una que no declaraste pero que aparece con un comentario `# via pydantic`.

**2.3.4** Instala las dependencias y el proyecto en modo editable:

```bash
python -m pip install -r requirements.txt -r requirements-dev.txt
python -m pip install -e .
```

**Verificación:**

```bash
python -c "import compras_auto; print(compras_auto.__version__)"
```

Resultado esperado: `0.1.0`

> **¿Por qué src-layout y `pip install -e .`?** Al estar el código dentro de `src/`, Python solo encuentra `compras_auto` si el paquete está instalado. Así detectas importaciones que funcionarían "por casualidad" desde la raíz.

> **Checkpoint 2.** El entorno está activo, `requirements.txt` existe y la verificación imprime `0.1.0`.

---

## Parte 3. Configuración, entornos y secretos (15 min)

En esta parte separas las tres categorías vistas en clase:

| Categoría | Dónde vive | Ejemplo en esta práctica |
| :--- | :--- | :--- |
| **Código** | `src/` | Las reglas de validación |
| **Configuración** | `config/dev.toml`, `config/prod.toml` | Límite de aprobación, tamaño máximo, carpeta de salida |
| **Secretos** | Variables de entorno (`.env` solo en desarrollo) | `COMPRAS_NOTIF_TOKEN` |

### 3.1 Archivos de configuración por entorno

**Archivo:** `config/dev.toml`

```toml
# Configuración del entorno de desarrollo. Aquí NO van secretos.

[entrada]
tamano_max_mb = 5.0          # frontera de alcance: archivos mayores se rechazan

[reglas]
limite_aprobacion = 5000.00  # montos mayores requieren aprobación humana

[salida]
directorio = "salida/dev"

[registro]
nivel = "INFO"
```

**Archivo:** `config/prod.toml`

```toml
# Configuración del entorno de producción. Aquí NO van secretos.

[entrada]
tamano_max_mb = 5.0

[reglas]
limite_aprobacion = 5000.00

[salida]
directorio = "salida/prod"

[registro]
nivel = "WARNING"
```

El código es el mismo en ambos entornos. Solo cambian los valores.

### 3.2 Variables de entorno y secretos

**3.2.1** Crea la plantilla `.env.example`. Esta **sí** se comparte con el equipo:

**Archivo:** `.env.example`

```bash
# Copia este archivo como .env y reemplaza los valores ficticios.
# .env.example SÍ se versiona; .env NUNCA.

# Secreto obligatorio (valor ficticio)
COMPRAS_NOTIF_TOKEN=reemplaza-con-tu-token

# Opcionales: sobrescriben al archivo TOML
# COMPRAS_LOG_LEVEL=DEBUG
# COMPRAS_SALIDA_DIR=salida/otra_carpeta
```

**3.2.2** Crea tu archivo local `.env` a partir de la plantilla:

**PowerShell**
```powershell
Copy-Item .env.example .env
```

**bash**
```bash
cp .env.example .env
```

Abre `.env` y reemplaza el valor ficticio por `tok-dev-123456`. Este archivo **nunca** se sube al repositorio.

> `APP_ENV` no va dentro de `.env`. La define la plataforma que ejecuta el proceso: si no existe, se asume `dev`. En `prod` no se carga `.env`; los secretos deben venir del sistema, del contenedor o de un gestor de secretos.

### 3.3 Excepciones propias

Crea `src/compras_auto/errores.py`. Cada excepción tiene un **código de salida** distinto, así quien opere el proceso sabe qué falló sin leer el código:

**Archivo:** `src/compras_auto/errores.py`

```python
"""Excepciones propias de la automatización.

Cada excepción tiene un código de salida para que quien ejecute el proceso
(una persona, un programador de tareas o un orquestador) sepa qué falló.
"""


class AutomatizacionError(Exception):
    """Clase base de los errores controlados de la automatización."""

    codigo_salida: int = 1


class ConfiguracionInvalidaError(AutomatizacionError):
    """La configuración o un secreto obligatorio no es válido."""

    codigo_salida = 1


class EntradaRechazadaError(AutomatizacionError):
    """El archivo de entrada no cumple las precondiciones."""

    codigo_salida = 3


class LimiteAlcanceExcedidoError(EntradaRechazadaError):
    """El archivo supera el tamaño máximo definido en el alcance."""


class ReconciliacionError(AutomatizacionError):
    """Los conteos de salida no cuadran con los registros leídos."""

    codigo_salida = 4
```

### 3.4 Cargador de configuración con fail-fast

Crea `src/compras_auto/configuracion.py`. Este módulo concentra:

- La lectura de TOML con `tomllib`.
- La validación con Pydantic v2: `Field(gt=..., le=...)`, `Literal`, y `extra="forbid"` para detectar claves mal escritas.
- La jerarquía de precedencia **CLI > variables de entorno > archivo > valores por defecto**. Python no la aplica solo: la implementamos nosotros en `resolver_configuracion()`.
- La carga de `.env` **solo** en desarrollo, con `override=False`.
- Los secretos obligatorios con `os.environ[...]`, que lanza `KeyError` si faltan. Las opciones usan `os.getenv(...)`.

**Archivo:** `src/compras_auto/configuracion.py`

```python
"""Carga y validación de la configuración y de los secretos (fail-fast)."""

import os
import tomllib
from decimal import Decimal
from pathlib import Path
from typing import Any, Literal

from dotenv import load_dotenv
from pydantic import BaseModel, ConfigDict, Field, ValidationError

from compras_auto.errores import ConfiguracionInvalidaError

NivelLog = Literal["DEBUG", "INFO", "WARNING", "ERROR"]
ENTORNOS_VALIDOS = ("dev", "prod")


class EntradaConfig(BaseModel):
    model_config = ConfigDict(extra="forbid")
    tamano_max_mb: float = Field(default=5.0, gt=0, le=100)


class ReglasConfig(BaseModel):
    model_config = ConfigDict(extra="forbid")
    limite_aprobacion: Decimal = Field(gt=0)


class SalidaConfig(BaseModel):
    model_config = ConfigDict(extra="forbid")
    directorio: Path


class RegistroConfig(BaseModel):
    model_config = ConfigDict(extra="forbid")
    nivel: NivelLog = "INFO"


class AppConfig(BaseModel):
    """Esquema completo; extra="forbid" detecta claves mal escritas."""

    model_config = ConfigDict(extra="forbid")
    entrada: EntradaConfig = Field(default_factory=EntradaConfig)
    reglas: ReglasConfig
    salida: SalidaConfig
    registro: RegistroConfig = Field(default_factory=RegistroConfig)


def cargar_entorno() -> str:
    """Determina el entorno y, solo en desarrollo, carga el archivo .env.

    Returns:
        "dev" o "prod".

    Raises:
        ConfiguracionInvalidaError: Si APP_ENV tiene un valor no permitido.
    """
    entorno = os.getenv("APP_ENV", "dev")
    if entorno not in ENTORNOS_VALIDOS:
        raise ConfiguracionInvalidaError(
            f"APP_ENV={entorno!r} no es válido; usa uno de {ENTORNOS_VALIDOS}"
        )
    if entorno == "dev":
        # override=False: una variable ya definida en el sistema tiene prioridad.
        load_dotenv(Path.cwd() / ".env", override=False)
    return entorno


def leer_toml(ruta: Path) -> dict[str, Any]:
    """Lee un archivo TOML y lo devuelve como diccionario.

    Raises:
        ConfiguracionInvalidaError: Si no existe o su sintaxis es incorrecta.
    """
    if not ruta.is_file():
        raise ConfiguracionInvalidaError(
            f"No existe el archivo de configuración: {ruta}"
        )
    try:
        with ruta.open("rb") as archivo:
            return tomllib.load(archivo)
    except tomllib.TOMLDecodeError as error:
        raise ConfiguracionInvalidaError(
            f"Sintaxis TOML inválida en {ruta}: {error}"
        ) from error


def _sobrescribir(
    datos: dict[str, Any], seccion: str, clave: str, valor: object
) -> None:
    """Reemplaza datos[seccion][clave] solo si se proporcionó un valor."""
    if valor is None:
        return
    tabla = datos.setdefault(seccion, {})
    if isinstance(tabla, dict):
        tabla[clave] = valor


def resolver_configuracion(
    ruta_config: Path, cli_salida: Path | None, cli_nivel: str | None
) -> AppConfig:
    """Aplica la jerarquía CLI > entorno > archivo > valores por defecto y valida.

    Raises:
        ConfiguracionInvalidaError: Si el resultado no cumple el esquema.
    """
    datos = leer_toml(ruta_config)  # 3. archivo (4. los defaults están en el esquema)
    _sobrescribir(datos, "salida", "directorio", os.getenv("COMPRAS_SALIDA_DIR"))  # 2.
    _sobrescribir(datos, "registro", "nivel", os.getenv("COMPRAS_LOG_LEVEL"))  # 2.
    _sobrescribir(datos, "salida", "directorio", cli_salida)  # 1. CLI
    _sobrescribir(datos, "registro", "nivel", cli_nivel)  # 1. CLI
    try:
        return AppConfig.model_validate(datos)
    except ValidationError as error:
        raise ConfiguracionInvalidaError(
            f"Configuración inválida en {ruta_config}:\n{error}"
        ) from error


def obtener_secreto(nombre: str) -> str:
    """Obtiene un secreto obligatorio del entorno sin mostrar su valor.

    Raises:
        ConfiguracionInvalidaError: Si la variable no existe o está vacía.
    """
    try:
        valor = os.environ[nombre]
    except KeyError:
        raise ConfiguracionInvalidaError(
            f"Falta la variable de entorno obligatoria {nombre}"
        ) from None
    if not valor.strip():
        raise ConfiguracionInvalidaError(f"La variable de entorno {nombre} está vacía")
    return valor
```

**Verificación 3.4.1.** La configuración se carga y se valida:

```bash
python -c "from pathlib import Path; from compras_auto.configuracion import resolver_configuracion; print(resolver_configuracion(Path('config/dev.toml'), None, None))"
```

Resultado esperado (en Windows verás `WindowsPath` en lugar de `PosixPath`):

```text
entrada=EntradaConfig(tamano_max_mb=5.0) reglas=ReglasConfig(limite_aprobacion=Decimal('5000.0')) salida=SalidaConfig(directorio=PosixPath('salida/dev')) registro=RegistroConfig(nivel='INFO')
```

**Verificación 3.4.2.** Un secreto ausente detiene el proceso con un mensaje claro:

```bash
python -c "from compras_auto.configuracion import obtener_secreto; obtener_secreto('NO_EXISTE')"
```

La última línea debe ser:

```text
compras_auto.errores.ConfiguracionInvalidaError: Falta la variable de entorno obligatoria NO_EXISTE
```

> **Checkpoint 3.** Las dos verificaciones dan el resultado esperado.

---

## Parte 4. Componentes con responsabilidades separadas (25 min)

Vas a pasar de la **fase lineal** (el script heredado) a la **fase de componentes**:

| Capa | Módulo | Responsabilidad | ¿Hace E/S? |
| :--- | :--- | :--- | :---: |
| Dominio | `reglas.py` | Validar y clasificar | No |
| Contratos | `contratos.py` | Interfaces con `typing.Protocol` | No |
| Ingesta | `lector.py` | Precondiciones del archivo y lectura | Sí |
| Salida | `salida.py` | Escribir resultados y notificar | Sí |
| Orquestación | `orquestador.py`, `cli.py` | Coordinar el flujo | No directamente |

### 4.1 Dominio: reglas de negocio puras

Crea `src/compras_auto/reglas.py`. No lee archivos, no imprime y no se conecta a nada, por eso se puede probar de forma aislada. Las columnas obligatorias se definen aquí porque son el **contrato del dominio**: si cambian, también cambia la lógica, así que no conviene parametrizarlas.

**Archivo:** `src/compras_auto/reglas.py`

```python
"""Capa de dominio: reglas de negocio puras, sin lectura ni escritura de archivos."""

import re
from dataclasses import dataclass
from decimal import Decimal, InvalidOperation
from enum import StrEnum

COLUMNAS_OBLIGATORIAS: tuple[str, ...] = ("id", "solicitante", "correo", "monto")
_PATRON_CORREO = re.compile(r"^[^@\s]+@[^@\s]+\.[^@\s]+$")


class Categoria(StrEnum):
    """Destino de cada registro procesado."""

    AUTOMATICA = "automatica"
    PENDIENTE = "pendiente_aprobacion"
    RECHAZADA = "rechazada"


@dataclass(frozen=True)
class Solicitud:
    """Solicitud de compra validada y normalizada."""

    id: int
    solicitante: str
    correo: str
    monto: Decimal


@dataclass(frozen=True)
class Rechazo:
    """Registro que no cumple las reglas y se envía a revisión humana."""

    numero_fila: int
    motivo: str
    valores: tuple[str, ...]


def normalizar_texto(texto: str) -> str:
    """Elimina espacios al inicio y al final y colapsa espacios internos.

    Args:
        texto: Texto original.

    Returns:
        Texto normalizado, por ejemplo "  Ana   López " -> "Ana López".
    """
    return " ".join(texto.split())


def validar_solicitud(datos: dict[str, str]) -> Solicitud:
    """Convierte un registro crudo en una Solicitud válida.

    Args:
        datos: Valores del registro indexados por nombre de columna.

    Returns:
        La solicitud con sus campos normalizados.

    Raises:
        ValueError: Si algún campo incumple las reglas; el mensaje es el motivo.
    """
    try:
        id_solicitud = int(datos["id"])
    except ValueError:
        raise ValueError(f"id no es un entero: {datos['id']!r}") from None
    if id_solicitud <= 0:
        raise ValueError("id debe ser mayor que cero")

    solicitante = normalizar_texto(datos["solicitante"])
    if not solicitante:
        raise ValueError("solicitante vacío")

    correo = normalizar_texto(datos["correo"]).lower()
    if not _PATRON_CORREO.match(correo):
        raise ValueError("correo con formato inválido")

    try:
        monto = Decimal(datos["monto"].strip())
    except InvalidOperation:
        raise ValueError(f"monto no numérico: {datos['monto']!r}") from None
    if not monto.is_finite() or monto <= 0:
        raise ValueError("monto debe ser un número mayor que cero")

    return Solicitud(id_solicitud, solicitante, correo, monto)


def clasificar(solicitud: Solicitud, limite_aprobacion: Decimal) -> Categoria:
    """Decide si la solicitud se procesa sola o requiere aprobación humana.

    Un monto exactamente igual al límite se procesa automáticamente.
    """
    if solicitud.monto > limite_aprobacion:
        return Categoria.PENDIENTE
    return Categoria.AUTOMATICA
```

Fíjate en tres decisiones:

- `Decimal` en lugar de `float` para los importes, con rechazo explícito de `NaN`, infinito, cero y negativos.
- `>` y no `>=`: un monto **exactamente** de $5,000 se procesa automáticamente.
- `validar_solicitud()` no "adivina" valores. Si algo no cumple, lanza `ValueError` con el motivo.

**Verificación 4.1:**

```bash
python -c "from compras_auto.reglas import validar_solicitud; print(validar_solicitud({'id': '1', 'solicitante': '  Ana   López ', 'correo': 'ANA@EMPRESA.MX', 'monto': '1500.00'}))"
```

Resultado esperado:

```text
Solicitud(id=1, solicitante='Ana López', correo='ana@empresa.mx', monto=Decimal('1500.00'))
```

### 4.2 Contratos entre capas

Crea `src/compras_auto/contratos.py`. Con `Protocol`, el orquestador depende de **lo que un componente sabe hacer**, no de una clase concreta. Cualquier objeto con esos métodos sirve, sin necesidad de herencia:

**Archivo:** `src/compras_auto/contratos.py`

```python
"""Contratos (interfaces) entre las capas, definidos con typing.Protocol."""

from collections.abc import Iterator
from dataclasses import dataclass
from typing import Protocol

from compras_auto.reglas import Categoria, Rechazo, Solicitud


@dataclass(frozen=True)
class FilaLeida:
    """Fila tal como llega del origen, antes de aplicar reglas."""

    numero: int
    valores: list[str]


class Lector(Protocol):
    """Cualquier origen de datos: CSV, API, base de datos..."""

    def validar_estructura(self) -> list[str]: ...

    def registros(self) -> Iterator[FilaLeida]: ...


class Escritor(Protocol):
    """Cualquier destino de resultados."""

    def iniciar(self) -> None: ...

    def guardar_solicitud(self, categoria: Categoria, solicitud: Solicitud) -> None: ...

    def guardar_rechazo(self, rechazo: Rechazo) -> None: ...

    def finalizar(self) -> dict[Categoria, int]: ...


class Notificador(Protocol):
    """Cualquier canal para avisar a los aprobadores."""

    def notificar_pendientes(self, cantidad: int) -> None: ...
```

### 4.3 Ingesta: fronteras y lectura bajo demanda

Crea `src/compras_auto/lector.py`. Reúne lo visto en las demos de límites de alcance y de validación de CSV:

- `verificar_limites_alcance()` aplica la frontera de tamaño con la excepción personalizada.
- `validar_estructura()` lee **solo el encabezado** y compara conjuntos para encontrar las columnas faltantes.
- `registros()` es un **generador** (`yield`): entrega una fila a la vez, sin cargar el archivo completo en memoria.
- Se usa `encoding="utf-8-sig"` (tolera el BOM de Excel) y `newline=""` (deja los saltos de línea al módulo `csv`).

**Archivo:** `src/compras_auto/lector.py`

```python
"""Capa de ingesta: precondiciones del archivo y lectura bajo demanda."""

import csv
from collections.abc import Iterator
from pathlib import Path

from compras_auto.contratos import FilaLeida
from compras_auto.errores import EntradaRechazadaError, LimiteAlcanceExcedidoError
from compras_auto.reglas import COLUMNAS_OBLIGATORIAS

BYTES_POR_MIB = 1024 * 1024


def verificar_limites_alcance(ruta: Path, limite_mb: float) -> Path:
    """Comprueba que el archivo exista y no supere el tamaño permitido.

    Args:
        ruta: Ruta del archivo de entrada.
        limite_mb: Tamaño máximo permitido en MiB (validado en la configuración).

    Returns:
        La misma ruta, ya validada.

    Raises:
        EntradaRechazadaError: Si el archivo no existe.
        LimiteAlcanceExcedidoError: Si el archivo supera el límite.
    """
    if not ruta.is_file():
        raise EntradaRechazadaError(f"No existe el archivo de entrada: {ruta}")
    tamano_mb = ruta.stat().st_size / BYTES_POR_MIB
    if tamano_mb > limite_mb:
        raise LimiteAlcanceExcedidoError(
            f"El archivo mide {tamano_mb:.2f} MiB y el máximo es {limite_mb:.2f} MiB"
        )
    return ruta


class LectorCsv:
    """Lee un CSV de solicitudes fila por fila (generador)."""

    def __init__(self, ruta: Path) -> None:
        self._ruta = ruta

    def validar_estructura(self) -> list[str]:
        """Lee solo el encabezado y verifica las columnas obligatorias.

        Raises:
            EntradaRechazadaError: Si faltan columnas o el archivo no es UTF-8.
        """
        try:
            with self._ruta.open("r", encoding="utf-8-sig", newline="") as archivo:
                encabezado = [col.strip() for col in next(csv.reader(archivo), [])]
        except UnicodeDecodeError as error:
            raise EntradaRechazadaError(
                f"El archivo no está en UTF-8: {error}"
            ) from error
        faltantes = set(COLUMNAS_OBLIGATORIAS) - set(encabezado)
        if faltantes:
            raise EntradaRechazadaError(
                f"Faltan columnas obligatorias: {', '.join(sorted(faltantes))}"
            )
        return encabezado

    def registros(self) -> Iterator[FilaLeida]:
        """Entrega las filas de datos una a una, sin cargar el archivo completo."""
        try:
            with self._ruta.open("r", encoding="utf-8-sig", newline="") as archivo:
                lector = csv.reader(archivo)
                next(lector, None)  # el encabezado ya se validó
                for valores in lector:
                    if not any(valor.strip() for valor in valores):
                        continue  # se ignoran las líneas vacías
                    yield FilaLeida(numero=lector.line_num, valores=valores)
        except UnicodeDecodeError as error:
            raise EntradaRechazadaError(
                f"El archivo no está en UTF-8: {error}"
            ) from error
```

**Verificación 4.3:**

```bash
python -c "from pathlib import Path; from compras_auto.lector import LectorCsv; l = LectorCsv(Path('datos/solicitudes_2026-10-23.csv')); print(l.validar_estructura()); print(next(l.registros()))"
```

Resultado esperado:

```text
['id', 'solicitante', 'correo', 'monto']
FilaLeida(numero=2, valores=['1', '  Ana   López ', 'ANA.LOPEZ@EMPRESA.MX', '1500.00'])
```

> `numero` es la **línea del archivo**: el encabezado es la línea 1, así que el registro con `id=1` está en la línea 2.

### 4.4 Salida y notificación

Crea `src/compras_auto/salida.py`. Contiene dos escritores que cumplen el **mismo contrato**:

- `EscritorCsv` escribe los tres archivos y al cerrar **vuelve a contar desde disco** lo que escribió (postcondición).
- `EscritorSimulado` cuenta pero no escribe nada. Se usa con `--dry-run` y en las pruebas.

`NotificadorConsola` recibe el token por su constructor (inyección de dependencias) y **nunca lo muestra**: solo registra su longitud. Los archivos se escriben con `utf-8-sig` para que Excel muestre bien los acentos.

**Archivo:** `src/compras_auto/salida.py`

```python
"""Capa de salida y notificación."""

import csv
import logging
from pathlib import Path
from typing import TextIO

from compras_auto.reglas import Categoria, Rechazo, Solicitud

logger = logging.getLogger(__name__)

ARCHIVOS_SALIDA: dict[Categoria, str] = {
    Categoria.AUTOMATICA: "aprobadas_automaticamente.csv",
    Categoria.PENDIENTE: "pendientes_aprobacion.csv",
    Categoria.RECHAZADA: "rechazadas.csv",
}
ENCABEZADO_SOLICITUD = ["id", "solicitante", "correo", "monto"]
ENCABEZADO_RECHAZO = ["numero_fila", "motivo", "registro_original"]


class EscritorCsv:
    """Escribe un CSV por categoría dentro de un directorio de salida."""

    def __init__(self, directorio: Path) -> None:
        self._directorio = directorio
        self._archivos: dict[Categoria, TextIO] = {}

    def iniciar(self) -> None:
        """Crea el directorio y los tres archivos con su encabezado."""
        self._directorio.mkdir(parents=True, exist_ok=True)
        for categoria, nombre in ARCHIVOS_SALIDA.items():
            archivo = (self._directorio / nombre).open(
                "w", encoding="utf-8-sig", newline=""
            )
            es_rechazo = categoria is Categoria.RECHAZADA
            encabezado = ENCABEZADO_RECHAZO if es_rechazo else ENCABEZADO_SOLICITUD
            csv.writer(archivo).writerow(encabezado)
            self._archivos[categoria] = archivo

    def guardar_solicitud(self, categoria: Categoria, solicitud: Solicitud) -> None:
        fila = [solicitud.id, solicitud.solicitante, solicitud.correo, solicitud.monto]
        csv.writer(self._archivos[categoria]).writerow(fila)

    def guardar_rechazo(self, rechazo: Rechazo) -> None:
        fila = [rechazo.numero_fila, rechazo.motivo, ",".join(rechazo.valores)]
        csv.writer(self._archivos[Categoria.RECHAZADA]).writerow(fila)

    def finalizar(self) -> dict[Categoria, int]:
        """Cierra los archivos y vuelve a contarlos desde disco (postcondición)."""
        for archivo in self._archivos.values():
            archivo.close()
        conteos: dict[Categoria, int] = {}
        for categoria, nombre in ARCHIVOS_SALIDA.items():
            ruta = self._directorio / nombre
            with ruta.open(encoding="utf-8-sig", newline="") as archivo:
                conteos[categoria] = sum(1 for _ in csv.reader(archivo)) - 1
        return conteos


class EscritorSimulado:
    """Escritor para --dry-run y pruebas: cuenta, pero no escribe archivos."""

    def __init__(self) -> None:
        self.conteos: dict[Categoria, int] = dict.fromkeys(Categoria, 0)

    def iniciar(self) -> None:
        logger.info("Modo simulación: no se escribirá ningún archivo.")

    def guardar_solicitud(self, categoria: Categoria, solicitud: Solicitud) -> None:
        self.conteos[categoria] += 1

    def guardar_rechazo(self, rechazo: Rechazo) -> None:
        self.conteos[Categoria.RECHAZADA] += 1

    def finalizar(self) -> dict[Categoria, int]:
        return dict(self.conteos)


class NotificadorConsola:
    """Simula el aviso a los aprobadores; en producción sería correo o chat."""

    def __init__(self, token: str) -> None:
        self._token = token  # se usa para autenticarse; nunca se registra

    def notificar_pendientes(self, cantidad: int) -> None:
        logger.info(
            "Notificación simulada a aprobadores: %d solicitud(es) pendiente(s) "
            "(credencial cargada, %d caracteres).",
            cantidad,
            len(self._token),
        )
```

### 4.5 Orquestador: coordina, reconcilia y aplica HITL

Crea `src/compras_auto/orquestador.py`. El orquestador **recibe** sus dependencias y no crea ninguna:

**Archivo:** `src/compras_auto/orquestador.py`

```python
"""Capa de orquestación: coordina las capas, sin conocer sus detalles."""

import logging
from dataclasses import dataclass
from decimal import Decimal

from compras_auto.contratos import Escritor, FilaLeida, Lector, Notificador
from compras_auto.errores import ReconciliacionError
from compras_auto.reglas import Categoria, Rechazo, clasificar, validar_solicitud

logger = logging.getLogger(__name__)


@dataclass
class Resumen:
    """Conteos de la ejecución, usados para el criterio de éxito."""

    leidas: int = 0
    automaticas: int = 0
    pendientes: int = 0
    rechazadas: int = 0

    def cuadra(self) -> bool:
        """Reconciliación: cada fila leída quedó en exactamente una categoría."""
        total = self.automaticas + self.pendientes + self.rechazadas
        return self.leidas == total


class OrquestadorCompras:
    """Recibe sus dependencias desde fuera (inyección de dependencias)."""

    def __init__(
        self,
        lector: Lector,
        escritor: Escritor,
        notificador: Notificador,
        limite_aprobacion: Decimal,
    ) -> None:
        self._lector = lector
        self._escritor = escritor
        self._notificador = notificador
        self._limite = limite_aprobacion

    def ejecutar(self) -> Resumen:
        """Ejecuta el proceso completo y verifica postcondiciones.

        Raises:
            EntradaRechazadaError: Si la entrada no cumple las precondiciones.
            ReconciliacionError: Si los conteos no cuadran.
        """
        encabezado = self._lector.validar_estructura()  # precondición antes de escribir
        resumen = Resumen()
        self._escritor.iniciar()
        try:
            for fila in self._lector.registros():
                resumen.leidas += 1
                self._procesar_fila(encabezado, fila, resumen)
        finally:
            escritos = self._escritor.finalizar()

        self._verificar_postcondiciones(resumen, escritos)
        if resumen.pendientes:
            self._notificador.notificar_pendientes(resumen.pendientes)
        return resumen

    def _procesar_fila(
        self, encabezado: list[str], fila: FilaLeida, resumen: Resumen
    ) -> None:
        if len(fila.valores) != len(encabezado):
            motivo = (
                f"se esperaban {len(encabezado)} columnas "
                f"y llegaron {len(fila.valores)}"
            )
            self._rechazar(fila, motivo, resumen)
            return
        try:
            solicitud = validar_solicitud(
                dict(zip(encabezado, fila.valores, strict=True))
            )
        except ValueError as error:
            self._rechazar(fila, str(error), resumen)
            return

        categoria = clasificar(solicitud, self._limite)
        logger.debug("Fila %d clasificada como %s", fila.numero, categoria)
        self._escritor.guardar_solicitud(categoria, solicitud)
        if categoria is Categoria.PENDIENTE:
            resumen.pendientes += 1
        else:
            resumen.automaticas += 1

    def _rechazar(self, fila: FilaLeida, motivo: str, resumen: Resumen) -> None:
        # Se registra el número de fila y el motivo, no los datos personales.
        logger.warning("Fila %d rechazada: %s", fila.numero, motivo)
        self._escritor.guardar_rechazo(
            Rechazo(fila.numero, motivo, tuple(fila.valores))
        )
        resumen.rechazadas += 1

    @staticmethod
    def _verificar_postcondiciones(
        resumen: Resumen, escritos: dict[Categoria, int]
    ) -> None:
        if not resumen.cuadra():
            raise ReconciliacionError(f"Los conteos no cuadran: {resumen}")
        esperados = {
            Categoria.AUTOMATICA: resumen.automaticas,
            Categoria.PENDIENTE: resumen.pendientes,
            Categoria.RECHAZADA: resumen.rechazadas,
        }
        if escritos != esperados:
            raise ReconciliacionError(
                f"Se esperaba escribir {esperados} y en destino hay {escritos}"
            )
```

Puntos clave:

- **Precondición primero:** `validar_estructura()` se ejecuta antes de crear cualquier archivo de salida.
- **Supervisión por excepción:** una fila inválida no detiene el lote. Se aísla con su motivo y el proceso continúa.
- **HITL:** las compras mayores al límite **no** se procesan. Quedan en `pendientes_aprobacion.csv` (estado persistente) y se notifica al responsable.
- **Registro seguro:** el log de un rechazo muestra el número de fila y el motivo, no los datos personales.
- **Criterio de éxito:** `cuadra()` y la comparación con los conteos en disco. Si no coinciden, se lanza `ReconciliacionError` (código 4).

### 4.6 Punto de entrada y parámetros de ejecución

**4.6.1** Crea `src/compras_auto/cli.py`. Separa la **fase de configuración** (`inicializar_parametros`) de la **fase de ejecución**, y en `main()` conecta los componentes concretos:

**Archivo:** `src/compras_auto/cli.py`

```python
"""Punto de entrada: parámetros, configuración y arranque del proceso."""

import argparse
import logging
from pathlib import Path
from typing import NamedTuple

from compras_auto.configuracion import (
    AppConfig,
    cargar_entorno,
    obtener_secreto,
    resolver_configuracion,
)
from compras_auto.errores import AutomatizacionError
from compras_auto.lector import LectorCsv, verificar_limites_alcance
from compras_auto.orquestador import OrquestadorCompras
from compras_auto.salida import EscritorCsv, EscritorSimulado, NotificadorConsola

logger = logging.getLogger("compras_auto")


class ConfigEjecucion(NamedTuple):
    """Configuración resuelta e inmutable de una ejecución (sin secretos)."""

    entorno: str
    ruta_entrada: Path
    ajustes: AppConfig
    dry_run: bool


def construir_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(
        prog="python -m compras_auto",
        description="Clasifica las solicitudes de compra del día.",
    )
    parser.add_argument("entrada", type=Path, help="CSV de solicitudes a procesar")
    parser.add_argument(
        "--config", type=Path, help="archivo TOML (por defecto config/<APP_ENV>.toml)"
    )
    parser.add_argument("--salida", type=Path, help="directorio de salida")
    parser.add_argument(
        "--nivel-log",
        choices=["DEBUG", "INFO", "WARNING", "ERROR"],
        help="nivel de log",
    )
    parser.add_argument(
        "--dry-run", action="store_true", help="procesa y reporta sin escribir archivos"
    )
    return parser


def inicializar_parametros(args: argparse.Namespace) -> ConfigEjecucion:
    """Resuelve entorno y configuración en un solo lugar."""
    entorno = cargar_entorno()
    ruta_config = args.config or Path("config") / f"{entorno}.toml"
    ajustes = resolver_configuracion(ruta_config, args.salida, args.nivel_log)
    return ConfigEjecucion(entorno, args.entrada, ajustes, args.dry_run)


def main(argv: list[str] | None = None) -> int:
    """Ejecuta la automatización y devuelve el código de salida."""
    logging.basicConfig(
        level=logging.INFO,
        format="%(asctime)s | %(levelname)-8s | %(name)s | %(message)s",
    )
    args = construir_parser().parse_args(argv)  # argparse termina con código 2
    try:
        config = inicializar_parametros(args)
        logging.getLogger().setLevel(config.ajustes.registro.nivel)
        token = obtener_secreto("COMPRAS_NOTIF_TOKEN")
        logger.info("Entorno: %s | Entrada: %s", config.entorno, config.ruta_entrada)

        verificar_limites_alcance(
            config.ruta_entrada, config.ajustes.entrada.tamano_max_mb
        )
        directorio = config.ajustes.salida.directorio / config.ruta_entrada.stem
        escritor = EscritorSimulado() if config.dry_run else EscritorCsv(directorio)
        orquestador = OrquestadorCompras(
            lector=LectorCsv(config.ruta_entrada),
            escritor=escritor,
            notificador=NotificadorConsola(token),
            limite_aprobacion=config.ajustes.reglas.limite_aprobacion,
        )
        resumen = orquestador.ejecutar()
    except AutomatizacionError as error:
        logger.error("%s", error)
        return error.codigo_salida
    except Exception:
        logger.exception("Error inesperado; revisa el runbook")
        return 1

    logger.info(
        "Leídas=%d | Automáticas=%d | Pendientes=%d | Rechazadas=%d | Cuadra: sí",
        resumen.leidas,
        resumen.automaticas,
        resumen.pendientes,
        resumen.rechazadas,
    )
    if not config.dry_run:
        logger.info("Resultados en: %s", directorio)
    return 0
```

Observa que `logging.basicConfig()` se configura **antes** de leer la configuración, para que también quede registrado un error de configuración. El token se obtiene del entorno y **nunca** como argumento de línea de comandos, porque los argumentos quedan en el historial de la terminal.

**4.6.2** Crea `src/compras_auto/__main__.py` para poder ejecutar el paquete con `python -m compras_auto`:

**Archivo:** `src/compras_auto/__main__.py`

```python
"""Permite ejecutar el paquete con: python -m compras_auto."""

from compras_auto.cli import main

raise SystemExit(main())
```

**Verificación 4.6:**

```bash
python -m compras_auto --help
```

Debe mostrar la ayuda generada por `argparse`, con el argumento `entrada` y las opciones `--config`, `--salida`, `--nivel-log` y `--dry-run`.

> **Checkpoint 4.** `--help` funciona. Si aparece `ModuleNotFoundError`, revisa la sección **Solución de problemas**.

---

## Parte 5. Ejecutar y comprobar el comportamiento ante fallos (12 min)

> Para ver el **código de salida** del último comando:
> - **PowerShell:** `$LASTEXITCODE`
> - **bash:** `echo $?`

### 5.1 Ejecución normal

```bash
python -m compras_auto datos/solicitudes_2026-10-23.csv
```

**Resultado esperado** (las fechas y horas cambian; en Windows la ruta usa `\`):

```text
... | INFO     | compras_auto | Entorno: dev | Entrada: datos/solicitudes_2026-10-23.csv
... | WARNING  | compras_auto.orquestador | Fila 4 rechazada: monto no numérico: 'MONTO_INVALIDO'
... | WARNING  | compras_auto.orquestador | Fila 5 rechazada: solicitante vacío
... | WARNING  | compras_auto.orquestador | Fila 6 rechazada: correo con formato inválido
... | WARNING  | compras_auto.orquestador | Fila 8 rechazada: monto debe ser un número mayor que cero
... | WARNING  | compras_auto.orquestador | Fila 9 rechazada: id no es un entero: 'X8'
... | WARNING  | compras_auto.orquestador | Fila 11 rechazada: monto debe ser un número mayor que cero
... | WARNING  | compras_auto.orquestador | Fila 12 rechazada: se esperaban 4 columnas y llegaron 5
... | INFO     | compras_auto.salida | Notificación simulada a aprobadores: 2 solicitud(es) pendiente(s) (credencial cargada, 14 caracteres).
... | INFO     | compras_auto | Leídas=11 | Automáticas=2 | Pendientes=2 | Rechazadas=7 | Cuadra: sí
... | INFO     | compras_auto | Resultados en: salida/dev/solicitudes_2026-10-23
```

Código de salida: **0**.

Abre los tres archivos generados en `salida/dev/solicitudes_2026-10-23/` y comprueba:

| Archivo | Registros | Contenido |
| :--- | :---: | :--- |
| `aprobadas_automaticamente.csv` | 2 | id 1 (nombre y correo ya normalizados) e id 6 (exactamente $5,000) |
| `pendientes_aprobacion.csv` | 2 | id 2 ($7,200.50) e id 9 ($12,000) |
| `rechazadas.csv` | 7 | Cada fila con su número de línea, motivo y registro original |

**Reconciliación:** 11 = 2 + 2 + 7.

**Repetibilidad:** ejecuta el mismo comando otra vez. Los archivos se sobrescriben y los conteos son idénticos; no se duplica nada.

### 5.2 Fronteras y precondiciones (fail-fast)

Ejecuta cada caso y anota el código de salida:

**Caso A. Falta una columna obligatoria**
```bash
python -m compras_auto datos/sin_columna_correo.csv
```
Esperado: `ERROR ... Faltan columnas obligatorias: correo` y código **3**. No se crea ninguna carpeta de salida.

**Caso B. El archivo supera la frontera de tamaño.** Primero genera un archivo de 6 MiB:
```bash
python -c "from pathlib import Path; Path('datos/grande.csv').write_bytes(b'x' * 6 * 1024 * 1024)"
python -m compras_auto datos/grande.csv
```
Esperado: `ERROR ... El archivo mide 6.00 MiB y el máximo es 5.00 MiB` y código **3**.

**Caso C. El archivo no existe**
```bash
python -m compras_auto datos/no_existe.csv
```
Esperado: `ERROR ... No existe el archivo de entrada: datos/no_existe.csv` y código **3** (en Windows la ruta aparece con `\`).

**Caso D. Configuración inválida.** Copia la configuración de desarrollo:

**PowerShell**
```powershell
Copy-Item config/dev.toml config/roto.toml
```
**bash**
```bash
cp config/dev.toml config/roto.toml
```

En `config/roto.toml`, cambia `limite_aprobacion = 5000.00` por `limite_aprobacion = -1` y agrega debajo la línea `limite_aprovacion = 10` (con el error de escritura intencional). Ejecuta:

```bash
python -m compras_auto datos/solicitudes_2026-10-23.csv --config config/roto.toml
```

Esperado: código **1** y un mensaje de Pydantic con **dos** errores: `Input should be greater than 0` y `Extra inputs are not permitted`. El proceso se detiene **antes** de leer cualquier dato.

### 5.3 Separación de entornos y secretos

**Caso E. Producción sin secreto.** En producción no se lee `.env`, así que el token de desarrollo no está disponible.

**PowerShell**
```powershell
$env:APP_ENV = "prod"
python -m compras_auto datos/solicitudes_2026-10-23.csv
```
**bash**
```bash
APP_ENV=prod python -m compras_auto datos/solicitudes_2026-10-23.csv
```

Esperado: `ERROR ... Falta la variable de entorno obligatoria COMPRAS_NOTIF_TOKEN` y código **1**.

**Caso F. Producción con el secreto inyectado por la plataforma**

**PowerShell**
```powershell
$env:COMPRAS_NOTIF_TOKEN = "tok-prod-999"
python -m compras_auto datos/solicitudes_2026-10-23.csv
Remove-Item Env:APP_ENV, Env:COMPRAS_NOTIF_TOKEN
```
**bash**
```bash
APP_ENV=prod COMPRAS_NOTIF_TOKEN=tok-prod-999 python -m compras_auto datos/solicitudes_2026-10-23.csv
```

Esperado: código **0**. Solo aparecen los `WARNING` de rechazo, porque `prod.toml` define `nivel = "WARNING"`, y los resultados se escriben en `salida/prod/...`. Mismo código, distinta configuración.

### 5.4 Jerarquía de parametrización

**Caso G. La variable de entorno sobrescribe el archivo**

**PowerShell**
```powershell
$env:COMPRAS_LOG_LEVEL = "WARNING"
python -m compras_auto datos/solicitudes_2026-10-23.csv --dry-run
```
**bash**
```bash
COMPRAS_LOG_LEVEL=WARNING python -m compras_auto datos/solicitudes_2026-10-23.csv --dry-run
```
Esperado: solo líneas `WARNING`, aunque `dev.toml` dice `INFO`.

**Caso H. El argumento CLI sobrescribe a la variable de entorno**

**PowerShell**
```powershell
python -m compras_auto datos/solicitudes_2026-10-23.csv --dry-run --nivel-log DEBUG
Remove-Item Env:COMPRAS_LOG_LEVEL
```
**bash**
```bash
COMPRAS_LOG_LEVEL=WARNING python -m compras_auto datos/solicitudes_2026-10-23.csv --dry-run --nivel-log DEBUG
```
Esperado: aparecen líneas `DEBUG ... Fila 2 clasificada como automatica` y `Modo simulación: no se escribirá ningún archivo.`

### 5.5 Preguntas de cierre

1. En el Caso A, ¿por qué es mejor rechazar el archivo completo que procesar las filas que sí tienen datos?
2. ¿Qué fila del AMFE mitiga cada uno de los casos A a F?
3. Si el límite de aprobación cambia a $8,000, ¿qué archivo modificas? Compáralo con tu respuesta de la Parte 1.

---

## Parte 6. Calidad, pruebas y documentación (8 min)

### 6.1 Pruebas aisladas con objetos simulados

**6.1.1** Crea `tests/test_reglas.py`. Prueba la capa de dominio sin archivos, e incluye casos positivos, negativos y el valor límite:

**Archivo:** `tests/test_reglas.py`

```python
"""Pruebas de la capa de dominio: no necesitan archivos ni red."""

from decimal import Decimal

import pytest

from compras_auto.reglas import (
    Categoria,
    clasificar,
    normalizar_texto,
    validar_solicitud,
)


def registro(**cambios: str) -> dict[str, str]:
    base = {
        "id": "1",
        "solicitante": "Ana López",
        "correo": "ana@empresa.mx",
        "monto": "100",
    }
    base.update(cambios)
    return base


def test_normaliza_espacios() -> None:
    assert normalizar_texto("  Ana   López ") == "Ana López"


def test_solicitud_valida_normaliza_correo() -> None:
    solicitud = validar_solicitud(registro(correo=" ANA@EMPRESA.MX "))
    assert solicitud.correo == "ana@empresa.mx"


@pytest.mark.parametrize(
    ("campo", "valor"),
    [
        ("id", "X8"),
        ("solicitante", "   "),
        ("correo", "ana-empresa.mx"),
        ("monto", "abc"),
        ("monto", "NaN"),
        ("monto", "-200"),
    ],
)
def test_rechaza_registros_invalidos(campo: str, valor: str) -> None:
    with pytest.raises(ValueError):
        validar_solicitud(registro(**{campo: valor}))


@pytest.mark.parametrize(
    ("monto", "esperada"),
    [("5000.00", Categoria.AUTOMATICA), ("5000.01", Categoria.PENDIENTE)],
)
def test_limite_de_aprobacion(monto: str, esperada: Categoria) -> None:
    solicitud = validar_solicitud(registro(monto=monto))
    assert clasificar(solicitud, Decimal("5000")) is esperada
```

**6.1.2** Crea `tests/test_orquestador.py`. Gracias a la inyección de dependencias, el orquestador se prueba con un lector y un notificador **falsos**:

**Archivo:** `tests/test_orquestador.py`

```python
"""Pruebas del orquestador con dependencias simuladas (inyección de dependencias)."""

from collections.abc import Iterator
from decimal import Decimal

from compras_auto.contratos import FilaLeida
from compras_auto.orquestador import OrquestadorCompras
from compras_auto.reglas import COLUMNAS_OBLIGATORIAS
from compras_auto.salida import EscritorSimulado


class LectorFalso:
    def __init__(self, filas: list[FilaLeida]) -> None:
        self._filas = filas

    def validar_estructura(self) -> list[str]:
        return list(COLUMNAS_OBLIGATORIAS)

    def registros(self) -> Iterator[FilaLeida]:
        yield from self._filas


class NotificadorFalso:
    def __init__(self) -> None:
        self.avisos: list[int] = []

    def notificar_pendientes(self, cantidad: int) -> None:
        self.avisos.append(cantidad)


def test_clasifica_reconcilia_y_notifica() -> None:
    filas = [
        FilaLeida(2, ["1", "Ana", "ana@empresa.mx", "100"]),
        FilaLeida(3, ["2", "Bruno", "bruno@empresa.mx", "9000"]),
        FilaLeida(4, ["3", "Carla", "carla@empresa.mx", "abc"]),
        FilaLeida(5, ["4", "Darío", "dario@empresa.mx"]),
    ]
    notificador = NotificadorFalso()
    orquestador = OrquestadorCompras(
        LectorFalso(filas), EscritorSimulado(), notificador, Decimal("5000")
    )

    resumen = orquestador.ejecutar()

    assert (resumen.automaticas, resumen.pendientes, resumen.rechazadas) == (1, 1, 2)
    assert resumen.cuadra()
    assert notificador.avisos == [1]
```

**6.1.3** Ejecuta las pruebas:

```bash
python -m pytest -v
```

Esperado: `11 passed`.

### 6.2 Análisis estático y métricas

**6.2.1** Compara el script heredado con tu solución:

```bash
ruff check legado
ruff check src tests
ruff format --check src tests
python -m mypy src tests
```

Esperado:

| Comando | Resultado |
| :--- | :--- |
| `ruff check legado` | `Found 4 errors.` (importaciones múltiples en una línea, sin ordenar y sin usar) |
| `ruff check src tests` | `All checks passed!` |
| `ruff format --check src tests` | `12 files already formatted` |
| `mypy src tests` | `Success: no issues found in 12 source files` |

**6.2.2** Mide la complejidad ciclomática y el índice de mantenibilidad:

```bash
radon cc src -s -a
radon mi src
```

Esperado: todas las funciones con grado **A** o **B** y todos los módulos con índice de mantenibilidad **A**. La función más compleja es `validar_solicitud` (B): ¿por qué crees que es aceptable en este caso?

> Ninguna de estas herramientas garantiza que el programa sea correcto. Detectan ciertos problemas antes de ejecutarlo; las pruebas y la validación funcional siguen siendo necesarias.

### 6.3 README y runbook

**6.3.1** Crea `README.md` en la raíz. Explica **cómo entender y ejecutar** el proyecto:

**Archivo:** `README.md`

````markdown
# compras-auto

Clasifica el archivo diario de solicitudes de compra en tres salidas:
aprobadas automáticamente, pendientes de aprobación humana y rechazadas.

## Requisitos
- Python 3.11 o superior
- Dependencias fijadas en `requirements.txt`

## Instalación
```bash
python -m venv .venv
# Activa el entorno (Windows: .venv\Scripts\Activate.ps1 | macOS/Linux: source .venv/bin/activate)
python -m pip install -r requirements.txt
python -m pip install -e .
```

## Configuración
| Fuente | Qué contiene | ¿Se versiona? |
|---|---|---|
| `config/dev.toml`, `config/prod.toml` | Límites, reglas, directorio de salida, nivel de log | Sí |
| `.env` (solo desarrollo) | `COMPRAS_NOTIF_TOKEN` y variables opcionales | **No** |
| `.env.example` | Plantilla con valores ficticios | Sí |
| `APP_ENV` | `dev` (por defecto) o `prod` | Lo define la plataforma |

Precedencia: argumentos CLI > variables de entorno > archivo TOML > valores por defecto.

## Ejecución
```bash
python -m compras_auto datos/solicitudes_2026-10-23.csv
python -m compras_auto datos/solicitudes_2026-10-23.csv --dry-run --nivel-log DEBUG
```

## Resultado esperado
En `salida/<entorno>/<nombre_del_archivo>/` se generan
`aprobadas_automaticamente.csv`, `pendientes_aprobacion.csv` y `rechazadas.csv`.
La operación del proceso y la atención de fallos se describen en `docs/runbook.md`.
````

**6.3.2** Crea `docs/runbook.md`. Explica **cómo operar el proceso y responder ante fallos**, y completa el responsable de escalamiento:

**Archivo:** `docs/runbook.md`

```markdown
# Runbook: compras-auto

## Ejecución y verificación
1. Ejecutar: `python -m compras_auto <archivo.csv>` desde la raíz del proyecto.
2. Verificar que la última línea del log diga `Cuadra: sí` y que el código de salida sea `0`.
3. Entregar `pendientes_aprobacion.csv` a los aprobadores y `rechazadas.csv` al área que envía el archivo.

## Códigos de salida
| Código | Significado | Acción |
|---|---|---|
| 0 | Éxito; la reconciliación cuadra | Ninguna |
| 1 | Configuración o secreto inválido | Revisar el mensaje: archivo TOML, `APP_ENV` o `COMPRAS_NOTIF_TOKEN` |
| 2 | Argumentos de línea de comandos inválidos | Ejecutar con `--help` y corregir el comando |
| 3 | Entrada rechazada (no existe, excede el tamaño o le faltan columnas) | Solicitar al área de origen un archivo corregido; **no** editarlo a mano |
| 4 | La reconciliación no cuadra | No usar las salidas; escalar al equipo de desarrollo |

## Logs
Se escriben en la consola (salida de error estándar). Nunca contienen el token ni datos
personales de los registros rechazados: solo número de fila y motivo.

## Recuperación
El proceso puede ejecutarse de nuevo con el mismo archivo: sobrescribe sus salidas y
produce el mismo resultado.

## Escalamiento
Si el código 1 persiste con una configuración que parece correcta, o aparece el código 4,
escalar a: ___ (responsable del proceso).
```

---

## Entregables

Comprime la carpeta `practica1_compras` **sin** `.venv/`, `.env` ni `salida/` y entrégala. Antes de entregar, revisa esta lista:

- [ ] `docs/analisis.md` completo: cuadrante, alcance, criterios, AMFE con NPR y decisión HITL
- [ ] Estructura `src/` con los 10 archivos del paquete `compras_auto`
- [ ] `pyproject.toml`, `requirements.txt` y `requirements-dev.txt` generados con pip-tools
- [ ] `config/dev.toml` y `config/prod.toml` sin secretos
- [ ] `.env.example` versionado y `.env` incluido en `.gitignore`
- [ ] La ejecución normal termina con `Cuadra: sí` y código 0
- [ ] Los casos A a H se comportan como se describe
- [ ] `pytest`, `ruff` y `mypy` sin errores
- [ ] `README.md` y `docs/runbook.md` completos

---

## Solución de problemas

| Síntoma | Causa probable | Solución |
| :--- | :--- | :--- |
| `ModuleNotFoundError: No module named 'compras_auto'` | El paquete no está instalado en el entorno activo, o falta `src/compras_auto/__init__.py` | Activa `.venv` y ejecuta `python -m pip install -e .` desde la raíz |
| `ModuleNotFoundError: No module named 'pydantic'` o `'dotenv'` | Estás fuera del entorno virtual | Activa `.venv`; comprueba con `python -m pip show pydantic` |
| PowerShell: `...Activate.ps1 cannot be loaded because running scripts is disabled` | Política de ejecución de Windows | Ejecuta `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` y vuelve a activar |
| `No module named 'tomllib'` | Python 3.10 o anterior | Instala Python 3.11 o superior y recrea `.venv` |
| `Falta la variable de entorno obligatoria COMPRAS_NOTIF_TOKEN` en desarrollo | No existe `.env`, no estás en la raíz del proyecto o quedó `APP_ENV=prod` de un caso anterior | Crea `.env`, ejecuta desde la raíz y en PowerShell ejecuta `Remove-Item Env:APP_ENV` |
| `No existe el archivo de configuración: config\dev.toml` | Ejecutas desde otra carpeta | Usa `cd` para volver a la raíz `practica1_compras` |
| `pip-compile` no se reconoce | pip-tools no está instalado en el entorno activo | `python -m pip install pip-tools` con `.venv` activo |

---

## Limpieza

**PowerShell**
```powershell
Remove-Item Env:APP_ENV, Env:COMPRAS_NOTIF_TOKEN, Env:COMPRAS_LOG_LEVEL -ErrorAction SilentlyContinue
Remove-Item datos/grande.csv, config/roto.toml -ErrorAction SilentlyContinue
deactivate
```

**bash**
```bash
rm -f datos/grande.csv config/roto.toml
deactivate
```

**Conserva el proyecto:** es el punto de partida de la Práctica 2, donde se agregan validación de esquemas, contratos de datos y reglas de calidad configurables.

---

## Resumen

| Lo que hiciste | Tema del capítulo |
| :--- | :--- |
| Evaluaste el proceso con la matriz impacto/esfuerzo antes de programar | Selección y priorización de casos de uso |
| Definiste entradas, reglas, salidas, excepciones y lo que queda fuera | Definición del alcance |
| Calculaste el NPR y convertiste cada mitigación en código | Modos y condiciones de fallo (AMFE) |
| Las compras mayores a $5,000 esperan una decisión humana | Automatización frente a aprobación humana |
| Lectura con generador y frontera de tamaño | Automatizaciones escalables |
| `src/`, `pyproject.toml`, `tests/`, `config/`, `docs/` | Estructura de proyectos |
| Dominio, ingesta, salida y orquestación con `Protocol` e inyección de dependencias | Diseño modular y separación de responsabilidades |
| Tipos verificados con mypy, docstrings estilo Google, Ruff y Radon | Anotaciones de tipo, calidad, estilo y mantenibilidad |
| Versiones resueltas con pip-tools | Gestión de dependencias |
| TOML + Pydantic, `.env`, `APP_ENV`, precedencia CLI > entorno > archivo > valores por defecto | Gestión de configuración y entornos |
| El token no está en el código, en los argumentos ni en los logs | Separación y uso seguro de secretos |
| Pasaste de un script lineal a componentes reutilizables | Evolución desde scripts hacia componentes |
