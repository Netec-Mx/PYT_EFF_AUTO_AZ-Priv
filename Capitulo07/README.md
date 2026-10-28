# Práctica 7. Productividad del desarrollo asistida por IA

| Dato | Detalle |
| :--- | :--- |
| **Curso** | Automatización de procesos con Python |
| **Capítulo** | 7. IA aplicada al desarrollo y automatizaciones reutilizables |
| **Duración** | 120 minutos |
| **Modalidad** | Individual |
| **Nivel de Bloom** | Aplicar / Evaluar |

---

## Objetivo

Aplicar IA de manera controlada sobre actividades concretas del ciclo de desarrollo Python y validar técnicamente el resultado antes de incorporarlo. Usarás **Claude Code** o **GitHub Copilot** sobre código existente para **revisar**, **refactorizar y depurar**, **generar pruebas** y **documentar**. Cada propuesta de la IA es un borrador: solo entra a la solución si cruza las capas de verificación del equipo.

> **La IA propone, la ingeniería verifica.** Las respuestas de un asistente cambian de una persona a otra y de un día a otro. Por eso esta guía no te pide copiar "lo que respondió la IA": te da los prompts, lo que debes revisar en cada respuesta y la salida **determinista** de las verificaciones, que es igual para todos.

## El caso: ticket ABC-142

El equipo de ventas tiene un script que calcula comisiones, las guarda en SQLite y envía el total por vendedor a un webhook. Con los datos de todos los días "funciona", pero el área de ventas reporta comisiones incorrectas en los bordes y el equipo de seguridad lo bloqueó: tiene un token en el código y TLS desactivado.

Tu tarea: corregirlo y dejarlo mantenible con ayuda de la IA, **sin cambiar su contrato** (la CLI, la tabla y el JSON que recibe el webhook), y demostrar con evidencia que el resultado es correcto, seguro y mantenible.

```text
                  código heredado (src/comisiones/proceso.py)
                                 │
     ┌───────────────────────────┼─────────────────────────────────┐
     │ IA (Claude Code o Copilot): revisión → refactor y corrección → pruebas → documentación │
     └───────────────────────────┼─────────────────────────────────┘
                                 ▼  borrador
   herramientas/verificar.py  ── confianza cero: 14 capas, de la más barata a la más cara ──
     controles intactos · caracterización · criterios de aceptación · políticas del equipo
     pruebas generadas · mutantes (¿las pruebas detectan errores?) · doctest
     Ruff · formato · mypy --strict · Bandit · Xenon (complejidad) · gobierno (AST) · dependencias
                                 │
              APROBADO ──► commit con "Asistido-por-IA" + EVIDENCIA.md
              RECHAZADO ─► el error real vuelve al prompt, o lo corriges tú
```

### Relación con el capítulo

| Actividad del laboratorio | Lección |
| :--- | :--- |
| Prompts con rol, contexto, tarea, restricciones, formato y criterios de aceptación | 7.1 Generación de código |
| `test_reglas.py` generado por la IA y evaluado con mutantes (`probar_las_pruebas.py`) | 7.2 Generación de pruebas |
| Revisión pedida con contexto, prioridades y formato; hallazgos confirmados con herramientas | 7.3 Revisión de código |
| Docstrings estilo Google con ejemplos que se ejecutan (doctest) | 7.4 Documentación |
| Refactorización protegida por una prueba de caracterización | 7.5 Refactorización |
| Corrección de los criterios que fallan: el error real vuelve al prompt | 7.6 Depuración |
| `verificar.py`: capas de confianza cero, de la más barata a la más cara | 7.8 Validación de código generado |
| Claude Code (modo plan, permisos) o GitHub Copilot (chat y agente) | 7.9, 7.10, 7.11 Herramientas |
| `AGENTS.md` y `CLAUDE.md`: las reglas del equipo, escritas una vez | 7.12 Integración en el flujo |
| Lista de revisión humana antes de aceptar un cambio | 7.13 Revisión humana |
| `test_criterios_aceptacion.py`: valores acordados con el negocio | 7.14 Validación funcional |
| Fronteras, valores no numéricos, montos en cero, ejecuciones repetidas | 7.15 Pruebas del código generado |
| Ruff (reglas S) y Bandit; secretos, TLS, timeout, SQL con parámetros | 7.16 Revisión de seguridad |
| `revisar_dependencias.py`: paquetes no aprobados y nombres inventados | 7.17 Dependencias y riesgos |
| Xenon con umbral de complejidad | 7.18 Mantenibilidad |
| Puerta de gobierno sobre el AST, controles intactos y evidencia en JSON | 7.19 Gobierno |

> Las lecciones 7.20 a 7.27 (bibliotecas compartidas, plantillas, interfaces y versionado) no forman parte de esta práctica; `reglas.py` como lógica pura separada de la entrada y salida es el primer paso hacia ellas (7.27).

## Criterios de evaluación

| Criterio | Evidencia en tu entrega |
| :--- | :--- |
| **Corrección** | Caracterización en verde (el comportamiento con datos típicos no cambió) y los 13 criterios de aceptación en verde |
| **Mantenibilidad** | Ruff, formato y `mypy --strict` sin hallazgos; Xenon dentro del umbral; docstrings ≥ 80 % y ejemplos que se ejecutan |
| **Seguridad** | Bandit sin hallazgos MEDIUM ni HIGH; sin secretos ni datos personales en el código o los logs; dependencias aprobadas; los controles del equipo sin modificar |
| **Valor operativo** | `EVIDENCIA.md`: prompts usados, qué propuso la IA, qué rechazaste o corregiste y por qué, puntaje de mutantes y la salida final de `verificar.py` en APROBADO |

---

## Requisitos

| Elemento | Requisito |
| :--- | :--- |
| Python | **3.12 o superior** (probado con 3.12, 3.13 y 3.14) |
| Git | Cualquier versión reciente (`git --version`) |
| Editor | Visual Studio Code |
| Asistente de IA | **Una** de las dos: **Claude Code** (cuenta Claude Pro, Max, Team o Enterprise, o Claude Console) o **GitHub Copilot** en VS Code (sirve el plan Copilot Free, con límites mensuales, o una licencia Pro, Business o Enterprise) |
| Sistema operativo | Windows 10/11, macOS o Linux |
| Red | Acceso a PyPI y al servicio de tu asistente. No se usan bases de datos ni servicios externos: el webhook se simula |

> **Qué no se comparte con la IA.** Los datos de esta práctica son sintéticos. En tu trabajo, nunca pegues en un asistente credenciales, datos de clientes ni código que tu organización no haya autorizado (7.6, 7.19).

> **Convenciones**
> - Todos los comandos se ejecutan desde la **carpeta raíz del proyecto** (`practica7_ia`) con el entorno virtual activo.
> - Cuando un comando cambia según el sistema operativo, verás la versión **PowerShell** y la versión **bash**.
> - Los bloques **Prompt** se pegan tal cual en el asistente. Las salidas de la IA variarán; las de `verificar.py`, no.

### Distribución del tiempo

| Parte | Contenido | Minutos |
| :--- | :--- | :---: |
| 0. Preparación | Proyecto, controles del equipo, entorno, Git y asistente | 15 |
| 1. Línea base | Verificación de confianza cero sobre el código heredado | 10 |
| 2. Revisión con IA | Hallazgos de la IA contra herramientas y pruebas | 15 |
| 3. Refactorización y depuración con IA | Plan, cambio, revisión humana y verificación | 30 |
| 4. Pruebas con IA | Prueba tautológica, pruebas generadas y mutantes | 20 |
| 5. Documentación con IA | Docstrings Google con ejemplos ejecutables | 10 |
| 6. Seguridad, gobierno y cierre técnico | Dependencias inventadas, controles intactos, verificación final y evidencia | 15 |
| Cierre | Preguntas | 5 |

> **Checkpoints.** Al final de las partes 3, 4 y 5 hay un punto de control. Si la IA se atora o tu tiempo se acaba, el instructor tiene la solución de referencia de cada parte.

---

## Parte 0. Preparación (15 min)

**0.1** Crea la carpeta `practica7_ia`, ábrela en VS Code (**File → Open Folder**) y crea la estructura:

**PowerShell**
```powershell
New-Item -ItemType Directory -Force -Path src/comisiones, tests/datos, herramientas
```

**bash**
```bash
mkdir -p src/comisiones tests/datos herramientas
```

**0.2** Archivos del proyecto. `pyproject.toml` fija las reglas de Ruff, mypy y pytest; `AGENTS.md` son las reglas del equipo que leen Copilot y Claude Code (`CLAUDE.md` lo importa con `@AGENTS.md`, 7.12).

**Archivo:** `pyproject.toml`

```toml
[build-system]
requires = ["setuptools>=77"]
build-backend = "setuptools.build_meta"

[project]
name = "comisiones"
version = "1.0.0"
description = "Cálculo de comisiones de vendedores y aviso por webhook"
requires-python = ">=3.12"
dependencies = ["requests>=2.32,<3"]

[tool.setuptools.packages.find]
where = ["src"]

[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["."]
addopts = "-ra --strict-markers --strict-config"

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "S"]   # S: reglas de seguridad (bandit)

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101", "S105", "S106"]     # assert y tokens falsos en pruebas

[tool.ruff.lint.isort]
known-first-party = ["comisiones", "tests"]

[tool.mypy]
strict = true
files = ["src"]
```

**Archivo:** `requirements.in`

```text
# Dependencias de ejecución (las mismas de pyproject.toml)
requests>=2.32,<3
```

**Archivo:** `requirements.txt`

```text
# Versiones fijadas para la Práctica 7 (hay ruedas para Windows, macOS 10.15+ y Linux; Python 3.12-3.14).
# Generado con:
#   pip-compile --strip-extras --no-annotate -o requirements.txt requirements.in
certifi==2026.10.23
charset-normalizer==3.5.2
idna==3.20
requests==2.34.2
urllib3==2.8.0
```

**Archivo:** `requirements-dev.in`

```text
# Pruebas y controles de ingeniería; respetan las versiones de requirements.txt
-c requirements.txt
bandit
mypy
pytest
radon
responses
ruff
types-requests
xenon
```

**Archivo:** `requirements-dev.txt`

```text
# Versiones fijadas para la Práctica 7 (hay ruedas para Windows, macOS 10.15+ y Linux; Python 3.12-3.14).
# Generado con:
#   pip-compile --strip-extras --no-annotate -o requirements-dev.txt requirements-dev.in
ast-serialize==0.13.1
bandit==1.9.4
certifi==2026.10.23
charset-normalizer==3.5.2
colorama==0.4.6
idna==3.20
iniconfig==2.3.1
librt==0.16.0
mando==0.7.1
markdown-it-py==4.2.0
mdurl==0.1.2
mypy==2.4.0
mypy-extensions==1.1.0
packaging==26.3
pathspec==1.1.1
pluggy==1.7.0
pygments==2.21.0
pytest==9.2.0
pyyaml==6.0.3
radon==6.0.1
requests==2.34.2
responses==0.26.3
rich==15.0.0
ruff==0.17.3
six==1.17.0
stevedore==5.9.1
types-requests==2.33.0.20261006
typing-extensions==4.16.0
urllib3==2.8.0
xenon==0.9.3
```

**Archivo:** `.gitignore`

```text
.venv/
__pycache__/
.pytest_cache/
.mypy_cache/
.ruff_cache/
*.egg-info/
# Bases de datos y salidas locales
*.db
```

**Archivo:** `README.md`

````markdown
# comisiones

Calcula las comisiones de los vendedores a partir de un CSV de ventas, las guarda en
SQLite y envía el total por vendedor a un webhook.

```bash
python -m comisiones ventas.csv comisiones.db https://hooks.empresa.internal/comisiones
```

**Ticket ABC-142.** El código heredado funciona con los datos típicos, pero no cumple
los criterios del área de ventas ni las políticas del equipo. Hay que corregirlo y
dejarlo mantenible **sin cambiar su contrato** (ver `AGENTS.md`). La verificación de
aceptación es `python herramientas/verificar.py`.
````

**Archivo:** `AGENTS.md`

```markdown
# comisiones: automatización Python de comisiones de vendedores

## Entorno
- Python 3.12 o superior. Dependencias solo en `pyproject.toml`; versiones fijas en `requirements*.txt`.
- Pruebas: `python -m pytest -q` · Lint: `ruff check src tests` · Formato: `ruff format src tests` · Tipos: `mypy`
- Verificación completa: `python herramientas/verificar.py` (debe terminar en APROBADO).

## Contrato que no cambia
- `comisiones.proceso.procesar(entrada, bd, url)` guarda las comisiones y devuelve el total por vendedor.
- `comisiones.proceso.main(argv)` = `python -m comisiones ENTRADA BD [URL]`.
  Códigos de salida: 0 entregado, 1 falló la lectura, la base de datos o el webhook, 2 argumentos inválidos.
- Tabla SQLite `comisiones(folio, vendedor, monto, comision)`.
- JSON del webhook: `{"total": número, "por_vendedor": {vendedor: número}}`.
- Reglas de negocio en `src/comisiones/reglas.py`: `calcular_comision(monto: Decimal, region: str) -> Decimal`.

## Convenciones
- Tipos en todas las firmas (`mypy --strict`); tipos nativos (`list`, `dict`).
- Dinero con `Decimal`; redondeo `ROUND_HALF_UP` a centavos.
- `logging`, nunca `print`, para diagnóstico. Sin datos personales (correos) ni secretos en los logs:
  una fila inválida se registra con su número de línea (`línea N`) y el motivo, nunca con sus valores.
- SQL con parámetros (`?`) y una sola transacción por carga.
- `requests` siempre con `timeout` y `raise_for_status()`; TLS verificado.
- El token del webhook llega por la variable de entorno `COMISIONES_TOKEN`.
- Docstrings estilo Google en todo lo público; ejemplos ejecutables (doctest) en las reglas de negocio.

## Límites
- NO modifiques `tests/conftest.py`, `tests/test_caracterizacion.py`, `tests/test_criterios_aceptacion.py`,
  `tests/test_politicas.py`, `tests/datos/`, `herramientas/`, `pyproject.toml` ni los `requirements*.txt`:
  son los controles del equipo. Si una prueba del equipo parece incorrecta, dilo; no la cambies.
- No agregues dependencias (ver `dependencias_aprobadas.txt`).
- No cambies el contrato de arriba.
- Al terminar: ejecuta `python herramientas/verificar.py --sin-mutantes` y muestra la salida completa.
```

**Archivo:** `CLAUDE.md`

```markdown
@AGENTS.md

## Solo para Claude Code
- Para cambios en más de un archivo, presenta primero el plan y espera mi aprobación.
- No instales paquetes ni ejecutes comandos que usen la red.
- Si una verificación falla, muestra el error real antes de proponer la corrección.
```

**Archivo:** `dependencias_aprobadas.txt`

```text
# Dependencias de terceros aprobadas para src/ (nombre del paquete en PyPI).
# Agregar una requiere revisión: ¿existe?, ¿es la correcta?, ¿mantenida?, ¿licencia?, ¿vulnerabilidades?
requests
```

**Archivo:** `EVIDENCIA.md`

````markdown
# Evidencia de la Práctica 7

| Dato | Valor |
| :--- | :--- |
| Participante | |
| Herramienta | Claude Code / GitHub Copilot (en línea, chat o agente) |
| Modelo (si la herramienta lo muestra) | |

## 1. Revisión de código asistida

Prompt usado:

```text

```

| # | Hallazgo de la IA (línea y motivo) | Severidad | ¿Lo confirma una herramienta o una prueba? ¿Cuál? | Decisión |
| :---: | :--- | :---: | :--- | :--- |
| 1 | | | | |

Defectos que la IA **no** reportó y que sí detectaron las herramientas o las pruebas:

Afirmaciones de la IA que resultaron falsas o exageradas:

## 2. Refactorización y corrección

Prompt usado:

```text

```

- Iteraciones (veces que devolviste un error real al asistente):
- Cambios que rechazaste o corregiste a mano, y por qué:

## 3. Pruebas generadas

Prompt usado:

```text

```

- Mutantes detectados con la primera propuesta: ___ / ___ · con la versión final: ___ / ___
- Mutantes que sobrevivieron y qué hiciste con cada uno:

## 4. Documentación

Prompt usado:

```text

```

- Algo que la documentación generada afirmaba y no era cierto (o que no se podía verificar):

## 5. Verificación final

Salida completa de `python herramientas/verificar.py`:

```text

```

Salida de `git diff --stat inicio`:

```text

```

## 6. Valor operativo

- Tiempo que estimas sin IA / con IA:
- Qué hizo bien la IA:
- Qué control detectó algo que la IA no vio:
````

**0.3** El código heredado, **tal como llegó**. No lo corrijas a mano: es el material de trabajo de la IA.

**Archivo:** `src/comisiones/__init__.py`

```python
"""comisiones: cálculo de comisiones de vendedores (código heredado del equipo de ventas)."""
```

**Archivo:** `src/comisiones/__main__.py`

```python
import sys

from comisiones.proceso import main

sys.exit(main())
```

**Archivo:** `src/comisiones/proceso.py`

```python
import csv
import logging
import sqlite3
import sys

import requests

TOKEN = "xoxb-comisiones-2026-9f8e7d"
URL_NOTIFICACION = "https://hooks.empresa.internal/comisiones"

logging.basicConfig(level=logging.INFO)
log = logging.getLogger("comisiones")


def procesar(entrada, bd, url=URL_NOTIFICACION, resumen={}):
    conn = sqlite3.connect(bd)
    conn.execute(
        "CREATE TABLE IF NOT EXISTS comisiones "
        "(folio TEXT PRIMARY KEY, vendedor TEXT, monto REAL, comision REAL)"
    )
    total = 0
    f = open(entrada)
    for row in csv.DictReader(f):
        try:
            monto = float(row["monto"])
            if row["region"] == "norte":
                if monto > 10000:
                    tasa = 0.02
                else:
                    tasa = 0.015
            elif row["region"] == "sur":
                if monto > 10000:
                    tasa = 0.025
                else:
                    tasa = 0.02
            else:
                tasa = 0.01
            comision = round(monto * tasa, 2)
            if comision < 5:
                comision = 5.0
            log.info(f"venta {row['folio']} de {row['email_vendedor']}: comision {comision}")
            conn.execute(
                f"INSERT OR REPLACE INTO comisiones VALUES "
                f"('{row['folio']}', '{row['vendedor']}', {monto}, {comision})"
            )
            conn.commit()
            total += comision
            resumen[row["vendedor"]] = resumen.get(row["vendedor"], 0) + comision
        except Exception:
            pass
    f.close()
    requests.post(
        url,
        json={"total": round(total, 2), "por_vendedor": resumen},
        headers={"Authorization": "Bearer " + TOKEN},
        verify=False,
    )
    return resumen


def main(argv=None):
    args = argv if argv is not None else sys.argv[1:]
    url = args[2] if len(args) > 2 else URL_NOTIFICACION
    resumen = procesar(args[0], args[1], url)
    print(f"comisiones calculadas para {len(resumen)} vendedores")
    return 0
```

**0.4** Los controles del equipo. Los escribió el equipo, no la IA, y **ni la IA ni tú los modifican**: son la definición de "correcto".

`conftest.py` tiene utilidades compartidas: un webhook simulado con `responses` (sin red) y un webhook HTTP real en `127.0.0.1` para ejecutar el programa en otro proceso.

**Archivo:** `tests/conftest.py`

```python
"""Utilidades de las pruebas del equipo. NO las modifica la IA (ver AGENTS.md)."""

import csv
import json
import sqlite3
import threading
from collections.abc import Iterator
from decimal import Decimal
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from pathlib import Path
from typing import Any

import pytest
import responses

DATOS = Path(__file__).parent / "datos"
URL = "https://hooks.ejemplo.test/comisiones"
TOKEN = "tok-de-prueba-0123456789"
COLUMNAS = ["folio", "fecha", "vendedor", "email_vendedor", "region", "monto"]


def centavos(valor: Any) -> str:
    """Normaliza un número (float, Decimal o texto) a dos decimales para comparar."""
    return str(Decimal(str(valor)).quantize(Decimal("0.01")))


def filas_bd(ruta: Path) -> list[list[str]]:
    """Lee la tabla comisiones normalizada: [folio, vendedor, monto, comision]."""
    conexion = sqlite3.connect(ruta)
    try:
        filas = conexion.execute(
            "SELECT folio, vendedor, monto, comision FROM comisiones ORDER BY folio"
        ).fetchall()
    finally:
        conexion.close()
    return [[f, v, centavos(m), centavos(c)] for f, v, m, c in filas]


def normalizar_notificacion(cuerpo: dict[str, Any]) -> dict[str, Any]:
    return {
        "total": centavos(cuerpo["total"]),
        "por_vendedor": {k: centavos(v) for k, v in sorted(cuerpo["por_vendedor"].items())},
    }


def escribir_csv(ruta: Path, filas: list[dict[str, str]]) -> Path:
    with ruta.open("w", encoding="utf-8", newline="") as archivo:
        escritor = csv.DictWriter(archivo, fieldnames=COLUMNAS)
        escritor.writeheader()
        escritor.writerows(filas)
    return ruta


def venta(folio: str, region: str, monto: str, vendedor: str = "Ana Torres") -> dict[str, str]:
    return {
        "folio": folio,
        "fecha": "2026-10-01",
        "vendedor": vendedor,
        "email_vendedor": "ana.torres@ejemplo.com",
        "region": region,
        "monto": monto,
    }


@pytest.fixture(autouse=True)
def token(monkeypatch: pytest.MonkeyPatch) -> str:
    monkeypatch.setenv("COMISIONES_TOKEN", TOKEN)
    return TOKEN


@pytest.fixture
def webhook() -> Iterator[responses.RequestsMock]:
    """Webhook simulado: responde 200 y guarda lo que recibe (sin red)."""
    with responses.RequestsMock(assert_all_requests_are_fired=False) as simulado:
        simulado.post(URL, json={"ok": True})
        yield simulado


class WebhookLocal:
    """Webhook HTTP real en 127.0.0.1 para ejecutar el programa en otro proceso."""

    def __init__(self) -> None:
        self.recibidos: list[dict[str, Any]] = []
        recibidos = self.recibidos

        class Manejador(BaseHTTPRequestHandler):
            def do_POST(self) -> None:  # noqa: N802 (nombre que exige http.server)
                largo = int(self.headers.get("Content-Length", "0"))
                recibidos.append(json.loads(self.rfile.read(largo)))
                self.send_response(200)
                self.end_headers()
                self.wfile.write(b'{"ok": true}')

            def log_message(self, *args: object) -> None:
                pass

        self.servidor = ThreadingHTTPServer(("127.0.0.1", 0), Manejador)
        self.url = f"http://127.0.0.1:{self.servidor.server_address[1]}/comisiones"

    def __enter__(self) -> "WebhookLocal":
        threading.Thread(target=self.servidor.serve_forever, daemon=True).start()
        return self

    def __exit__(self, *args: object) -> None:
        self.servidor.shutdown()
        self.servidor.server_close()
```

La **prueba de caracterización** (7.5) congela lo que el programa hace hoy con datos típicos: lo que queda en la base de datos y lo que recibe el webhook. Los valores esperados se obtuvieron ejecutando la versión heredada.

**Archivo:** `tests/datos/ventas_tipicas.csv`

```text
folio,fecha,vendedor,email_vendedor,region,monto
F-1001,2026-10-01,Ana Torres,ana.torres@ejemplo.com,norte,1200.00
F-1002,2026-10-01,Luis Perez,luis.perez@ejemplo.com,sur,850.00
F-1003,2026-10-02,Ana Torres,ana.torres@ejemplo.com,norte,15000.00
F-1004,2026-10-02,Sofia Ruiz,sofia.ruiz@ejemplo.com,centro,3400.00
F-1005,2026-10-03,Luis Perez,luis.perez@ejemplo.com,sur,12000.00
F-1006,2026-10-03,Marco Diaz,marco.diaz@ejemplo.com,occidente,200.00
F-1007,2026-10-04,Sofia Ruiz,sofia.ruiz@ejemplo.com,norte,10000.00
F-1008,2026-10-04,Marco Diaz,marco.diaz@ejemplo.com,sur,2450.00
F-1009,2026-10-05,Ana Torres,ana.torres@ejemplo.com,centro,N/D
F-1010,2026-10-05,Luis Perez,luis.perez@ejemplo.com,norte,640.00
F-1011,2026-10-06,Sofia Ruiz,sofia.ruiz@ejemplo.com,sur,4980.00
F-1012,2026-10-06,Marco Diaz,marco.diaz@ejemplo.com,occidente,7600.00
```

**Archivo:** `tests/datos/caracterizacion_esperada.json`

```json
{
  "origen": "salida de la versión heredada (src/comisiones/proceso.py antes de cualquier cambio)",
  "bd": [
    ["F-1001", "Ana Torres", "1200.00", "18.00"],
    ["F-1002", "Luis Perez", "850.00", "17.00"],
    ["F-1003", "Ana Torres", "15000.00", "300.00"],
    ["F-1004", "Sofia Ruiz", "3400.00", "34.00"],
    ["F-1005", "Luis Perez", "12000.00", "300.00"],
    ["F-1006", "Marco Diaz", "200.00", "5.00"],
    ["F-1007", "Sofia Ruiz", "10000.00", "150.00"],
    ["F-1008", "Marco Diaz", "2450.00", "49.00"],
    ["F-1010", "Luis Perez", "640.00", "9.60"],
    ["F-1011", "Sofia Ruiz", "4980.00", "99.60"],
    ["F-1012", "Marco Diaz", "7600.00", "76.00"]
  ],
  "notificacion": {"total": "1058.20", "por_vendedor": {"Ana Torres": "318.00", "Luis Perez": "326.60", "Marco Diaz": "130.00", "Sofia Ruiz": "283.60"}}
}
```

**Archivo:** `tests/test_caracterizacion.py`

```python
"""Prueba de caracterización: la salida de hoy, congelada ANTES de cambiar el código.

Ejecuta el programa en otro proceso, igual que un orquestador, contra un webhook
local. Compara lo que queda en la base de datos y lo que recibe el webhook con lo
que producía la versión heredada (tests/datos/caracterizacion_esperada.json).
"""

import json
import os
import subprocess
import sys
from pathlib import Path

from tests.conftest import DATOS, TOKEN, WebhookLocal, filas_bd, normalizar_notificacion

ESPERADO = json.loads((DATOS / "caracterizacion_esperada.json").read_text(encoding="utf-8"))


def test_misma_salida_que_la_version_heredada(tmp_path: Path) -> None:
    bd = tmp_path / "comisiones.db"
    entorno = os.environ | {"COMISIONES_TOKEN": TOKEN, "NO_PROXY": "127.0.0.1,localhost"}
    with WebhookLocal() as webhook:
        proceso = subprocess.run(  # noqa: S603 (comando fijo del propio proyecto)
            [sys.executable, "-m", "comisiones", str(DATOS / "ventas_tipicas.csv"), str(bd),
             webhook.url],
            capture_output=True, text=True, env=entorno, timeout=60, check=False,
        )  # fmt: skip
    assert proceso.returncode == 0, proceso.stderr
    assert filas_bd(bd) == ESPERADO["bd"]
    assert len(webhook.recibidos) == 1
    assert normalizar_notificacion(webhook.recibidos[0]) == ESPERADO["notificacion"]
```

Los **criterios de aceptación** (7.14) son ejemplos acordados con el área de ventas, no deducidos del código:

**Archivo:** `tests/test_criterios_aceptacion.py`

```python
"""Criterios de aceptación acordados con el área de ventas (no deducidos del código).

Reglas: norte 1.5 % (2 % si el monto supera 10,000), sur 2 % (2.5 % si supera
10,000), cualquier otra región 1 %. Comisión mínima 5.00 y máxima 500.00,
redondeo comercial a centavos (mitad hacia arriba). Un monto que no es un número
positivo se rechaza: no se guarda ni se suma.
"""

from pathlib import Path

import pytest
import responses

from comisiones.proceso import procesar
from tests.conftest import URL, centavos, escribir_csv, filas_bd, venta

CRITERIOS = [
    # (región, monto, comisión esperada; None = fila rechazada)
    pytest.param("norte", "1000.00", "15.00", id="tipico-norte"),
    pytest.param("norte", "100.00", "5.00", id="minimo"),
    pytest.param("norte", "10000.00", "150.00", id="frontera-10000-tasa-baja"),
    pytest.param("norte", "10000.01", "200.00", id="frontera-10000.01-tasa-alta"),
    pytest.param("sur", "10000.00", "200.00", id="frontera-sur"),
    pytest.param("sur", "30000.00", "500.00", id="maximo"),
    pytest.param("norte", "1001.00", "15.02", id="redondeo-mitad-hacia-arriba"),
    pytest.param("centro", "2000.00", "20.00", id="otra-region"),
    pytest.param("norte", "0", None, id="monto-cero-rechazado"),
    pytest.param("sur", "-50.00", None, id="monto-negativo-rechazado"),
    pytest.param("norte", "N/D", None, id="monto-no-numerico-rechazado"),
]


@pytest.mark.parametrize(("region", "monto", "esperado"), CRITERIOS)
def test_criterio_de_comision(
    tmp_path: Path, webhook: responses.RequestsMock, region: str, monto: str, esperado: str | None
) -> None:
    entrada = escribir_csv(tmp_path / "ventas.csv", [venta("F-1", region, monto)])
    procesar(entrada, tmp_path / "c.db", URL)
    filas = filas_bd(tmp_path / "c.db")
    if esperado is None:
        assert filas == []
    else:
        assert [fila[3] for fila in filas] == [esperado]


def test_nombres_con_apostrofo_se_guardan(tmp_path: Path, webhook: responses.RequestsMock) -> None:
    entrada = escribir_csv(
        tmp_path / "ventas.csv", [venta("F-1", "norte", "2000.00", vendedor="Ana O'Brien")]
    )
    procesar(entrada, tmp_path / "c.db", URL)
    assert filas_bd(tmp_path / "c.db") == [["F-1", "Ana O'Brien", "2000.00", "30.00"]]


def test_dos_ejecuciones_dan_el_mismo_resumen(
    tmp_path: Path, webhook: responses.RequestsMock
) -> None:
    entrada = escribir_csv(tmp_path / "ventas.csv", [venta("F-1", "norte", "1000.00")])
    primera = procesar(entrada, tmp_path / "a.db", URL)
    segunda = procesar(entrada, tmp_path / "b.db", URL)
    assert {k: centavos(v) for k, v in primera.items()} == {"Ana Torres": "15.00"}
    assert {k: centavos(v) for k, v in segunda.items()} == {"Ana Torres": "15.00"}
```

Las **políticas del equipo** de `AGENTS.md`, convertidas en pruebas: lo que debe cumplirse siempre se verifica, no se pide de favor (7.12):

**Archivo:** `tests/test_politicas.py`

```python
"""Políticas del equipo (AGENTS.md) convertidas en pruebas: lo que debe cumplirse siempre."""

import json
import logging
import re
from pathlib import Path

import pytest
import responses

from comisiones.proceso import main, procesar
from tests.conftest import TOKEN, URL, escribir_csv, venta


def _correr(tmp_path: Path, filas: list[dict[str, str]]) -> None:
    procesar(escribir_csv(tmp_path / "ventas.csv", filas), tmp_path / "c.db", URL)


def test_sin_datos_personales_en_los_logs(
    tmp_path: Path, webhook: responses.RequestsMock, caplog: pytest.LogCaptureFixture,
    capsys: pytest.CaptureFixture[str],
) -> None:  # fmt: skip
    caplog.set_level(logging.DEBUG)
    _correr(tmp_path, [venta("F-1", "norte", "1000.00")])
    salida = caplog.text + capsys.readouterr().err
    assert "@" not in salida


def test_token_desde_el_entorno_y_fuera_de_los_logs(
    tmp_path: Path, webhook: responses.RequestsMock, caplog: pytest.LogCaptureFixture
) -> None:
    caplog.set_level(logging.DEBUG)
    _correr(tmp_path, [venta("F-1", "norte", "1000.00")])
    peticion = webhook.calls[0].request
    assert peticion.headers["Authorization"] == f"Bearer {TOKEN}"
    assert TOKEN not in caplog.text


def test_peticion_con_timeout_y_tls_verificado(
    tmp_path: Path, webhook: responses.RequestsMock
) -> None:
    _correr(tmp_path, [venta("F-1", "norte", "1000.00")])
    opciones = webhook.calls[0].request.req_kwargs
    assert opciones.get("timeout") is not None
    assert opciones.get("verify") is not False


def test_notificacion_es_json_con_total_y_por_vendedor(
    tmp_path: Path, webhook: responses.RequestsMock
) -> None:
    _correr(tmp_path, [venta("F-1", "norte", "1000.00"), venta("F-2", "sur", "1000.00")])
    cuerpo = json.loads(webhook.calls[0].request.body)
    assert set(cuerpo) == {"total", "por_vendedor"}
    numeros = [cuerpo["total"], *cuerpo["por_vendedor"].values()]
    assert all(isinstance(n, int | float) for n in numeros)  # contrato: números, no texto


def test_fila_invalida_se_reporta_con_su_linea_sin_el_valor(
    tmp_path: Path, webhook: responses.RequestsMock, caplog: pytest.LogCaptureFixture
) -> None:
    caplog.set_level(logging.DEBUG)
    _correr(tmp_path, [venta("F-1", "norte", "1000.00"), venta("F-2", "norte", "N/D")])
    assert re.search(r"(?i)l[ií]nea 3\b", caplog.text)  # línea 1 = encabezado
    assert "N/D" not in caplog.text


def test_webhook_caido_termina_con_codigo_1(tmp_path: Path) -> None:
    entrada = escribir_csv(tmp_path / "ventas.csv", [venta("F-1", "norte", "1000.00")])
    with responses.RequestsMock() as simulado:
        simulado.post(URL, status=503)
        assert main([str(entrada), str(tmp_path / "c.db"), URL]) == 1


def test_sin_argumentos_termina_con_codigo_2() -> None:
    try:
        codigo = main([])
    except SystemExit as salida:  # argparse termina así ante un error de uso
        codigo = salida.code
    assert codigo == 2
```

**0.5** Las herramientas de verificación. Ninguna ejecuta el código que revisa, salvo las pruebas.

La **puerta de gobierno** (7.19) aplica reglas propias sobre el AST: funciones prohibidas, `shell=True`, `verify=False`, excepciones silenciadas, valores por defecto mutables y cobertura de docstrings.

**Archivo:** `herramientas/puerta_gobierno.py`

```python
"""Puerta de gobierno: reglas propias del equipo sobre el AST (no ejecuta el código).

Uso: python herramientas/puerta_gobierno.py [carpeta]   (por omisión: src)
"""

import ast
import json
import sys
from pathlib import Path
from typing import Any

PROHIBIDAS = {"eval", "exec"}
MUTABLES = (ast.List, ast.Dict, ast.Set)
COBERTURA_MINIMA = 0.8


class Auditor(ast.NodeVisitor):
    def __init__(self) -> None:
        self.hallazgos: list[dict[str, Any]] = []
        self.publicas = 0
        self.documentadas = 0

    def _anotar(self, nodo: ast.AST, regla: str, mensaje: str) -> None:
        self.hallazgos.append({"linea": getattr(nodo, "lineno", 0), "regla": regla,
                               "mensaje": mensaje})  # fmt: skip

    def _definicion(self, nodo: ast.FunctionDef | ast.AsyncFunctionDef | ast.ClassDef) -> None:
        if not nodo.name.startswith("_"):
            self.publicas += 1
            if ast.get_docstring(nodo):
                self.documentadas += 1
            else:
                self._anotar(nodo, "DOC-01", f"'{nodo.name}' sin docstring")
        if not isinstance(nodo, ast.ClassDef):
            for valor in nodo.args.defaults + [d for d in nodo.args.kw_defaults if d]:
                if isinstance(valor, MUTABLES):
                    self._anotar(valor, "ERR-02", f"valor por defecto mutable en '{nodo.name}'")
        self.generic_visit(nodo)

    visit_FunctionDef = visit_AsyncFunctionDef = visit_ClassDef = _definicion

    def visit_Call(self, nodo: ast.Call) -> None:
        if isinstance(nodo.func, ast.Name) and nodo.func.id in PROHIBIDAS:
            self._anotar(nodo, "SEG-01", f"uso de {nodo.func.id}()")
        for clave in nodo.keywords:
            if isinstance(clave.value, ast.Constant):
                if clave.arg == "shell" and clave.value.value is True:
                    self._anotar(nodo, "SEG-02", "subprocess con shell=True")
                if clave.arg == "verify" and clave.value.value is False:
                    self._anotar(nodo, "SEG-03", "TLS desactivado (verify=False)")
        self.generic_visit(nodo)

    def visit_ExceptHandler(self, nodo: ast.ExceptHandler) -> None:
        silencioso = len(nodo.body) == 1 and isinstance(nodo.body[0], ast.Pass)
        generico = nodo.type is None or (
            isinstance(nodo.type, ast.Name) and nodo.type.id in {"Exception", "BaseException"}
        )
        if silencioso and generico:
            self._anotar(nodo, "ERR-01", "excepción genérica silenciada")
        self.generic_visit(nodo)


def auditar(carpeta: Path) -> dict[str, Any]:
    archivos: list[dict[str, Any]] = []
    publicas = documentadas = 0
    for ruta in sorted(carpeta.rglob("*.py")):
        auditor = Auditor()
        auditor.visit(ast.parse(ruta.read_text(encoding="utf-8"), filename=str(ruta)))
        publicas += auditor.publicas
        documentadas += auditor.documentadas
        archivos.append({"archivo": ruta.as_posix(), "hallazgos": auditor.hallazgos})
    cobertura = documentadas / publicas if publicas else 1.0
    criticos = [
        h for a in archivos for h in a["hallazgos"] if h["regla"].startswith(("SEG", "ERR"))
    ]
    aprobado = not criticos and cobertura >= COBERTURA_MINIMA
    return {"cobertura_docstrings": round(cobertura, 2), "archivos": archivos,
            "estado": "APROBADO" if aprobado else "RECHAZADO"}  # fmt: skip


def main() -> int:
    sys.stdout.reconfigure(errors="replace")
    carpeta = Path(sys.argv[1] if len(sys.argv) > 1 else "src")
    informe = auditar(carpeta)
    for archivo in informe["archivos"]:
        for h in archivo["hallazgos"]:
            print(f"{archivo['archivo']}:{h['linea']} {h['regla']} {h['mensaje']}")
    print(f"docstrings: {informe['cobertura_docstrings']:.0%} (mínimo {COBERTURA_MINIMA:.0%})")
    print(f"puerta de gobierno: {informe['estado']}")
    Path("evidencia").mkdir(exist_ok=True)
    Path("evidencia/gobierno.json").write_text(
        json.dumps(informe, ensure_ascii=False, indent=2), encoding="utf-8"
    )
    return 0 if informe["estado"] == "APROBADO" else 1


if __name__ == "__main__":
    sys.exit(main())
```

El **revisor de dependencias** (7.17) compara los imports con la lista aprobada y, con `--pypi`, comprueba si un paquete sugerido existe:

**Archivo:** `herramientas/revisar_dependencias.py`

```python
"""Compara los imports del código con la lista de dependencias aprobadas.

Lee los imports con ast (no ejecuta nada). Con --pypi consulta si cada paquete
no aprobado existe en PyPI: un nombre inventado por la IA es un riesgo de
slopsquatting.

Uso: python herramientas/revisar_dependencias.py [--pypi]
"""

import ast
import json
import sys
import urllib.error
import urllib.request
from pathlib import Path

# Nombre del import -> nombre del paquete que se instala
IMPORT_A_PAQUETE = {
    "yaml": "pyyaml",
    "dateutil": "python-dateutil",
    "sklearn": "scikit-learn",
    "PIL": "pillow",
    "dotenv": "python-dotenv",
    "bs4": "beautifulsoup4",
}


def imports_de(carpeta: Path) -> set[str]:
    nombres: set[str] = set()
    for ruta in carpeta.rglob("*.py"):
        for nodo in ast.walk(ast.parse(ruta.read_text(encoding="utf-8"))):
            if isinstance(nodo, ast.Import):
                nombres.update(alias.name.split(".")[0] for alias in nodo.names)
            elif isinstance(nodo, ast.ImportFrom) and nodo.module and nodo.level == 0:
                nombres.add(nodo.module.split(".")[0])
    locales = {p.name for p in carpeta.iterdir() if p.is_dir()}
    return {n for n in nombres if n not in sys.stdlib_module_names and n not in locales}


def en_pypi(paquete: str) -> str:
    url = f"https://pypi.org/pypi/{paquete}/json"
    try:
        with urllib.request.urlopen(url, timeout=10) as respuesta:  # noqa: S310 (URL fija https)
            datos = json.load(respuesta)
    except urllib.error.HTTPError as error:
        return "NO EXISTE en PyPI" if error.code == 404 else f"HTTP {error.code}"
    except urllib.error.URLError:
        return "sin red: verificar a mano"
    fechas = [a["upload_time"][:10] for v in datos["releases"].values() for a in v]
    return f"existe; primera versión {min(fechas) if fechas else '?'}"


def main() -> int:
    sys.stdout.reconfigure(errors="replace")
    lista = Path("dependencias_aprobadas.txt").read_text(encoding="utf-8").splitlines()
    aprobadas = {
        linea.split("#")[0].strip().lower() for linea in lista if linea.split("#")[0].strip()
    }
    no_aprobadas = 0
    for nombre in sorted(imports_de(Path("src"))):
        paquete = IMPORT_A_PAQUETE.get(nombre, nombre).lower().replace("_", "-")
        if paquete in aprobadas:
            print(f"{nombre:<22} -> {paquete:<22} aprobado")
            continue
        no_aprobadas += 1
        detalle = f"; PyPI: {en_pypi(paquete)}" if "--pypi" in sys.argv else ""
        print(f"{nombre:<22} -> {paquete:<22} NO aprobado{detalle}")
    print(f"dependencias no aprobadas: {no_aprobadas}")
    return 1 if no_aprobadas else 0


if __name__ == "__main__":
    sys.exit(main())
```

**Probar las pruebas** (7.2): introduce errores pequeños en un módulo, uno a la vez, y comprueba que las pruebas fallen. Lo modifica temporalmente y siempre lo restaura.

**Archivo:** `herramientas/probar_las_pruebas.py`

```python
"""Prueba las pruebas: introduce errores pequeños (mutantes) y comprueba que fallen.

Cada mutante cambia UNA cosa del módulo (un > por >=, una constante, el modo de
redondeo, min por max...). Si las pruebas siguen en verde con el error dentro,
el mutante "sobrevive": esas pruebas no protegen ese detalle.

Uso: python herramientas/probar_las_pruebas.py src/comisiones/reglas.py tests/test_reglas.py
"""

import ast
import copy
import json
import os
import re
import subprocess
import sys
import tempfile
from pathlib import Path

MINIMO = 0.8  # porcentaje de mutantes que las pruebas deben detectar
COMPARACIONES: dict[type[ast.cmpop], type[ast.cmpop]] = {
    ast.Gt: ast.GtE, ast.GtE: ast.Gt, ast.Lt: ast.LtE, ast.LtE: ast.Lt,
    ast.Eq: ast.NotEq, ast.NotEq: ast.Eq,
}  # fmt: skip
SIMBOLOS = {ast.Gt: ">", ast.GtE: ">=", ast.Lt: "<", ast.LtE: "<=", ast.Eq: "==", ast.NotEq: "!="}
ARITMETICA: dict[type[ast.operator], type[ast.operator]] = {
    ast.Add: ast.Sub,
    ast.Sub: ast.Add,
    ast.Mult: ast.Div,
    ast.Div: ast.Mult,
}
NUMERO = re.compile(r"-?\d+(\.\d+)?")  # solo textos numéricos, p. ej. Decimal("500.00")
REDONDEO = {"ROUND_HALF_UP": "ROUND_HALF_EVEN", "ROUND_HALF_EVEN": "ROUND_HALF_UP"}


def _otro_digito(texto: str) -> str:
    """'500.00' -> '500.01': cambia el último dígito de un número escrito como texto."""
    ultimo = texto[-1]
    return texto[:-1] + str((int(ultimo) + 1) % 10)


def _docstrings(arbol: ast.AST) -> set[int]:
    ids = set()
    for nodo in ast.walk(arbol):
        cuerpo = getattr(nodo, "body", None)
        if isinstance(cuerpo, list) and cuerpo and isinstance(cuerpo[0], ast.Expr):
            ids.add(id(cuerpo[0].value))
    return ids


def mutaciones(arbol: ast.AST) -> list[tuple[int, str]]:
    """Enumera (posición en ast.walk, descripción) de cada mutante posible."""
    resultado = []
    excluidos = _docstrings(arbol)
    for posicion, nodo in enumerate(ast.walk(arbol)):
        if id(nodo) in excluidos:
            continue
        linea = getattr(nodo, "lineno", 0)
        if isinstance(nodo, ast.Compare):
            for i, op in enumerate(nodo.ops):
                if type(op) in COMPARACIONES:
                    nuevo = SIMBOLOS[COMPARACIONES[type(op)]]
                    resultado.append(
                        (posicion, f"L{linea}: '{SIMBOLOS[type(op)]}' -> '{nuevo}' #{i}")
                    )
        elif isinstance(nodo, ast.BinOp) and type(nodo.op) in ARITMETICA:
            resultado.append((posicion, f"L{linea}: {type(nodo.op).__name__} -> "
                                        f"{ARITMETICA[type(nodo.op)].__name__}"))  # fmt: skip
        elif isinstance(nodo, ast.Constant) and not isinstance(nodo.value, bool):
            if isinstance(nodo.value, int | float):
                resultado.append((posicion, f"L{linea}: {nodo.value!r} -> {nodo.value + 1!r}"))
            elif isinstance(nodo.value, str) and NUMERO.fullmatch(nodo.value):
                nuevo = _otro_digito(nodo.value)
                resultado.append((posicion, f"L{linea}: '{nodo.value}' -> '{nuevo}'"))
        elif isinstance(nodo, ast.Name | ast.Attribute):
            nombre = nodo.id if isinstance(nodo, ast.Name) else nodo.attr
            if nombre in REDONDEO:
                resultado.append((posicion, f"L{linea}: {nombre} -> {REDONDEO[nombre]}"))
            elif isinstance(nodo, ast.Name) and nombre in {"min", "max"}:
                otro = "max" if nombre == "min" else "min"
                resultado.append((posicion, f"L{linea}: {nombre}() -> {otro}()"))
    return sorted(resultado, key=lambda m: (int(m[1][1:].split(":")[0]), m[0]))  # por línea


def aplicar(arbol: ast.AST, posicion: int, descripcion: str) -> str:
    mutado = copy.deepcopy(arbol)
    nodo = list(ast.walk(mutado))[posicion]
    if isinstance(nodo, ast.Compare):
        i = int(descripcion.rsplit("#", 1)[1])
        nodo.ops[i] = COMPARACIONES[type(nodo.ops[i])]()
    elif isinstance(nodo, ast.BinOp):
        nodo.op = ARITMETICA[type(nodo.op)]()
    elif isinstance(nodo, ast.Constant):
        valor = nodo.value
        nodo.value = valor + 1 if isinstance(valor, int | float) else _otro_digito(valor)
    elif isinstance(nodo, ast.Name):
        nodo.id = REDONDEO.get(nodo.id) or ("max" if nodo.id == "min" else "min")
    elif isinstance(nodo, ast.Attribute):
        nodo.attr = REDONDEO[nodo.attr]
    if isinstance(nodo, ast.Name) and nodo.id in REDONDEO and isinstance(mutado, ast.Module):
        # el modo de redondeo nuevo debe estar importado: si no, el mutante muere por
        # NameError y no porque las pruebas detecten el cambio de comportamiento
        importacion = ast.parse("from decimal import ROUND_HALF_EVEN, ROUND_HALF_UP").body[0]
        mutado.body.insert(1 if ast.get_docstring(mutado) else 0, importacion)
    return ast.unparse(mutado)


def correr_pruebas(pruebas: str, cache: str) -> int:
    entorno = os.environ | {"PYTHONPYCACHEPREFIX": cache, "PYTHONDONTWRITEBYTECODE": "1"}
    try:
        proceso = subprocess.run(  # noqa: S603 (comando fijo: pytest del proyecto)
            [sys.executable, "-m", "pytest", "-q", "-x", "-p", "no:cacheprovider", pruebas],
            capture_output=True, text=True, env=entorno, timeout=120, check=False,
        )  # fmt: skip
    except subprocess.TimeoutExpired:
        return 1  # un mutante que cuelga las pruebas cuenta como detectado
    return proceso.returncode


def main() -> int:
    sys.stdout.reconfigure(errors="replace")
    if len(sys.argv) != 3:
        print(__doc__)
        return 2
    modulo, pruebas = Path(sys.argv[1]), sys.argv[2]
    original = modulo.read_text(encoding="utf-8")
    arbol = ast.parse(original)
    with tempfile.TemporaryDirectory() as cache:
        if correr_pruebas(pruebas, f"{cache}/original") != 0:
            print("Las pruebas deben pasar con el código original antes de mutarlo.")
            return 2
        resultados = []
        try:
            for n, (posicion, descripcion) in enumerate(mutaciones(arbol)):
                modulo.write_text(aplicar(arbol, posicion, descripcion), encoding="utf-8")
                detectado = correr_pruebas(pruebas, f"{cache}/m{n}") != 0
                resultados.append({"mutante": descripcion.split(" #")[0], "detectado": detectado})
                marca = "✓ detectado " if detectado else "✗ SOBREVIVE"
                print(f"  {marca}  {resultados[-1]['mutante']}")
        finally:
            modulo.write_text(original, encoding="utf-8")  # siempre se restaura
    detectados = sum(r["detectado"] for r in resultados)
    puntaje = detectados / len(resultados) if resultados else 0.0
    print(
        f"mutantes detectados: {detectados}/{len(resultados)} ({puntaje:.0%}); mínimo {MINIMO:.0%}"
    )
    Path("evidencia").mkdir(exist_ok=True)
    Path("evidencia/mutantes.json").write_text(
        json.dumps({"modulo": modulo.as_posix(), "pruebas": pruebas, "puntaje": round(puntaje, 2),
                    "resultados": resultados}, ensure_ascii=False, indent=2),
        encoding="utf-8",
    )  # fmt: skip
    return 0 if resultados and puntaje >= MINIMO else 1


if __name__ == "__main__":
    sys.exit(main())
```

El **verificador de confianza cero** (7.8) ejecuta todas las capas en orden y deja la evidencia en `evidencia/verificacion.json`:

**Archivo:** `herramientas/verificar.py`

```python
"""Verificación de confianza cero: las capas que todo código (de la IA o no) debe cruzar.

Uso: python herramientas/verificar.py [--sin-mutantes]
Escribe evidencia/verificacion.json y termina con 0 solo si TODAS las capas pasan.
"""

import json
import os
import shutil
import subprocess
import sys
from collections.abc import Callable
from datetime import UTC, datetime
from functools import partial
from pathlib import Path

PY = sys.executable
REGLAS, PRUEBAS_REGLAS = "src/comisiones/reglas.py", "tests/test_reglas.py"
# Controles del equipo: ni la IA ni la persona los cambian para "pasar" la verificación
PROTEGIDOS = [
    "tests/conftest.py", "tests/test_caracterizacion.py", "tests/test_criterios_aceptacion.py",
    "tests/test_politicas.py", "tests/datos", "herramientas", "pyproject.toml",
    "requirements.txt", "requirements-dev.txt", "dependencias_aprobadas.txt", "AGENTS.md",
]  # fmt: skip


def correr(*args: str) -> tuple[int, str]:
    entorno = os.environ | {"PYTHONIOENCODING": "utf-8"}  # acentos y ✓ también en Windows
    proceso = subprocess.run(  # noqa: S603 (comandos fijos de las herramientas del proyecto)
        [PY, *args], capture_output=True, text=True, encoding="utf-8", errors="replace",
        env=entorno, check=False,
    )  # fmt: skip
    return proceso.returncode, (proceso.stdout + proceso.stderr).strip()


def ultima(texto: str) -> str:
    lineas = [linea.strip("= ") for linea in texto.splitlines() if linea.strip()]
    return lineas[-1] if lineas else ""


def pytest(*rutas: str) -> tuple[bool, str]:
    codigo, salida = correr("-m", "pytest", "-q", "-p", "no:cacheprovider", *rutas)
    return codigo == 0, ultima(salida)


def doctest() -> tuple[bool, str]:
    codigo, salida = correr("-m", "pytest", "-q", "-p", "no:cacheprovider",
                            "--doctest-modules", "src")  # fmt: skip
    if codigo == 5:
        return False, "sin ejemplos ejecutables en los docstrings"
    return codigo == 0, ultima(salida)


def bandit() -> tuple[bool, str]:
    _, salida = correr("-m", "bandit", "-q", "-r", "src", "-f", "json")
    resultados = json.loads(salida[salida.find("{") :])["results"] if "{" in salida else []
    bloquean = [r for r in resultados if r["issue_severity"] in {"MEDIUM", "HIGH"}]
    revisar = [r for r in resultados if r["issue_severity"] == "LOW"]
    detalle = ", ".join(f"{r['test_id']} L{r['line_number']} {r['issue_severity']}"
                        for r in bloquean + revisar)  # fmt: skip
    return not bloquean, f"{len(bloquean)} bloquean, {len(revisar)} para revisar. {detalle}".strip()


def mutantes() -> tuple[bool, str]:
    if not (Path(REGLAS).exists() and Path(PRUEBAS_REGLAS).exists()):
        return False, f"faltan {REGLAS} o {PRUEBAS_REGLAS}"
    codigo, salida = correr("herramientas/probar_las_pruebas.py", REGLAS, PRUEBAS_REGLAS)
    return codigo == 0, ultima(salida)


def herramienta(*args: str, clave: str = "") -> tuple[bool, str]:
    """Ejecuta una herramienta; resume con la línea que contiene `clave` o la última."""
    codigo, salida = correr(*args)
    lineas = [linea for linea in salida.splitlines() if clave and clave in linea]
    return codigo == 0, " · ".join(lineas) or ultima(salida) or "sin hallazgos"


def xenon() -> tuple[bool, str]:
    codigo, salida = correr("-m", "xenon", "--max-absolute", "B", "--max-modules", "B",
                            "--max-average", "A", "src")  # fmt: skip
    errores = [linea.split("ERROR:xenon:")[-1] for linea in salida.splitlines() if "ERROR" in linea]
    return codigo == 0, "; ".join(errores) or "funciones y módulos <= B, promedio A"


def git(*args: str) -> subprocess.CompletedProcess[str]:
    ejecutable = shutil.which("git")
    if ejecutable is None:
        return subprocess.CompletedProcess(list(args), 1, "", "git no está instalado")
    return subprocess.run(  # noqa: S603 (git del sistema, argumentos fijos)
        [ejecutable, *args], capture_output=True, text=True, check=False
    )


def controles_intactos() -> tuple[bool, str]:
    if not shutil.which("git"):
        return False, "git no está instalado"
    if git("rev-parse", "-q", "--verify", "refs/tags/inicio").returncode != 0:
        return False, "falta la etiqueta 'inicio' (Parte 0)"
    cambiados = git("diff", "--name-only", "inicio", "--", *PROTEGIDOS).stdout.split()
    nuevos = git("ls-files", "--others", "--exclude-standard", "--", *PROTEGIDOS).stdout.split()
    if cambiados or nuevos:
        return False, "modificados: " + ", ".join(sorted(set(cambiados + nuevos)))
    return True, "sin cambios desde la etiqueta 'inicio'"


def pruebas_generadas() -> tuple[bool, str]:
    return pytest(PRUEBAS_REGLAS) if Path(PRUEBAS_REGLAS).exists() else (False, "no existe")


CAPAS: list[tuple[str, Callable[[], tuple[bool, str]]]] = [
    ("controles del equipo intactos", controles_intactos),
    ("caracterización (comportamiento heredado)", partial(pytest, "tests/test_caracterizacion.py")),
    ("criterios de aceptación del negocio", partial(pytest, "tests/test_criterios_aceptacion.py")),
    ("políticas del equipo", partial(pytest, "tests/test_politicas.py")),
    ("pruebas generadas (test_reglas.py)", pruebas_generadas),
    ("calidad de las pruebas (mutantes)", mutantes),
    ("ejemplos ejecutables (doctest)", doctest),
    ("Ruff: errores y seguridad", partial(herramienta, "-m", "ruff", "check", "src", "tests",
                                          clave="Found")),
    ("Ruff: formato", partial(herramienta, "-m", "ruff", "format", "--check", "src", "tests")),
    ("mypy --strict", partial(herramienta, "-m", "mypy")),
    ("Bandit (MEDIUM y HIGH bloquean)", bandit),
    ("complejidad (Xenon)", xenon),
    ("gobierno: reglas del equipo y docstrings", partial(herramienta,
                                                         "herramientas/puerta_gobierno.py",
                                                         clave=": ")),
    ("dependencias aprobadas", partial(herramienta, "herramientas/revisar_dependencias.py")),
]  # fmt: skip


def main() -> int:
    sys.stdout.reconfigure(errors="replace")
    capas = [c for c in CAPAS if "--sin-mutantes" not in sys.argv or "mutantes" not in c[0]]
    informe = []
    for nombre, ejecutar in capas:
        aprobado, resumen = ejecutar()
        informe.append({"capa": nombre, "aprobado": aprobado, "resumen": resumen})
        print(f"  {'✓' if aprobado else '✗'} {nombre:<44} {resumen}")
    fallidas = sum(not c["aprobado"] for c in informe)
    evidencia = {
        "fecha": datetime.now(UTC).isoformat(timespec="seconds"),
        "commit": git("rev-parse", "--short", "HEAD").stdout.strip() or "sin commit",
        "aprobado": fallidas == 0,
        "capas": informe,
    }
    Path("evidencia").mkdir(exist_ok=True)
    Path("evidencia/verificacion.json").write_text(
        json.dumps(evidencia, ensure_ascii=False, indent=2), encoding="utf-8"
    )
    total = len(informe)
    estado = "APROBADO" if fallidas == 0 else f"RECHAZADO: {fallidas} de {total} capas fallaron"
    print(f"{estado}. Evidencia: evidencia/verificacion.json")
    return 0 if fallidas == 0 else 1


if __name__ == "__main__":
    sys.exit(main())
```

**0.6** Crea el entorno e instala:

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
python -m pip install --no-deps -e .
python -c "import comisiones, requests, responses; print('ok')"
```

Resultado esperado: `ok`.

**0.7** Guarda el punto de partida y márcalo con la etiqueta `inicio`. La capa *controles del equipo intactos* compara contra ella:

```bash
git init -b main
git add .
git commit -m "ABC-142: código heredado y controles del equipo"
git tag inicio
```

> Si Git pide tu identidad: `git config --global user.name "Tu Nombre"` y `git config --global user.email "tu@correo.com"`.

**0.8** Prepara tu asistente y comprueba que **lee las reglas del proyecto**.

| | Claude Code | GitHub Copilot en VS Code |
| :--- | :--- | :--- |
| Instalar | macOS, Linux o WSL: `curl -fsSL https://claude.ai/install.sh \| bash` · Windows PowerShell: `irm https://claude.ai/install.ps1 \| iex` | Extensión **GitHub Copilot** desde Extensions; inicia sesión con tu cuenta de GitHub |
| Verificar | `claude --version` y `claude doctor` | El icono de Copilot en la barra de estado, sin errores |
| Abrir | `claude` en la terminal, dentro de `practica7_ia`; la primera vez inicia sesión en el navegador | Vista de chat con **Ctrl+Alt+I** (macOS: **Ctrl+Cmd+I**) |
| Reglas | Lee `CLAUDE.md`, que importa `@AGENTS.md` | Lee `AGENTS.md` de la raíz (ajuste `chat.useAgentsMdFile`, activo por omisión) |

Haz esta pregunta en tu asistente:

> **Prompt**
> ```text
> Según las reglas de este proyecto, ¿con qué comando verifico un cambio y qué archivos no debo modificar?
> ```

**Qué revisar:** la respuesta debe mencionar `python herramientas/verificar.py` y los archivos de la sección *Límites* de `AGENTS.md`. Si no los menciona, el asistente no está leyendo las reglas: en Claude Code, confirma que lo abriste en la carpeta del proyecto; en Copilot, revisa el ajuste `chat.useAgentsMdFile` o agrega `AGENTS.md` al chat con **#file**.

---

## Parte 1. Línea base: confianza cero (10 min)

Antes de pedirle nada a la IA, mide el punto de partida con las mismas capas que cruzará el resultado:

```bash
python herramientas/verificar.py --sin-mutantes
```

Resultado esperado (código de salida 1):

```text
  ✓ controles del equipo intactos                sin cambios desde la etiqueta 'inicio'
  ✓ caracterización (comportamiento heredado)    1 passed in 0.67s
  ✗ criterios de aceptación del negocio          6 failed, 7 passed in 0.13s
  ✗ políticas del equipo                         6 failed, 1 passed in 0.08s
  ✗ pruebas generadas (test_reglas.py)           no existe
  ✗ ejemplos ejecutables (doctest)               sin ejemplos ejecutables en los docstrings
  ✗ Ruff: errores y seguridad                    Found 6 errors.
  ✓ Ruff: formato                                7 files already formatted
  ✗ mypy --strict                                Found 5 errors in 2 files (checked 3 source files)
  ✗ Bandit (MEDIUM y HIGH bloquean)              2 bloquean, 2 para revisar. B501 L56 HIGH, B113 L52 MEDIUM, B105 L8 LOW, B110 L49 LOW
  ✗ complejidad (Xenon)                          average complexity is ranked B
  ✗ gobierno: reglas del equipo y docstrings     docstrings: 0% (mínimo 80%) · puerta de gobierno: RECHAZADO
  ✓ dependencias aprobadas                       dependencias no aprobadas: 0
RECHAZADO: 9 de 13 capas fallaron. Evidencia: evidencia/verificacion.json
```

Observa dos cosas:

- La **caracterización pasa**: con los datos de todos los días, el programa produce lo que produce. Esa es la red de seguridad para refactorizar: debe seguir en verde.
- Los **criterios de aceptación fallan**: el programa "funciona", pero no cumple el requisito. Correr a la primera no es lo mismo que ser correcto (7.13).

Consulta qué falla exactamente:

```bash
python -m pytest -q
```

Resultado esperado: `12 failed, 9 passed`, entre ellos `test_criterio_de_comision[maximo]`, `[redondeo-mitad-hacia-arriba]`, `[monto-cero-rechazado]`, `test_dos_ejecuciones_dan_el_mismo_resumen` y `test_sin_datos_personales_en_los_logs`.

> La capa de mutantes se omite (`--sin-mutantes`) hasta la Parte 4, cuando existan las pruebas que evalúa.

---

## Parte 2. Revisión de código con IA (15 min)

**2.1** Pide una revisión **sin edición**. El prompt sigue el formato del tema 7.3: contexto que cambia la severidad, prioridades en orden y un formato de respuesta concreto.

- **Claude Code:** abre `claude` y activa el **modo plan** con **Shift+Tab** (no edita archivos). Pega el prompt.
- **Copilot:** en la vista de chat elige el modo de preguntas (**Ask**), agrega el archivo con **#file:src/comisiones/proceso.py** y pega el prompt.

> **Prompt**
> ```text
> Actúa como revisor senior de un equipo de datos. No edites ningún archivo.
> Contexto: src/comisiones/proceso.py corre cada noche con unas 50,000 ventas; el CSV llega de
> un sistema externo que no controlamos y el resultado se envía a un webhook interno.
> Las reglas del equipo están en AGENTS.md.
> Revisa src/comisiones/proceso.py y busca, en este orden:
> 1. Seguridad: secretos, inyección, TLS, datos personales en los logs.
> 2. Corrección: manejo de dinero, errores silenciados, estado que se comparte entre llamadas.
> 3. Robustez y rendimiento: transacciones, commit por fila, timeout, qué queda si falla a la
>    mitad y códigos de salida.
> Para cada hallazgo da: línea, severidad (alta/media/baja), por qué importa y el cambio mínimo.
> No reescribas la función. Si no estás seguro de algo, plantéalo como pregunta.
> ```

**2.2** Ahora ejecuta las herramientas deterministas y compara:

```bash
ruff check src --output-format concise
bandit -q -r src -f custom --msg-template "L{line} {test_id} {severity} {msg}"
python herramientas/puerta_gobierno.py
```

Resultado esperado:

```text
src/comisiones/proceso.py:8:9: S105 Possible hardcoded password assigned to: "TOKEN"
src/comisiones/proceso.py:15:57: B006 Do not use mutable data structures for argument defaults
src/comisiones/proceso.py:43:17: S608 Possible SQL injection vector through string-based query construction
src/comisiones/proceso.py:49:9: S110 `try`-`except`-`pass` detected, consider logging the exception
src/comisiones/proceso.py:52:5: S113 Probable use of `requests` call without timeout
src/comisiones/proceso.py:56:9: S501 Probable use of `requests` call with `verify=False` disabling SSL certificate checks
Found 6 errors.

L8 B105 LOW Possible hardcoded password: 'xoxb-comisiones-2026-9f8e7d'
L49 B110 LOW Try, Except, Pass detected.
L56 B501 HIGH Call to requests with verify=False disabling SSL certificate checks, security issue.
L52 B113 MEDIUM Call to requests without timeout

src/comisiones/proceso.py:15 DOC-01 'procesar' sin docstring
src/comisiones/proceso.py:15 ERR-02 valor por defecto mutable en 'procesar'
src/comisiones/proceso.py:49 ERR-01 excepción genérica silenciada
src/comisiones/proceso.py:52 SEG-03 TLS desactivado (verify=False)
src/comisiones/proceso.py:61 DOC-01 'main' sin docstring
docstrings: 0% (mínimo 80%)
puerta de gobierno: RECHAZADO
```

**2.3** Llena la sección 1 de `EVIDENCIA.md`: por cada hallazgo de la IA, ¿lo confirma una herramienta (2.2) o una prueba que falla (Parte 1)? Después responde:

- ¿Qué defectos **no** detecta ninguna herramienta estática y solo aparecen como pruebas en rojo? (pista: el redondeo con `float`, el máximo de 500.00, los montos en cero, el correo en el log, el webhook que responde 503 y el programa termina con 0).
- ¿Qué detectó la IA que ninguna herramienta ni prueba cubre? (pista: el `commit()` por fila con 50,000 ventas, el `open()` sin `with` ni `encoding`).
- ¿Afirmó algo falso o exagerado?

> Es la idea del tema 7.3: las herramientas deterministas primero, la IA amplía la mirada y **la persona decide**.

---

## Parte 3. Refactorización y depuración con IA (30 min)

**3.1** Pide el cambio como un ticket bien escrito. Las pruebas del equipo son los criterios de aceptación; el plan se revisa **antes** de editar.

- **Claude Code:** sigue en **modo plan**; cuando el plan te parezca correcto, apruébalo para que edite.
- **Copilot:** cambia al **modo agente (Agent)** y revisa cada edición y cada comando antes de aceptarlo.

> **Prompt**
> ```text
> Tarea (ticket ABC-142): corrige y refactoriza src/comisiones/proceso.py sin cambiar su contrato.
> Contexto: lee AGENTS.md. Los criterios del negocio y las políticas ya están escritos como pruebas
> en tests/test_criterios_aceptacion.py y tests/test_politicas.py; tests/test_caracterizacion.py
> congela la salida actual con datos típicos y debe seguir en verde.
> Requisitos:
> - Crea src/comisiones/reglas.py con calcular_comision(monto: Decimal, region: str) -> Decimal,
>   lógica pura sin archivos ni red, y úsala desde proceso.py.
> - Separa proceso.py en funciones pequeñas: leer el CSV, guardar en SQLite, resumir y notificar.
> - Corrige lo que piden las pruebas: Decimal con ROUND_HALF_UP, mínimo 5.00 y máximo 500.00,
>   montos no positivos o no numéricos rechazados y registrados con su línea (sin el valor),
>   SQL con parámetros en una sola transacción, token desde COMISIONES_TOKEN, timeout y
>   raise_for_status, sin datos personales en los logs, códigos de salida 0/1/2 y sin valores
>   por defecto mutables.
> Restricciones: no modifiques las pruebas existentes, herramientas/, pyproject.toml ni los
> requirements; no agregues dependencias; conserva las firmas de procesar y main, la tabla
> comisiones y el JSON del webhook (números, no texto).
> Antes de editar, muéstrame el plan. Al terminar ejecuta
> python herramientas/verificar.py --sin-mutantes y muéstrame la salida completa.
> ```

**3.2 Revisión humana del plan** (7.13). Antes de aprobarlo, comprueba que:

- No propone tocar archivos de *Límites* ni agregar dependencias.
- Mantiene `procesar(entrada, bd, url)` y `main(argv)`.
- Separa `calcular_comision` en `reglas.py`.

Si propone "ajustar" una prueba del equipo porque "está mal", **recházalo**: la prueba define el requisito.

**3.3 Revisión humana del cambio.** Cuando termine, lee el diff completo; no aceptes lo que no entiendes:

```bash
git status
git diff --stat
git diff src/comisiones/proceso.py
```

Usa la lista de cuatro pilares (7.13):

| Pilar | Pregunta | Dónde mirar |
| :--- | :--- | :--- |
| Corrección | ¿El dinero es `Decimal`? ¿La tasa alta aplica solo si el monto **supera** 10,000? ¿El máximo está? | `reglas.py` |
| Seguridad | ¿El token sale de `os.environ`? ¿Hay `timeout`? ¿Los logs tienen correos o valores de la fila? | `notificar()` y los `log.` |
| Robustez | ¿Una sola transacción? ¿Qué devuelve `main` si el webhook falla? | `guardar()` y `main()` |
| Mantenibilidad | ¿Funciones pequeñas con un solo propósito? ¿Tipos en todas las firmas? | todo el diff |

**3.4** Verifica **tú**, aunque el asistente ya haya mostrado su resultado: un reporte del agente no es evidencia.

```bash
python herramientas/verificar.py --sin-mutantes
```

Resultado esperado: todas las capas en ✓ **excepto** `pruebas generadas (test_reglas.py)`, que aún no existe (`RECHAZADO: 1 de 13 capas fallaron`).

Si falla otra capa, aplica el ciclo del tema 7.12: copia el **error real** (la línea de `verificar.py` y, si es una prueba, la salida de `python -m pytest -q` con el nombre de la prueba) y pégalo al asistente:

> **Prompt**
> ```text
> La verificación falla con esta salida: <pega la salida>.
> Corrige solo lo necesario para que pase, sin modificar las pruebas ni herramientas/.
> Explica la causa en una frase antes del cambio.
> ```

Máximo tres iteraciones; si después de eso sigue fallando, corrígelo a mano o pide el checkpoint al instructor. Anota en `EVIDENCIA.md` cuántas iteraciones hicieron falta y qué tuviste que corregir tú.

> **Si `ruff format` es la única capa en rojo**, ejecuta `ruff format src tests` y vuelve a verificar. El formato lo resuelve una herramienta, no la IA.

**3.5** Guarda el cambio dejando constancia de la asistencia (7.19):

```bash
git add src
git commit -m "ABC-142: reglas en reglas.py, Decimal, transacción y webhook seguro" -m "Asistido-por-IA: Claude Code"
```

(Escribe la herramienta que usaste.)

> **Checkpoint 3.** `verificar.py --sin-mutantes`: 12 de 13 capas en ✓; solo falta `test_reglas.py`.

---

## Parte 4. Pruebas generadas y prueba de las pruebas (20 min)

**4.1 Primero, una prueba que no protege nada.** Crea `tests/test_reglas.py` con esta propuesta típica de una IA sin indicaciones: una prueba que verifica su propio mock y un solo caso feliz.

```python
from decimal import Decimal
from unittest.mock import MagicMock, patch

from comisiones.reglas import calcular_comision


@patch("comisiones.reglas.calcular_comision")
def test_calcular_comision(mock_calculo: MagicMock) -> None:
    mock_calculo.return_value = Decimal("15.00")
    assert mock_calculo(Decimal("1000.00"), "norte") == Decimal("15.00")  # prueba el mock


def test_caso_feliz() -> None:
    assert calcular_comision(Decimal("1000.00"), "norte") == Decimal("15.00")
```

```bash
python -m pytest -q tests/test_reglas.py
python herramientas/probar_las_pruebas.py src/comisiones/reglas.py tests/test_reglas.py
```

Resultado esperado: `2 passed`, y luego la mayoría de los mutantes con `✗ SOBREVIVE`. Con la solución de referencia de `reglas.py` el resultado es `mutantes detectados: 4/16 (25%); mínimo 80%` (con tu `reglas.py` el número de mutantes puede ser otro). Las pruebas están en verde **con los errores dentro**: no protegen nada (7.2).

**4.2** Pide pruebas a partir de las **reglas de negocio**, no del código. Si se las pides a partir del código, la IA confirmará lo que el código ya hace, incluidos sus errores (7.14).

> **Prompt**
> ```text
> Reemplaza tests/test_reglas.py con pruebas pytest para calcular_comision de
> src/comisiones/reglas.py. Los valores esperados salen de estas REGLAS DE NEGOCIO, no del código:
> norte 1.5 % (2 % si el monto supera 10,000), sur 2 % (2.5 % si supera 10,000), otras regiones 1 %;
> comisión mínima 5.00 y máxima 500.00; redondeo a centavos con mitad hacia arriba;
> un monto menor o igual a 0 lanza ValueError.
> Usa pytest.mark.parametrize con un comentario por caso, incluye las fronteras de cada regla y los
> errores con pytest.raises(match=...). Sin mocks: es una función pura. No modifiques reglas.py.
> Ejecuta python -m pytest -q tests/test_reglas.py y muéstrame el resultado.
> ```

**Qué revisar:** que cada valor esperado se pueda calcular a mano con las reglas. Calcula al menos dos tú mismo.

**4.3** Prueba las pruebas:

```bash
python herramientas/probar_las_pruebas.py src/comisiones/reglas.py tests/test_reglas.py
```

Cada mutante que sobrevive es una de dos cosas:

- **Un hueco de las pruebas:** un cambio de comportamiento que ninguna prueba detecta. Se agrega un caso.
- **Un mutante equivalente:** el cambio no altera nada observable. Se documenta y se deja.

Con la solución de referencia, el resultado es:

```text
  ✓ detectado   L5: '10000' -> '10001'
  ...
  ✓ detectado   L48: '>' -> '>='
  ✓ detectado   L49: Mult -> Div
  ✗ SOBREVIVE  L49: '0.01' -> '0.02'
  ✓ detectado   L49: ROUND_HALF_UP -> ROUND_HALF_EVEN
  ✓ detectado   L50: min() -> max()
  ✓ detectado   L50: max() -> min()
mutantes detectados: 15/16 (94%); mínimo 80%
```

`quantize(Decimal("0.01"))` y `quantize(Decimal("0.02"))` hacen lo mismo (solo importa el exponente: dos decimales), así que ese mutante es **equivalente**. En cambio, si sobrevive `ROUND_HALF_UP -> ROUND_HALF_EVEN`, es un hueco: falta un caso donde los dos redondeos den resultados distintos (por ejemplo, norte con 1003.00 da 15.045 → 15.05 con mitad hacia arriba y 15.04 con mitad al par). Ojo: 1001.00 da 15.015, que con los dos redondeos queda en 15.02.

**4.4** Si tienes huecos, devuélvelos al asistente:

> **Prompt**
> ```text
> Estos mutantes sobrevivieron: <pega las líneas ✗ SOBREVIVE>.
> Para cada uno dime si es un hueco de las pruebas o un mutante equivalente, y por qué.
> Agrega casos solo para los huecos, con valores calculados a partir de las reglas de negocio.
> No modifiques reglas.py.
> ```

Repite 4.3 hasta llegar al 80 % o más y anota en `EVIDENCIA.md` el puntaje inicial, el final y la decisión sobre cada sobreviviente.

> No edites `reglas.py` mientras corre `probar_las_pruebas.py`: lo modifica temporalmente y lo restaura al terminar (también si lo interrumpes con Ctrl+C).

```bash
git add tests/test_reglas.py
git commit -m "ABC-142: pruebas de calcular_comision" -m "Asistido-por-IA: Claude Code"
```

> **Checkpoint 4.** `pytest -q tests/test_reglas.py` en verde y mutantes ≥ 80 %.

---

## Parte 5. Documentación verificable con IA (10 min)

> **Prompt**
> ```text
> Documenta src/comisiones/reglas.py y src/comisiones/proceso.py con docstrings estilo Google
> (Args, Returns, Raises y, en los generadores, Yields). En calcular_comision agrega una sección
> Examples con al menos tres ejemplos de doctest tomados de tests/test_criterios_aceptacion.py,
> incluido un caso de error. Describe solo lo que el código hace; si algo no está claro, pregúntame.
> No cambies el código.
> ```

Verifica que los ejemplos se ejecutan y que la cobertura de docstrings cumple:

```bash
python -m pytest --doctest-modules src -q
python herramientas/puerta_gobierno.py
```

Resultado esperado: `N passed` (un elemento por cada función con ejemplos) y `puerta de gobierno: APROBADO` con docstrings al 80 % o más.

**Qué revisar** (7.4): la documentación generada puede describir lo que el código *parece* hacer. Lee cada docstring y marca cualquier afirmación que no puedas comprobar en el código (por ejemplo, "reintenta el envío", "valida el formato del folio" o "redondea al par"). Anota en `EVIDENCIA.md` lo que encontraste y corrígelo.

Comprueba que un ejemplo incorrecto se detecta: cambia temporalmente un resultado del doctest (por ejemplo `Decimal('15.00')` por `Decimal('15.01')`), ejecuta de nuevo `python -m pytest --doctest-modules src -q` (debe fallar) y **revierte** el cambio.

```bash
git add src
git commit -m "ABC-142: docstrings con ejemplos ejecutables" -m "Asistido-por-IA: Claude Code"
```

> **Checkpoint 5.** doctest en verde y puerta de gobierno APROBADO.

---

## Parte 6. Seguridad, gobierno y verificación final (15 min)

**6.1 Una dependencia que no existe** (7.17). Un asistente puede sugerir un paquete con un nombre plausible que nadie publicó; si alguien lo publica después con código malicioso, `pip install` lo instala (slopsquatting). Agrega temporalmente esta línea debajo de `import requests` en `src/comisiones/proceso.py`:

```python
import snowflake_autoloader  # sugerido por la IA
```

```bash
python herramientas/revisar_dependencias.py --pypi
```

Resultado esperado (código 1; requiere salida a internet):

```text
requests               -> requests               aprobado
snowflake_autoloader   -> snowflake-autoloader   NO aprobado; PyPI: NO EXISTE en PyPI
dependencias no aprobadas: 1
```

Sin red, la consulta dice `sin red: verificar a mano`, pero la capa igual bloquea: no está en la lista aprobada. **Borra la línea.**

**6.2 Las instrucciones orientan; los controles obligan** (7.12, 7.19). Simula que un asistente "arregló" una prueba para que pase: agrega una línea cualquiera al final de `tests/test_politicas.py`, por ejemplo `# ajuste`, y ejecuta:

```bash
python herramientas/verificar.py --sin-mutantes
```

Resultado esperado: la primera capa en rojo:

```text
  ✗ controles del equipo intactos                modificados: tests/test_politicas.py
```

Revierte el cambio:

```bash
git checkout -- tests/test_politicas.py
```

**6.3 Verificación final** con todas las capas, incluidos los mutantes:

```bash
ruff format src tests
python herramientas/verificar.py
```

Resultado esperado (los tiempos y el número de mutantes pueden variar con tu código):

```text
  ✓ controles del equipo intactos                sin cambios desde la etiqueta 'inicio'
  ✓ caracterización (comportamiento heredado)    1 passed in 0.68s
  ✓ criterios de aceptación del negocio          13 passed in 0.09s
  ✓ políticas del equipo                         7 passed in 0.04s
  ✓ pruebas generadas (test_reglas.py)           18 passed in 0.02s
  ✓ calidad de las pruebas (mutantes)            mutantes detectados: 15/16 (94%); mínimo 80%
  ✓ ejemplos ejecutables (doctest)               1 passed in 0.07s
  ✓ Ruff: errores y seguridad                    All checks passed!
  ✓ Ruff: formato                                9 files already formatted
  ✓ mypy --strict                                Success: no issues found in 4 source files
  ✓ Bandit (MEDIUM y HIGH bloquean)              0 bloquean, 0 para revisar.
  ✓ complejidad (Xenon)                          funciones y módulos <= B, promedio A
  ✓ gobierno: reglas del equipo y docstrings     docstrings: 100% (mínimo 80%) · puerta de gobierno: APROBADO
  ✓ dependencias aprobadas                       dependencias no aprobadas: 0
APROBADO. Evidencia: evidencia/verificacion.json
```

**6.4** Guarda la evidencia y mide el tamaño del cambio:

```bash
git add evidencia EVIDENCIA.md
git commit -m "ABC-142: evidencia de verificación" -m "Asistido-por-IA: Claude Code"
git diff --stat inicio
git log --oneline
```

Copia en `EVIDENCIA.md` la salida de 6.3 y de `git diff --stat inicio`, y completa la sección 6 (valor operativo).

---

## Preguntas de cierre (5 min)

1. **Corrección.** El programa heredado pasaba la caracterización y fallaba 6 criterios de aceptación. ¿Qué te dice eso sobre aceptar código porque "corre"?
2. **Pruebas.** La prueba tautológica de 4.1 estaba en verde. ¿Qué la delató? ¿Por qué no basta con medir cobertura de líneas?
3. **Herramientas.** ¿Qué parte del trabajo hizo mejor la IA y qué parte tuviste que corregir tú? ¿Qué control detectó algo que la IA no vio?
4. **Gobierno.** `AGENTS.md` pedía no modificar las pruebas del equipo. ¿Por qué además existe la capa *controles del equipo intactos*?

---

## Entregables

Comprime `practica7_ia` **sin** `.venv/` (o comparte el repositorio). Antes de entregar, revisa:

- [ ] `src/comisiones/reglas.py` con `calcular_comision` y `proceso.py` refactorizado
- [ ] `tests/test_reglas.py` con mutantes ≥ 80 % y la decisión sobre cada sobreviviente
- [ ] `python herramientas/verificar.py` en **APROBADO** y `evidencia/verificacion.json`
- [ ] `git diff --stat inicio` sin cambios en `tests/conftest.py`, `tests/test_caracterizacion.py`, `tests/test_criterios_aceptacion.py`, `tests/test_politicas.py`, `tests/datos/` ni `herramientas/`
- [ ] Commits con la línea `Asistido-por-IA`
- [ ] `EVIDENCIA.md` completa: prompts, hallazgos confirmados y descartados, iteraciones, mutantes, verificación final y valor operativo
- [ ] Respuestas a las preguntas de cierre

---

## Solución de problemas

| Síntoma | Causa probable | Solución |
| :--- | :--- | :--- |
| `ModuleNotFoundError: No module named 'comisiones'` | El paquete no está instalado en el entorno activo | Activa `.venv` y ejecuta `python -m pip install --no-deps -e .` |
| `ModuleNotFoundError: No module named 'tests'` | No estás en la raíz del proyecto | Ejecuta desde `practica7_ia` (`pythonpath = ["."]` en `pyproject.toml`) |
| `falta la etiqueta 'inicio' (Parte 0)` | No creaste la etiqueta | `git tag inicio` sobre el commit del código heredado (`git log --oneline` para verlo) |
| `modificados: ...` en *controles intactos* sin haberlos tocado | El asistente los editó, o creaste los archivos después de la etiqueta | Revisa con `git diff inicio -- tests herramientas`; restaura con `git checkout inicio -- <archivo>` |
| La caracterización falla tras el refactor | El cambio alteró el contrato: otra tabla, otro JSON, otra CLI | Compara con `git diff`; pide al asistente conservar la tabla `comisiones` y el JSON con números |
| `test_notificacion_es_json_con_total_y_por_vendedor` falla | El JSON envía los montos como texto (`"15.00"`) | El contrato pide números; convierte con `float()` solo al serializar |
| `Las pruebas deben pasar con el código original antes de mutarlo.` | `test_reglas.py` tiene pruebas en rojo | Corrige primero las pruebas: `python -m pytest -q tests/test_reglas.py` |
| Un mutante sobrevive y no encuentras el caso que lo mata | Puede ser equivalente | Si no cambia ningún resultado posible, documéntalo como equivalente en `EVIDENCIA.md` |
| `reglas.py` quedó raro tras interrumpir los mutantes | Se cortó el proceso antes de restaurar | `git checkout -- src/comisiones/reglas.py` (o tu último commit) |
| El asistente no respeta `AGENTS.md` | No lo está leyendo | 0.8: abre Claude Code en la raíz; en Copilot revisa `chat.useAgentsMdFile` o adjunta `#file:AGENTS.md` |
| Copilot dejó de responder a mitad de la práctica | Límite mensual del plan Free | Continúa con el checkpoint del instructor y anota el límite en `EVIDENCIA.md` |
| PowerShell no deja activar `.venv` | Política de ejecución de Windows | `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` |

---

## Limpieza

**PowerShell**
```powershell
deactivate
Remove-Item -Recurse -Force .venv, .pytest_cache, .mypy_cache, .ruff_cache -ErrorAction SilentlyContinue
```

**bash**
```bash
deactivate
rm -rf .venv .pytest_cache .mypy_cache .ruff_cache
```

---

## Resumen

| Lo que hiciste | Lección |
| :--- | :--- |
| Escribiste prompts como tickets: rol, contexto, tarea, restricciones, formato y criterios | 7.1 |
| Generaste pruebas con IA y las evaluaste con mutantes, no con el color de pytest | 7.2, 7.15 |
| Pediste una revisión útil y confirmaste cada hallazgo con herramientas y pruebas | 7.3 |
| Documentaste con ejemplos que se ejecutan | 7.4 |
| Refactorizaste con una prueba de caracterización como red de seguridad | 7.5 |
| Corregiste errores devolviendo el error real al asistente | 7.6 |
| Aplicaste confianza cero: 14 capas antes de aceptar | 7.8 |
| Usaste Claude Code o Copilot según la actividad, con las reglas en `AGENTS.md` | 7.9 a 7.12 |
| Revisaste el diff con cuatro pilares y validaste contra el requisito del negocio | 7.13, 7.14 |
| Eliminaste secretos, TLS desactivado, SQL armado con texto y datos personales en logs | 7.16 |
| Detectaste un paquete inventado antes de instalarlo | 7.17 |
| Mediste la complejidad con Xenon | 7.18 |
| Dejaste evidencia: puerta de gobierno, controles intactos, JSON y commits con `Asistido-por-IA` | 7.19 |
