# Etapa 2 — Sub-bloques 2.3 y 2.6
# Arquitectura de módulos y estructura del repositorio

**TFG:** Diseño de un motor multiprotocolo de Tecnología Operativa en un ambiente
simulado en el LabCIBE de la Universidad Nacional de Costa Rica.
**Estado:** Borrador para aprobación del equipo
**Fecha:** 2026-08-17

Marcado: `[DOC]` documentos oficiales · `[EQUIPO]` decisión del equipo ·
`[REC]` recomendación del asistente · `[VERIFICAR]` pendiente de comprobación.

---

## Parte I — Arquitectura de módulos (2.3)

### 1. Principios de diseño

Tres reglas gobiernan la separación de módulos. Las tres provienen de exigencias
del proyecto, no de preferencias de estilo.

**R1 — Los adaptadores de protocolo no conocen la lógica de identificación.**
Un módulo Modbus sabe hablar Modbus y nada más. Si la lógica de "esto parece un
PLC" vive dentro del cliente Modbus, agregar OPC UA obliga a duplicarla.

**R2 — La identificación no conoce las conexiones.** `[DOC]` La hoja de ruta lo
exige explícitamente en el sub-bloque 4.3: el módulo de clasificación debe ser
testeable sin red. Debe recibir estructuras de datos, no sockets.

**R3 — La normalización es la costura que hace real el "multiprotocolo".**
Sin una capa que convierta resultados heterogéneos a un modelo común, el motor
son dos programas distintos compartiendo carpeta.

### 2. Flujo de ejecución

```
   Configuración de objetivos
              │
              ▼
      ┌───────────────┐
      │  DISCOVERY    │  orquesta, controla ritmo y límites
      └───────┬───────┘
              │
      ┌───────┴────────┐
      ▼                ▼
┌───────────┐   ┌───────────┐
│ PROTOCOLS │   │ PROTOCOLS │      adaptadores independientes
│  modbus   │   │   opcua   │
└─────┬─────┘   └─────┬─────┘
      │               │
      └───────┬───────┘
              ▼
      ┌───────────────┐
      │ NORMALIZATION │  resultado crudo → modelo común
      └───────┬───────┘
              │
      ┌───────┴───────┬────────────────┐
      ▼               ▼                │
┌──────────────┐ ┌──────────┐          │
│IDENTIFICATION│ │ EXPOSURE │          │
│  rol + nivel │ │  E1..E5  │          │
└──────┬───────┘ └────┬─────┘          │
       └──────┬───────┘                │
              ▼                        │
      ┌───────────────┐                │
      │    STORAGE    │◄───────────────┘
      └───────┬───────┘      AUDIT registra toda consulta
              ▼                en cualquier punto del flujo
      ┌───────────────┐
      │      API      │ ──► DASHBOARD
      └───────────────┘
```

### 3. Responsabilidad de cada módulo

| Módulo | Responsabilidad | No le corresponde |
|---|---|---|
| `core` | Modelo de dominio: activo, punto de datos, exposición, enumeraciones de grados G0–G3 y niveles I0–I3 | Lógica de red, persistencia |
| `protocols` | Hablar cada protocolo industrial y devolver resultados crudos tipados | Interpretar qué significan los datos |
| `discovery` | Orquestar el recorrido de objetivos, aplicar límites de ritmo y tiempos de espera | Conocer detalles de cada protocolo |
| `normalization` | Traducir resultados crudos al modelo común de `core` | Inferir roles |
| `identification` | Aplicar reglas heurísticas y asignar rol y nivel de certeza | Conectarse a nada |
| `exposure` | Calcular las cinco dimensiones E1–E5 de exposición observable | Emitir juicios de riesgo |
| `storage` | Persistir activos, puntos, exposición e histórico de ejecuciones | Lógica de negocio |
| `audit` | Registrar cada consulta emitida: destino, función, momento, resultado | Todo lo demás |
| `api` | Exponer los resultados al dashboard | Ejecutar descubrimiento |
| `cli` | Punto de entrada de ejecución | Contener lógica |

### 4. Nota sobre el módulo `audit`

Es el más fácil de omitir y el que más peso tiene en la defensa. El anteproyecto
compromete evaluar la **seguridad del proceso de descubrimiento**, no solo su
eficacia. `[DOC]` Eso exige poder demostrar exactamente qué consultas se
enviaron, a qué ritmo y con qué resultado.

Además produce los datos crudos que la Etapa 6 necesita conservar como evidencia,
y respalda el criterio "ninguna consulta se ejecutó fuera del testbed autorizado".

Debe existir desde el primer día de la Etapa 4, no agregarse al final.

### 5. Interfaz común de protocolo `[REC]`

Todo adaptador de `protocols` implementa el mismo contrato, para que `discovery`
no distinga entre Modbus y OPC UA:

- `probe(objetivo)` → confirma existencia y devuelve el grado alcanzado (G0–G3)
- `collect(objetivo)` → devuelve puntos de datos observados
- `describe(objetivo)` → devuelve metadatos de identidad, si el protocolo los ofrece

`describe()` devolverá poco o nada en Modbus TCP y bastante en OPC UA. Esa
asimetría es esperada y está documentada en el sub-bloque 2.2; el modelo común
debe admitir campos vacíos sin tratarlos como error.

---

## Parte II — Estructura del repositorio (2.6)

### 6. Decisiones técnicas `[EQUIPO]`

| Decisión | Valor propuesto | Justificación |
|---|---|---|
| Lenguaje | Python 3.11+ | Bibliotecas maduras para ambos protocolos; el equipo tiene base de backend |
| Cliente Modbus | `pymodbus` | Referencia de facto, soporta cliente TCP síncrono y asíncrono |
| Cliente OPC UA | `asyncua` | Implementación mantenida de OPC UA en Python |
| Base de datos | SQLite en desarrollo, PostgreSQL si se requiere concurrencia | Sin servidor, archivo portable, adecuado a un prototipo académico |
| API | FastAPI | Documentación automática, útil como evidencia en la memoria |
| Dashboard | React | Decisión alineada con el énfasis de uno de los proponentes |
| Pruebas | `pytest` | Estándar del ecosistema |

Ninguna de estas tecnologías está exigida por el anteproyecto. Todas deben
registrarse como decisión del equipo con su justificación. `[DOC]`

### 7. Estructura de carpetas propuesta `[REC]`

```
motor-ot-multiprotocolo/
├── README.md
├── LICENSE
├── .gitignore
├── requirements.txt
├── pyproject.toml
│
├── backend/
│   ├── config/
│   │   ├── settings.py            # parámetros generales
│   │   └── targets.example.yaml   # objetivos del testbed (plantilla)
│   │
│   ├── core/
│   │   ├── models.py              # Activo, PuntoDato, Exposicion
│   │   ├── enums.py               # GradoDeteccion, NivelIdentificacion, Protocolo
│   │   └── exceptions.py
│   │
│   ├── protocols/
│   │   ├── base.py                # contrato común: probe / collect / describe
│   │   ├── modbus/
│   │   │   ├── client.py
│   │   │   ├── probes.py
│   │   │   └── mapping.py         # tablas Modbus → puntos de dato
│   │   └── opcua/
│   │       ├── client.py
│   │       ├── probes.py
│   │       └── mapping.py
│   │
│   ├── discovery/
│   │   ├── orchestrator.py
│   │   ├── targets.py
│   │   └── rate_limiter.py        # control de ritmo de consultas
│   │
│   ├── normalization/
│   │   └── normalizer.py
│   │
│   ├── identification/
│   │   ├── rules.py               # reglas heurísticas declarativas
│   │   └── classifier.py
│   │
│   ├── exposure/
│   │   └── evaluator.py           # dimensiones E1–E5
│   │
│   ├── storage/
│   │   ├── schema.sql
│   │   ├── repository.py
│   │   └── migrations/
│   │
│   ├── audit/
│   │   └── query_log.py           # traza de cada consulta emitida
│   │
│   ├── api/
│   │   ├── main.py
│   │   └── routers/
│   │
│   └── cli/
│       └── run_discovery.py
│
├── frontend/                      # dashboard React
│   ├── package.json
│   └── src/
│
├── tests/
│   ├── unit/
│   │   ├── test_identification.py # sin red, según regla R2
│   │   └── test_normalization.py
│   └── integration/
│       └── test_modbus_testbed.py
│
├── docs/
│   ├── etapa2/                    # definiciones, arquitectura, requisitos
│   ├── etapa3/                    # testbed, mapa de direcciones, reinicio
│   ├── diagrams/
│   └── decisiones.md              # bitácora de decisiones técnicas
│
└── evidence/
    ├── etapa3/                    # capturas de Modbus Poll y Factory I/O
    └── etapa5/                    # resultados de pruebas
```

### 8. Justificación de tres elecciones no obvias

**`docs/` y `evidence/` dentro del repositorio.** Un TFG se evalúa por su
trazabilidad, no solo por su código. Versionar la documentación junto al código
permite demostrar que el diseño precedió a la implementación, con historial de
commits fechado. Es el respaldo más fuerte posible de la secuencia real de trabajo.

**`identification/rules.py` separado de `classifier.py`.** Las reglas heurísticas
van a cambiar mucho durante las Etapas 4 y 5. Mantenerlas declarativas y aisladas
permite modificarlas sin tocar el motor de aplicación, y probarlas con datos
sintéticos sin conectarse al testbed.

**`targets.example.yaml` en vez de `targets.yaml`.** El archivo real, con las
direcciones del testbed, no se versiona. Evita publicar direcciones de la
infraestructura del laboratorio y obliga a que cada ejecución declare su alcance
de forma explícita — coherente con "ninguna consulta fuera del testbed autorizado".

### 9. Estrategia de ramas `[REC]`

| Rama | Propósito |
|---|---|
| `main` | Solo estados entregables. Cada entrega al comité se etiqueta con un tag |
| `develop` | Integración continua del trabajo del equipo |
| `feature/<clave-jira>-<descripcion>` | Una rama por tarea de Jira |
| `docs/<tema>` | Cambios exclusivos de documentación |

Ejemplo: `feature/MOT-14-cliente-modbus`

**Convención de nombres ligada a Jira.** No es burocracia: produce trazabilidad
automática entre objetivo específico → tarea de Jira → rama → commits → resultado.
Ese encadenamiento es un criterio de salida explícito de la Etapa 7.

**Etiquetas propuestas:** `v0.1-etapa3`, `v0.2-etapa4`, `v0.3-etapa5`, una por
cierre de etapa. Permiten mostrar el estado del proyecto en cualquier momento de
la línea de tiempo durante la defensa.

### 10. Advertencia de alcance

Esta estructura contempla motor, base de datos y dashboard, según la decisión
D-11 del equipo. El anteproyecto compromete un **prototipo funcional**, no una
plataforma. `[DOC]`

Si el tiempo se vuelve un problema, `api/` y `frontend/` son las carpetas
prescindibles: el motor, la persistencia y la auditoría son el núcleo evaluable.
Conviene tenerlo presente ahora y no cuando queden tres semanas.

---

## 11. Pendientes de este documento

| # | Pendiente | Responsable |
|---|---|---|
| P-06 | Nombre definitivo del repositorio | Equipo |
| P-07 | Plataforma: GitHub, GitLab u otra | Equipo |
| P-08 | ¿Repositorio único o separado para frontend? | Equipo |
| P-09 | Confirmar clave de proyecto en Jira para la convención de ramas | Equipo |
