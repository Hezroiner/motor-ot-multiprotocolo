# Bitácora de decisiones técnicas

Registro de las decisiones tomadas por el equipo durante el desarrollo del TFG
*"Diseño de un motor multiprotocolo de Tecnología Operativa en un ambiente simulado
en el LabCIBE de la Universidad Nacional de Costa Rica."*

**Propósito.** El anteproyecto no exige ninguna tecnología concreta. Toda selección
es una decisión del equipo y debe poder sustentarse ante el comité asesor. Este
archivo evita redecidir lo mismo en cada sesión y deja constancia fechada del
criterio aplicado.

**Cómo se usa.** Una entrada por decisión, numerada de forma correlativa y sin
reutilizar números. Las decisiones no se borran: si una se revierte, se agrega una
nueva entrada que la sustituye y la anterior se marca como *Sustituida*.

**Estados posibles:** `Adoptada` · `Propuesta` · `Sustituida` · `Pendiente`

---

## Índice

| # | Decisión | Estado | Fecha |
|---|---|---|---|
| D-01 … D-11 | Decisiones de las sesiones previas de Etapa 2 | ⚠️ Pendiente de trasladar | — |
| D-12 | Lenguaje de implementación: Python 3.11+ | Adoptada | 2026-08-17 |
| D-13 | Cliente Modbus TCP: `pymodbus` | Adoptada | 2026-08-17 |
| D-14 | Cliente OPC UA: `asyncua` | Adoptada | 2026-08-17 |
| D-15 | Persistencia: SQLite en desarrollo | Adoptada | 2026-08-17 |
| D-16 | API: FastAPI | Adoptada | 2026-08-17 |
| D-17 | Dashboard: React | Adoptada | 2026-08-17 |
| D-18 | Pruebas: `pytest` | Adoptada | 2026-08-17 |
| D-19 | Nombre del repositorio | Adoptada | 2026-08-17 |
| D-20 | Repositorio único (monorepo) | Adoptada | 2026-08-17 |
| D-21 | Plataforma de alojamiento: GitHub | Adoptada | 2026-08-17 |
| D-22 | Estrategia de ramas y etiquetas | Adoptada | 2026-08-17 |

---

## ⚠️ D-01 a D-11 — Pendiente de trasladar

El documento `docs/etapa2/2.3_2.6_arquitectura_y_repositorio.md` referencia una
**decisión D-11** del equipo (alcance del prototipo: motor + base de datos +
dashboard). Eso indica que existen decisiones D-01 a D-11 tomadas en sesiones
previas de la Etapa 2 que aún no están registradas aquí.

**Acción pendiente:** recuperarlas de las notas de esas sesiones y transcribirlas
con el mismo formato antes de cerrar la Etapa 2. La numeración de este archivo
arranca en D-12 precisamente para no invadir ese rango.

---

## D-12 — Lenguaje de implementación: Python 3.11 o superior

**Estado:** Adoptada · **Fecha:** 2026-08-17 · **Responsable:** Equipo

**Contexto.** El motor debe hablar dos protocolos industriales, normalizar sus
resultados, aplicar reglas de clasificación y persistir los hallazgos. Se necesita
un lenguaje con bibliotecas maduras para ambos protocolos y con el que el equipo
pueda avanzar sin una curva de aprendizaje que consuma el tiempo del TFG.

**Alternativas consideradas.**

| Opción | Por qué se descartó |
|---|---|
| C / C++ (p. ej. libmodbus, open62541) | Bibliotecas de referencia y mayor control, pero el costo de desarrollo y depuración no se justifica en un prototipo académico |
| Java | Existen clientes para ambos protocolos, pero el equipo no tiene base previa y el ciclo de iteración es más lento |
| Node.js / JavaScript | Unificaría lenguaje con el dashboard, pero el soporte de OPC UA es menos maduro que en Python |
| Go | Buen rendimiento y despliegue simple; ecosistema OPC UA menos consolidado |

**Justificación.** Python concentra las dos bibliotecas cliente más utilizadas para
Modbus TCP y OPC UA, permite escribir reglas de clasificación de forma legible —
lo que importa porque esas reglas se documentan en la memoria y se defienden ante
el comité — y el equipo cuenta con experiencia previa en backend.

Se fija 3.11 como mínimo por el soporte de tipado y por `asyncio`, relevante para
las consultas concurrentes controladas del módulo `discovery`.

**Consecuencias.** El dashboard queda en otro lenguaje (ver D-17), lo que obliga a
una frontera explícita entre backend y frontend. Se considera aceptable: esa
frontera ya estaba prevista en la arquitectura.

---

## D-13 — Cliente Modbus TCP: `pymodbus`

**Estado:** Adoptada · **Fecha:** 2026-08-17 · **Responsable:** Equipo

**Contexto.** El adaptador `protocols/modbus/` necesita emitir lecturas controladas
(por ejemplo, códigos de función de lectura de registros) contra el testbed de
Factory I/O, con control explícito de tiempos de espera.

**Alternativas consideradas.**

| Opción | Valoración |
|---|---|
| `pyModbusTCP` | Más liviana y simple, pero cubre menos casos y ofrece menos control sobre el manejo de errores y reintentos |
| `umodbus` | Orientada a implementaciones mínimas; menos documentación disponible |
| Implementación propia sobre sockets | Educativa, pero desviaría tiempo del objetivo real del TFG hacia reimplementar un protocolo ya resuelto |

**Justificación.** `pymodbus` es la referencia de facto en Python, ofrece cliente TCP
síncrono y asíncrono, expone el manejo de excepciones Modbus de forma explícita —
necesario para el módulo `audit`, que debe registrar el resultado de cada consulta,
incluidos los fallos — y tiene documentación suficiente para un equipo sin
experiencia previa en el protocolo.

**Riesgo registrado.** `[VERIFICAR]` La API de `pymodbus` cambió de forma
significativa entre sus versiones mayores (2.x → 3.x). Gran parte de los ejemplos
que circulan en línea corresponden a la versión antigua y no funcionan tal cual.
**Acción:** fijar la versión exacta en `requirements.txt` al iniciar la Etapa 4 y
seguir únicamente la documentación oficial correspondiente a esa versión.

---

## D-14 — Cliente OPC UA: `asyncua`

**Estado:** Adoptada · **Fecha:** 2026-08-17 · **Responsable:** Equipo

**Contexto.** El adaptador `protocols/opcua/` debe conectarse a un servidor OPC UA,
recorrer su espacio de direcciones y recuperar metadatos de identidad. A diferencia
de Modbus TCP, OPC UA sí ofrece información descriptiva del dispositivo, lo que lo
convierte en la fuente principal del método `describe()` de la interfaz común.

**Alternativas consideradas.**

| Opción | Valoración |
|---|---|
| `python-opcua` | Predecesora de `asyncua`, del mismo autor; ya no recibe desarrollo activo. Descartada |
| `open62541` (C) con enlaces a Python | Implementación muy completa, pero añade compilación y complejidad de integración |
| Cliente sobre el SDK oficial de la OPC Foundation | Licenciamiento y costo no compatibles con un prototipo académico |

**Justificación.** `asyncua` es la continuación mantenida del proyecto anterior,
está escrita sobre `asyncio` — coherente con D-12 — e implementa las operaciones
que el motor necesita: conexión, exploración de nodos y lectura de atributos.

**Riesgo registrado.** `[VERIFICAR]` Queda pendiente confirmar en la Etapa 3 que el
testbed exponga efectivamente un extremo OPC UA. El documento de la Etapa 2 ya
anticipa que la ruta OPC UA puede requerir un componente intermedio. Si el escenario
de Factory I/O no lo ofrece de forma nativa, esta decisión no cambia, pero sí se
suma una decisión adicional sobre cómo se publica ese servidor.

---

## D-15 — Persistencia: SQLite en desarrollo

**Estado:** Adoptada · **Fecha:** 2026-08-17 · **Responsable:** Equipo

**Contexto.** El módulo `storage` debe persistir activos descubiertos, puntos de
dato, exposición observable e histórico de ejecuciones. El módulo `audit` genera
además la traza de cada consulta emitida, que la Etapa 6 conserva como evidencia.

**Alternativas consideradas.**

| Opción | Valoración |
|---|---|
| PostgreSQL | Más robusta y con mejor manejo de concurrencia, pero exige un servidor instalado y configurado en cada equipo y en el LabCIBE |
| Archivos JSON o CSV | Suficientes para guardar resultados, pero no permiten consultas relacionales ni comparar ejecuciones entre sí, que es justamente lo que pide la Etapa 6 |
| MongoDB | El modelo de datos del proyecto es relacional (activo → puntos → exposición); no hay ventaja real |

**Justificación.** SQLite no requiere servidor, el archivo de base de datos es
portable y puede acompañar la evidencia de las pruebas, y soporta SQL completo para
las consultas comparativas de la evaluación. Para un prototipo ejecutado por dos
personas contra un testbed controlado, la concurrencia no es un factor.

**Condición de revisión.** Si en la Etapa 5 aparece necesidad real de accesos
concurrentes o de ejecución multiusuario, se migra a PostgreSQL. Para que esa
migración sea viable, el esquema debe escribirse en SQL estándar y evitar
particularidades de SQLite.

---

## D-16 — API: FastAPI

**Estado:** Adoptada · **Fecha:** 2026-08-17 · **Responsable:** Equipo

**Alternativas consideradas.** Flask (más simple, sin documentación automática ni
validación de tipos), Django REST Framework (demasiada infraestructura para el
alcance), o prescindir de API y que el dashboard lea la base de datos directamente.

**Justificación.** FastAPI genera documentación interactiva de los endpoints de
forma automática, lo que constituye evidencia utilizable en la memoria y en la
demostración final sin trabajo adicional. Su validación por tipos se apoya en el
tipado ya presente por D-12.

**Advertencia de alcance registrada.** El documento de arquitectura señala que
`api/` y `frontend/` son las carpetas prescindibles si el tiempo escasea: el núcleo
evaluable es motor, persistencia y auditoría. Esta decisión no altera esa prioridad.

---

## D-17 — Dashboard: React

**Estado:** Adoptada · **Fecha:** 2026-08-17 · **Responsable:** Equipo

**Alternativas consideradas.** Streamlit o Dash (mucho más rápidos de construir y en
Python, pero con menos control de la interfaz), plantillas HTML servidas por FastAPI
(lo más simple), o React.

**Justificación.** La selección se alinea con el énfasis en Desarrollo de Sistemas
basados en Web de uno de los proponentes, lo que aporta coherencia entre el perfil
académico del equipo y el producto entregado.

**Riesgo registrado.** Es la decisión con peor relación esfuerzo/beneficio si el
cronograma se comprime, ya que Streamlit produciría una vista funcional en una
fracción del tiempo. Si al llegar a la Etapa 5 el dashboard no ha comenzado,
corresponde reevaluar esta decisión antes que sacrificar pruebas o evaluación.

---

## D-18 — Pruebas: `pytest`

**Estado:** Adoptada · **Fecha:** 2026-08-17 · **Responsable:** Equipo

**Justificación.** Estándar del ecosistema Python, con sintaxis más concisa que
`unittest` y buen manejo de datos de prueba mediante *fixtures*.

**Relación con la arquitectura.** La regla R2 del documento de arquitectura exige
que el módulo `identification` sea testeable sin red. Esa exigencia solo se puede
demostrar si existen pruebas unitarias que corran con datos sintéticos, sin
testbed. La separación `tests/unit/` y `tests/integration/` responde a eso.

---

## D-19 — Nombre del repositorio (resuelve P-06)

**Estado:** Adoptada · **Fecha:** 2026-08-17

**Decisión.** `motor-ot-multiprotocolo`

**Justificación.** Describe el sistema y no el trámite académico, por lo que el
repositorio sigue teniendo sentido como base reproducible para futuras
investigaciones del LabCIBE después de la defensa. Conserva el término
*multiprotocolo*, que es el elemento diferenciador del trabajo. Sin tildes, mayúsculas
ni espacios, para evitar problemas en URLs y en clonado entre sistemas operativos.

**Ubicación.** `https://github.com/Hezroiner/motor-ot-multiprotocolo`

---

## D-20 — Repositorio único (resuelve P-08)

**Estado:** Adoptada · **Fecha:** 2026-08-17

**Decisión.** Backend, frontend, documentación y evidencia conviven en un solo
repositorio.

**Justificación.** Las etiquetas de cierre de etapa (`v0.1-etapa3`, etc.) solo tienen
sentido si capturan el estado completo del proyecto en un momento dado. Con dos
repositorios, cada cierre exigiría etiquetas coordinadas y la trazabilidad
*objetivo específico → tarea de Jira → rama → commits → resultado* quedaría partida.
Esa trazabilidad es criterio de salida explícito de la Etapa 7.

---

## D-21 — Plataforma de alojamiento: GitHub (resuelve P-07)

**Estado:** Adoptada · **Fecha:** 2026-08-17

**Justificación.** Integración disponible con Jira para vincular ramas y commits a
las tareas, familiaridad del equipo, y disponibilidad de historial público de commits
fechados como evidencia de la secuencia real de trabajo.

**Pendientes asociados.** La visibilidad del repositorio y su licencia **no** quedan
resueltas por esta decisión (ver sección de pendientes).

---

## D-22 — Estrategia de ramas y etiquetas

**Estado:** Adoptada · **Fecha:** 2026-08-17

| Rama | Propósito |
|---|---|
| `main` | Únicamente estados entregables; cada entrega al comité lleva etiqueta |
| `develop` | Integración del trabajo del equipo |
| `feature/<clave-jira>-<descripcion>` | Una rama por tarea de Jira |
| `docs/<tema>` | Cambios exclusivos de documentación |

Etiquetas previstas: `v0.1-etapa3`, `v0.2-etapa4`, `v0.3-etapa5`.

**Justificación.** La convención de nombres ligada a Jira produce trazabilidad
automática entre tarea y código. Las etiquetas por etapa permiten mostrar el estado
del proyecto en cualquier punto de la línea de tiempo durante la defensa.

**Configuración asociada.** `develop` se establece como rama por defecto en GitHub,
de modo que los *pull requests* apunten allí y `main` conserve su carácter de rama
de entregas.

---

## Pendientes de decisión

| # | Pendiente | Depende de | Prioridad |
|---|---|---|---|
| P-01 | Licencia del código | Comité asesor / políticas de propiedad intelectual de la UNA | Alta |
| P-02 | Visibilidad del repositorio durante el desarrollo (público o privado) | Comité asesor | Alta |
| P-03 | Título definitivo para entregables formales (CTFG-DOC-06 vs. documento interno) | Comité asesor | Alta |
| P-09 | Clave del proyecto en Jira para la convención de ramas | Equipo | Alta |

**Sobre P-01 y P-02.** El anteproyecto establece que el proyecto debe respetar las
políticas institucionales sobre licencias de software, propiedad intelectual y
publicación de resultados. Mientras no haya confirmación, se recomienda mantener el
repositorio privado: pasar de privado a público es reversible; lo contrario, no.

**Sobre P-09.** La clave debe fijarse antes de crear la primera rama `feature/`.
Cambiarla a mitad del proyecto rompe la trazabilidad que la Etapa 7 exige demostrar.
