# Práctica 2. Automatización de calidad y validación de contratos de datos

| Dato | Detalle |
| :--- | :--- |
| **Curso** | Automatización de procesos con Python |
| **Capítulo** | 2. Automatización de archivos, datos y Data Quality |
| **Duración** | 120 minutos |
| **Modalidad** | Individual |
| **Nivel de Bloom** | Aplicar / Analizar |

---

## Objetivo

Automatizar controles repetibles de estructura y calidad sobre conjuntos de datos ficticios destinados a un flujo de procesamiento empresarial.

Vas a construir una automatización **configurable** con estas capacidades:

- Ingiere archivos y valida su esquema y su contrato de datos.
- Aplica reglas de calidad.
- Detecta problemas de completitud, duplicados y anomalías.
- Transforma los datos.
- Genera automáticamente resultados y excepciones.

**La misma lógica se reutiliza sobre dos conjuntos de datos distintos cambiando solo la configuración.**

## Lo que vas a construir

Un validador llamado **calidad-datos**. Cada archivo recorre estas etapas y cada etapa es un componente que viste en el capítulo:

```text
 datos/entrada/archivo.csv
        │
        ▼
 1. Controles previos ─────────── existe, tamaño, hash ─┐
 2. Lectura ───────────────────── codificación, delimitador, estructura ─┤──► cuarentena/  (archivo completo)
 3. Esquema ───────────────────── columnas faltantes / nuevas ─┘
        │
        ▼
 4. Transformación ────────────── espacios, minúsculas, fechas a ISO 8601
 5. Duplicados ────────────────── por clave
 6. Contrato por fila (Pydantic) ─ tipos y obligatoriedad ───────────────┐
 7. Reglas (YAML) ─────────────── semánticas y de negocio: error / warning ┴──► rechazados.csv (fila y motivo)
        │
        ▼
 8. Calidad del lote ──────────── completitud, % rechazados, anomalías IQR
        │
        ├── aprobado ──► validos.parquet ──► 9. Reconciliación
        └── no aprobado ──► no se publica
        │
        ▼
 10. Reporte ──────────────────── reporte.json + reporte.html (Jinja2)
```

### Relación con el capítulo

| Componente del laboratorio | Lección del capítulo |
| :--- | :--- |
| `detectar_formato()`: `utf-8-sig` / `cp1252` y `csv.Sniffer` | 2.1 Procesamiento de CSV |
| `pd.read_csv(dtype=str)`, Parquet con pyarrow | 2.1, 2.4 Procesamiento de Parquet |
| `transformar()`: limpieza, normalización y fechas en varios formatos | 2.7 Transformación de datos |
| `comparar_esquema()`: cambios compatibles e incompatibles | 2.11 Comparación de estructuras, 2.12 Cambios incompatibles |
| `construir_contrato()`: contrato por fila con Pydantic v2 | 2.10 Validación de esquemas |
| Cuarentena con log estructurado JSON | 2.13 Tratamiento de estructuras no válidas |
| `ejecutar_controles_previos()`: existencia, tamaño, hash SHA-256 | 2.14 Controles previos al procesamiento |
| `MotorDeReglas` y registro `@registrar` | 2.15 Reglas de validación, 2.22 Configuración de reglas reutilizables |
| `reconciliar()`: conteo, suma de control y claves | 2.17 Reconciliación de datos |
| `detectar_anomalias_iqr()` | 2.18 Comprobaciones de anomalías |
| `comprobar_completitud()` | 2.19 Comprobaciones de completitud |
| `marcar_duplicados()` | 2.20 Detección de duplicados |
| `escribir_reporte()` con Jinja2 y `autoescape` | 2.21 Generación automatizada de reportes |

## Criterios de evaluación

| Criterio | Evidencia en tu entrega |
| :--- | :--- |
| **Corrección** | Se identifican los 7 rechazos de ventas y los 4 de clientes, cada uno con su motivo, y los 4 archivos defectuosos terminan en cuarentena |
| **Resiliencia** | Un archivo o una fila inválida no detienen el proceso: se aíslan y se registran |
| **Mantenibilidad** | En la Parte 7 cambias reglas y umbrales **solo en YAML**; agregar un validador no modifica el motor |
| **Valor operativo** | El mismo motor valida dos datasets distintos y genera un reporte que sustituye la revisión manual |

---

## Requisitos

| Elemento | Requisito |
| :--- | :--- |
| Python | **3.11 o superior** (probado con 3.11 y 3.13) |
| Editor | Visual Studio Code con la extensión de Python |
| Terminal | Terminal integrada de VS Code (PowerShell en Windows; bash/zsh en macOS o Linux) |
| Red | Acceso a PyPI para instalar paquetes |

> **Convenciones**
> - Todos los comandos se ejecutan desde la **carpeta raíz del proyecto** (`practica2_calidad`).
> - Cuando un comando cambia según el sistema operativo, verás la versión **PowerShell** y la versión **bash**.
> - Este proyecto sigue los estándares de la Práctica 1: src-layout, `pyproject.toml`, pip-tools, configuración validada con Pydantic y códigos de salida.

### Distribución del tiempo

| Parte | Contenido | Minutos |
| :--- | :--- | :---: |
| 0. Preparación | Estructura, entorno y dependencias | 10 |
| 1. Datos y configuración | Datos de prueba, contratos YAML y su validación | 15 |
| 2. Ingesta y cuarentena | Controles previos, lectura, esquema | 18 |
| 3. Transformación y contrato | Limpieza, normalización y Pydantic | 12 |
| 4. Reglas y calidad | Motor de reglas, completitud, duplicados, anomalías | 17 |
| 5. Publicación y orquestación | Salidas, reconciliación, reporte y CLI | 15 |
| 6. Ejecución | Dos datasets y cuatro archivos defectuosos | 15 |
| 7. Cambiar reglas sin tocar la lógica | Ejercicios de configuración | 10 |
| 8. Pruebas | pytest, ruff y mypy | 8 |


---

## Parte 0. Preparación (10 min)

**0.1** Crea la carpeta `practica2_calidad`, ábrela en VS Code (**File → Open Folder**) y abre una terminal.

**0.2** Crea la estructura de carpetas:

**PowerShell**
```powershell
New-Item -ItemType Directory -Force -Path src/calidad_datos/plantillas, tests, config/datasets, scripts
```

**bash**
```bash
mkdir -p src/calidad_datos/plantillas tests config/datasets scripts
```

**0.3** Crea los archivos base del proyecto:

**Archivo:** `pyproject.toml`

```toml
[build-system]
requires = ["setuptools>=69"]
build-backend = "setuptools.build_meta"

[project]
name = "calidad-datos"
version = "0.1.0"
description = "Validador configurable de contratos y calidad de datos"
requires-python = ">=3.11"
dependencies = [
    "jinja2>=3.1,<4",
    "pandas>=2.2,<4",
    "pyarrow>=15",
    "pydantic>=2.7,<3",
    "pyyaml>=6.0,<7",
]

[tool.setuptools.packages.find]
where = ["src"]

[tool.setuptools.package-data]
calidad_datos = ["plantillas/*.html"]

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
```

**Archivo:** `requirements-dev.in`

```text
# Herramientas de desarrollo; respetan las versiones de requirements.txt
-c requirements.txt
mypy
pip-tools
pytest
ruff
types-PyYAML
```

**Archivo:** `.gitignore`

```text
.venv/
__pycache__/
*.egg-info/
.mypy_cache/
.ruff_cache/
.pytest_cache/

# Datos de cada ejecución
datos/entrada/
cuarentena/
salida/
```

**Archivo:** `src/calidad_datos/__init__.py`

```python
"""Validador configurable de contratos y calidad de datos."""

__version__ = "0.1.0"
```

**0.4** Crea el entorno virtual, fija las versiones e instala:

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
python -m pip install pip-tools
pip-compile --strip-extras pyproject.toml -o requirements.txt
pip-compile --strip-extras requirements-dev.in -o requirements-dev.txt
python -m pip install -r requirements.txt -r requirements-dev.txt
python -m pip install -e .
```

**Verificación:**

```bash
python -c "import calidad_datos, pandas, pyarrow, pydantic, yaml, jinja2; print('ok', calidad_datos.__version__)"
```

Resultado esperado: `ok 0.1.0`

---

## Parte 1. Datos de prueba y contratos en configuración (15 min)

### 1.1 Generar los archivos de entrada

Los archivos de prueba se generan con un script, como en las demos del capítulo. Así todos parten del mismo estado y puedes regenerarlos cuando quieras. El archivo de clientes se escribe en **cp1252 con `;`**, como lo exporta Excel en español.

**Archivo:** `scripts/preparar_datos.py`

```python
"""Genera los archivos de entrada del laboratorio en datos/entrada/.

Se puede ejecutar cuantas veces se quiera: borra la carpeta y la vuelve a crear,
así siempre se parte del mismo estado.
"""

import hashlib
import shutil
from pathlib import Path

ENTRADA = Path("datos/entrada")

VENTAS = """id_venta,fecha,sucursal,cliente_email,monto,canal,observaciones
V-10001,2026-10-24,1,ana.lopez@empresa.mx,1250.50,tienda,
V-10002,24/10/2026,2,BRUNO.DIAZ@EMPRESA.MX,349.90,Web,
V-10003,2026-10-24,1,,89.00,app,
V-10004,2026-10-24,3,carla.ruiz@empresa.mx,MONTO_INVALIDO,tienda,
V-10005,2026-10-24,2,dario.mena@empresa.mx,-150.00,web,
V-10006,2026-10-24,1,elena.soto-empresa.mx,560.00,tienda,
V-10002,24/10/2026,2,bruno.diaz@empresa.mx,349.90,web,reenvío del POS
V-10007,2026-13-45,2,fernando.gil@empresa.mx,410.00,app,
V-10008,2026-10-24,4,gabriela.paz@empresa.mx,95000.00,tienda,
V-10009,2026-10-24,1,hugo.vera@empresa.mx,780.25,telefono,
V-10010,2026-10-24,0,irene.vela@empresa.mx,230.00,web,
,2026-10-24,2,jorge.luna@empresa.mx,120.00,app,
V-10011,2026-10-24,3,karla.rios@empresa.mx,640.00,web,
V-10012,2026-10-24,2,luis.mora@empresa.mx,515.75,tienda,
V-10013,2026-10-24,1,maria.nava@empresa.mx,298.40,app,
V-10014,24/10/2026,4,nora.salas@empresa.mx,412.30,web,
V-10015,2026-10-24,3,omar.cruz@empresa.mx,875.00,tienda,
V-10016,2026-10-24,1,paula.reyes@empresa.mx,150.00,app,
V-10017,2026-10-24,2,quique.lara@empresa.mx,1020.00,web,
V-10018,2026-10-24,4,rosa.mejia@empresa.mx,333.33,tienda,
V-10019,31/12/2099,3,sergio.tapia@empresa.mx,705.00,web,
"""

# Exportado desde Excel en español: separador ";" y codificación cp1252.
CLIENTES = """id_cliente;nombre;email;telefono;limite_credito;segmento
1;Ana López;ana.lopez@empresa.mx;5512345678;50000;Premium
2;Peña y Asociados;contacto@pena.mx;;120000;empresa
3;Luis Núñez;luis.nunez@correo;5598765432;15000;basico
4;María José Ávila;mjavila@empresa.mx;;-500;basico
5;Óscar Ibáñez;oscar.ibanez@empresa.mx;5544332211;20000;vip
2;Peña y Asociados;contacto@pena.mx;;120000;empresa
6;;sin.nombre@empresa.mx;;10000;basico
7;Raúl Gómez;raul.gomez@empresa.mx;5511223344;30000;premium
8;Sofía Treviño;sofia.trevino@empresa.mx;55 1234;25000;basico
9;Tomás Ureña;tomas.urena@empresa.mx;5587654321;18000;basico
10;Úrsula Vega;ursula.vega@empresa.mx;;40000;premium
11;Víctor Zúñiga;victor.zuniga@empresa.mx;5522334455;22000;basico
"""

# Archivos con fallas de archivo, estructura o esquema.
ESTRUCTURA_ROTA = """id_venta,fecha,sucursal,cliente_email,monto,canal,observaciones
V-20001,2026-10-24,1,ana.lopez@empresa.mx,100.00,web,
V-20002,2026-10-24,1,bruno.diaz@empresa.mx,200.00,web,,columna,de,más
"""

SIN_MONTO = """id_venta,fecha,sucursal,cliente_email,canal
V-30001,2026-10-24,1,ana.lopez@empresa.mx,web
"""


def main() -> None:
    shutil.rmtree(ENTRADA, ignore_errors=True)
    ENTRADA.mkdir(parents=True)
    ventas = ENTRADA / "ventas_2026-10-24.csv"
    ventas.write_text(VENTAS, encoding="utf-8")
    (ENTRADA / "clientes_2026-10-24.csv").write_text(CLIENTES, encoding="cp1252")
    (ENTRADA / "ventas_estructura_rota.csv").write_text(
        ESTRUCTURA_ROTA, encoding="utf-8"
    )
    (ENTRADA / "ventas_sin_monto.csv").write_text(SIN_MONTO, encoding="utf-8")
    (ENTRADA / "ventas_vacio.csv").write_text("", encoding="utf-8")
    # Manifiesto que "envía" el sistema origen junto con el archivo de ventas.
    huella = hashlib.sha256(ventas.read_bytes()).hexdigest()
    (ENTRADA / (ventas.name + ".sha256")).write_text(
        f"{huella}  {ventas.name}\n", encoding="utf-8"
    )
    for archivo in sorted(ENTRADA.iterdir()):
        print(f"{archivo.name:<36} {archivo.stat().st_size:>5} bytes")


if __name__ == "__main__":
    main()
```

Ejecútalo:

```bash
python scripts/preparar_datos.py
```

Resultado esperado (en Windows los tamaños pueden ser algo mayores por los saltos de línea `\r\n`):

```text
clientes_2026-10-24.csv                726 bytes
ventas_2026-10-24.csv                 1249 bytes
ventas_2026-10-24.csv.sha256            88 bytes
ventas_estructura_rota.csv             189 bytes
ventas_sin_monto.csv                    90 bytes
ventas_vacio.csv                         0 bytes
```

> `ventas_2026-10-24.csv.sha256` es el **manifiesto** que envía el sistema origen. Con él se comprueba que el archivo no se alteró en el camino.

### 1.2 Contratos y reglas como configuración

Cada dataset tiene un YAML con cinco secciones:

| Sección | Qué define | Nivel de validación |
| :--- | :--- | :--- |
| `esquema` | Columnas, tipo, obligatoriedad, longitud y mínimos | Sintáctica (contrato con Pydantic) |
| `transformaciones` | Formatos de fecha aceptados y columnas en minúsculas | Normalización |
| `reglas` | Mismo formato que `rules_config.yaml` del capítulo: `name`, `field`, `validator`, `severity`, `parameters` | Semántica y de negocio |
| `calidad` | Completitud mínima, clave de duplicados, anomalías, % máximo de rechazos | Lote |
| `controles_previos` | Tamaño mínimo y máximo del archivo | Archivo |

**1.2.1** Crea la configuración de ventas:

**Archivo:** `config/datasets/ventas.yaml`

```yaml
# Contrato y reglas de calidad del dataset de ventas diarias.
version: "1.0"
dataset: ventas
clave: id_venta
campo_control: monto          # suma de control para la reconciliación

controles_previos:
  tamano_min_bytes: 20
  tamano_max_mb: 50

# Validación sintáctica (contrato por fila con Pydantic): tipo y obligatoriedad
esquema:
  id_venta:      {tipo: texto,   requerido: true, min_largo: 5}
  fecha:         {tipo: fecha,   requerido: true}
  sucursal:      {tipo: entero,  requerido: true, minimo: 1}
  cliente_email: {tipo: texto,   requerido: false}
  monto:         {tipo: decimal, requerido: true}
  canal:         {tipo: texto,   requerido: true}

transformaciones:
  formatos_fecha: ["%Y-%m-%d", "%d/%m/%Y"]   # acordados con el sistema origen
  minusculas: [cliente_email, canal]

# Validación semántica y de negocio: mismo formato que rules_config.yaml
reglas:
  - name: verificar_monto_positivo
    field: monto
    validator: range
    severity: error            # error: el registro se rechaza
    parameters: {min: 0.01, max: 1000000.00}

  - name: canal_en_catalogo
    field: canal
    validator: in_set
    severity: error
    parameters: {values: [tienda, web, app]}

  - name: validar_formato_correo
    field: cliente_email
    validator: regex
    severity: warning          # warning: se registra y el registro continúa
    parameters:
      pattern: '^[\w.+-]+@[\w-]+(\.[\w-]+)+$'

# Controles del lote: deciden si se publica
calidad:
  completitud_minima: {cliente_email: 90}
  duplicados_por: [id_venta]
  anomalias: {campo: monto, k: 3.0}
  max_pct_rechazados: 40
```

**1.2.2** Crea la configuración de clientes. Observa que usa **los mismos validadores** e incluso **la misma regla de correo**:

**Archivo:** `config/datasets/clientes.yaml`

```yaml
# Contrato y reglas de calidad del catálogo de clientes.
# Usa el MISMO motor y los MISMOS validadores que ventas: solo cambia la configuración.
version: "1.0"
dataset: clientes
clave: id_cliente
campo_control: limite_credito

controles_previos:
  tamano_min_bytes: 20
  tamano_max_mb: 50

esquema:
  id_cliente:     {tipo: entero,  requerido: true, minimo: 1}
  nombre:         {tipo: texto,   requerido: true, min_largo: 2}
  email:          {tipo: texto,   requerido: true}
  telefono:       {tipo: texto,   requerido: false}
  limite_credito: {tipo: decimal, requerido: true}
  segmento:       {tipo: texto,   requerido: true}

transformaciones:
  minusculas: [email, segmento]

reglas:
  - name: limite_credito_valido
    field: limite_credito
    validator: range
    severity: error
    parameters: {min: 0, max: 500000}

  - name: segmento_en_catalogo
    field: segmento
    validator: in_set
    severity: error
    parameters: {values: [basico, premium, empresa]}

  - name: validar_formato_correo     # la misma regla que en ventas
    field: email
    validator: regex
    severity: warning
    parameters:
      pattern: '^[\w.+-]+@[\w-]+(\.[\w-]+)+$'

  - name: telefono_10_digitos
    field: telefono
    validator: regex
    severity: warning
    parameters:
      pattern: '^\d{10}$'

calidad:
  completitud_minima: {email: 100, telefono: 50}
  duplicados_por: [id_cliente]
  anomalias: {campo: limite_credito, k: 3.0}
  max_pct_rechazados: 40
```

> `severity: error` rechaza el registro. `severity: warning` lo deja pasar y lo anota en el reporte. La decisión de **publicar el lote** la toman los umbrales de la sección `calidad`.

### 1.3 Excepciones y validación de la configuración

**1.3.1** Crea `src/calidad_datos/errores.py`. Cada error tiene su código de salida:

**Archivo:** `src/calidad_datos/errores.py`

```python
"""Excepciones controladas y sus códigos de salida."""


class CalidadError(Exception):
    """Clase base de los errores controlados."""

    codigo_salida: int = 1


class ConfiguracionInvalidaError(CalidadError):
    """El archivo YAML del dataset no es válido."""

    codigo_salida = 1


class ArchivoEnCuarentenaError(CalidadError):
    """El archivo no pasó un control previo, estructural o de esquema."""

    codigo_salida = 3

    def __init__(self, motivo: str, detalle: str) -> None:
        super().__init__(f"{motivo}: {detalle}")
        self.motivo = motivo
        self.detalle = detalle


class ReconciliacionError(CalidadError):
    """Lo publicado no coincide con lo validado."""

    codigo_salida = 4


class LoteNoAprobadoError(CalidadError):
    """El lote no cumple los umbrales de calidad; no se publica."""

    codigo_salida = 5
```

**1.3.2** Crea `src/calidad_datos/configuracion.py`. Valida el YAML con Pydantic **antes** de leer cualquier dato (fail-fast). El YAML se lee con `yaml.safe_load`, nunca con `yaml.load`, y cada validador mencionado debe existir:

**Archivo:** `src/calidad_datos/configuracion.py`

```python
"""Esquema de la configuración YAML de cada dataset (fail-fast con Pydantic)."""

from pathlib import Path
from typing import Any, Literal

import yaml
from pydantic import BaseModel, ConfigDict, Field, ValidationError, model_validator

from calidad_datos.errores import ConfiguracionInvalidaError

TipoColumna = Literal["texto", "entero", "decimal", "fecha"]


class Estricto(BaseModel):
    model_config = ConfigDict(extra="forbid")


class ControlesPreviosConfig(Estricto):
    tamano_min_bytes: int = Field(default=10, ge=1)
    tamano_max_mb: float = Field(default=50.0, gt=0)


class ColumnaConfig(Estricto):
    tipo: TipoColumna
    requerido: bool = True
    min_largo: int | None = Field(default=None, ge=1)
    minimo: float | None = None
    maximo: float | None = None


class TransformacionesConfig(Estricto):
    formatos_fecha: list[str] = Field(default_factory=lambda: ["%Y-%m-%d"])
    minusculas: list[str] = Field(default_factory=list)


class ReglaConfig(Estricto):
    """Mismo formato que rules_config.yaml visto en el capítulo."""

    name: str
    field: str
    validator: str
    severity: Literal["error", "warning"]
    parameters: dict[str, Any] = Field(default_factory=dict)


class AnomaliasConfig(Estricto):
    campo: str
    k: float = Field(default=3.0, gt=0)


class CalidadConfig(Estricto):
    completitud_minima: dict[str, float] = Field(default_factory=dict)
    duplicados_por: list[str] = Field(default_factory=list)
    anomalias: AnomaliasConfig | None = None
    max_pct_rechazados: float = Field(default=10.0, ge=0, le=100)


class DatasetConfig(Estricto):
    version: str
    dataset: str
    clave: str
    campo_control: str  # columna numérica para la suma de control
    controles_previos: ControlesPreviosConfig = Field(
        default_factory=ControlesPreviosConfig
    )
    esquema: dict[str, ColumnaConfig] = Field(min_length=1)
    transformaciones: TransformacionesConfig = Field(
        default_factory=TransformacionesConfig
    )
    reglas: list[ReglaConfig] = Field(default_factory=list)
    calidad: CalidadConfig = Field(default_factory=CalidadConfig)

    @model_validator(mode="after")
    def columnas_conocidas(self) -> "DatasetConfig":
        """Toda columna mencionada en la configuración debe existir en el esquema."""
        mencionadas = {self.clave, self.campo_control}
        mencionadas |= {regla.field for regla in self.reglas}
        mencionadas |= set(self.transformaciones.minusculas)
        mencionadas |= set(self.calidad.completitud_minima)
        mencionadas |= set(self.calidad.duplicados_por)
        if self.calidad.anomalias:
            mencionadas.add(self.calidad.anomalias.campo)
        desconocidas = mencionadas - set(self.esquema)
        if desconocidas:
            raise ValueError(
                f"columnas no declaradas en esquema: {sorted(desconocidas)}"
            )
        return self


def cargar_configuracion(ruta: Path, validadores: set[str]) -> DatasetConfig:
    """Lee el YAML con safe_load, lo valida y comprueba que los validadores existan.

    Raises:
        ConfiguracionInvalidaError: Si algo no es válido, antes de leer datos.
    """
    if not ruta.is_file():
        raise ConfiguracionInvalidaError(f"No existe la configuración: {ruta}")
    try:
        with ruta.open(encoding="utf-8") as archivo:
            datos = yaml.safe_load(archivo)  # safe_load: nunca yaml.load
        config = DatasetConfig.model_validate(datos)
    except yaml.YAMLError as error:
        raise ConfiguracionInvalidaError(f"YAML inválido en {ruta}: {error}") from error
    except ValidationError as error:
        raise ConfiguracionInvalidaError(
            f"Configuración inválida en {ruta}:\n{error}"
        ) from error
    desconocidos = {regla.validator for regla in config.reglas} - validadores
    if desconocidos:
        raise ConfiguracionInvalidaError(
            f"Validadores no registrados: {sorted(desconocidos)}"
        )
    return config
```

**Verificación 1.3:**

```bash
python -c "from pathlib import Path; from calidad_datos.configuracion import cargar_configuracion; v = {'not_null', 'range', 'regex', 'in_set'}; [print(c.dataset, len(c.esquema), 'columnas,', len(c.reglas), 'reglas') for c in (cargar_configuracion(Path(f'config/datasets/{n}.yaml'), v) for n in ('ventas', 'clientes'))]"
```

Resultado esperado:

```text
ventas 6 columnas, 3 reglas
clientes 6 columnas, 4 reglas
```

> **Checkpoint 1.** Existen los 6 archivos de entrada y las dos configuraciones se validan.

---

## Parte 2. Ingesta, controles previos y esquema (18 min)

### 2.1 Controles previos

Crea `src/calidad_datos/controles_previos.py`. Revisa el archivo **como objeto**, sin abrir su contenido: existencia, tamaño y hash SHA-256 leído por bloques contra el manifiesto.

**Archivo:** `src/calidad_datos/controles_previos.py`

```python
"""Controles previos: revisan el archivo como objeto antes de leer su contenido."""

import hashlib
import os
from pathlib import Path

from calidad_datos.configuracion import ControlesPreviosConfig
from calidad_datos.errores import ArchivoEnCuarentenaError


def sha256(ruta: Path, bloque: int = 64 * 1024) -> str:
    """Calcula el hash leyendo por bloques (memoria acotada)."""
    h = hashlib.sha256()
    with ruta.open("rb") as archivo:
        while datos := archivo.read(bloque):
            h.update(datos)
    return h.hexdigest()


def ejecutar_controles_previos(ruta: Path, config: ControlesPreviosConfig) -> None:
    """Existencia, tamaño e integridad contra el manifiesto <archivo>.sha256.

    Raises:
        FileNotFoundError: Si el archivo no existe o no es legible.
        ArchivoEnCuarentenaError: Si está vacío, es demasiado grande o fue alterado.
    """
    if not (ruta.is_file() and os.access(ruta, os.R_OK)):
        raise FileNotFoundError(f"No existe o no es legible: {ruta}")

    tamano = ruta.stat().st_size
    if tamano < config.tamano_min_bytes:
        raise ArchivoEnCuarentenaError(
            "vacio", f"{tamano} bytes; mínimo {config.tamano_min_bytes}"
        )
    if tamano > config.tamano_max_mb * 1024 * 1024:
        raise ArchivoEnCuarentenaError(
            "demasiado_grande", f"{tamano} bytes; máximo {config.tamano_max_mb} MiB"
        )

    manifiesto = ruta.with_name(ruta.name + ".sha256")
    if manifiesto.is_file():
        esperado = manifiesto.read_text(encoding="utf-8").split()[0]
        if sha256(ruta) != esperado:
            raise ArchivoEnCuarentenaError(
                "hash_distinto", "el contenido no coincide con el manifiesto"
            )
```

> El capítulo también presenta el control de **estabilidad** (comparar el tamaño en dos momentos). Aquí no se incluye porque los archivos no llegan en tránsito; en un directorio de recepción real sí conviene usarlo.

### 2.2 Lectura: codificación, delimitador y estructura

Crea `src/calidad_datos/lectura.py`:

- `detectar_formato()` prueba `utf-8-sig` y luego `cp1252`, y detecta el delimitador con `csv.Sniffer` sobre el encabezado.
- `leer_csv()` lee **todo como texto** (`dtype=str`, `keep_default_na=False`) para que el contrato decida los tipos. Una fila con columnas de más produce `ParserError`: es **invalidez estructural** y el archivo completo va a cuarentena.

**Archivo:** `src/calidad_datos/lectura.py`

```python
"""Ingesta: detecta codificación y delimitador, y lee el CSV como texto."""

import csv
from pathlib import Path

import pandas as pd

from calidad_datos.errores import ArchivoEnCuarentenaError

# El orden importa: cp1252 casi nunca falla, por eso se prueba al final.
CODIFICACIONES = ("utf-8-sig", "cp1252")


def detectar_formato(ruta: Path) -> tuple[str, str]:
    """Devuelve (codificación, delimitador).

    Raises:
        ArchivoEnCuarentenaError: Si no se reconoce la codificación o el delimitador.
    """
    for codificacion in CODIFICACIONES:
        try:
            with ruta.open(encoding=codificacion, newline="") as archivo:
                muestra = archivo.read(64 * 1024)
                archivo.read()  # decodifica el resto: falla aquí si no es válida
            break
        except UnicodeDecodeError:
            continue
    else:
        raise ArchivoEnCuarentenaError("codificacion", "no es UTF-8 ni cp1252")
    encabezado = muestra.splitlines()[0] if muestra else ""
    try:
        # Se analiza solo el encabezado: una fila mal formada no debe confundir
        # la detección del delimitador; esa falla la reporta la lectura.
        dialecto = csv.Sniffer().sniff(encabezado, delimiters=",;|\t")
    except csv.Error as error:
        raise ArchivoEnCuarentenaError("delimitador", str(error)) from error
    return codificacion, dialecto.delimiter


def leer_csv(ruta: Path) -> pd.DataFrame:
    """Lee todas las columnas como texto; los tipos los decide el contrato.

    Raises:
        ArchivoEnCuarentenaError: Si está mal formado (invalidez estructural).
    """
    codificacion, delimitador = detectar_formato(ruta)
    try:
        df = pd.read_csv(
            ruta,
            sep=delimitador,
            encoding=codificacion,
            dtype=str,
            keep_default_na=False,  # "NaN" o "NA" se quedan como texto
            on_bad_lines="error",
        )
    except pd.errors.ParserError as error:
        raise ArchivoEnCuarentenaError("estructura", str(error).strip()) from error
    df.columns = [str(columna).strip() for columna in df.columns]
    return df
```

**Verificación 2.2:**

```bash
python -c "from pathlib import Path; from calidad_datos.lectura import detectar_formato, leer_csv; r = Path('datos/entrada/clientes_2026-10-24.csv'); print(detectar_formato(r)); print(leer_csv(r).head(2).to_string())"
```

Resultado esperado:

```text
('cp1252', ';')
  id_cliente            nombre                 email    telefono limite_credito segmento
0          1         Ana López  ana.lopez@empresa.mx  5512345678          50000  Premium
1          2  Peña y Asociados      contacto@pena.mx                     120000  empresa
```

Los acentos y la ñ se leen bien porque se detectó `cp1252`.

### 2.3 Comparación de esquema

Crea `src/calidad_datos/esquema.py`. Aplica la tabla de clasificación de cambios del capítulo:

| Cambio | Clasificación | Acción |
| :--- | :--- | :--- |
| Falta una columna requerida | Incompatible | El archivo va a cuarentena |
| Columna duplicada | Incompatible | El archivo va a cuarentena |
| Falta una columna opcional | Compatible | Se registra y continúa |
| Llega una columna nueva | Compatible | Se registra y se ignora |

**Archivo:** `src/calidad_datos/esquema.py`

```python
"""Comparación entre la estructura recibida y la esperada."""

from dataclasses import dataclass, field

from calidad_datos.configuracion import ColumnaConfig


@dataclass
class ResultadoEsquema:
    incompatibles: list[str] = field(default_factory=list)
    compatibles: list[str] = field(default_factory=list)

    @property
    def es_compatible(self) -> bool:
        return not self.incompatibles


def comparar_esquema(
    recibidas: list[str], esperado: dict[str, ColumnaConfig]
) -> ResultadoEsquema:
    """Clasifica las diferencias como en la tabla de cambios de esquema.

    - Falta una columna requerida: incompatible (se detiene el lote).
    - Falta una columna opcional o llega una columna nueva: compatible (se registra).
    """
    resultado = ResultadoEsquema()
    for columna, definicion in esperado.items():
        if columna not in recibidas:
            destino = (
                resultado.incompatibles
                if definicion.requerido
                else resultado.compatibles
            )
            destino.append(f"falta '{columna}' (requerido={definicion.requerido})")
    for columna in recibidas:
        if columna not in esperado:
            resultado.compatibles.append(f"columna nueva '{columna}' (se ignora)")
    duplicadas = sorted({c for c in recibidas if recibidas.count(c) > 1})
    for columna in duplicadas:
        resultado.incompatibles.append(f"columna duplicada '{columna}'")
    return resultado
```

---

## Parte 3. Transformación y contrato por fila (12 min)

### 3.1 Transformación

Crea `src/calidad_datos/transformacion.py`. Genera un DataFrame **nuevo** (la fuente no se modifica) con espacios eliminados, columnas en minúsculas y fechas convertidas a ISO 8601 desde los formatos acordados. Si una fecha no coincide con ningún formato, **no se adivina**: el contrato la rechazará con su motivo.

**Archivo:** `src/calidad_datos/transformacion.py`

```python
"""Transformación: limpieza y normalización sin modificar la fuente."""

from datetime import datetime

import pandas as pd

from calidad_datos.configuracion import DatasetConfig


def normalizar_fecha(valor: str, formatos: list[str]) -> str:
    """Convierte una fecha en cualquiera de los formatos acordados a ISO 8601.

    Si ningún formato coincide, devuelve el valor original: el contrato lo rechazará
    con su motivo, en lugar de adivinar.
    """
    for formato in formatos:
        try:
            return datetime.strptime(valor, formato).date().isoformat()
        except ValueError:
            continue
    return valor


def transformar(df: pd.DataFrame, config: DatasetConfig) -> pd.DataFrame:
    """Devuelve un DataFrame nuevo, limpio y normalizado."""
    limpio = df.copy()
    for columna in limpio.columns:
        limpio[columna] = limpio[columna].str.strip()
    for columna in config.transformaciones.minusculas:
        if columna in limpio:
            limpio[columna] = limpio[columna].str.lower()
    formatos = config.transformaciones.formatos_fecha
    for columna, definicion in config.esquema.items():
        if definicion.tipo == "fecha" and columna in limpio:
            limpio[columna] = limpio[columna].map(
                lambda valor: normalizar_fecha(valor, formatos) if valor else valor
            )
    return limpio
```

### 3.2 Contrato por fila con Pydantic v2

Crea `src/calidad_datos/contrato.py`. En lugar de escribir una clase por dataset, `create_model` **genera la clase Pydantic a partir del esquema YAML**. Así, un dataset nuevo no requiere código. `model_validate` hace la coerción vista en clase (`"1"` → `1`, `"1250.50"` → `Decimal`) y reporta **todos** los errores de la fila.

**Archivo:** `src/calidad_datos/contrato.py`

```python
"""Contrato por fila con Pydantic v2, generado a partir del esquema YAML."""

from datetime import date
from decimal import Decimal
from typing import Any

from pydantic import BaseModel, ConfigDict, Field, ValidationError, create_model

from calidad_datos.configuracion import ColumnaConfig

TIPOS: dict[str, type] = {
    "texto": str,
    "entero": int,
    "decimal": Decimal,
    "fecha": date,
}


def construir_contrato(
    nombre: str, esquema: dict[str, ColumnaConfig]
) -> type[BaseModel]:
    """Crea la clase Pydantic del dataset: un contrato nuevo no requiere código."""
    campos: dict[str, Any] = {}
    for columna, definicion in esquema.items():
        restricciones: dict[str, Any] = {
            "min_length": definicion.min_largo,
            "ge": definicion.minimo,
            "le": definicion.maximo,
        }
        restricciones = {k: v for k, v in restricciones.items() if v is not None}
        tipo = TIPOS[definicion.tipo]
        if definicion.requerido:
            campos[columna] = (tipo, Field(**restricciones))
        else:
            campos[columna] = (tipo | None, Field(default=None, **restricciones))
    return create_model(
        f"Contrato_{nombre}",
        __config__=ConfigDict(extra="ignore", str_strip_whitespace=True),
        **campos,
    )


def validar_fila(
    contrato: type[BaseModel], fila: dict[str, str]
) -> tuple[dict[str, Any] | None, list[str]]:
    """Devuelve (registro tipado, []) o (None, lista de todos los errores).

    Las celdas vacías se omiten: si el campo es requerido, Pydantic informa
    "Field required"; si es opcional, toma su valor por defecto (None).
    """
    presentes = {columna: valor for columna, valor in fila.items() if valor != ""}
    try:
        return contrato.model_validate(presentes).model_dump(), []
    except ValidationError as error:
        return None, [f"{e['loc'][0]}: {e['msg']}" for e in error.errors()]
```

**Verificación 3.2:**

```bash
python -c "from pathlib import Path; from calidad_datos.configuracion import cargar_configuracion; from calidad_datos.contrato import construir_contrato, validar_fila; c = cargar_configuracion(Path('config/datasets/ventas.yaml'), {'range', 'regex', 'in_set'}); m = construir_contrato(c.dataset, c.esquema); print(validar_fila(m, {'id_venta': 'V-1', 'fecha': '2026-10-24', 'sucursal': '0', 'cliente_email': '', 'monto': 'abc', 'canal': 'web'})); print(validar_fila(m, {'id_venta': 'V-10001', 'fecha': '2026-10-24', 'sucursal': '1', 'cliente_email': '', 'monto': '1250.50', 'canal': 'tienda'}))"
```

Resultado esperado:

```text
(None, ['id_venta: String should have at least 5 characters', 'sucursal: Input should be greater than or equal to 1', 'monto: Input should be a valid decimal'])
({'id_venta': 'V-10001', 'fecha': datetime.date(2026, 10, 24), 'sucursal': 1, 'cliente_email': None, 'monto': Decimal('1250.50'), 'canal': 'tienda'}, [])
```

> **Checkpoint 3.** La primera fila trae los **tres** errores a la vez y la segunda queda tipada (`date`, `int`, `Decimal`).

---

## Parte 4. Reglas configurables y calidad del lote (17 min)

### 4.1 Registro de validadores y motor de reglas

Crea `src/calidad_datos/reglas.py`. Es el patrón Strategy del capítulo:

- El decorador `@registrar("nombre")` agrega cada validador al diccionario `VALIDADORES` con el nombre que se usa en el YAML.
- `MotorDeReglas` evalúa **todas** las reglas de un registro sin detenerse en el primer fallo.
- Un valor ausente (`None`) solo lo evalúa `not_null`: la ausencia ya la miden el contrato y la completitud.

**Archivo:** `src/calidad_datos/reglas.py`

```python
"""Reglas de validación configurables: registro de validadores (patrón Strategy)."""

import re
from collections.abc import Callable
from dataclasses import dataclass
from decimal import Decimal
from typing import Any

from calidad_datos.configuracion import ReglaConfig

Validador = Callable[[Any, dict[str, Any]], bool]
VALIDADORES: dict[str, Validador] = {}


def registrar(nombre: str) -> Callable[[Validador], Validador]:
    """Decorador: agrega la función al registro con el nombre usado en el YAML."""

    def decorador(funcion: Validador) -> Validador:
        VALIDADORES[nombre] = funcion
        return funcion

    return decorador


@registrar("not_null")
def no_nulo(valor: Any, params: dict[str, Any]) -> bool:
    return valor is not None and str(valor).strip() != ""


@registrar("range")
def en_rango(valor: Any, params: dict[str, Any]) -> bool:
    try:
        numero = Decimal(str(valor))
    except ArithmeticError:
        return False
    minimo = Decimal(str(params.get("min", "-Infinity")))
    maximo = Decimal(str(params.get("max", "Infinity")))
    return minimo <= numero <= maximo


@registrar("regex")
def coincide(valor: Any, params: dict[str, Any]) -> bool:
    return re.fullmatch(params["pattern"], str(valor)) is not None


@registrar("in_set")
def en_catalogo(valor: Any, params: dict[str, Any]) -> bool:
    return valor in params["values"]


@dataclass(frozen=True)
class Incumplimiento:
    fila: int
    regla: str
    campo: str
    valor: str
    severidad: str


class MotorDeReglas:
    """Evalúa todas las reglas de un registro sin detenerse en el primer fallo."""

    def __init__(self, reglas: list[ReglaConfig]) -> None:
        self._reglas = reglas

    def evaluar(self, fila: int, registro: dict[str, Any]) -> list[Incumplimiento]:
        incumplimientos = []
        for regla in self._reglas:
            valor = registro.get(regla.field)
            if valor is None and regla.validator != "not_null":
                continue  # la ausencia la miden el contrato y la completitud
            if not VALIDADORES[regla.validator](valor, regla.parameters):
                incumplimientos.append(
                    Incumplimiento(
                        fila, regla.name, regla.field, str(valor), regla.severity
                    )
                )
        return incumplimientos
```

**Verificación 4.1:**

```bash
python -c "from calidad_datos.reglas import VALIDADORES; print(sorted(VALIDADORES))"
```

Resultado esperado: `['in_set', 'not_null', 'range', 'regex']`

### 4.2 Completitud, duplicados y anomalías

Crea `src/calidad_datos/calidad.py`:

- **Completitud:** porcentaje de celdas con dato por columna; las vacías o con solo espacios cuentan como ausentes.
- **Duplicados exactos:** por clave, con `duplicated(keep="first")`. Ningún duplicado se descarta en silencio: va a `rechazados.csv`.
- **Anomalías puntuales:** límites IQR `Q1 − k·IQR` y `Q3 + k·IQR`. Una anomalía **no se rechaza**: se señala para que una persona la revise.

**Archivo:** `src/calidad_datos/calidad.py`

```python
"""Controles de calidad del lote: completitud, duplicados y anomalías."""

from typing import Any

import pandas as pd


def comprobar_completitud(
    df: pd.DataFrame, minimos: dict[str, float]
) -> dict[str, dict[str, Any]]:
    """Porcentaje de celdas con dato por columna contra su mínimo configurado.

    Las celdas vacías o con solo espacios cuentan como dato ausente.
    """
    resultado = {}
    for columna, minimo in minimos.items():
        if columna in df and len(df):
            presentes = df[columna].fillna("").str.strip().ne("")
            pct = round(100 * float(presentes.mean()), 2)
        else:
            pct = 0.0
        resultado[columna] = {
            "presente_pct": pct,
            "minimo_pct": minimo,
            "ok": pct >= minimo,
        }
    return resultado


def marcar_duplicados(df: pd.DataFrame, columnas: list[str]) -> pd.Series:
    """True en cada repetición de la clave; la primera aparición se conserva.

    Las filas con la clave vacía no se marcan: las rechaza el contrato.
    """
    if not columnas:
        return pd.Series(False, index=df.index)
    con_clave = df[columnas].ne("").all(axis=1)
    return df.duplicated(subset=columnas, keep="first") & con_clave


def detectar_anomalias_iqr(
    valores: pd.Series, k: float
) -> tuple[list[Any], tuple[float, float] | None]:
    """Índices fuera de [Q1 − k·IQR, Q3 + k·IQR] y los límites usados.

    Con menos de 4 valores no hay base estadística y no se evalúa.
    """
    numeros = pd.to_numeric(valores, errors="coerce").dropna()
    if len(numeros) < 4:
        return [], None
    q1, q3 = numeros.quantile([0.25, 0.75])
    iqr = q3 - q1
    inferior, superior = float(q1 - k * iqr), float(q3 + k * iqr)
    fuera = numeros[(numeros < inferior) | (numeros > superior)]
    return list(fuera.index), (round(inferior, 2), round(superior, 2))
```

**Verificación 4.2:**

```bash
python -c "import pandas as pd; from calidad_datos.calidad import detectar_anomalias_iqr; print(detectar_anomalias_iqr(pd.Series([120, 150, 98, 135, 4000, 110]), 3.0))"
```

Resultado esperado: `([4], (11.25, 247.5))`. El índice 4 (`4000`) queda fuera de los límites.

---

## Parte 5. Publicación, reconciliación, reporte y orquestación (15 min)

### 5.1 Salidas y cuarentena

Crea `src/calidad_datos/salida.py`:

- `enviar_a_cuarentena()` **mueve** el archivo con la fecha y el motivo en el nombre, y emite un log estructurado en JSON.
- `escribir_parquet()` publica los válidos con tipos reales (`date32`, `decimal128`, `int64`).
- `escribir_reporte()` genera JSON y HTML con Jinja2 y `autoescape`, porque el reporte contiene datos de origen externo.

**Archivo:** `src/calidad_datos/salida.py`

```python
"""Salidas: Parquet de válidos, CSV de rechazados, cuarentena y reporte."""

import json
import logging
import shutil
from datetime import UTC, datetime
from pathlib import Path
from typing import Any

import pandas as pd
from jinja2 import Environment, FileSystemLoader, select_autoescape

logger = logging.getLogger(__name__)
PLANTILLAS = Path(__file__).parent / "plantillas"


def enviar_a_cuarentena(
    archivo: Path, carpeta: Path, motivo: str, detalle: str
) -> Path:
    """Mueve el archivo (y su manifiesto) con fecha y motivo en el nombre."""
    carpeta.mkdir(parents=True, exist_ok=True)
    marca = datetime.now(UTC).strftime("%Y%m%dT%H%M%SZ")
    destino = carpeta / f"{marca}_{motivo}_{archivo.name}"
    shutil.move(archivo, destino)  # atómico solo dentro del mismo sistema de archivos
    manifiesto = archivo.with_name(archivo.name + ".sha256")
    if manifiesto.is_file():
        shutil.move(manifiesto, destino.with_name(destino.name + ".sha256"))
    evento = {
        "estado": "QUARANTINED",
        "archivo": archivo.name,
        "motivo": motivo,
        "detalle": detalle,
    }
    logger.warning(json.dumps(evento, ensure_ascii=False))
    return destino


def escribir_parquet(registros: list[dict[str, Any]], destino: Path) -> None:
    pd.DataFrame(registros).to_parquet(destino, engine="pyarrow", index=False)


def escribir_rechazados(rechazados: list[dict[str, Any]], destino: Path) -> None:
    columnas = ["fila", "motivos", "registro_original"]
    pd.DataFrame(rechazados, columns=columnas).to_csv(
        destino, index=False, encoding="utf-8-sig"
    )


def escribir_reporte(reporte: dict[str, Any], directorio: Path) -> None:
    """Guarda el reporte en JSON (para máquinas) y HTML (para personas)."""
    (directorio / "reporte.json").write_text(
        json.dumps(reporte, ensure_ascii=False, indent=2, default=str),
        encoding="utf-8",
    )
    entorno = Environment(
        loader=FileSystemLoader(PLANTILLAS),
        autoescape=select_autoescape(["html"]),  # los datos vienen de fuera
    )
    html = entorno.get_template("reporte_dq.html").render(r=reporte)
    (directorio / "reporte.html").write_text(html, encoding="utf-8")
```

### 5.2 Plantilla del reporte

Crea `src/calidad_datos/plantillas/reporte_dq.html`. Los datos (el diccionario del reporte) quedan separados de la presentación (la plantilla):

**Archivo:** `src/calidad_datos/plantillas/reporte_dq.html`

```html
<!doctype html>
<html lang="es"><head><meta charset="utf-8">
<title>Calidad · {{ r.dataset }} · {{ r.archivo }}</title>
<style>
  body { font-family: Arial, sans-serif; margin: 2rem; color: #1b2430; }
  table { border-collapse: collapse; margin-bottom: 1.5rem; }
  th, td { border: 1px solid #d0d6dd; padding: .35rem .7rem; text-align: left; }
  .ok { color: #1a7f37; } .falla { color: #c62828; } .aviso { color: #9a6700; }
</style></head>
<body>
  <h1>Reporte de calidad · {{ r.dataset }}</h1>
  <p>Archivo: <strong>{{ r.archivo }}</strong> ·
     Decisión: <strong class="{{ 'ok' if r.aprobado else 'falla' }}">
     {{ 'PUBLICADO' if r.aprobado else 'NO PUBLICADO' }}</strong></p>
  {% for motivo in r.motivos_no_aprobado %}<p class="falla">{{ motivo }}</p>{% endfor %}

  <h2>Conteos</h2>
  <table>
    <tr><th>Leídas</th><th>Válidas</th><th>Rechazadas</th><th>% rechazadas</th><th>Cuadra</th></tr>
    <tr><td>{{ r.conteos.leidas }}</td><td>{{ r.conteos.validas }}</td>
        <td>{{ r.conteos.rechazadas }}</td><td>{{ r.conteos.pct_rechazadas }}</td>
        <td class="{{ 'ok' if r.cuadra else 'falla' }}">{{ 'sí' if r.cuadra else 'no' }}</td></tr>
  </table>

  <h2>Completitud</h2>
  <table>
    <tr><th>Columna</th><th>% presente</th><th>% mínimo</th></tr>
    {% for columna, d in r.completitud.items() %}
    <tr class="{{ 'ok' if d.ok else 'falla' }}"><td>{{ columna }}</td>
        <td>{{ d.presente_pct }}</td><td>{{ d.minimo_pct }}</td></tr>
    {% endfor %}
  </table>

  <h2>Advertencias</h2>
  <ul>
    {% for aviso in r.esquema_avisos %}<li class="aviso">Esquema: {{ aviso }}</li>{% endfor %}
    {% for aviso in r.advertencias %}<li class="aviso">{{ aviso }}</li>{% endfor %}
    {% if r.anomalias.filas %}
    <li class="aviso">Anomalías en {{ r.anomalias.campo }} (IQR {{ r.anomalias.limites_iqr }}):
        filas {{ r.anomalias.filas | join(', ') }}</li>
    {% endif %}
  </ul>

  {% if r.reconciliacion %}
  <h2>Reconciliación</h2>
  <p class="{{ 'ok' if r.reconciliacion.exito else 'falla' }}">
    Conteo {{ r.reconciliacion.conteo.validados }} → {{ r.reconciliacion.conteo.publicados }} ·
    Suma de {{ r.reconciliacion.suma_control.campo }}
    {{ r.reconciliacion.suma_control.validados }} → {{ r.reconciliacion.suma_control.publicados }}</p>
  {% endif %}
  <p>Detalle de cada rechazo: <code>rechazados.csv</code></p>
</body></html>
```

### 5.3 Reconciliación

Crea `src/calidad_datos/reconciliacion.py`. Vuelve a leer el Parquet publicado y lo compara con lo validado en tres niveles: conteo, suma de control con `Decimal` (sin comparar floats con `==`) y claves faltantes o huérfanas.

**Archivo:** `src/calidad_datos/reconciliacion.py`

```python
"""Reconciliación: lo publicado debe coincidir con lo validado."""

from decimal import Decimal
from pathlib import Path
from typing import Any

import pandas as pd


def reconciliar(
    validos: list[dict[str, Any]], parquet: Path, clave: str, campo_control: str
) -> dict[str, Any]:
    """Nivel 1 (conteo), nivel 2 (suma de control) y nivel 3 (claves)."""
    publicado = pd.read_parquet(parquet)
    suma_validos = sum(
        (
            Decimal(str(r[campo_control]))
            for r in validos
            if r[campo_control] is not None
        ),
        Decimal("0"),
    )
    suma_publicada = sum(
        (Decimal(str(v)) for v in publicado[campo_control].dropna()), Decimal("0")
    )
    claves_validas = {str(r[clave]) for r in validos}
    claves_publicadas = {str(v) for v in publicado[clave]}
    resultado = {
        "conteo": {"validados": len(validos), "publicados": len(publicado)},
        "suma_control": {
            "campo": campo_control,
            "validados": str(suma_validos),
            "publicados": str(suma_publicada),
        },
        "faltantes_en_destino": sorted(claves_validas - claves_publicadas),
        "huerfanos_en_destino": sorted(claves_publicadas - claves_validas),
    }
    resultado["exito"] = (
        len(validos) == len(publicado)
        and suma_validos == suma_publicada
        and not resultado["faltantes_en_destino"]
        and not resultado["huerfanos_en_destino"]
    )
    return resultado
```

### 5.4 Orquestador

Crea `src/calidad_datos/pipeline.py`. Coordina las etapas en el orden del diagrama. **No contiene ninguna regla de un dataset concreto**: todo lo específico llega en `DatasetConfig`.

**Archivo:** `src/calidad_datos/pipeline.py`

```python
"""Orquestación del flujo de calidad: un mismo motor para cualquier dataset."""

import logging
from pathlib import Path
from typing import Any

import pandas as pd

from calidad_datos.calidad import (
    comprobar_completitud,
    detectar_anomalias_iqr,
    marcar_duplicados,
)
from calidad_datos.configuracion import DatasetConfig
from calidad_datos.contrato import construir_contrato, validar_fila
from calidad_datos.controles_previos import ejecutar_controles_previos
from calidad_datos.errores import (
    ArchivoEnCuarentenaError,
    LoteNoAprobadoError,
    ReconciliacionError,
)
from calidad_datos.esquema import comparar_esquema
from calidad_datos.lectura import leer_csv
from calidad_datos.reconciliacion import reconciliar
from calidad_datos.reglas import MotorDeReglas
from calidad_datos.salida import (
    enviar_a_cuarentena,
    escribir_parquet,
    escribir_rechazados,
    escribir_reporte,
)
from calidad_datos.transformacion import transformar

logger = logging.getLogger(__name__)


class PipelineCalidad:
    """Recibe la configuración del dataset; la lógica no cambia entre datasets."""

    def __init__(self, config: DatasetConfig, salida: Path, cuarentena: Path) -> None:
        self.config = config
        self.salida = salida
        self.cuarentena = cuarentena
        self.contrato = construir_contrato(config.dataset, config.esquema)
        self.motor = MotorDeReglas(config.reglas)

    def ejecutar(self, archivo: Path) -> dict[str, Any]:
        """Procesa un archivo y devuelve el reporte de calidad.

        Raises:
            FileNotFoundError: Si el archivo no existe.
            ArchivoEnCuarentenaError: Si falla un control previo, estructural o de
                esquema; el archivo se mueve a cuarentena.
            LoteNoAprobadoError: Si el lote no cumple los umbrales de calidad.
            ReconciliacionError: Si lo publicado no coincide con lo validado.
        """
        crudo, avisos_esquema = self._ingerir(archivo)
        df = transformar(crudo, self.config)
        duplicado = marcar_duplicados(df, self.config.calidad.duplicados_por)

        validos: list[dict[str, Any]] = []
        filas_validas: list[int] = []
        rechazados: list[dict[str, Any]] = []
        advertencias: list[str] = []
        for i, fila in enumerate(df.to_dict(orient="records")):
            numero = i + 2  # la línea 1 es el encabezado
            motivos, registro = self._evaluar_fila(
                numero, fila, bool(duplicado.iloc[i])
            )
            if registro is None:
                original = ",".join(str(v) for v in crudo.iloc[i])
                rechazados.append(
                    {
                        "fila": numero,
                        "motivos": " | ".join(motivos),
                        "registro_original": original,
                    }
                )
                logger.warning("Fila %d rechazada: %s", numero, " | ".join(motivos))
            else:
                advertencias.extend(motivos)
                validos.append(registro)
                filas_validas.append(numero)

        reporte = self._construir_reporte(
            archivo,
            df,
            validos,
            filas_validas,
            rechazados,
            advertencias,
            avisos_esquema,
        )
        for advertencia in advertencias:
            logger.warning("Advertencia: %s", advertencia)
        if reporte["anomalias"]["filas"]:
            logger.warning(
                "Anomalía (IQR %s) en %s: filas %s",
                reporte["anomalias"]["limites_iqr"],
                reporte["anomalias"]["campo"],
                reporte["anomalias"]["filas"],
            )
        self._publicar(archivo, validos, rechazados, reporte)
        return reporte

    def _ingerir(self, archivo: Path) -> tuple[pd.DataFrame, list[str]]:
        """Controles previos, lectura y esquema; cualquier fallo aísla el archivo."""
        try:
            ejecutar_controles_previos(archivo, self.config.controles_previos)
            crudo = leer_csv(archivo)
            esquema = comparar_esquema(list(crudo.columns), self.config.esquema)
            if not esquema.es_compatible:
                raise ArchivoEnCuarentenaError(
                    "esquema_incompatible", "; ".join(esquema.incompatibles)
                )
        except ArchivoEnCuarentenaError as error:
            enviar_a_cuarentena(archivo, self.cuarentena, error.motivo, error.detalle)
            raise
        for aviso in esquema.compatibles:
            logger.warning("Cambio de esquema compatible: %s", aviso)
        return crudo, esquema.compatibles

    def _evaluar_fila(
        self, numero: int, fila: dict[str, str], es_duplicado: bool
    ) -> tuple[list[str], dict[str, Any] | None]:
        """Devuelve (motivos, registro). registro es None si la fila se rechaza."""
        if es_duplicado:
            clave = self.config.clave
            return [f"duplicado: {clave}={fila[clave]!r} ya apareció"], None
        registro, errores = validar_fila(self.contrato, fila)
        if registro is None:
            return [f"contrato: {e}" for e in errores], None
        incumplimientos = self.motor.evaluar(numero, registro)
        textos = [f"{i.regla}: {i.campo}={i.valor!r}" for i in incumplimientos]
        errores_regla = [
            t
            for t, i in zip(textos, incumplimientos, strict=True)
            if i.severidad == "error"
        ]
        if errores_regla:
            return [f"regla: {t}" for t in errores_regla], None
        return [f"fila {numero}: {t}" for t in textos], registro

    def _construir_reporte(
        self,
        archivo: Path,
        df: pd.DataFrame,
        validos: list[dict[str, Any]],
        filas_validas: list[int],
        rechazados: list[dict[str, Any]],
        advertencias: list[str],
        avisos_esquema: list[str],
    ) -> dict[str, Any]:
        calidad = self.config.calidad
        completitud = comprobar_completitud(df, calidad.completitud_minima)
        anomalias: list[int] = []
        limites = None
        if calidad.anomalias and validos:
            valores = pd.Series(
                [r[calidad.anomalias.campo] for r in validos], index=filas_validas
            )
            anomalias, limites = detectar_anomalias_iqr(valores, calidad.anomalias.k)
        leidas = len(df)
        pct_rechazados = round(100 * len(rechazados) / leidas, 2) if leidas else 0.0
        motivos_no_aprobado = []
        if pct_rechazados > calidad.max_pct_rechazados:
            motivos_no_aprobado.append(
                f"rechazados {pct_rechazados}% > máximo {calidad.max_pct_rechazados}%"
            )
        for columna, dato in completitud.items():
            if not dato["ok"]:
                motivos_no_aprobado.append(
                    f"completitud de {columna} {dato['presente_pct']}% "
                    f"< mínimo {dato['minimo_pct']}%"
                )
        return {
            "dataset": self.config.dataset,
            "archivo": archivo.name,
            "conteos": {
                "leidas": leidas,
                "validas": len(validos),
                "rechazadas": len(rechazados),
                "pct_rechazadas": pct_rechazados,
            },
            "cuadra": leidas == len(validos) + len(rechazados),
            "esquema_avisos": avisos_esquema,
            "completitud": completitud,
            "advertencias": advertencias,
            "anomalias": {
                "campo": calidad.anomalias.campo if calidad.anomalias else None,
                "filas": anomalias,
                "limites_iqr": limites,
            },
            "aprobado": not motivos_no_aprobado,
            "motivos_no_aprobado": motivos_no_aprobado,
            "reconciliacion": None,
        }

    def _publicar(
        self,
        archivo: Path,
        validos: list[dict[str, Any]],
        rechazados: list[dict[str, Any]],
        reporte: dict[str, Any],
    ) -> None:
        directorio = self.salida / self.config.dataset / archivo.stem
        directorio.mkdir(parents=True, exist_ok=True)
        escribir_rechazados(rechazados, directorio / "rechazados.csv")
        parquet = directorio / "validos.parquet"
        parquet.unlink(missing_ok=True)  # nunca dejar publicado un lote anterior
        if reporte["aprobado"] and validos:
            escribir_parquet(validos, parquet)
            reporte["reconciliacion"] = reconciliar(
                validos, parquet, self.config.clave, self.config.campo_control
            )
        escribir_reporte(reporte, directorio)
        if not reporte["cuadra"]:
            raise ReconciliacionError("leídas ≠ válidas + rechazadas")
        if reporte["reconciliacion"] and not reporte["reconciliacion"]["exito"]:
            parquet.unlink(missing_ok=True)
            raise ReconciliacionError(f"{reporte['reconciliacion']}")
        if not reporte["aprobado"]:
            raise LoteNoAprobadoError("; ".join(reporte["motivos_no_aprobado"]))
```

Puntos clave:

- **Cuarentena de archivos:** cualquier fallo en controles previos, lectura o esquema mueve el archivo completo y detiene **solo ese archivo**.
- **Rechazo de filas:** un duplicado, un incumplimiento del contrato o una regla con `severity: error` envían la fila a `rechazados.csv` con todos sus motivos. El resto del lote continúa.
- **Decisión del lote:** si el % de rechazos supera el máximo o falla la completitud, el lote **no se publica**. Además se borra cualquier `validos.parquet` anterior, para que nunca quede publicado un lote viejo.
- **Reconciliación:** si lo publicado no coincide con lo validado, se retira el Parquet.

### 5.5 Punto de entrada

**Archivo:** `src/calidad_datos/cli.py`

```python
"""Punto de entrada: python -m calidad_datos <archivo> --dataset <nombre>."""

import argparse
import logging
from pathlib import Path

from calidad_datos.configuracion import cargar_configuracion
from calidad_datos.errores import CalidadError
from calidad_datos.pipeline import PipelineCalidad
from calidad_datos.reglas import VALIDADORES

logger = logging.getLogger("calidad_datos")


def construir_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(
        prog="python -m calidad_datos",
        description="Valida contrato y calidad de un archivo con reglas configurables.",
    )
    parser.add_argument("archivo", type=Path, help="CSV a procesar")
    parser.add_argument(
        "--dataset", required=True, help="nombre del YAML en config/datasets/"
    )
    parser.add_argument("--config-dir", type=Path, default=Path("config/datasets"))
    parser.add_argument("--salida", type=Path, default=Path("salida"))
    parser.add_argument("--cuarentena", type=Path, default=Path("cuarentena"))
    return parser


def main(argv: list[str] | None = None) -> int:
    logging.basicConfig(
        level=logging.INFO, format="%(levelname)-8s | %(name)s | %(message)s"
    )
    args = construir_parser().parse_args(argv)
    try:
        config = cargar_configuracion(
            args.config_dir / f"{args.dataset}.yaml", set(VALIDADORES)
        )
        pipeline = PipelineCalidad(config, args.salida, args.cuarentena)
        reporte = pipeline.ejecutar(args.archivo)
    except FileNotFoundError as error:
        logger.error("%s", error)
        return 3
    except CalidadError as error:
        logger.error("%s", error)
        return error.codigo_salida

    c = reporte["conteos"]
    logger.info(
        "%s | leídas=%d válidas=%d rechazadas=%d (%.2f%%) | publicado: sí",
        reporte["dataset"],
        c["leidas"],
        c["validas"],
        c["rechazadas"],
        c["pct_rechazadas"],
    )
    return 0
```

**Archivo:** `src/calidad_datos/__main__.py`

```python
"""Permite ejecutar el paquete con: python -m calidad_datos."""

from calidad_datos.cli import main

raise SystemExit(main())
```

**Verificación 5.5:**

```bash
python -m calidad_datos --help
```

Debe mostrar el argumento `archivo` y las opciones `--dataset`, `--config-dir`, `--salida` y `--cuarentena`.

> **Checkpoint 5.** `--help` funciona.

---

## Parte 6. Ejecutar sobre dos datasets y cuatro archivos defectuosos (15 min)

### Códigos de salida

| Código | Significado |
| :---: | :--- |
| 0 | Lote validado y publicado; la reconciliación cuadra |
| 1 | Configuración YAML inválida o inexistente |
| 2 | Argumentos de línea de comandos inválidos |
| 3 | Archivo inexistente o enviado a cuarentena |
| 4 | La reconciliación no cuadra |
| 5 | El lote no cumple los umbrales de calidad; no se publica |

Para ver el código de salida del último comando: **PowerShell** `$LASTEXITCODE` | **bash** `echo $?`

### 6.1 Ventas

```bash
python -m calidad_datos datos/entrada/ventas_2026-10-24.csv --dataset ventas
```

Resultado esperado (código **0**):

```text
WARNING  | calidad_datos.pipeline | Cambio de esquema compatible: columna nueva 'observaciones' (se ignora)
WARNING  | calidad_datos.pipeline | Fila 5 rechazada: contrato: monto: Input should be a valid decimal
WARNING  | calidad_datos.pipeline | Fila 6 rechazada: regla: verificar_monto_positivo: monto='-150.00'
WARNING  | calidad_datos.pipeline | Fila 8 rechazada: duplicado: id_venta='V-10002' ya apareció
WARNING  | calidad_datos.pipeline | Fila 9 rechazada: contrato: fecha: Input should be a valid date or datetime, month value is outside expected range of 1-12
WARNING  | calidad_datos.pipeline | Fila 11 rechazada: regla: canal_en_catalogo: canal='telefono'
WARNING  | calidad_datos.pipeline | Fila 12 rechazada: contrato: sucursal: Input should be greater than or equal to 1
WARNING  | calidad_datos.pipeline | Fila 13 rechazada: contrato: id_venta: Field required
WARNING  | calidad_datos.pipeline | Advertencia: fila 7: validar_formato_correo: cliente_email='elena.soto-empresa.mx'
WARNING  | calidad_datos.pipeline | Anomalía (IQR (-1147.61, 2317.58)) en monto: filas [10]
INFO     | calidad_datos | ventas | leídas=21 válidas=14 rechazadas=7 (33.33%) | publicado: sí
```

Revisa `salida/ventas/ventas_2026-10-24/`:

| Archivo | Qué comprobar |
| :--- | :--- |
| `rechazados.csv` | 7 filas, cada una con número de línea, motivo y registro original |
| `validos.parquet` | 14 registros. `V-10002` aparece **una sola vez** y con fecha `2026-10-24` (venía como `24/10/2026`) |
| `reporte.html` | Ábrelo en el navegador: decisión **PUBLICADO**, completitud de `cliente_email` 95.24 % y reconciliación `14 → 14` |
| `reporte.json` | La misma información, para otros sistemas |

Observa tres decisiones distintas:

- **Fila 7** (correo sin `@`): se **publica** con una advertencia, porque esa regla es `warning`.
- **Fila 10** ($95,000): se **publica**, pero se señala como **anomalía** para revisión humana.
- **Fila 6** (monto negativo): se **rechaza**, porque esa regla es `error`.

**Repetibilidad:** ejecuta el comando otra vez. Las salidas se sobrescriben con el mismo resultado.

### 6.2 Clientes: el mismo motor, otra configuración

```bash
python -m calidad_datos datos/entrada/clientes_2026-10-24.csv --dataset clientes
```

Resultado esperado (código **0**):

```text
WARNING  | calidad_datos.pipeline | Fila 5 rechazada: regla: limite_credito_valido: limite_credito='-500'
WARNING  | calidad_datos.pipeline | Fila 6 rechazada: regla: segmento_en_catalogo: segmento='vip'
WARNING  | calidad_datos.pipeline | Fila 7 rechazada: duplicado: id_cliente='2' ya apareció
WARNING  | calidad_datos.pipeline | Fila 8 rechazada: contrato: nombre: Field required
WARNING  | calidad_datos.pipeline | Advertencia: fila 4: validar_formato_correo: email='luis.nunez@correo'
WARNING  | calidad_datos.pipeline | Advertencia: fila 10: telefono_10_digitos: telefono='55 1234'
WARNING  | calidad_datos.pipeline | Anomalía (IQR (-43500.0, 107000.0)) en limite_credito: filas [3]
INFO     | calidad_datos | clientes | leídas=12 válidas=8 rechazadas=4 (33.33%) | publicado: sí
```

No escribiste código nuevo: cambiaron el archivo (`cp1252`, `;`), el esquema y las reglas, y el motor es el mismo. La **anomalía** de la fila 3 (Peña y Asociados, $120,000) es un cliente empresarial legítimo. Por eso una anomalía se **señala** y no se rechaza automáticamente.

### 6.3 Archivos defectuosos: cuarentena

Ejecuta cada caso. Todos deben terminar con código **3** y mover el archivo a `cuarentena/`.

**Caso A. Invalidez estructural** (una fila con columnas de más)
```bash
python -m calidad_datos datos/entrada/ventas_estructura_rota.csv --dataset ventas
```
```text
WARNING  | calidad_datos.salida | {"estado": "QUARANTINED", "archivo": "ventas_estructura_rota.csv", "motivo": "estructura", "detalle": "Error tokenizing data. C error: Expected 7 fields in line 3, saw 10"}
ERROR    | calidad_datos | estructura: Error tokenizing data. C error: Expected 7 fields in line 3, saw 10
```

**Caso B. Cambio de esquema incompatible** (falta `monto`)
```bash
python -m calidad_datos datos/entrada/ventas_sin_monto.csv --dataset ventas
```
```text
WARNING  | calidad_datos.salida | {"estado": "QUARANTINED", "archivo": "ventas_sin_monto.csv", "motivo": "esquema_incompatible", "detalle": "falta 'monto' (requerido=True)"}
ERROR    | calidad_datos | esquema_incompatible: falta 'monto' (requerido=True)
```

**Caso C. Archivo vacío** (control previo de tamaño)
```bash
python -m calidad_datos datos/entrada/ventas_vacio.csv --dataset ventas
```
```text
WARNING  | calidad_datos.salida | {"estado": "QUARANTINED", "archivo": "ventas_vacio.csv", "motivo": "vacio", "detalle": "0 bytes; mínimo 20"}
ERROR    | calidad_datos | vacio: 0 bytes; mínimo 20
```

**Caso D. Archivo alterado en tránsito** (hash contra el manifiesto). Cambia un monto del archivo de ventas y vuelve a procesarlo:
```bash
python -c "from pathlib import Path; p = Path('datos/entrada/ventas_2026-10-24.csv'); p.write_text(p.read_text(encoding='utf-8').replace('1250.50', '12.50'), encoding='utf-8')"
python -m calidad_datos datos/entrada/ventas_2026-10-24.csv --dataset ventas
```
```text
WARNING  | calidad_datos.salida | {"estado": "QUARANTINED", "archivo": "ventas_2026-10-24.csv", "motivo": "hash_distinto", "detalle": "el contenido no coincide con el manifiesto"}
ERROR    | calidad_datos | hash_distinto: el contenido no coincide con el manifiesto
```

Revisa la carpeta `cuarentena/`. Cada archivo lleva la fecha UTC y el motivo en el nombre; el archivo de ventas se movió junto con su manifiesto:

```text
<fecha>_estructura_ventas_estructura_rota.csv
<fecha>_esquema_incompatible_ventas_sin_monto.csv
<fecha>_vacio_ventas_vacio.csv
<fecha>_hash_distinto_ventas_2026-10-24.csv
<fecha>_hash_distinto_ventas_2026-10-24.csv.sha256
```

**6.4** Regenera los archivos de entrada para la siguiente parte:

```bash
python scripts/preparar_datos.py
```

---

## Parte 7. Cambiar reglas y umbrales sin tocar la lógica (10 min)

En cada ejercicio cambias **solo configuración**, salvo en el 7.1, donde agregas un validador al registro sin tocar el motor ni el orquestador.

### 7.1 Agregar un validador y una regla nueva

El registro `V-10019` trae la fecha `31/12/2099`: es válida para el contrato, pero imposible para el negocio.

**7.1.1** En `src/calidad_datos/reglas.py`, debajo de `from dataclasses import dataclass`, agrega:

```python
from datetime import date
```

**7.1.2** Al final del bloque de validadores (después de `en_catalogo`), agrega:

```python
@registrar("not_future")
def no_futuro(valor: Any, params: dict[str, Any]) -> bool:
    return bool(valor <= date.today())
```

**7.1.3** En `config/datasets/ventas.yaml`, agrega esta regla al final de la sección `reglas`, justo antes del comentario `# Controles del lote`:

```yaml
  - name: fecha_no_futura
    field: fecha
    validator: not_future
    severity: error
    parameters: {}
```

**7.1.4** Ejecuta ventas otra vez:

```bash
python -m calidad_datos datos/entrada/ventas_2026-10-24.csv --dataset ventas
```

Esperado: un rechazo nuevo, `Fila 22 rechazada: regla: fecha_no_futura: fecha='2099-12-31'`, y el resumen `leídas=21 válidas=13 rechazadas=8 (38.10%) | publicado: sí`.

> `V-10019` es el registro 21 del archivo; el log dice **fila 22** porque la línea 1 es el encabezado.

**Conserva este cambio.** Los siguientes ejercicios parten de aquí.

### 7.2 Endurecer el umbral del lote

En `ventas.yaml` cambia `max_pct_rechazados: 40` por `max_pct_rechazados: 30` y ejecuta ventas. Esperado:

```text
ERROR    | calidad_datos | rechazados 38.1% > máximo 30.0%
```

Código **5**. En `salida/ventas/ventas_2026-10-24/` ya **no existe** `validos.parquet`, y `reporte.html` dice **NO PUBLICADO**. **Regresa el valor a 40.**

### 7.3 Exigir más completitud

Cambia `completitud_minima: {cliente_email: 90}` por `{cliente_email: 98}` y ejecuta ventas. Esperado:

```text
ERROR    | calidad_datos | completitud de cliente_email 95.24% < mínimo 98.0%
```

Código **5**. En `reporte.html` el `<` del motivo se ve como texto, porque `autoescape` lo escapó. **Regresa el valor a 90.**

### 7.4 Error de configuración: fail-fast

Cambia `validator: in_set` por `validator: catalogo` en la regla `canal_en_catalogo` y ejecuta ventas. Esperado:

```text
ERROR    | calidad_datos | Validadores no registrados: ['catalogo']
```

Código **1**: el proceso se detiene **antes de leer el archivo**. **Regresa el valor a `in_set`.**

### 7.5 Pregunta de cierre

Haz una lista de los archivos de Python que modificaste en los ejercicios 7.2 a 7.4. ¿Qué criterio de evaluación demuestra esa lista?

---

## Parte 8. Pruebas y calidad del código (8 min)

**8.1** Crea `tests/test_reglas.py`:

**Archivo:** `tests/test_reglas.py`

```python
"""Pruebas del registro de validadores y del motor de reglas."""

from decimal import Decimal

import pytest

from calidad_datos.configuracion import ReglaConfig
from calidad_datos.reglas import VALIDADORES, MotorDeReglas


@pytest.mark.parametrize(
    ("validador", "valor", "parametros", "esperado"),
    [
        ("range", Decimal("10.00"), {"min": 0.01, "max": 100}, True),
        ("range", Decimal("-1"), {"min": 0.01, "max": 100}, False),
        ("in_set", "web", {"values": ["tienda", "web"]}, True),
        ("in_set", "fax", {"values": ["tienda", "web"]}, False),
        ("regex", "ana@empresa.mx", {"pattern": r"^[\w.+-]+@[\w-]+(\.[\w-]+)+$"}, True),
        ("regex", "ana@empresa", {"pattern": r"^[\w.+-]+@[\w-]+(\.[\w-]+)+$"}, False),
        ("not_null", "  ", {}, False),
    ],
)
def test_validadores(
    validador: str, valor: object, parametros: dict, esperado: bool
) -> None:
    assert VALIDADORES[validador](valor, parametros) is esperado


def test_motor_acumula_errores_y_omite_ausentes() -> None:
    reglas = [
        ReglaConfig(
            name="monto",
            field="monto",
            validator="range",
            severity="error",
            parameters={"min": 1},
        ),
        ReglaConfig(
            name="canal",
            field="canal",
            validator="in_set",
            severity="error",
            parameters={"values": ["web"]},
        ),
        ReglaConfig(
            name="correo",
            field="correo",
            validator="regex",
            severity="warning",
            parameters={"pattern": ".+@.+"},
        ),
    ]
    resultado = MotorDeReglas(reglas).evaluar(
        2, {"monto": Decimal("0"), "canal": "fax", "correo": None}
    )
    assert [i.regla for i in resultado] == ["monto", "canal"]
```

**8.2** Crea `tests/test_contrato.py`:

**Archivo:** `tests/test_contrato.py`

```python
"""Pruebas del contrato generado a partir del esquema."""

from datetime import date
from decimal import Decimal

from calidad_datos.configuracion import ColumnaConfig
from calidad_datos.contrato import construir_contrato, validar_fila

ESQUEMA = {
    "id": ColumnaConfig(tipo="texto", min_largo=3),
    "fecha": ColumnaConfig(tipo="fecha"),
    "monto": ColumnaConfig(tipo="decimal"),
    "nota": ColumnaConfig(tipo="texto", requerido=False),
}
CONTRATO = construir_contrato("prueba", ESQUEMA)


def test_fila_valida_queda_tipada() -> None:
    registro, errores = validar_fila(
        CONTRATO, {"id": "A-1", "fecha": "2026-10-24", "monto": "10.50", "nota": ""}
    )
    assert errores == []
    assert registro == {
        "id": "A-1",
        "fecha": date(2026, 10, 24),
        "monto": Decimal("10.50"),
        "nota": None,
    }


def test_fila_invalida_reporta_todos_los_errores() -> None:
    registro, errores = validar_fila(
        CONTRATO, {"id": "", "fecha": "2026-13-01", "monto": "abc", "nota": ""}
    )
    assert registro is None
    assert [e.split(":")[0] for e in errores] == ["id", "fecha", "monto"]
```

**8.3** Crea `tests/test_pipeline.py`. Usa `tmp_path` para que las pruebas no toquen tus carpetas reales:

**Archivo:** `tests/test_pipeline.py`

```python
"""Prueba de extremo a extremo con archivos temporales (tmp_path)."""

from pathlib import Path

import pytest

from calidad_datos.configuracion import cargar_configuracion
from calidad_datos.errores import ArchivoEnCuarentenaError, LoteNoAprobadoError
from calidad_datos.pipeline import PipelineCalidad
from calidad_datos.reglas import VALIDADORES

CONFIG = Path("config/datasets/ventas.yaml")
ENCABEZADO = "id_venta,fecha,sucursal,cliente_email,monto,canal\n"


def crear(tmp_path: Path, contenido: str) -> Path:
    archivo = tmp_path / "entrada" / "lote.csv"
    archivo.parent.mkdir()
    archivo.write_text(contenido, encoding="utf-8")
    return archivo


def pipeline(tmp_path: Path) -> PipelineCalidad:
    config = cargar_configuracion(CONFIG, set(VALIDADORES))
    return PipelineCalidad(config, tmp_path / "salida", tmp_path / "cuarentena")


def test_lote_valido_se_publica_y_reconcilia(tmp_path: Path) -> None:
    archivo = crear(
        tmp_path,
        ENCABEZADO
        + "V-00001,2026-10-24,1,ana@empresa.mx,100.00,web\n"
        + "V-00002,24/10/2026,2,luis@empresa.mx,50.50,tienda\n",
    )
    reporte = pipeline(tmp_path).ejecutar(archivo)
    assert reporte["aprobado"] and reporte["reconciliacion"]["exito"]
    assert reporte["reconciliacion"]["suma_control"]["publicados"] == "150.50"


def test_lote_con_demasiados_rechazos_no_se_publica(tmp_path: Path) -> None:
    archivo = crear(
        tmp_path,
        ENCABEZADO
        + "V-00001,2026-10-24,1,ana@empresa.mx,100.00,web\n"
        + "V-00002,2026-10-24,1,luis@empresa.mx,-5,web\n",
    )
    with pytest.raises(LoteNoAprobadoError):
        pipeline(tmp_path).ejecutar(archivo)
    assert not (tmp_path / "salida/ventas/lote/validos.parquet").exists()


def test_esquema_incompatible_va_a_cuarentena(tmp_path: Path) -> None:
    archivo = crear(tmp_path, "id_venta,fecha\nV-00001,2026-10-24\n")
    with pytest.raises(ArchivoEnCuarentenaError):
        pipeline(tmp_path).ejecutar(archivo)
    assert not archivo.exists()
    assert len(list((tmp_path / "cuarentena").iterdir())) == 1
```

**8.4** Ejecuta:

```bash
python -m pytest -q
ruff check src tests scripts
python -m mypy src tests scripts
```

Esperado: `13 passed`, `All checks passed!` y `Success: no issues found`.

---

## Entregables

Comprime `practica2_calidad` **sin** `.venv/`, `datos/entrada/`, `cuarentena/` ni `salida/`. Antes de entregar, revisa:

- [ ] `config/datasets/ventas.yaml` (con la regla `fecha_no_futura`) y `clientes.yaml`
- [ ] Paquete `src/calidad_datos/` con 15 archivos `.py` y la plantilla HTML
- [ ] `requirements.txt` y `requirements-dev.txt` generados con pip-tools
- [ ] Ventas: 8 rechazos con motivo, publicado y reconciliado
- [ ] Clientes: 4 rechazos con motivo, publicado y reconciliado
- [ ] Casos A a D en cuarentena con código 3
- [ ] Ejercicios 7.2 a 7.4 comprobados y revertidos
- [ ] `pytest`, `ruff` y `mypy` sin errores

---

## Solución de problemas

| Síntoma | Causa probable | Solución |
| :--- | :--- | :--- |
| `ModuleNotFoundError: No module named 'calidad_datos'` | El paquete no está instalado en el entorno activo | Activa `.venv` y ejecuta `python -m pip install -e .` desde la raíz |
| `TemplateNotFound: reporte_dq.html` | La plantilla no está en `src/calidad_datos/plantillas/` o tiene otro nombre | Revisa la ruta y el nombre exactos |
| `No existe o no es legible: datos/entrada/...` | El archivo ya se movió a cuarentena en una ejecución anterior | Ejecuta `python scripts/preparar_datos.py` |
| Ventas termina en `hash_distinto` sin haberlo modificado | Editaste o guardaste el CSV en VS Code o Excel después de generarlo | Ejecuta `python scripts/preparar_datos.py` |
| `Validadores no registrados` en el ejercicio 7.1 | Falta el decorador `@registrar("not_future")` o el nombre no coincide con el YAML | Usa exactamente `not_future` en ambos lugares |
| PowerShell no deja activar `.venv` | Política de ejecución de Windows | `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` |

---

## Limpieza

**PowerShell**
```powershell
Remove-Item -Recurse -Force cuarentena, salida -ErrorAction SilentlyContinue
deactivate
```

**bash**
```bash
rm -rf cuarentena salida
deactivate
```

---

## Resumen

| Lo que hiciste | Lección |
| :--- | :--- |
| Detectaste `cp1252` y `;` sin configurarlos | 2.1 CSV |
| Leíste todo como texto y publicaste Parquet tipado | 2.1, 2.4 |
| Normalizaste fechas en dos formatos sin adivinar los inválidos | 2.7 Transformación |
| Generaste el contrato Pydantic desde YAML y obtuviste todos los errores de cada fila | 2.10 Validación de esquemas |
| Clasificaste columnas faltantes y nuevas como incompatibles o compatibles | 2.11, 2.12 |
| Aislaste archivos rotos con log JSON y continuaste con los demás | 2.13 Estructuras no válidas |
| Revisaste tamaño y hash antes de abrir el archivo | 2.14 Controles previos |
| Aplicaste reglas `error` / `warning` desde YAML con un registro de validadores | 2.15, 2.22 |
| Comprobaste conteo, suma de control y claves después de publicar | 2.17 Reconciliación |
| Señalaste valores atípicos con IQR sin rechazarlos | 2.18 Anomalías |
| Mediste la completitud por columna contra un mínimo | 2.19 Completitud |
| Separaste duplicados exactos sin descartarlos en silencio | 2.20 Duplicados |
| Generaste un reporte JSON y HTML con Jinja2 y `autoescape` | 2.21 Reportes |
| Usaste el mismo motor para ventas y clientes | 2.22 Reglas reutilizables |
