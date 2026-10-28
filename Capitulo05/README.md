# Práctica 5. Pruebas y escalamiento de una automatización

| Dato | Detalle |
| :--- | :--- |
| **Curso** | Automatización de procesos con Python |
| **Capítulo** | 5. Pruebas, paralelismo y optimización de rendimiento |
| **Duración** | 120 minutos |
| **Modalidad** | Individual |
| **Nivel de Bloom** | Aplicar / Analizar |

---

## Objetivo

Implementar una suite de pruebas para un pipeline de datos y escalarlo de forma segura:

- **Pruebas:** unitarias, de integración, con mocks, con datos sintéticos y de regresión.
- **Resiliencia:** validar que los fallos de la API (timeout, 5xx, 429, JSON inválido) se manejan como indica el diseño.
- **Rendimiento:** medir, aplicar una optimización y comparar el antes y el después **sin regresiones**.

Todo es local: no necesitas cuentas, Docker ni servicios en la nube.

## Lo que vas a construir

Un pipeline de ventas, su suite de pruebas y su medición de rendimiento:

```text
 datos/ventas.csv  (10,000 ventas sintéticas con Faker y semilla 42)
        │
        ▼
 leer_csv ─► deduplicar ─► validar_venta ─► enriquecer ─────► cargar ─► reporte JSON
                │              │               │                 │
                │              │               │  GET /productos/{sku}    SQLite con UPSERT
                │              │               ▼                 │        en lotes (batched)
                │              │        API de catálogo simulada (50 ms por petición)
                │              │
                └── rechazos con su motivo: duplicado, cantidad, precio, fecha, producto

 Suite pytest
   unitarias ─── reglas y valores límite (parametrize)
   mocks ─────── cliente HTTP: reintentos, Retry-After, 404, JSON corrupto, autospec
   integración ─ CSV en tmp_path + responses + SQLite real; API real en un hilo
   regresión ─── golden master del reporte completo
        │
        ▼
 Medir (cProfile) ─► optimizar SOLO lo que el perfil señala ─► comparar (tiempo y memoria)
                                                               └─ la suite prueba que el
                                                                  resultado no cambió
```

### Relación con el capítulo

| Componente del laboratorio | Lección del capítulo |
| :--- | :--- |
| `test_validacion.py`: AAA, `pytest.raises` con `match`, excepción propia que hereda de `ValueError` | 5.1 Pruebas unitarias |
| `test_pipeline.py` y `test_api_real.py`: `tmp_path`, SQLite real, `responses`, fixture con `yield` y `scope="module"`, tres niveles de verificación | 5.2 Pruebas de integración |
| `test_cliente.py`: `patch` donde se usa, `side_effect` en lista, `autospec`, reloj inyectado y `call_args_list` | 5.3 Mocking |
| `datos.py`: Faker con `seed_instance`, fechas absolutas, defectos en posiciones fijas, dominios `example.*` | 5.4 Datos de prueba |
| `test_regresion.py`: golden master, `--actualizar-golden`, datos dinámicos enmascarados | 5.6 Pruebas de regresión |
| `parametrize` con `id=`: partición de equivalencia, valores límite, `NaN`, `bool`, tipos inválidos | 5.7 Casos positivos y negativos |
| `pyproject.toml` (`--strict-markers`, `--strict-config`, marcadores, `xfail_strict`), `--junitxml`, workflow de GitHub Actions, códigos de salida | 5.8 Preparación para CI/CD |
| `cargar()` recibe un generador y escribe con `itertools.batched` | 5.9 Generadores, 5.10 Lotes |
| `enriquecer_concurrente()`: `ThreadPoolExecutor`, `Future`, `as_completed`, `result()` | 5.13 Concurrencia |
| `medir_memoria()` con `tracemalloc` | 5.14 Gestión de memoria |
| `perfilar()` con `cProfile` y `pstats`; confirmación con `perf_counter` | 5.15 Profiling |
| Por qué hilos y no `asyncio` ni procesos; qué se ganó y qué se pagó | 5.11, 5.12, 5.18 Compromisos |

> Las pruebas de contrato con pact-python (5.5) y los límites con `resource` (5.17) no forman parte de esta práctica: se cubren en las demos del capítulo.

## Criterios de evaluación

| Criterio | Evidencia en tu entrega |
| :--- | :--- |
| **Corrección** | Las pruebas verifican reglas de negocio y valores límite (5,000 y 20,000 del descuento; cantidad y precio en sus extremos), no solo que un mock fue llamado |
| **Resiliencia** | Timeout, error de conexión, 5xx, 429 con `Retry-After`, 404 y JSON corrupto tienen cada uno una prueba; una API caída detiene el pipeline sin cargar nada (código 3) |
| **Mantenibilidad** | Pruebas deterministas e independientes (semilla, `tmp_path`, sin esperas reales); datos sin información personal; configuración en `pyproject.toml`; marcadores para CI |
| **Valor operativo** | La optimización se justifica con un perfil; la comparación muestra el antes y el después; la suite y el golden master demuestran que el resultado no cambió, y detectan una "optimización" incorrecta |

---

## Requisitos

| Elemento | Requisito |
| :--- | :--- |
| Python | **3.12 o superior** (probado con 3.12, 3.13 y 3.14; se usa `itertools.batched`) |
| Sistema operativo | Windows 10/11, macOS o Linux |
| Editor | Visual Studio Code con la extensión de Python |
| Terminal | **Dos** terminales integradas de VS Code (PowerShell en Windows; bash/zsh en macOS o Linux) |
| Red | Acceso a PyPI para instalar paquetes. Durante la práctica todo es local |

> **Convenciones**
> - Todos los comandos se ejecutan desde la **carpeta raíz del proyecto** (`practica5_pruebas`) con el entorno virtual activo.
> - Las pruebas se ejecutan siempre con **`python -m pytest`**: usa el Python del entorno activo.
> - **Terminal 1** ejecuta la API simulada a partir de la Parte 4; **Terminal 2**, todo lo demás.
> - Cuando un comando cambia según el sistema operativo, verás la versión **PowerShell** y la versión **bash**.
> - Los tiempos de las salidas de ejemplo cambian según el equipo; los conteos y los resultados, no.

### Distribución del tiempo

| Parte | Contenido | Minutos |
| :--- | :--- | :---: |
| 0. Preparación | Estructura, entorno y dependencias | 10 |
| 1. Datos sintéticos | Faker con semilla y sus pruebas | 10 |
| 2. Reglas de negocio | Pruebas unitarias, casos positivos, negativos y límites | 15 |
| 3. Cliente de la API | Mocks: reintentos, Retry-After, errores permanentes, JSON corrupto | 15 |
| 4. Pipeline base | Pipeline, CLI y pruebas de integración | 20 |
| 5. Regresión | Golden master y un error que solo detecta una prueba de límite | 10 |
| 6. Medir, optimizar y comparar | cProfile, optimización, comparación y una optimización incorrecta | 25 |
| 7. Listo para CI | Marcadores, JUnit XML, códigos de salida y workflow | 10 |
| Cierre | Preguntas | 5 |

> **Checkpoints.** Al final de las partes 1 a 6 hay un punto de control. Si te atrasas, el instructor puede darte la carpeta del checkpoint.

---

## Parte 0. Preparación (10 min)

**0.1** Crea la carpeta `practica5_pruebas`, ábrela en VS Code (**File → Open Folder**) y abre una terminal.

**0.2** Crea la estructura de carpetas:

**PowerShell**
```powershell
New-Item -ItemType Directory -Force -Path src/pipeline_ventas, tests/datos, .github/workflows
```

**bash**
```bash
mkdir -p src/pipeline_ventas tests/datos .github/workflows
```

**0.3** Crea los archivos base. Observa la sección `[tool.pytest.ini_options]`: la configuración de pytest vive aquí desde el principio (tema 5.8).

**Archivo:** `pyproject.toml`

```toml
[build-system]
requires = ["setuptools>=69"]
build-backend = "setuptools.build_meta"

[project]
name = "pipeline-ventas"
version = "0.1.0"
description = "Pipeline de ventas con su suite de pruebas y su medición de rendimiento"
requires-python = ">=3.12"
dependencies = [
    "faker>=30",          # datos sintéticos reproducibles
    "requests>=2.32,<3",
]

[tool.setuptools.packages.find]
where = ["src"]

[tool.pytest.ini_options]
minversion = "8.0"
testpaths = ["tests"]
pythonpath = ["."]  # para importar api_simulada en las pruebas
addopts = "-ra --strict-markers --strict-config"
markers = [
    "integracion: usa la API simulada real en un hilo; en CI va en un paso aparte",
    "regresion: compara la salida completa contra el golden master",
]
xfail_strict = true

[tool.ruff]
line-length = 88
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP"]

[tool.mypy]
python_version = "3.12"
disallow_untyped_defs = true
ignore_missing_imports = true
```

**0.4** Crea los archivos de versiones. Se entregan resueltos para que todos instalen exactamente lo que se probó:

**Archivo:** `requirements.txt`

```text
# Versiones probadas para la Práctica 5 (Windows, macOS y Linux; Python 3.12-3.14).
# Generado con:
#   pip-compile --strip-extras --no-annotate pyproject.toml -o requirements.txt
certifi==2026.10.23
charset-normalizer==3.5.2
faker==40.43.0
idna==3.20
requests==2.34.2
urllib3==2.8.0
```

**Archivo:** `requirements-dev.in`

```text
# Herramientas de desarrollo; respetan las versiones de requirements.txt
-c requirements.txt
mypy
pip-tools
pytest
responses
ruff
types-requests
```

**Archivo:** `requirements-dev.txt`

```text
# Versiones probadas para la Práctica 5 (Windows, macOS y Linux; Python 3.12-3.14).
# Generado con:
#   pip-compile --strip-extras --no-annotate requirements-dev.in -o requirements-dev.txt
ast-serialize==0.13.1
build==1.6.1
certifi==2026.10.23
charset-normalizer==3.5.2
click==8.5.0
idna==3.20
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
pyyaml==6.0.3
requests==2.34.2
responses==0.26.3
ruff==0.17.3
types-requests==2.33.0.20261006
typing-extensions==4.16.0
urllib3==2.8.0
wheel==0.48.0
```

**Archivo:** `.gitignore`

```text
.venv/
__pycache__/
*.egg-info/
.mypy_cache/
.ruff_cache/
.pytest_cache/

# Datos generados y resultados de cada ejecución (el golden master SÍ se versiona)
datos/
salida/
reportes/
*.prof
```

**Archivo:** `src/pipeline_ventas/__init__.py`

```python
"""Pipeline de ventas con pruebas y medición de rendimiento (Práctica 5)."""

__version__ = "0.1.0"
```

**Archivo:** `tests/__init__.py`

```python
"""Pruebas de la Práctica 5."""
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
python -c "import pipeline_ventas, faker, requests, responses, pytest; print('ok', pipeline_ventas.__version__, 'pytest', pytest.__version__)"
```

Resultado esperado: `ok 0.1.0 pytest 9.1.1`

---

## Parte 1. Datos sintéticos reproducibles (10 min)

### 1.1 El generador

Usar datos reales de producción en pruebas es un riesgo de cumplimiento (LFPDPPP) y además los vuelve inestables. El generador usa estas técnicas:

- **Faker con una semilla propia** (`seed_instance`): misma semilla, mismos datos.
- **Fechas absolutas:** con `"-1y"` o `"today"` el dato cambiaría cada día.
- **`random.Random(semilla)`** para los demás campos.
- **Defectos en posiciones fijas:** así las pruebas saben exactamente qué esperar.

**Archivo:** `src/pipeline_ventas/datos.py`

```python
"""Datos sintéticos reproducibles con Faker y una semilla (tema 5.4).

Sin datos personales reales: los correos son de dominios example.* reservados.
Los defectos se insertan en posiciones fijas para que las pruebas sepan qué esperar.
"""

import csv
import random
from datetime import date
from pathlib import Path

from faker import Faker

COLUMNAS = [
    "venta_id",
    "fecha",
    "sucursal",
    "sku",
    "cantidad",
    "precio_unitario",
    "cliente_email",
]
SUCURSALES = ["CDMX-Centro", "GDL-Norte", "MTY-Valle", "PUE-Sur", "QRO-Centro"]
SKUS = [f"SKU-{n:03d}" for n in range(1, 41)]

# Cada cuántas filas aparece cada defecto (números primos: casi no coinciden)
DEFECTOS = {
    "duplicado": 97,  # repite el venta_id de la fila anterior
    "cantidad_cero": 89,
    "precio_texto": 83,
    "fecha_imposible": 79,
    "sku_inexistente": 73,  # la API responderá 404
}


def generar_ventas(n: int, semilla: int) -> list[dict[str, str]]:
    fake = Faker("es_MX")
    fake.seed_instance(semilla)  # semilla de esta instancia: mismo resultado siempre
    azar = random.Random(semilla)
    ventas: list[dict[str, str]] = []
    for i in range(1, n + 1):
        venta = {
            "venta_id": f"V-{i:06d}",
            # fechas absolutas: con "-1y" o "today" el dato cambiaría cada día
            "fecha": fake.date_between(date(2026, 1, 1), date(2026, 6, 30)).isoformat(),
            "sucursal": azar.choice(SUCURSALES),
            "sku": azar.choice(SKUS),
            "cantidad": str(azar.randint(1, 20)),
            "precio_unitario": f"{azar.uniform(10, 2500):.2f}",
            "cliente_email": fake.email(),
        }
        if i % DEFECTOS["duplicado"] == 0:
            venta["venta_id"] = ventas[-1]["venta_id"]
        elif i % DEFECTOS["cantidad_cero"] == 0:
            venta["cantidad"] = "0"
        elif i % DEFECTOS["precio_texto"] == 0:
            venta["precio_unitario"] = "N/A"
        elif i % DEFECTOS["fecha_imposible"] == 0:
            venta["fecha"] = "2026-02-30"
        elif i % DEFECTOS["sku_inexistente"] == 0:
            venta["sku"] = "SKU-999"
        ventas.append(venta)
    return ventas


def escribir_csv(ruta: Path, ventas: list[dict[str, str]]) -> None:
    ruta.parent.mkdir(parents=True, exist_ok=True)
    with open(ruta, "w", encoding="utf-8", newline="") as archivo:
        escritor = csv.DictWriter(archivo, fieldnames=COLUMNAS)
        escritor.writeheader()
        escritor.writerows(ventas)
```

```bash
python -c "from pipeline_ventas.datos import generar_ventas; [print(' | '.join(v.values())) for v in generar_ventas(3, semilla=42)]"
```

Resultado esperado: las mismas tres filas en todos los equipos.

```text
V-000001 | 2026-04-26 | CDMX-Centro | SKU-002 | 9 | 619.78 | laureanooscar@example.org
V-000002 | 2026-01-26 | GDL-Norte | SKU-007 | 18 | 226.48 | trejoeric@example.net
V-000003 | 2026-03-17 | PUE-Sur | SKU-003 | 1 | 243.30 | cabanmariano@example.org
```

### 1.2 Pruebas de los datos

**Archivo:** `tests/test_datos.py`

```python
from pipeline_ventas.datos import DEFECTOS, generar_ventas


def test_misma_semilla_mismos_datos() -> None:
    assert generar_ventas(50, semilla=42) == generar_ventas(50, semilla=42)


def test_otra_semilla_otros_datos() -> None:
    assert generar_ventas(50, semilla=42) != generar_ventas(50, semilla=7)


def test_los_defectos_estan_donde_se_esperan() -> None:
    ventas = generar_ventas(300, semilla=7)
    assert ventas[96]["venta_id"] == ventas[95]["venta_id"]  # fila 97: duplicado
    assert ventas[88]["cantidad"] == "0"
    assert ventas[82]["precio_unitario"] == "N/A"
    assert ventas[78]["fecha"] == "2026-02-30"
    assert ventas[72]["sku"] == "SKU-999"
    assert set(DEFECTOS) == {
        "duplicado",
        "cantidad_cero",
        "precio_texto",
        "fecha_imposible",
        "sku_inexistente",
    }


def test_sin_datos_personales_reales() -> None:
    dominios = {v["cliente_email"].split("@")[1] for v in generar_ventas(500, 42)}
    assert all(d.startswith("example.") for d in dominios)  # dominios reservados
```

> **Checkpoint 1.**
>
> ```bash
> python -m pytest tests/test_datos.py -v
> ```
>
> Resultado esperado: `4 passed`. La última prueba garantiza que ningún correo es real: todos son de dominios `example.*`, reservados para documentación.

---

## Parte 2. Reglas de negocio y pruebas unitarias (15 min)

### 2.1 Las reglas

Funciones puras, sin red ni base de datos: lo ideal para una prueba unitaria. Observa:

- **`VentaInvalidaError` hereda de `ValueError`:** el código que ya captura `ValueError` sigue funcionando.
- **`is_finite()`** rechaza `NaN` e infinito.
- **`isinstance(subtotal, bool)` va primero**, porque `True` también es `int`.
- **El dinero siempre en `Decimal`**, con `ROUND_HALF_UP`.

**Archivo:** `src/pipeline_ventas/validacion.py`

```python
"""Reglas de negocio puras: lo que cubren las pruebas unitarias (5.1 y 5.7)."""

from dataclasses import dataclass
from datetime import date
from decimal import ROUND_HALF_UP, Decimal, InvalidOperation

CANTIDAD_MAXIMA = 1000
PRECIO_MAXIMO = Decimal("100000")
CENTAVOS = Decimal("0.01")


class VentaInvalidaError(ValueError):
    """La fila existe pero no cumple las reglas: se rechaza con su motivo."""


@dataclass(frozen=True, slots=True)
class Venta:
    venta_id: str
    fecha: date
    sucursal: str
    sku: str
    cantidad: int
    precio_unitario: Decimal

    @property
    def subtotal(self) -> Decimal:
        return (self.precio_unitario * self.cantidad).quantize(CENTAVOS)


def validar_venta(fila: dict[str, str]) -> Venta:
    """Convierte una fila del CSV en Venta o lanza VentaInvalidaError."""
    try:
        cantidad = int(fila["cantidad"])
    except ValueError:
        raise VentaInvalidaError(
            f"cantidad no es entera: {fila['cantidad']!r}"
        ) from None
    if not 1 <= cantidad <= CANTIDAD_MAXIMA:
        raise VentaInvalidaError(f"cantidad {cantidad} fuera de [1, {CANTIDAD_MAXIMA}]")
    try:
        precio = Decimal(fila["precio_unitario"])
    except InvalidOperation:
        raise VentaInvalidaError(
            f"precio no numérico: {fila['precio_unitario']!r}"
        ) from None
    if not precio.is_finite() or not 0 < precio <= PRECIO_MAXIMO:  # NaN e infinito
        raise VentaInvalidaError(f"precio {precio} fuera de (0, {PRECIO_MAXIMO}]")
    try:
        fecha = date.fromisoformat(fila["fecha"])
    except ValueError:
        raise VentaInvalidaError(f"fecha inválida: {fila['fecha']!r}") from None
    return Venta(
        fila["venta_id"], fecha, fila["sucursal"], fila["sku"], cantidad, precio
    )


def calcular_descuento(subtotal: Decimal) -> Decimal:
    """5 % desde 5,000 y 10 % desde 20,000 (los errores suelen estar en < o <=)."""
    if isinstance(subtotal, bool) or not isinstance(subtotal, Decimal):
        raise TypeError(f"subtotal debe ser Decimal, no {type(subtotal).__name__}")
    if not subtotal.is_finite() or subtotal < 0:
        raise ValueError(f"subtotal inválido: {subtotal}")
    if subtotal >= 20000:
        tasa = Decimal("0.10")
    elif subtotal >= 5000:
        tasa = Decimal("0.05")
    else:
        return Decimal("0.00")
    return (subtotal * tasa).quantize(CENTAVOS, ROUND_HALF_UP)
```

### 2.2 Casos positivos, negativos y valores límite

Cada caso de `parametrize` lleva un `id=` legible. Se prueba justo debajo, en y justo encima de cada frontera, porque los errores suelen estar en `<` frente a `<=` (tema 5.7):

**Archivo:** `tests/test_validacion.py`

```python
from decimal import Decimal

import pytest

from pipeline_ventas.validacion import (
    VentaInvalidaError,
    calcular_descuento,
    validar_venta,
)


def fila(**cambios: str) -> dict[str, str]:
    base = {
        "venta_id": "V-000001",
        "fecha": "2026-03-15",
        "sucursal": "CDMX-Centro",
        "sku": "SKU-001",
        "cantidad": "2",
        "precio_unitario": "125.50",
        "cliente_email": "ana@example.com",
    }
    return base | cambios


def test_venta_valida_convierte_tipos() -> None:
    # Arrange
    entrada = fila()
    # Act
    venta = validar_venta(entrada)
    # Assert
    assert (venta.cantidad, venta.precio_unitario, venta.subtotal) == (
        2,
        Decimal("125.50"),
        Decimal("251.00"),
    )


@pytest.mark.parametrize(
    ("campo", "valor"),
    [
        pytest.param("cantidad", "1", id="cantidad-minima"),
        pytest.param("cantidad", "1000", id="cantidad-maxima"),
        pytest.param("precio_unitario", "0.01", id="precio-minimo"),
        pytest.param("precio_unitario", "100000", id="precio-maximo"),
        pytest.param("fecha", "2028-02-29", id="bisiesto"),
    ],
)
def test_limites_validos(campo: str, valor: str) -> None:
    validar_venta(fila(**{campo: valor}))  # no debe lanzar


@pytest.mark.parametrize(
    ("campo", "valor", "mensaje"),
    [
        pytest.param("cantidad", "0", "fuera de", id="cantidad-cero"),
        pytest.param("cantidad", "1001", "fuera de", id="cantidad-encima"),
        pytest.param("cantidad", "2.5", "no es entera", id="cantidad-decimal"),
        pytest.param("precio_unitario", "0", "fuera de", id="precio-cero"),
        pytest.param("precio_unitario", "-15", "fuera de", id="precio-negativo"),
        pytest.param("precio_unitario", "100000.01", "fuera de", id="precio-encima"),
        pytest.param("precio_unitario", "NaN", "fuera de", id="precio-nan"),
        pytest.param("precio_unitario", "N/A", "no numérico", id="precio-texto"),
        pytest.param("fecha", "2026-02-30", "fecha inválida", id="fecha-imposible"),
        pytest.param("fecha", "15/03/2026", "fecha inválida", id="fecha-formato"),
    ],
)
def test_casos_negativos(campo: str, valor: str, mensaje: str) -> None:
    with pytest.raises(VentaInvalidaError, match=mensaje):
        validar_venta(fila(**{campo: valor}))


def test_error_propio_tambien_es_value_error() -> None:
    with pytest.raises(ValueError):  # quien ya capturaba ValueError sigue funcionando
        validar_venta(fila(cantidad="cero"))


@pytest.mark.parametrize(
    ("subtotal", "esperado"),
    [
        pytest.param("0.00", "0.00", id="cero"),
        pytest.param("4999.99", "0.00", id="justo-debajo-de-5000"),
        pytest.param("5000.00", "250.00", id="limite-5000"),
        pytest.param("19999.99", "1000.00", id="justo-debajo-de-20000"),
        pytest.param("20000.00", "2000.00", id="limite-20000"),
        pytest.param("5000.10", "250.01", id="redondeo-half-up"),
    ],
)
def test_descuento_en_los_limites(subtotal: str, esperado: str) -> None:
    assert calcular_descuento(Decimal(subtotal)) == Decimal(esperado)


@pytest.mark.parametrize(
    ("valor", "excepcion"),
    [
        pytest.param(Decimal("-1"), ValueError, id="negativo"),
        pytest.param(Decimal("NaN"), ValueError, id="nan"),
        pytest.param(5000.0, TypeError, id="float"),
        pytest.param(True, TypeError, id="booleano"),
    ],
)
def test_descuento_rechaza_entradas_invalidas(valor: object, excepcion: type) -> None:
    with pytest.raises(excepcion):
        calcular_descuento(valor)  # type: ignore[arg-type]
```

> **Checkpoint 2.**
>
> ```bash
> python -m pytest tests/test_validacion.py -v
> ```
>
> Resultado esperado: `27 passed`. En la lista verás un nombre por caso, por ejemplo `test_descuento_en_los_limites[limite-5000]` o `test_casos_negativos[precio-nan]`.

---

## Parte 3. El cliente de la API y los mocks (15 min)

### 3.1 El cliente

`ClienteCatalogo` consulta el producto de cada SKU. Cada respuesta tiene su tratamiento:

| Respuesta | Qué hace |
| :--- | :--- |
| `ConnectionError`, `Timeout`, 429, 500, 502, 503, 504 | Reintenta con espera exponencial (0.5 s, 1 s). Si llega `Retry-After`, espera lo que indique |
| 404 | `ProductoNoEncontradoError`: el SKU no existe. Es un problema del dato y no se reintenta |
| Otro 4xx | `APIError` sin reintentar |
| 200 con cuerpo inválido | `APIRespuestaInvalidaError` |

Dos detalles de diseño:

- **`dormir` se inyecta en el constructor:** así las pruebas verifican las esperas sin esperar de verdad.
- **La causa original se conserva** con `raise ... from`.

**Archivo:** `src/pipeline_ventas/cliente.py`

```python
"""Cliente de la API de catálogo: timeouts, reintentos y validación (5.3)."""

import logging
import time
from collections.abc import Callable
from decimal import Decimal, InvalidOperation
from typing import Any

import requests

logger = logging.getLogger(__name__)

TRANSITORIOS = {429, 500, 502, 503, 504}


class APIError(Exception):
    """La API no respondió como se esperaba: el pipeline se detiene."""


class APIRespuestaInvalidaError(APIError):
    """HTTP 200, pero el cuerpo no es el JSON que dice el contrato."""


class ProductoNoEncontradoError(APIError):
    """HTTP 404: el SKU no existe. Es un problema del dato, no de la API."""


class ClienteCatalogo:
    def __init__(
        self,
        url_base: str,
        timeout: tuple[float, float] = (3.05, 5.0),
        max_intentos: int = 3,
        espera_s: float = 0.5,
        dormir: Callable[[float], None] = time.sleep,  # inyectado: la prueba no espera
    ) -> None:
        self.url_base = url_base.rstrip("/")
        self.timeout = timeout
        self.max_intentos = max_intentos
        self.espera_s = espera_s
        self.dormir = dormir

    def obtener_producto(self, sku: str) -> dict[str, Any]:
        url = f"{self.url_base}/productos/{sku}"
        for intento in range(1, self.max_intentos + 1):
            espera = self.espera_s * 2 ** (intento - 1)  # 0.5 s, 1 s, 2 s...
            try:
                respuesta = requests.get(url, timeout=self.timeout)
            except (requests.ConnectionError, requests.Timeout) as error:
                causa: Exception = error
            else:
                if respuesta.status_code == 404:  # dato: no se reintenta
                    raise ProductoNoEncontradoError(f"{sku} no existe en el catálogo")
                if (
                    respuesta.status_code >= 400
                    and respuesta.status_code not in TRANSITORIOS
                ):
                    raise APIError(f"{sku}: HTTP {respuesta.status_code}")  # permanente
                if respuesta.status_code not in TRANSITORIOS:
                    return self._validar(respuesta, sku)
                causa = APIError(f"HTTP {respuesta.status_code}")
                retry_after = respuesta.headers.get("Retry-After", "")
                if retry_after.isdigit():
                    espera = float(retry_after)  # el servidor manda
            if intento == self.max_intentos:
                raise APIError(
                    f"{sku}: sin respuesta válida tras {intento} intentos"
                ) from causa
            motivo = causa if isinstance(causa, APIError) else type(causa).__name__
            logger.warning("%s: %s; reintento en %.1f s", sku, motivo, espera)
            self.dormir(espera)
        raise AssertionError("inalcanzable")  # pragma: no cover

    @staticmethod
    def _validar(respuesta: requests.Response, sku: str) -> dict[str, Any]:
        try:
            datos = respuesta.json()
        except ValueError:
            raise APIRespuestaInvalidaError(f"{sku}: el cuerpo no es JSON") from None
        faltan = {"sku", "categoria", "costo_unitario"} - set(datos)
        if faltan or datos["sku"] != sku:
            raise APIRespuestaInvalidaError(
                f"{sku}: respuesta incompleta, faltan {faltan}"
            )
        try:
            costo = Decimal(datos["costo_unitario"])
        except (InvalidOperation, TypeError):
            raise APIRespuestaInvalidaError(
                f"{sku}: costo_unitario no numérico"
            ) from None
        return {"sku": sku, "categoria": datos["categoria"], "costo_unitario": costo}
```

### 3.2 Pruebas con mocks

Se parchea `requests.get` **donde se usa**: `pipeline_ventas.cliente.requests.get`, no `requests.get`. Con `side_effect` en lista, cada llamada devuelve el siguiente elemento o lanza la excepción de esa posición. Con `autospec=True`, el mock exige la misma firma que la función real:

**Archivo:** `tests/test_cliente.py`

```python
"""Mocks: se parchea requests.get DONDE SE USA, en pipeline_ventas.cliente."""

from decimal import Decimal
from typing import Any
from unittest.mock import MagicMock, Mock, call, patch

import pytest
import requests

from pipeline_ventas.cliente import (
    APIError,
    APIRespuestaInvalidaError,
    ClienteCatalogo,
    ProductoNoEncontradoError,
)

URL = "https://catalogo.test"
PRODUCTO = {"sku": "SKU-001", "categoria": "Redes", "costo_unitario": "52.00"}


def respuesta(status: int = 200, cuerpo: Any = PRODUCTO, **cabeceras: str) -> MagicMock:
    simulada = MagicMock(status_code=status, headers=cabeceras)
    simulada.json.return_value = cuerpo
    return simulada


@pytest.fixture
def dormir() -> Mock:
    return Mock()  # registra las esperas sin esperar de verdad


@patch("pipeline_ventas.cliente.requests.get", autospec=True)
def test_respuesta_correcta(mock_get: MagicMock, dormir: Mock) -> None:
    mock_get.return_value = respuesta()
    producto = ClienteCatalogo(URL, dormir=dormir).obtener_producto("SKU-001")
    assert producto == {
        "sku": "SKU-001",
        "categoria": "Redes",
        "costo_unitario": Decimal("52.00"),
    }
    mock_get.assert_called_once_with(f"{URL}/productos/SKU-001", timeout=(3.05, 5.0))
    dormir.assert_not_called()


def test_recupera_tras_dos_fallos_de_red(dormir: Mock) -> None:
    efectos = [
        requests.ConnectionError("red caída"),
        requests.Timeout("lenta"),
        respuesta(),
    ]
    with patch("pipeline_ventas.cliente.requests.get", side_effect=efectos) as mock_get:
        ClienteCatalogo(URL, dormir=dormir).obtener_producto("SKU-001")
    assert mock_get.call_count == 3
    assert dormir.call_args_list == [call(0.5), call(1.0)]  # backoff exponencial


def test_respeta_retry_after_del_429(dormir: Mock) -> None:
    efectos = [respuesta(503), respuesta(429, **{"Retry-After": "2"}), respuesta()]
    with patch("pipeline_ventas.cliente.requests.get", side_effect=efectos):
        ClienteCatalogo(URL, dormir=dormir).obtener_producto("SKU-001")
    assert dormir.call_args_list == [call(0.5), call(2.0)]


def test_agota_reintentos_y_conserva_la_causa(dormir: Mock) -> None:
    with patch("pipeline_ventas.cliente.requests.get", return_value=respuesta(500)):
        with pytest.raises(APIError, match="tras 3 intentos") as info:
            ClienteCatalogo(URL, dormir=dormir).obtener_producto("SKU-001")
    assert str(info.value.__cause__) == "HTTP 500"
    assert dormir.call_count == 2  # 3 intentos, 2 esperas


@pytest.mark.parametrize(
    ("status", "excepcion"),
    [
        pytest.param(404, ProductoNoEncontradoError, id="404-dato"),
        pytest.param(400, APIError, id="400-permanente"),
        pytest.param(403, APIError, id="403-permanente"),
    ],
)
def test_errores_permanentes_no_se_reintentan(
    status: int, excepcion: type, dormir: Mock
) -> None:
    with patch(
        "pipeline_ventas.cliente.requests.get", return_value=respuesta(status)
    ) as mock_get:
        with pytest.raises(excepcion):
            ClienteCatalogo(URL, dormir=dormir).obtener_producto("SKU-001")
    assert mock_get.call_count == 1


def test_json_corrupto(dormir: Mock) -> None:
    corrupta = respuesta()
    corrupta.json.side_effect = ValueError("Expecting value")  # HTML de un proxy
    with patch("pipeline_ventas.cliente.requests.get", return_value=corrupta):
        with pytest.raises(APIRespuestaInvalidaError, match="no es JSON"):
            ClienteCatalogo(URL, dormir=dormir).obtener_producto("SKU-001")


@pytest.mark.parametrize(
    "cuerpo",
    [
        pytest.param({"sku": "SKU-001", "categoria": "Redes"}, id="falta-costo"),
        pytest.param(PRODUCTO | {"sku": "SKU-002"}, id="otro-sku"),
        pytest.param(PRODUCTO | {"costo_unitario": "gratis"}, id="costo-texto"),
    ],
)
def test_respuestas_incompletas(cuerpo: dict[str, str], dormir: Mock) -> None:
    with patch(
        "pipeline_ventas.cliente.requests.get", return_value=respuesta(cuerpo=cuerpo)
    ):
        with pytest.raises(APIRespuestaInvalidaError):
            ClienteCatalogo(URL, dormir=dormir).obtener_producto("SKU-001")


def test_autospec_hace_cumplir_la_firma() -> None:
    with patch("pipeline_ventas.cliente.requests.get", autospec=True) as mock_get:
        with pytest.raises(TypeError, match="url"):
            mock_get()  # requests.get exige url, igual que el real
```

> **Checkpoint 3.**
>
> ```bash
> python -m pytest tests/test_cliente.py -v
> ```
>
> Resultado esperado: `12 passed` en milisegundos, aunque el código "espera" 0.5, 1 y 2 segundos. Fíjate en `test_recupera_tras_dos_fallos_de_red`: `dormir.call_args_list == [call(0.5), call(1.0)]` comprueba el backoff exponencial.

---

## Parte 4. El pipeline base y las pruebas de integración (20 min)

### 4.1 La API simulada (el sistema externo)

`api_simulada.py` representa la API de catálogo: 40 productos, 50 ms por respuesta y 404 para un SKU que no existe. **Cópiala tal cual.** Su función `respuesta_catalogo()` también la usan las pruebas, para que el mock responda exactamente lo mismo que el servidor:

**Archivo:** `api_simulada.py`

```python
"""API de catálogo SIMULADA (solo biblioteca estándar): el sistema externo.

GET /productos/SKU-001 -> {"sku": "SKU-001", "categoria": ..., "costo_unitario": ...}
Cada respuesta tarda --latencia-ms (50 por defecto), como una API real en la red.
Un SKU que no existe responde 404.

Uso:
    python api_simulada.py                    # puerto 8000, 50 ms por petición
    python api_simulada.py --latencia-ms 0
"""

import argparse
import json
import time
from decimal import Decimal
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from typing import Any

CATEGORIAS = ["Cómputo", "Redes", "Software", "Accesorios", "Servicios"]
CATALOGO = {
    f"SKU-{n:03d}": {
        "sku": f"SKU-{n:03d}",
        "categoria": CATEGORIAS[n % len(CATEGORIAS)],
        "costo_unitario": str(Decimal(n * 37 % 900 + 15).quantize(Decimal("0.01"))),
    }
    for n in range(1, 41)
}


def respuesta_catalogo(ruta: str) -> tuple[int, dict[str, Any]]:
    """Lógica de la API sin red: también la usan las pruebas con responses."""
    sku = ruta.rstrip("/").rsplit("/", 1)[-1]
    if not ruta.startswith("/productos/") or sku not in CATALOGO:
        return 404, {"error": f"producto {sku} no encontrado"}
    return 200, CATALOGO[sku]


def crear_servidor(puerto: int, latencia_s: float) -> ThreadingHTTPServer:
    class Manejador(BaseHTTPRequestHandler):
        def log_message(self, format: str, *args: Any) -> None:
            pass  # sin ruido: miles de peticiones

        def do_GET(self) -> None:
            time.sleep(latencia_s)  # latencia de red simulada
            codigo, cuerpo = respuesta_catalogo(self.path)
            datos = json.dumps(cuerpo, ensure_ascii=False).encode("utf-8")
            self.send_response(codigo)
            self.send_header("Content-Type", "application/json; charset=utf-8")
            self.send_header("Content-Length", str(len(datos)))
            self.end_headers()
            self.wfile.write(datos)

    servidor = ThreadingHTTPServer(("127.0.0.1", puerto), Manejador)
    servidor.daemon_threads = True
    return servidor


def main() -> None:
    parser = argparse.ArgumentParser(
        description="API de catálogo simulada (Práctica 5)"
    )
    parser.add_argument("--puerto", type=int, default=8000)
    parser.add_argument("--latencia-ms", type=int, default=50)
    args = parser.parse_args()
    servidor = crear_servidor(args.puerto, args.latencia_ms / 1000)
    print(
        f"[api] Catálogo de {len(CATALOGO)} productos en http://127.0.0.1:{args.puerto}"
        f" ({args.latencia_ms} ms por petición). Ctrl+C para detener."
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

### 4.2 El pipeline en su versión base

Es **correcto y fácil de leer, pero no rápido**. Observa dos cosas:

- **Las estrategias se inyectan:** `deduplicar` y `enriquecer` se reciben como parámetros. En la Parte 6, la versión optimizada reemplazará solo esas dos piezas.
- **`cargar()` recibe un generador** y escribe en lotes de 500 con `itertools.batched`, dentro de una sola transacción (temas 5.9 y 5.10).

**Archivo:** `src/pipeline_ventas/pipeline.py`

```python
"""El pipeline en su versión BASE: correcta y fácil de leer... pero lenta.

leer → deduplicar → validar → enriquecer con la API → cargar → reporte.
deduplicar y enriquecer se reciben como parámetros: así la versión optimizada
(optimizado.py) reemplaza solo esas dos piezas y todo lo demás es idéntico.
"""

import csv
import logging
import sqlite3
import uuid
from collections import Counter, defaultdict
from collections.abc import Callable, Iterable
from contextlib import closing
from datetime import UTC, datetime
from decimal import Decimal
from itertools import batched
from pathlib import Path
from typing import Any

from pipeline_ventas.cliente import ClienteCatalogo, ProductoNoEncontradoError
from pipeline_ventas.validacion import (
    CENTAVOS,
    Venta,
    VentaInvalidaError,
    calcular_descuento,
    validar_venta,
)

logger = logging.getLogger(__name__)

Fila = dict[str, str]
Deduplicador = Callable[[list[Fila]], tuple[list[Fila], list[Fila]]]
Enriquecedor = Callable[[ClienteCatalogo, list[str]], dict[str, dict[str, Any] | None]]
TAMANO_LOTE = 500


def leer_csv(ruta: Path) -> list[Fila]:
    with open(ruta, encoding="utf-8", newline="") as archivo:
        return list(csv.DictReader(archivo))


def deduplicar_base(filas: list[Fila]) -> tuple[list[Fila], list[Fila]]:
    """Conserva la primera aparición de cada venta_id."""
    vistos: list[str] = []
    unicas, duplicadas = [], []
    for fila in filas:
        if fila["venta_id"] in vistos:  # búsqueda en lista: recorre todo lo visto
            duplicadas.append(fila)
        else:
            vistos.append(fila["venta_id"])
            unicas.append(fila)
    return unicas, duplicadas


def enriquecer_secuencial(
    cliente: ClienteCatalogo, skus: list[str]
) -> dict[str, dict[str, Any] | None]:
    """Una petición por SKU, una detrás de otra: casi todo el tiempo es espera."""
    productos: dict[str, dict[str, Any] | None] = {}
    for sku in skus:
        try:
            productos[sku] = cliente.obtener_producto(sku)
        except ProductoNoEncontradoError:
            productos[sku] = None  # las ventas de ese SKU se rechazan
    return productos


def cargar(ruta_bd: Path, filas: Iterable[dict[str, Any]]) -> int:
    """UPSERT por lotes en una sola transacción: repetir la carga no duplica."""
    ruta_bd.parent.mkdir(parents=True, exist_ok=True)
    total = 0
    with closing(sqlite3.connect(ruta_bd)) as conn, conn:
        conn.execute(
            "CREATE TABLE IF NOT EXISTS ventas (venta_id TEXT PRIMARY KEY, fecha TEXT, "
            "sucursal TEXT, sku TEXT, categoria TEXT, cantidad INTEGER, "
            "total TEXT, margen TEXT)"
        )
        for lote in batched(filas, TAMANO_LOTE):  # Python 3.12+
            conn.executemany(
                "INSERT INTO ventas VALUES (:venta_id, :fecha, :sucursal, :sku, "
                ":categoria, :cantidad, :total, :margen) ON CONFLICT(venta_id) DO "
                "UPDATE SET total = excluded.total, margen = excluded.margen",
                lote,
            )
            total += len(lote)
    return total


def _enriquecer_venta(venta: Venta, producto: dict[str, Any]) -> dict[str, Any]:
    descuento = calcular_descuento(venta.subtotal)
    total = venta.subtotal - descuento
    margen = (total - producto["costo_unitario"] * venta.cantidad).quantize(CENTAVOS)
    return {
        "venta_id": venta.venta_id,
        "fecha": venta.fecha.isoformat(),
        "sucursal": venta.sucursal,
        "sku": venta.sku,
        "categoria": producto["categoria"],
        "cantidad": venta.cantidad,
        "subtotal": venta.subtotal,
        "descuento": descuento,
        "total": total,
        "margen": margen,
    }


def construir_reporte(
    leidas: int,
    cargadas: list[dict[str, Any]],
    rechazos: list[dict[str, str]],
    skus_consultados: int,
) -> dict[str, Any]:
    por_sucursal: dict[str, dict[str, Any]] = defaultdict(
        lambda: {
            "ventas": 0,
            "subtotal": Decimal(0),
            "descuento": Decimal(0),
            "total": Decimal(0),
            "margen": Decimal(0),
        }
    )
    for fila in cargadas:
        acumulado = por_sucursal[fila["sucursal"]]
        acumulado["ventas"] += 1
        for campo in ("subtotal", "descuento", "total", "margen"):
            acumulado[campo] += fila[campo]
    return {
        "metadata": {  # cambia en cada ejecución: la prueba de regresión la enmascara
            "generado": datetime.now(UTC).isoformat(timespec="seconds"),
            "ejecucion": str(uuid.uuid4()),
        },
        "resumen": {
            "leidas": leidas,
            "cargadas": len(cargadas),
            "rechazadas": len(rechazos),
            "skus_consultados": skus_consultados,
        },
        "rechazos_por_tipo": dict(
            sorted(Counter(r["motivo"].split()[0] for r in rechazos).items())
        ),
        "por_sucursal": {  # dinero como texto con dos decimales: comparación exacta
            sucursal: {k: v if k == "ventas" else f"{v:.2f}" for k, v in datos.items()}
            for sucursal, datos in sorted(por_sucursal.items())
        },
        "rechazos": sorted(rechazos, key=lambda r: (r["venta_id"], r["motivo"])),
    }


def ejecutar(
    ruta_csv: Path,
    cliente: ClienteCatalogo,
    ruta_bd: Path,
    deduplicar: Deduplicador = deduplicar_base,
    enriquecer: Enriquecedor = enriquecer_secuencial,
) -> dict[str, Any]:
    filas = leer_csv(ruta_csv)
    unicas, duplicadas = deduplicar(filas)
    rechazos = [{"venta_id": f["venta_id"], "motivo": "duplicado"} for f in duplicadas]
    validas: list[Venta] = []
    for fila in unicas:
        try:
            validas.append(validar_venta(fila))
        except VentaInvalidaError as error:  # una fila mala no tumba el lote
            rechazos.append({"venta_id": fila["venta_id"], "motivo": str(error)})
    skus = sorted({v.sku for v in validas})
    productos = enriquecer(cliente, skus)  # APIError (no 404): se detiene aquí
    enriquecidas: list[dict[str, Any]] = []
    for venta in validas:
        producto = productos[venta.sku]
        if producto is None:
            rechazos.append(
                {
                    "venta_id": venta.venta_id,
                    "motivo": f"producto {venta.sku} no existe",
                }
            )
        else:
            enriquecidas.append(_enriquecer_venta(venta, producto))
    cargar(
        ruta_bd,
        (
            {**f, "total": str(f["total"]), "margen": str(f["margen"])}
            for f in enriquecidas
        ),
    )
    logger.info(
        "leídas=%d cargadas=%d rechazadas=%d",
        len(filas),
        len(enriquecidas),
        len(rechazos),
    )
    return construir_reporte(len(filas), enriquecidas, rechazos, len(skus))
```

### 4.3 Medición y punto de entrada

Las funciones de medición las usarás en la Parte 6; el comando `ejecutar` ya las usa para tomar el tiempo:

**Archivo:** `src/pipeline_ventas/medicion.py`

```python
"""Medir antes de optimizar: cProfile, perf_counter y tracemalloc (5.14 y 5.15)."""

import cProfile
import io
import pstats
import time
import tracemalloc
from collections.abc import Callable
from pathlib import Path
from typing import Any

# Solo las funciones de nuestro paquete: lo de requests o csv no lo podemos cambiar
NUESTRO_CODIGO = r"(pipeline|cliente|validacion|optimizado)\.py"


def perfilar(funcion: Callable[[], Any], destino: Path, top: int = 8) -> str:
    """Perfil por tiempo acumulado; se guarda también para SnakeViz o pstats."""
    with cProfile.Profile() as perfil:  # solo perfila el hilo principal
        funcion()
    destino.parent.mkdir(parents=True, exist_ok=True)
    perfil.dump_stats(destino)
    salida = io.StringIO()
    estadisticas = pstats.Stats(perfil, stream=salida).strip_dirs()
    estadisticas.sort_stats(pstats.SortKey.CUMULATIVE).print_stats(NUESTRO_CODIGO, top)
    return salida.getvalue()


def medir_tiempo(funcion: Callable[[], Any]) -> tuple[Any, float]:
    inicio = time.perf_counter()
    resultado = funcion()
    return resultado, time.perf_counter() - inicio


def medir_memoria(funcion: Callable[[], Any]) -> tuple[Any, float]:
    """Pico de memoria de Python en MB. tracemalloc frena el código: medir aparte."""
    tracemalloc.start()
    try:
        resultado = funcion()
        _, pico = tracemalloc.get_traced_memory()
    finally:
        tracemalloc.stop()
    return resultado, pico / 1024**2


def sin_metadata(reporte: dict[str, Any]) -> dict[str, Any]:
    """Lo que debe ser idéntico entre versiones: todo menos fecha e identificador."""
    return {k: v for k, v in reporte.items() if k != "metadata"}
```

`cli.py` importa `optimizado` solo cuando se pide `--modo optimizada`, porque ese módulo se crea en la Parte 6:

**Archivo:** `src/pipeline_ventas/cli.py`

```python
"""Punto de entrada: python -m pipeline_ventas <comando>."""

import argparse
import json
import logging
from collections.abc import Callable
from pathlib import Path
from typing import Any

from pipeline_ventas import medicion, pipeline
from pipeline_ventas.cliente import APIError, ClienteCatalogo
from pipeline_ventas.datos import escribir_csv, generar_ventas

SALIDA = Path("salida")


def preparar(modo: str, entrada: Path, api: str) -> Callable[[], dict[str, Any]]:
    """Devuelve el pipeline listo para correr en el modo pedido."""
    cliente = ClienteCatalogo(api)
    bd = SALIDA / f"ventas_{modo}.db"
    if modo == "base":
        return lambda: pipeline.ejecutar(entrada, cliente, bd)
    from pipeline_ventas import optimizado  # se crea en la Parte 6

    return lambda: pipeline.ejecutar(
        entrada,
        cliente,
        bd,
        optimizado.deduplicar_rapido,
        optimizado.enriquecer_concurrente,
    )


def comando_ejecutar(args: argparse.Namespace) -> int:
    reporte, segundos = medicion.medir_tiempo(
        preparar(args.modo, args.entrada, args.api)
    )
    destino = SALIDA / f"reporte_{args.modo}.json"
    destino.write_text(
        json.dumps(reporte, indent=2, ensure_ascii=False) + "\n", encoding="utf-8"
    )
    resumen = reporte["resumen"]
    print(
        f"modo={args.modo}: leídas={resumen['leidas']} cargadas={resumen['cargadas']} "
        f"rechazadas={resumen['rechazadas']} skus={resumen['skus_consultados']} "
        f"en {segundos:.2f} s"
    )
    print(f"rechazos por tipo: {reporte['rechazos_por_tipo']}")
    print(f"reporte: {destino.as_posix()}")
    return 0


def comando_perfilar(args: argparse.Namespace) -> int:
    texto = medicion.perfilar(
        preparar(args.modo, args.entrada, args.api),
        SALIDA / f"perfil_{args.modo}.prof",
        args.top,
    )
    print(texto.strip())
    return 0


def comando_comparar(args: argparse.Namespace) -> int:
    filas = []
    reportes = {}
    for modo in ("base", "optimizada"):
        reporte, segundos = medicion.medir_tiempo(
            preparar(modo, args.entrada, args.api)
        )
        _, pico_mb = medicion.medir_memoria(preparar(modo, args.entrada, args.api))
        reportes[modo] = medicion.sin_metadata(reporte)
        filas.append((modo, segundos, pico_mb))
    print(f"{'modo':<11} {'tiempo':>9} {'pico memoria':>13}")
    for modo, segundos, pico_mb in filas:
        print(f"{modo:<11} {segundos:>7.2f} s {pico_mb:>10.2f} MB")
    print(f"aceleración: x{filas[0][1] / filas[1][1]:.1f}")
    iguales = reportes["base"] == reportes["optimizada"]
    print(f"reportes idénticos (sin metadata): {'sí' if iguales else 'NO'}")
    return 0 if iguales else 4


def construir_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(prog="python -m pipeline_ventas")
    sub = parser.add_subparsers(dest="comando", required=True)
    generar = sub.add_parser("generar", help="crea el CSV de ventas sintético")
    generar.add_argument("--filas", type=int, default=10_000)
    generar.add_argument("--semilla", type=int, default=42)
    generar.add_argument("--salida", type=Path, default=Path("datos/ventas.csv"))
    for nombre, ayuda in (
        ("ejecutar", "corre el pipeline"),
        ("perfilar", "corre el pipeline con cProfile"),
        ("comparar", "mide base frente a optimizada"),
    ):
        p = sub.add_parser(nombre, help=ayuda)
        p.add_argument("--entrada", type=Path, default=Path("datos/ventas.csv"))
        p.add_argument("--api", default="http://127.0.0.1:8000")
        if nombre != "comparar":
            p.add_argument("--modo", choices=["base", "optimizada"], default="base")
        if nombre == "perfilar":
            p.add_argument("--top", type=int, default=8)
    return parser


def main(argv: list[str] | None = None) -> int:
    args = construir_parser().parse_args(argv)
    logging.basicConfig(
        level=logging.WARNING, format="%(levelname)s %(name)s: %(message)s"
    )
    if args.comando == "generar":
        escribir_csv(args.salida, generar_ventas(args.filas, args.semilla))
        print(
            f"{args.filas} ventas escritas en {args.salida.as_posix()} "
            f"(semilla {args.semilla})"
        )
        return 0
    if not args.entrada.exists():
        print(
            f"No existe {args.entrada}: use antes 'python -m pipeline_ventas generar'"
        )
        return 1
    SALIDA.mkdir(exist_ok=True)
    comandos = {
        "ejecutar": comando_ejecutar,
        "perfilar": comando_perfilar,
        "comparar": comando_comparar,
    }
    try:
        return comandos[args.comando](args)
    except APIError as error:
        causa = error.__cause__  # se conservó con raise ... from
        detalle = f": {causa}" if isinstance(causa, APIError) else ""
        print(f"Error de la API: {error} (causa: {type(causa).__name__}{detalle})")
        return 3
```

**Archivo:** `src/pipeline_ventas/__main__.py`

```python
"""Permite ejecutar el paquete con: python -m pipeline_ventas."""

from pipeline_ventas.cli import main

raise SystemExit(main())
```

### 4.4 Fixtures compartidas

`conftest.py` concentra lo que usan varias pruebas:

- **`csv_ventas`:** 300 ventas sintéticas en una carpeta temporal única (`tmp_path`).
- **`api_catalogo`:** intercepta `requests` con `responses` y contesta con los datos de la API simulada. Una llamada no registrada hace fallar la prueba.
- **`cliente`:** un cliente sin esperas reales.
- **`estrategias`:** hace que cada prueba que la usa corra con las dos versiones del pipeline. Mientras no exista `optimizado.py`, la versión optimizada se marca como omitida.
- **`--actualizar-golden`:** registra la opción que usarás en la Parte 5.

**Archivo:** `tests/conftest.py`

```python
"""Fixtures compartidas: datos sintéticos, API simulada con responses y estrategias."""

import json
import re
from collections.abc import Iterator
from pathlib import Path
from typing import Any

import pytest
import responses

from api_simulada import respuesta_catalogo
from pipeline_ventas import pipeline
from pipeline_ventas.cliente import ClienteCatalogo
from pipeline_ventas.datos import escribir_csv, generar_ventas

API = "https://catalogo.test"


def pytest_addoption(parser: pytest.Parser) -> None:
    parser.addoption(
        "--actualizar-golden",
        action="store_true",
        help="Reescribe el golden master tras revisar y aprobar el cambio",
    )


@pytest.fixture
def csv_ventas(tmp_path: Path) -> Path:
    """300 ventas sintéticas con semilla fija: los mismos datos en cada ejecución."""
    ruta = tmp_path / "ventas.csv"
    escribir_csv(ruta, generar_ventas(300, semilla=7))
    return ruta


@pytest.fixture
def api_catalogo() -> Iterator[responses.RequestsMock]:
    """Intercepta requests: la API responde con los datos de api_simulada, sin red."""

    def responder(peticion: Any) -> tuple[int, dict[str, str], str]:
        codigo, cuerpo = respuesta_catalogo(peticion.path_url)
        return codigo, {"Content-Type": "application/json"}, json.dumps(cuerpo)

    # Una llamada no registrada falla; no se exige usarla (las omitidas no la usan)
    with responses.RequestsMock(assert_all_requests_are_fired=False) as simulada:
        simulada.add_callback(
            responses.GET, re.compile(rf"{re.escape(API)}/productos/.+"), responder
        )
        yield simulada


@pytest.fixture
def cliente() -> ClienteCatalogo:
    return ClienteCatalogo(API, dormir=lambda segundos: None)  # sin esperas reales


@pytest.fixture(params=["base", "optimizada"])
def estrategias(request: pytest.FixtureRequest) -> dict[str, Any]:
    """Cada prueba que la usa corre con las dos versiones del pipeline."""
    if request.param == "base":
        return {}
    optimizado = pytest.importorskip(
        "pipeline_ventas.optimizado", reason="optimizado.py se crea en la Parte 6"
    )
    return {
        "deduplicar": optimizado.deduplicar_rapido,
        "enriquecer": optimizado.enriquecer_concurrente,
    }


def ejecutar(csv: Path, cliente: ClienteCatalogo, bd: Path, **estrategias: Any) -> Any:
    return pipeline.ejecutar(csv, cliente, bd, **estrategias)
```

### 4.5 Pruebas de integración

`test_pipeline.py` verifica en tres niveles (tema 5.2):

1. **El resultado:** los conteos y los rechazos por tipo.
2. **Las interacciones:** una llamada HTTP por SKU, no por venta.
3. **El estado en la base de datos:** cuántas filas quedaron.

También comprueba que repetir la carga no duplica y que una API caída detiene todo **sin cargar nada**:

**Archivo:** `tests/test_pipeline.py`

```python
"""Integración: CSV real en tmp_path, SQLite real y la API interceptada con responses.

Se verifica en tres niveles: el resultado, las llamadas HTTP y el estado en la BD.
"""

import re
import sqlite3
from pathlib import Path
from typing import Any

import pytest
import responses

from pipeline_ventas.cliente import APIError, ClienteCatalogo
from tests.conftest import API, ejecutar


def contar(bd: Path) -> int:
    with sqlite3.connect(bd) as conn:
        return int(conn.execute("SELECT COUNT(*) FROM ventas").fetchone()[0])


def test_pipeline_completo(
    csv_ventas: Path,
    tmp_path: Path,
    api_catalogo: responses.RequestsMock,
    cliente: ClienteCatalogo,
    estrategias: dict[str, Any],
) -> None:
    bd = tmp_path / "ventas.db"
    reporte = ejecutar(csv_ventas, cliente, bd, **estrategias)
    resumen = reporte["resumen"]
    assert resumen == {
        "leidas": 300,
        "cargadas": 284,
        "rechazadas": 16,
        "skus_consultados": 41,
    }  # 1. resultado
    assert reporte["rechazos_por_tipo"] == {
        "cantidad": 3,
        "duplicado": 3,
        "fecha": 3,
        "precio": 3,
        "producto": 4,
    }
    assert len(api_catalogo.calls) == 41  # 2. una llamada por SKU, no por venta
    assert contar(bd) == 284  # 3. estado en la base de datos


def test_repetir_la_carga_no_duplica(
    csv_ventas: Path,
    tmp_path: Path,
    api_catalogo: responses.RequestsMock,
    cliente: ClienteCatalogo,
) -> None:
    bd = tmp_path / "ventas.db"
    ejecutar(csv_ventas, cliente, bd)
    ejecutar(csv_ventas, cliente, bd)  # UPSERT
    assert contar(bd) == 284


def test_api_caida_detiene_sin_cargar(
    csv_ventas: Path, tmp_path: Path, estrategias: dict[str, Any]
) -> None:
    bd = tmp_path / "ventas.db"
    cliente = ClienteCatalogo(API, dormir=lambda segundos: None)
    with responses.RequestsMock() as simulada:
        simulada.get(re.compile(rf"{re.escape(API)}/productos/.+"), status=503)
        with pytest.raises(APIError, match="tras 3 intentos"):
            ejecutar(csv_ventas, cliente, bd, **estrategias)
    assert not bd.exists()  # nada a medias: la carga va después del enriquecimiento
```

`test_api_real.py` va un paso más allá: levanta la API simulada **de verdad**, por HTTP, en un hilo y un puerto libre. Como es un recurso caro, la fixture tiene `scope="module"`. Lleva el marcador `integracion`:

**Archivo:** `tests/test_api_real.py`

```python
"""Integración con la API simulada REAL (HTTP de verdad, en un hilo y puerto libre)."""

import threading
from collections.abc import Iterator
from pathlib import Path
from typing import Any

import pytest

from api_simulada import crear_servidor
from pipeline_ventas.cliente import ClienteCatalogo
from pipeline_ventas.medicion import sin_metadata
from tests.conftest import ejecutar

pytestmark = pytest.mark.integracion


@pytest.fixture(scope="module")
def url_api() -> Iterator[str]:
    """Recurso caro: un servidor para todo el módulo (scope="module")."""
    servidor = crear_servidor(0, latencia_s=0.005)  # puerto 0: uno libre
    hilo = threading.Thread(target=servidor.serve_forever, daemon=True)
    hilo.start()
    yield f"http://127.0.0.1:{servidor.server_address[1]}"
    servidor.shutdown()  # teardown: se ejecuta aunque la prueba falle
    servidor.server_close()


def test_mismo_reporte_por_http_real(
    csv_ventas: Path, tmp_path: Path, url_api: str, estrategias: dict[str, Any]
) -> None:
    cliente = ClienteCatalogo(url_api)
    base = sin_metadata(ejecutar(csv_ventas, cliente, tmp_path / "base.db"))
    otra = sin_metadata(
        ejecutar(csv_ventas, cliente, tmp_path / "otra.db", **estrategias)
    )
    assert otra == base
    assert base["resumen"]["cargadas"] == 284
```

```bash
python -m pytest tests/test_pipeline.py tests/test_api_real.py -v
```

Resultado esperado:

```text
tests/test_pipeline.py::test_pipeline_completo[base] PASSED              [ 14%]
tests/test_pipeline.py::test_pipeline_completo[optimizada] SKIPPED (...) [ 28%]
tests/test_pipeline.py::test_repetir_la_carga_no_duplica PASSED          [ 42%]
tests/test_pipeline.py::test_api_caida_detiene_sin_cargar[base] PASSED   [ 57%]
tests/test_pipeline.py::test_api_caida_detiene_sin_cargar[optimizada] SKIPPED [ 71%]
tests/test_api_real.py::test_mismo_reporte_por_http_real[base] PASSED    [ 85%]
tests/test_api_real.py::test_mismo_reporte_por_http_real[optimizada] SKIPPED [100%]
SKIPPED [1] tests/test_pipeline.py:23: optimizado.py se crea en la Parte 6
…
========================= 4 passed, 3 skipped in 1.33s =========================
```

Las omitidas (`SKIPPED`) son esperadas: `-ra` las resume al final con su motivo.

### 4.6 Ejecutar el pipeline de verdad

**4.6.1** Genera los datos (Terminal 2):

```bash
python -m pipeline_ventas generar
```

Resultado esperado: `10000 ventas escritas en datos/ventas.csv (semilla 42)`

**4.6.2** En la **Terminal 1**, con el entorno activo, levanta la API y déjala corriendo:

```bash
python api_simulada.py
```

```text
[api] Catálogo de 40 productos en http://127.0.0.1:8000 (50 ms por petición). Ctrl+C para detener.
```

**4.6.3** En la **Terminal 2**:

```bash
python -m pipeline_ventas ejecutar --modo base
```

```text
modo=base: leídas=10000 cargadas=9413 rechazadas=587 skus=41 en 2.88 s
rechazos por tipo: {'cantidad': 111, 'duplicado': 103, 'fecha': 123, 'precio': 118, 'producto': 132}
reporte: salida/reporte_base.json
```

Anota el tiempo: es tu **línea base**.

**4.6.4** Detén la API (**Ctrl+C** en la Terminal 1) y vuelve a ejecutar el pipeline en la Terminal 2:

```bash
python -m pipeline_ventas ejecutar --modo base
```

```text
WARNING pipeline_ventas.cliente: SKU-001: ConnectionError; reintento en 0.5 s
WARNING pipeline_ventas.cliente: SKU-001: ConnectionError; reintento en 1.0 s
Error de la API: SKU-001: sin respuesta válida tras 3 intentos (causa: ConnectionError)
```

El código de salida es `3`. Vuelve a levantar la API en la Terminal 1 con `python api_simulada.py`.

### Códigos de salida de `python -m pipeline_ventas`

| Código | Significado |
| :---: | :--- |
| 0 | Correcto |
| 1 | No existe el CSV de entrada |
| 2 | Argumentos inválidos |
| 3 | La API no respondió como se esperaba |
| 4 | `comparar`: los reportes de las dos versiones no son idénticos |

Para ver el código de salida del último comando: **PowerShell** `$LASTEXITCODE` | **bash** `echo $?`

> **Checkpoint 4.** `9413` ventas cargadas y `587` rechazadas, con la API arriba; código `3`, con la API abajo.

---

## Parte 5. Pruebas de regresión con golden master (10 min)

### 5.1 La prueba

Compara el reporte **completo** contra una referencia aprobada. Funciona así:

- **Enmascara lo dinámico:** fecha e identificador de la ejecución.
- **Si la referencia no existe, o pasas `--actualizar-golden`:** la escribe y marca la prueba como omitida (`SKIPPED`), para que alguien revise el cambio antes de aprobarlo.

**Archivo:** `tests/test_regresion.py`

```python
"""Regresión con golden master: la salida completa contra una referencia aprobada."""

import json
from pathlib import Path
from typing import Any

import pytest
import responses

from pipeline_ventas.cliente import ClienteCatalogo
from pipeline_ventas.medicion import sin_metadata
from tests.conftest import ejecutar

GOLDEN = Path(__file__).parent / "datos" / "golden_reporte.json"


@pytest.mark.regresion
def test_reporte_igual_al_golden_master(
    csv_ventas: Path,
    tmp_path: Path,
    api_catalogo: responses.RequestsMock,
    cliente: ClienteCatalogo,
    estrategias: dict[str, Any],
    request: pytest.FixtureRequest,
) -> None:
    obtenido = sin_metadata(
        ejecutar(csv_ventas, cliente, tmp_path / "v.db", **estrategias)
    )
    if request.config.getoption("--actualizar-golden") or not GOLDEN.exists():
        GOLDEN.parent.mkdir(parents=True, exist_ok=True)
        GOLDEN.write_text(
            json.dumps(obtenido, indent=2, ensure_ascii=False) + "\n", encoding="utf-8"
        )
        pytest.skip("golden master actualizado: revise el diff y haga commit")
    esperado = json.loads(GOLDEN.read_text(encoding="utf-8"))
    assert obtenido == esperado
```

### 5.2 Crear y aprobar el golden master

```bash
python -m pytest -m regresion --actualizar-golden
```

```text
SKIPPED [1] tests/test_regresion.py:34: golden master actualizado: revise el diff y haga commit
SKIPPED [1] tests/test_regresion.py:17: optimizado.py se crea en la Parte 6
====================== 2 skipped, 50 deselected in 0.18s =======================
```

Abre `tests/datos/golden_reporte.json` y revísalo: es la salida aprobada. Empieza así:

```json
{
  "resumen": {
    "leidas": 300,
    "cargadas": 284,
    "rechazadas": 16,
    "skus_consultados": 41
  },
  "rechazos_por_tipo": {
    "cantidad": 3,
    "duplicado": 3,
    "fecha": 3,
    "precio": 3,
    "producto": 4
  },
```

Este archivo **sí se versiona**. Ahora ejecuta la regresión sin la opción:

```bash
python -m pytest -m regresion
```

Resultado esperado: `1 passed, 1 skipped, 50 deselected`.

### 5.3 Un error que el golden master NO detecta

Abre `src/pipeline_ventas/validacion.py` y cambia `>=` por `>` en el límite de 5,000:

```python
    elif subtotal > 5000:
```

Ejecuta la suite completa:

```bash
python -m pytest -q
```

```text
FAILED tests/test_validacion.py::test_descuento_en_los_limites[limite-5000]
1 failed, 47 passed, 4 skipped in 1.62s
```

El golden master **pasó**: en los datos de prueba ninguna venta suma exactamente 5,000.00. La prueba de **valor límite** fue la que detectó el error. **Revierte el cambio** (`>=`) y comprueba que todo vuelve a pasar.

> **Checkpoint 5.** `tests/datos/golden_reporte.json` existe y `python -m pytest -q` termina en `48 passed, 4 skipped`.

---

## Parte 6. Medir, optimizar y comparar (25 min)

### 6.1 Medir antes de optimizar

Con la API corriendo en la Terminal 1:

```bash
python -m pipeline_ventas perfilar --modo base
```

```text
365102 function calls (364743 primitive calls) in 3.036 seconds

   Ordered by: cumulative time
   List reduced from 615 to 16 due to restriction <'(pipeline|cliente|validacion|optimizado)\\.py'>
   List reduced from 16 to 8 due to restriction <8>

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.014    0.014    3.033    3.033 pipeline.py:150(ejecutar)
        1    0.001    0.001    2.224    2.224 pipeline.py:56(enriquecer_secuencial)
       41    0.001    0.000    2.219    0.054 cliente.py:43(obtener_producto)
        1    0.584    0.584    0.588    0.588 pipeline.py:43(deduplicar_base)
     9413    0.024    0.000    0.083    0.000 pipeline.py:90(_enriquecer_venta)
        1    0.002    0.002    0.041    0.041 pipeline.py:38(leer_csv)
        1    0.003    0.003    0.037    0.037 pipeline.py:69(cargar)
     9897    0.019    0.000    0.033    0.000 validacion.py:30(validar_venta)
```

Lee el perfil (tema 5.15):

| Hallazgo | Qué indica | Tipo de cuello de botella |
| :--- | :--- | :--- |
| `enriquecer_secuencial`: **cumtime** 2.2 s, pero **tottime** 0.001 s | El tiempo no se gasta en la función, sino **esperando** a las 41 peticiones (≈ 54 ms cada una) | E/S |
| `deduplicar_base`: **tottime** 0.58 s | El tiempo se gasta **dentro** de la función: `in` sobre una lista recorre todo lo visto, O(n²) | CPU |
| El resto | `validar_venta`, `cargar` y `leer_csv` suman menos de 0.2 s | No vale la pena tocarlo |

El perfil también se guardó en `salida/perfil_base.prof`, por si quieres abrirlo con SnakeViz.

### 6.2 Elegir la herramienta para cada cuello de botella

| Cuello de botella | Opción | ¿Por qué sí o por qué no? |
| :--- | :--- | :--- |
| Esperas de red con `requests` (síncrona) | **Hilos** (`ThreadPoolExecutor`) | Durante la E/S se libera el GIL y las esperas se solapan; casi no cambia el código |
| | `asyncio` | Obligaría a cambiar a una biblioteca asíncrona (httpx) y a reescribir el cliente |
| | Procesos | Pagan arranque y serialización para un trabajo que solo espera |
| Búsqueda O(n) en una lista | **`set`** | Búsqueda O(1) promedio, con una línea distinta |
| | Procesos | No arreglan un algoritmo ineficiente: lo repartirían entre núcleos |

### 6.3 La optimización

Crea `optimizado.py`. Solo contiene las **dos piezas** que el perfil señaló; el pipeline, las reglas y la carga no cambian:

**Archivo:** `src/pipeline_ventas/optimizado.py`

```python
"""La optimización: SOLO las dos piezas que el perfilado señaló (Parte 6).

1. deduplicar_rapido: conjunto (set) en lugar de lista: búsqueda O(1) promedio.
2. enriquecer_concurrente: las esperas de red se solapan con hilos; requests es
   síncrona, así que hilos y no asyncio (tema 5.13).
"""

from concurrent.futures import ThreadPoolExecutor, as_completed
from typing import Any

from pipeline_ventas.cliente import ClienteCatalogo, ProductoNoEncontradoError

Fila = dict[str, str]
MAX_HILOS = 8  # suficiente para la API sin rebasar su capacidad


def deduplicar_rapido(filas: list[Fila]) -> tuple[list[Fila], list[Fila]]:
    vistos: set[str] = set()
    unicas, duplicadas = [], []
    for fila in filas:
        if fila["venta_id"] in vistos:  # O(1) promedio
            duplicadas.append(fila)
        else:
            vistos.add(fila["venta_id"])
            unicas.append(fila)
    return unicas, duplicadas


def enriquecer_concurrente(
    cliente: ClienteCatalogo, skus: list[str], max_hilos: int = MAX_HILOS
) -> dict[str, dict[str, Any] | None]:
    productos: dict[str, dict[str, Any] | None] = {}
    with ThreadPoolExecutor(max_workers=max_hilos) as executor:
        futuros = {executor.submit(cliente.obtener_producto, sku): sku for sku in skus}
        try:
            for futuro in as_completed(futuros):  # en orden de llegada
                sku = futuros[futuro]
                try:
                    productos[sku] = futuro.result()  # relanza la excepción del hilo
                except ProductoNoEncontradoError:
                    productos[sku] = None
        except BaseException:
            executor.shutdown(cancel_futures=True)  # no seguir llamando a la API
            raise
    return productos  # solo el hilo principal escribe en el dict: no hace falta Lock
```

Observa el final de `enriquecer_concurrente()`: los hilos solo **devuelven** resultados y únicamente el hilo principal escribe en el diccionario, así que no hace falta un `Lock` (tema 5.13).

### 6.4 La suite demuestra que nada cambió

```bash
python -m pytest -q
```

Resultado esperado: `52 passed`. Las cuatro pruebas que antes se omitían ahora corren con la versión optimizada, **incluido el golden master**.

### 6.5 Medir después

```bash
python -m pipeline_ventas ejecutar --modo optimizada
python -m pipeline_ventas perfilar --modo optimizada
```

```text
modo=optimizada: leídas=10000 cargadas=9413 rechazadas=587 skus=41 en 0.49 s
rechazos por tipo: {'cantidad': 111, 'duplicado': 103, 'fecha': 123, 'precio': 118, 'producto': 132}
…
   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.012    0.012    0.584    0.584 pipeline.py:150(ejecutar)
        1    0.000    0.000    0.343    0.343 optimizado.py:29(enriquecer_concurrente)
     41/1    0.001    0.000    0.336    0.336 cliente.py:43(obtener_producto)
     9413    0.023    0.000    0.080    0.000 pipeline.py:90(_enriquecer_venta)
…
```

`deduplicar` ya ni aparece entre las 8 primeras. Ten en cuenta que cProfile mide sobre todo el hilo principal: para el trabajo de los hilos, la medida confiable es el reloj (`perf_counter`).

### 6.6 Comparar el antes y el después

```bash
python -m pipeline_ventas comparar
```

```text
modo           tiempo  pico memoria
base           2.83 s      15.92 MB
optimizada     0.48 s      15.93 MB
aceleración: x5.9
reportes idénticos (sin metadata): sí
```

`comparar` hace tres cosas:

1. Mide el tiempo con `perf_counter`.
2. Mide la memoria **en otra ejecución** con `tracemalloc`, porque `tracemalloc` frena el código y falsearía el tiempo.
3. Verifica que los dos reportes sean idénticos.

El resultado: casi **6 veces** más rápido, con la misma memoria. El `set` de 10,000 identificadores es pequeño frente a las filas que el pipeline ya tiene en memoria.

### 6.7 Una "optimización" incorrecta

Supón que alguien propone una deduplicación "más corta". Reemplaza **temporalmente** la función `deduplicar_rapido` de `optimizado.py` por esta:

```python
def deduplicar_rapido(filas: list[Fila]) -> tuple[list[Fila], list[Fila]]:
    por_id = {fila["venta_id"]: fila for fila in filas}  # "más corto y más rápido"
    unicas = list(por_id.values())
    return unicas, filas[len(unicas) :]
```

```bash
python -m pytest -q
python -m pipeline_ventas comparar
```

```text
FAILED tests/test_api_real.py::test_mismo_reporte_por_http_real[optimizada]
FAILED tests/test_regresion.py::test_reporte_igual_al_golden_master[optimizada]
2 failed, 50 passed in 3.29s
…
reportes idénticos (sin metadata): NO
```

`comparar` termina con el código `4`. El diccionario conserva la **última** aparición de cada venta, no la primera. Los conteos son los mismos, por eso `test_pipeline_completo` pasa, pero los montos cambian. El golden master y la comparación lo detectan.

**Restaura** la versión original de `deduplicar_rapido` y comprueba otra vez que obtienes `52 passed`.

> **Checkpoint 6.** `comparar` muestra la aceleración y `reportes idénticos (sin metadata): sí`.

---

## Parte 7. Lista para CI/CD (10 min)

### 7.1 Ejecución por marcadores y reporte JUnit

En CI, las pruebas rápidas van primero y las de integración en un paso aparte:

```bash
python -m pytest -m "not integracion" --junitxml=reportes/rapidas.xml
python -m pytest -m integracion --junitxml=reportes/integracion.xml
```

Resultado esperado: `50 passed, 2 deselected` y `2 passed, 50 deselected`. En la carpeta `reportes/` quedan los dos XML que leen GitHub, GitLab o Jenkins.

### 7.2 El código de salida de pytest decide el pipeline

Comprueba estos casos y anota el código de salida de cada uno:

```bash
python -m pytest -m contrato -q
python -m pytest --opcion-inventada -q
```

| Código | Significado | Ejemplo |
| :---: | :--- | :--- |
| 0 | Todas las pruebas pasaron | 7.1 |
| 1 | Alguna prueba falló | 5.3 y 6.7 |
| 2 | Ejecución interrumpida | Ctrl+C a mitad de la suite |
| 4 | Error de uso o de configuración | `--opcion-inventada` |
| 5 | No se recolectó ninguna prueba | `-m contrato`: ninguna prueba tiene ese marcador |

Gracias a `--strict-markers`, un marcador **mal escrito en una prueba** es un error, no un filtro que deja la suite vacía en silencio.

### 7.3 El workflow de GitHub Actions

Este workflow ejecuta lo mismo en cada push y en cada pull request, con Python 3.12 y 3.14. Publica los reportes **aunque algo falle** (`if: always()`). No se ejecuta en el laboratorio: queda listo para cuando subas el proyecto a GitHub.

**Archivo:** `.github/workflows/pruebas.yml`

```yaml
# GitHub Actions: la suite corre en cada push a main y en cada pull request.
# El código de salida de pytest decide si el cambio avanza (tema 5.8).
name: pruebas

on:
  push:
    branches: [main]
  pull_request:

jobs:
  pruebas:
    runs-on: ubuntu-latest
    timeout-minutes: 15            # evita trabajos colgados
    strategy:
      matrix:
        python-version: ["3.12", "3.14"]
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-python@v6
        with:
          python-version: ${{ matrix.python-version }}
          cache: pip               # reutiliza paquetes entre ejecuciones
      - run: pip install -r requirements.txt -r requirements-dev.txt
      - run: pip install -e .
      - run: ruff check .
      - name: Unitarias, mocks y regresión (rápidas)
        run: pytest -m "not integracion" --junitxml=reportes/rapidas.xml
      - name: Integración con la API simulada
        run: pytest -m integracion --junitxml=reportes/integracion.xml
      - uses: actions/upload-artifact@v4
        if: always()               # publica los reportes también si algo falla
        with:
          name: reportes-pytest-${{ matrix.python-version }}
          path: reportes/
```

### 7.4 Calidad del código

```bash
ruff check .
python -m mypy src tests api_simulada.py
```

Resultado esperado: `All checks passed!` y `Success: no issues found in 18 source files`.

---

## Preguntas de cierre (5 min)

1. **Corrección.** En 5.3 el golden master no detectó el error del límite, y en 6.7 la prueba de conteos no detectó la mala optimización. ¿Qué te dice eso sobre depender de un solo tipo de prueba?
2. **Compromisos (5.18).** La versión optimizada es casi 6 veces más rápida. ¿Qué se pagó a cambio, en complejidad y en facilidad de depuración? ¿En qué situación te quedarías con la versión base?
3. **Escalamiento.** Si el catálogo creciera a 5,000 SKUs, ¿qué parámetro de `optimizado.py` revisarías primero y por qué? ¿Qué le pasaría a la API si lo subieras a 500?
4. **Memoria.** `leer_csv` carga todo el archivo en una lista. ¿Qué cambiarías para procesar un CSV de 10 GB con la memoria acotada (temas 5.9, 5.10 y 5.16)?

---

## Entregables

Comprime `practica5_pruebas` **sin** `.venv/`, `datos/`, `salida/` ni `reportes/`. Antes de entregar, revisa:

- [ ] Paquete `src/pipeline_ventas/` con 9 archivos `.py`, incluido `optimizado.py`
- [ ] `tests/` con 6 archivos de prueba, `conftest.py` y `tests/datos/golden_reporte.json`
- [ ] `python -m pytest -q`: `52 passed`
- [ ] Salida de `python -m pipeline_ventas comparar` con la aceleración y `reportes idénticos (sin metadata): sí`
- [ ] Ejercicios 5.3 y 6.7 comprobados y revertidos
- [ ] Códigos de salida de 7.2 anotados
- [ ] `.github/workflows/pruebas.yml`
- [ ] Respuestas a las preguntas de cierre
- [ ] `ruff` y `mypy` sin errores

---

## Solución de problemas

| Síntoma | Causa probable | Solución |
| :--- | :--- | :--- |
| `ModuleNotFoundError: No module named 'pipeline_ventas'` | El paquete no está instalado en el entorno activo | Activa `.venv` y ejecuta `python -m pip install -e .` desde la raíz |
| `ModuleNotFoundError: No module named 'api_simulada'` en las pruebas | Falta `pythonpath = ["."]` en `pyproject.toml` o no estás en la raíz | Revisa `[tool.pytest.ini_options]` y ejecuta desde `practica5_pruebas` |
| `ImportError: cannot import name 'batched' from 'itertools'` | Python 3.11 o anterior | Usa Python 3.12 o superior |
| `Error de la API: … (causa: ConnectionError)` fuera del paso 4.6.4 | La API no está corriendo en la Terminal 1 | Ejecuta `python api_simulada.py` en la Terminal 1 |
| `OSError: [Errno 98] Address already in use` (o `WinError 10048`) | Ya hay otra API simulada abierta en el puerto 8000 | Ciérrala con **Ctrl+C**, o usa `--puerto 8010` en la API y `--api http://127.0.0.1:8010` en el pipeline |
| `No existe datos/ventas.csv` | No generaste los datos | `python -m pipeline_ventas generar` |
| `test_reporte_igual_al_golden_master` falla sin haber cambiado nada | El golden master se generó con otra versión de Faker o con datos distintos | Revisa el diff; si el cambio es correcto, `python -m pytest -m regresion --actualizar-golden` |
| `PytestUnknownMarkWarning` o error de marcador | Un marcador no está declarado en `pyproject.toml` | Decláralo en `markers` o corrige el nombre |
| La aceleración es menor que la del ejemplo | El equipo, el antivirus o la red local añaden latencia | Lo importante es la comparación en tu equipo y que los reportes sean idénticos |
| PowerShell no deja activar `.venv` | Política de ejecución de Windows | `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` |

---

## Limpieza

Detén la API en la Terminal 1 con **Ctrl+C**. Después, en la Terminal 2:

**PowerShell**
```powershell
Remove-Item -Recurse -Force datos, salida, reportes -ErrorAction SilentlyContinue
deactivate
```

**bash**
```bash
rm -rf datos salida reportes
deactivate
```

---

## Resumen

| Lo que hiciste | Lección |
| :--- | :--- |
| Probaste reglas puras con AAA, `pytest.raises` y una excepción propia | 5.1 |
| Verificaste el pipeline con CSV en `tmp_path`, SQLite real, `responses` y una API real en un hilo | 5.2 |
| Simulaste red caída, 5xx, 429, 404 y JSON corrupto con `patch`, `side_effect` y `autospec`, sin esperar | 5.3 |
| Generaste datos sintéticos reproducibles, sin información personal | 5.4 |
| Protegiste la salida completa con un golden master que exige aprobación | 5.6 |
| Cubriste particiones y valores límite con `parametrize` | 5.7 |
| Dejaste la suite lista para CI: configuración estricta, marcadores, JUnit XML, códigos de salida y workflow | 5.8 |
| Cargaste por lotes desde un generador | 5.9, 5.10 |
| Solapaste las esperas de red con `ThreadPoolExecutor` y `as_completed` | 5.13 |
| Mediste memoria con `tracemalloc` y tiempo con `cProfile` y `perf_counter` | 5.14, 5.15 |
| Elegiste hilos y no `asyncio` ni procesos, y evaluaste lo que costó la optimización | 5.11, 5.12, 5.18 |
