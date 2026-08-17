# Motor OT multiprotocolo

Motor de descubrimiento de activos, identificación de dispositivos y evaluación
de exposición observable para entornos de Tecnología Operativa (OT), validado en
un ambiente simulado del Laboratorio de Investigación en Ciberseguridad (LabCIBE)
de la Universidad Nacional de Costa Rica.

> **Estado:** Etapa 2 — Diseño del sistema. No existe prototipo funcional todavía.
> Este repositorio contiene, por ahora, documentación de diseño.

---

## 1. Contexto académico

Trabajo Final de Graduación (TFG) para optar al grado de Licenciatura en
Informática, Escuela de Informática, Facultad de Ciencias Exactas y Naturales,
Universidad Nacional de Costa Rica.

| | |
|---|---|
| Modalidad | Proyecto |
| Énfasis | Sistemas de Información · Desarrollo de Sistemas basados en Web |
| Entorno de validación | LabCIBE, Universidad Nacional de Costa Rica |

**Proponentes**

- Adriana de los Ángeles Morera Elizondo
- Hezron Araya Castañeda

**Comité asesor**

| Rol | Persona |
|---|---|
| Tutor | Alex Daniel Villegas Carranza |
| Lector interno | Lawrence Fowks Peña |
| Lector externo | Doanson Torres Carrillo |
| Profesor investigador | Johnny Villalobos M. |

> **Nota sobre el título.** El anteproyecto oficial (CTFG-DOC-06) y el documento
> de trabajo interno emplean títulos distintos. La diferencia está registrada en
> `docs/decisiones.md` y pendiente de confirmación con el comité asesor. Hasta
> entonces, los entregables formales utilizan el título del anteproyecto oficial:
> *"Diseño de un motor multiprotocolo de Tecnología Operativa en un ambiente
> simulado en el LabCIBE de la Universidad Nacional de Costa Rica."*

---

## 2. Problema que aborda

Los entornos OT integran dispositivos de distintos fabricantes que se comunican
mediante protocolos industriales especializados. Esa heterogeneidad dificulta
saber qué activos existen en la red, qué función cumplen y qué servicios exponen.

Las técnicas de descubrimiento habituales en redes de TI no siempre son
apropiadas: pueden introducir tráfico o latencia sobre dispositivos diseñados
para operar de forma continua.

Este proyecto desarrolla un motor que combina **observación pasiva** y **consultas
activas controladas** sobre **Modbus TCP** y **OPC UA**, registrando además el
impacto operativo del propio proceso de descubrimiento.

---

## 3. Alcance

### Incluye

- Descubrimiento e identificación de activos sobre Modbus TCP y OPC UA.
- Registro de exposición observable de servicios e interfaces.
- Normalización de resultados de ambos protocolos a un modelo común.
- Auditoría de cada consulta emitida (destino, función, momento, resultado).
- Validación en entorno simulado con métricas de precisión, falsos positivos,
  falsos negativos, completitud de identificación e impacto operativo.

### No incluye

- Validación sobre infraestructura industrial real en producción.
- Protocolos industriales distintos de Modbus TCP y OPC UA.
- Explotación de vulnerabilidades, intrusión o pruebas destructivas.
- Funciones de SIEM, IDS, gestión de vulnerabilidades o respuesta a incidentes.
- Sustitución de plataformas comerciales de monitoreo continuo.

El compromiso del anteproyecto es un **prototipo funcional**, no una plataforma.

---

## 4. Advertencia de uso

Este software emite consultas activas hacia dispositivos que hablan protocolos
industriales. Su ejecución está restringida al testbed autorizado del LabCIBE.

- El archivo `backend/config/targets.yaml`, que declara los objetivos reales, **no
  se versiona**. Cada ejecución debe declarar su alcance de forma explícita a
  partir de `targets.example.yaml`.
- No debe ejecutarse contra redes de producción, ni contra equipos sobre los que
  no se cuente con autorización escrita.

---

## 5. Arquitectura

El motor se organiza en módulos independientes bajo tres reglas de diseño:
los adaptadores de protocolo no conocen la lógica de identificación; la
identificación no conoce las conexiones; la normalización es la capa que hace
real el carácter multiprotocolo.

```
Configuración de objetivos → DISCOVERY → PROTOCOLS (modbus | opcua)
    → NORMALIZATION → IDENTIFICATION + EXPOSURE → STORAGE → API → DASHBOARD
                                   AUDIT registra toda consulta
```

| Módulo | Responsabilidad |
|---|---|
| `core` | Modelo de dominio y enumeraciones |
| `protocols` | Hablar cada protocolo y devolver resultados crudos |
| `discovery` | Orquestar objetivos, ritmo y tiempos de espera |
| `normalization` | Traducir resultados crudos al modelo común |
| `identification` | Reglas heurísticas de rol y nivel de certeza |
| `exposure` | Dimensiones E1–E5 de exposición observable |
| `storage` | Persistencia de activos, puntos e histórico |
| `audit` | Traza de cada consulta emitida |
| `api` · `cli` | Puntos de entrada |

Detalle completo en [`docs/etapa2/2.3_2.6_arquitectura_y_repositorio.md`](docs/etapa2/2.3_2.6_arquitectura_y_repositorio.md).

---

## 6. Tecnologías

Ninguna de estas tecnologías está exigida por el anteproyecto. Todas son
decisión del equipo y están justificadas en `docs/decisiones.md`.

| Componente | Selección | Estado |
|---|---|---|
| Lenguaje | Python 3.11+ | Propuesta |
| Cliente Modbus | `pymodbus` | Propuesta |
| Cliente OPC UA | `asyncua` | Propuesta |
| Persistencia | SQLite (desarrollo) | Propuesta |
| API | FastAPI | Propuesta |
| Dashboard | React | Propuesta |
| Pruebas | `pytest` | Propuesta |
| Simulación | Factory I/O | Definido por el anteproyecto |

---

## 7. Estructura del repositorio

```
motor-ot-multiprotocolo/
├── backend/          # motor: core, protocols, discovery, normalization,
│                     # identification, exposure, storage, audit, api, cli
├── frontend/         # dashboard
├── tests/            # unit (sin red) e integration (contra el testbed)
├── docs/             # diseño, testbed, diagramas, bitácora de decisiones
└── evidence/         # capturas y resultados de pruebas por etapa
```

Las carpetas de `backend/` se incorporan conforme avanza la Etapa 4. La
documentación se versiona junto al código de forma deliberada: el historial de
commits fechado es la evidencia de que el diseño precedió a la implementación.

---

## 8. Instalación y ejecución

Pendiente. Se documentará al iniciar la Etapa 4 (desarrollo e implementación).

---

## 9. Ramas y etiquetas

| Rama | Propósito |
|---|---|
| `main` | Únicamente estados entregables; cada entrega al comité lleva su etiqueta |
| `develop` | Integración del trabajo del equipo |
| `feature/<clave-jira>-<descripcion>` | Una rama por tarea de Jira |
| `docs/<tema>` | Cambios exclusivos de documentación |

Etiquetas previstas: `v0.1-etapa3`, `v0.2-etapa4`, `v0.3-etapa5`.

---

## 10. Licencia

**Pendiente de confirmación con el comité asesor.** El proyecto debe respetar las
políticas institucionales de la Universidad Nacional en materia de propiedad
intelectual y publicación de resultados, por lo que la licencia y la visibilidad
del repositorio no se definen unilateralmente.

---

## 11. Referencias principales

- National Institute of Standards and Technology. (2015). *Guide to Industrial
  Control Systems (ICS) Security* (NIST SP 800-82 Rev. 2).
- International Society of Automation. (2021). *ISA/IEC 62443 series of standards*.
- Modbus Organization. (2024). *Modbus application protocol specification V1.1b3*.
- OPC Foundation. (2023). *OPC Unified Architecture specification*.
- Real Games. (2024). *Factory I/O documentation*.
