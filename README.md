<a href="https://netec.com">
  <img src="https://static.wixstatic.com/media/96edd2_934dddb3230b4b16b1d1c56c51143738~mv2.png" alt="Logo de Netec" width="300"/>
</a>

# Automatización de procesos con Python

## Plataforma de laboratorios

Este curso proporciona formación práctica y avanzada en el uso de Python para automatizar, estandarizar y escalar procesos internos relacionados con ingeniería y datos.

El programa se enfoca en evolucionar desde scripts hacia componentes de automatización mantenibles, comprobables mediante pruebas, seguros, observables y escalables, capaces de integrarse con pipelines de CI/CD, flujos de datos, herramientas operacionales, APIs y servicios programados.

Las prácticas se desarrollan con datos y escenarios ficticios. El curso incorpora además el uso responsable de Claude Code y GitHub Copilot como apoyo durante el desarrollo con Python.


## Navegación rápida

- [Capítulo 1. Diseño de automatizaciones y estándares de ingeniería Python](#capitulo-1)
- [Capítulo 2. Automatización de archivos, datos y Data Quality](#capitulo-2)
- [Capítulo 3. Integración con APIs, servicios, bases de datos y warehouses](#capitulo-3)
- [Capítulo 4. Ingeniería de confiabilidad, observabilidad y remediación](#capitulo-4)
- [Capítulo 5. Pruebas, paralelismo y optimización de rendimiento](#capitulo-5)
- [Capítulo 6. Programación de ejecuciones, orquestación, seguridad y despliegue](#capitulo-6)
- [Capítulo 7. IA aplicada al desarrollo y automatizaciones reutilizables](#capitulo-7)
- [Resumen de tiempos](#resumen-de-tiempos)
- [Materiales y anexos](#materiales-y-anexos)
- [Contacto y más información](#contacto)

---

## Lista de laboratorios

<a id="capitulo-1"></a>

### Capítulo 1. Diseño de automatizaciones y estándares de ingeniería Python

- Práctica: [Práctica 1. De un proceso manual a una automatización mantenible](Capitulo01/README.md)
- Duración estimada de la práctica: 90 min
- Actividades: analizar un proceso manual, definir su alcance, sus modos de fallo y los puntos de aprobación humana, y estructurar una solución modular con anotaciones de tipo, configuración externa, variables de entorno, gestión de dependencias y documentación.
- Resultado: la automatización compras-auto, que valida y clasifica solicitudes de compra, envía a aprobación las que superan el límite, separa los registros inválidos y comprueba que ninguno se pierda.

<a id="capitulo-2"></a>

### Capítulo 2. Automatización de archivos, datos y Data Quality

- Práctica: [Práctica 2. Automatización de calidad y validación de contratos de datos](Capitulo02/README.md)
- Duración estimada de la práctica: 120 min
- Actividades: ingerir archivos, validar su esquema y su contrato de datos, aplicar reglas de calidad, detectar problemas de completitud, duplicados y anomalías, y transformar los datos.
- Resultado: el validador calidad-datos, que genera resultados, excepciones y reportes, y se reutiliza sobre dos conjuntos de datos distintos cambiando solo la configuración.

<a id="capitulo-3"></a>

### Capítulo 3. Integración con APIs, servicios, bases de datos y warehouses

- Práctica: [Práctica 3. Automatización de integración de datos con Snowflake](Capitulo03/README.md)
- Duración estimada de la práctica: 120 min
- Actividades: extraer datos de una API ficticia con autenticación, paginación, límites de tasa, tiempos de espera y reintentos, validar las respuestas con Pydantic y cargarlas en un emulador local de Snowflake mediante el conector oficial.
- Resultado: el integrador integracion-ventas, con control de esquema, reconciliación e idempotencia, que puede ejecutarse repetidamente de forma segura.

<a id="capitulo-4"></a>

### Capítulo 4. Ingeniería de confiabilidad, observabilidad y remediación

- Práctica: [Práctica 4. Monitoreo de pipelines y remediación controlada de pipelines](Capitulo04/README.md)
- Duración estimada de la práctica: 90 min
- Actividades: detectar fallos en un pipeline, recolectar información diagnóstica sin exponer datos sensibles, decidir si cada fallo es recuperable y aplicar reintentos, alertas o remediación controlada, como se haría en Amazon MWAA con CloudWatch, sin necesidad de una cuenta de AWS.
- Resultado: el monitor monitoreo-pipeline, que cierra el ciclo de detectar, diagnosticar, decidir, actuar y registrar, y deja cada decisión en una auditoría verificable.

<a id="capitulo-5"></a>

### Capítulo 5. Pruebas, paralelismo y optimización de rendimiento

- Práctica: [Práctica 5. Pruebas y escalamiento de una automatización](Capitulo05/README.md)
- Duración estimada de la práctica: 120 min
- Actividades: construir pruebas unitarias, de integración, con objetos simulados, con datos sintéticos y de regresión, validar el manejo de fallos de una API, medir el rendimiento y aplicar una optimización.
- Resultado: un pipeline de ventas con su suite de pruebas y una comparación del antes y el después de la optimización, sin regresiones.

<a id="capitulo-6"></a>

### Capítulo 6. Programación de ejecuciones, orquestación, seguridad y despliegue

- Práctica: [Práctica 6. De automatización local a solución desplegable y orquestada](Capitulo06/README.md)
- Duración estimada de la práctica: 180 min, más una parte opcional de 30 min con Airflow local, que no se incluye en el total
- Actividades: empaquetar una automatización como paquete interno con CLI, contenerizarla con Docker, configurar CI/CD con GitHub Actions y definir un DAG para Amazon MWAA.
- Resultado: la solución cargaventas, con wheel verificado, imagen de contenedor, flujos de integración y entrega, y un DAG que coordina tareas, dependencias, estados y manejo de fallos. Todo se ejecuta en el equipo local.

<a id="capitulo-7"></a>

### Capítulo 7. IA aplicada al desarrollo y automatizaciones reutilizables

- Práctica: [Práctica 7. Productividad del desarrollo asistida por IA](Capitulo07/README.md)
- Duración estimada de la práctica: 120 min
- Actividades: usar Claude Code o GitHub Copilot sobre código existente para revisar, refactorizar, depurar, generar pruebas y documentar, verificando técnicamente cada propuesta antes de incorporarla.
- Resultado: un script heredado de cálculo de comisiones corregido y mantenible, sin cambios en su contrato, con evidencia de que es correcto y seguro.

---


## 📬 Contacto y más información

Si tienes alguna pregunta o necesitas más detalles, no dudes en [contactarnos](mailto:soporte@netec.com). También puedes encontrar más recursos en nuestra [página](https://netec.com).

---

¡Gracias por visitar nuestra plataforma! No olvides revisar todos los laboratorios y comenzar tu viaje de aprendizaje hoy mismo.
