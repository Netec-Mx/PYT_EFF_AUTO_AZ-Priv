<a href="https://netec.com">
  <img src="https://static.wixstatic.com/media/96edd2_934dddb3230b4b16b1d1c56c51143738~mv2.png" alt="Logo de Netec" width="300"/>
</a>

# Automatización de procesos con Python

## Plataforma de laboratorios

Este curso proporciona formación práctica y avanzada en el uso de Python para automatizar, estandarizar y escalar procesos internos relacionados con ingeniería y datos.

El programa se enfoca en evolucionar desde scripts hacia componentes de automatización mantenibles, comprobables mediante pruebas, seguros, observables y escalables, capaces de integrarse con pipelines de CI/CD, flujos de datos, herramientas operacionales, APIs y servicios programados.

Las prácticas se desarrollan con datos y escenarios ficticios. El curso incorpora además el uso responsable de Claude Code y GitHub Copilot como apoyo durante el desarrollo con Python.

---

## Lista de laboratorios

<a id="capitulo-1"></a>

### 📘 Capítulo 1. Diseño de automatizaciones y estándares de ingeniería Python

- Práctica: [Práctica 1. De un proceso manual a una automatización mantenible](Capitulo01/README.md)
- Duración estimada: 90 min
- Descripción: Analizar un proceso repetitivo relacionado con datos y transformarlo en la base de una automatización Python estructurada y reutilizable. Primero vas a identificar qué conviene automatizar, definir el alcance, los modos de fallo y los puntos que requieren aprobación humana. Después construirás una solución modular con anotaciones de tipo, configuración externa, variables de entorno, gestión de dependencias y documentación.

<a id="capitulo-2"></a>

### 📘 Capítulo 2. Automatización de archivos, datos y Data Quality

- Práctica: [Práctica 2. Automatización de calidad y validación de contratos de datos](Capitulo02/README.md)
- Duración estimada: 120 min
- Descripción: Automatizar controles repetibles de estructura y calidad sobre conjuntos de datos ficticios destinados a un flujo de procesamiento empresarial. Vas a construir una automatización configurable que ingiere archivos y valida su esquema y su contrato de datos, aplica reglas de calidad, detecta problemas de completitud, duplicados y anomalías, transforma los datos y genera automáticamente resultados y excepciones. La misma lógica se reutiliza sobre dos conjuntos de datos distintos cambiando solo la configuración.

<a id="capitulo-3"></a>

### 📘 Capítulo 3. Integración con APIs, servicios, bases de datos y warehouses

- Práctica: [Práctica 3. Automatización de integración de datos con Snowflake](Capitulo03/README.md)
- Duración estimada: 120 min
- Descripción: Automatizar la integración de datos de una API ficticia con Snowflake o un entorno simulado. La automatización cubre la extracción (autenticación, paginación, límites de tasa, timeouts y reintentos), la validación (contrato de la respuesta con Pydantic), la carga (control de esquema, reconciliación e idempotencia) y las credenciales (en variables de entorno, nunca en el código). El destino es un emulador local de Snowflake.

<a id="capitulo-4"></a>

### 📘 Capítulo 4. Ingeniería de confiabilidad, observabilidad y remediación

- Práctica: [Práctica 4. Monitoreo de pipelines y remediación controlada de pipelines](Capitulo04/README.md)
- Duración estimada: 90 min
- Descripción: Implementar el monitoreo de un pipeline de datos como se haría en Amazon MWAA con CloudWatch: detectar fallos con las señales de código de salida, tiempo máximo, volumen y latido; recolectar información diagnóstica sin exponer datos sensibles; decidir si cada fallo es recuperable; aplicar reintentos, alertas o remediación controlada según el caso; y registrar cada decisión operativa en una auditoría verificable. No necesitas cuenta de AWS.

<a id="capitulo-5"></a>

### 📘 Capítulo 5. Pruebas, paralelismo y optimización de rendimiento

- Práctica: [Práctica 5. Pruebas y escalamiento de una automatización](Capitulo05/README.md)
- Duración estimada: 120 min
- Descripción: Implementar una suite de pruebas para un pipeline de datos y escalarlo de forma segura. Incluye pruebas unitarias, de integración, con mocks, con datos sintéticos y de regresión; validar que los fallos de la API (timeout, 5xx, 429, JSON inválido) se manejan como indica el diseño; y medir, aplicar una optimización y comparar el antes y el después sin regresiones. Todo es local: no necesitas cuentas, Docker ni servicios en la nube.

<a id="capitulo-6"></a>

### 📘 Capítulo 6. Programación de ejecuciones, orquestación, seguridad y despliegue

- Práctica: [Práctica 6. De automatización local a solución desplegable y orquestada](Capitulo06/README.md)
- Duración estimada: 180 min
- Descripción: Preparar una automatización Python para su ejecución reproducible: empaquetarla como paquete interno con CLI, versión única, lockfile con hashes y un wheel verificado; contenerizarla con Docker en una imagen multi-stage que corre sin root y recibe sus secretos desde fuera; configurar CI/CD con GitHub Actions con pruebas, calidad, seguridad y una entrega controlada con prueba de humo y reversión; y definir un DAG para Amazon MWAA que coordine las tareas, sus dependencias, sus estados y el manejo de fallos con base en el código de salida. No necesitas cuenta de AWS: todo se ejecuta en tu equipo. La parte de la nube (ECR, ECS Fargate, MWAA) queda escrita, validada y lista para conectarse.

<a id="capitulo-7"></a>

### 📘 Capítulo 7. IA aplicada al desarrollo y automatizaciones reutilizables

- Práctica: [Práctica 7. Productividad del desarrollo asistida por IA](Capitulo07/README.md)
- Duración estimada: 120 min
- Descripción: Aplicar IA de manera controlada sobre actividades concretas del ciclo de desarrollo Python y validar técnicamente el resultado antes de incorporarlo. Usarás Claude Code o GitHub Copilot sobre código existente para revisar, refactorizar y depurar, generar pruebas y documentar. Cada propuesta de la IA es un borrador: solo entra a la solución si cruza las capas de verificación del equipo.

---

## 📬 Contacto y más información

Si tienes alguna pregunta o necesitas más detalles, no dudes en [contactarnos](mailto:soporte@netec.com). También puedes encontrar más recursos en nuestra [página](https://netec.com).

---

¡Gracias por visitar nuestra plataforma! No olvides revisar todos los laboratorios y comenzar tu viaje de aprendizaje hoy mismo.
