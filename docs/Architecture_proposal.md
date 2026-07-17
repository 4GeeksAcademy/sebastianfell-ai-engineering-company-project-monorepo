# Propuesta de Arquitectura de Backend — HealthCore

## Contexto

HealthCore opera 12 clínicas ambulatorias en EE. UU. y Reino Unido. El backend registrará y analizará incidentes de pacientes y alimentará el dashboard de experiencia. Dos condiciones marcan todo: manejamos **PHI bajo HIPAA y datos personales bajo UK GDPR** (cumplimiento, no preferencia), y el **frontend ya existe como sistema aparte**.

## Patrón: arquitectura en capas por dominio

Propongo una **única aplicación de backend organizada en capas y separada internamente por dominios de negocio**. Descarto la alternativa de dividir el sistema en varios servicios independientes: resuelve problemas de escala que HealthCore no tiene hoy y agrega complejidad de red impagable para un equipo pequeño. La separación por dominios deja abierta la puerta a extraer una parte si aparece la necesidad real.

Las capas —rutas, servicios (lógica de negocio), acceso a datos y modelos/esquemas— sacan las reglas de negocio de los endpoints para poder testearlas sin levantar HTTP.

## Frontend y backend separados

El backend expone una **API REST con JSON**; no renderiza vistas. Consecuencias explícitas: **autenticación por tokens**, **CORS** con lista explícita de orígenes (dos países, varios frontends — nada de `*`), **secretos en variables de entorno** fuera del repo, y el **contrato OpenAPI** como frontera estable entre equipos.

## Estructura de carpetas

Investigué la convención estándar de proyectos FastAPI (agrupar por dominio, separar `router` / `service` / `models` / `schemas`, aislar la configuración en un módulo `core`) y la adopto:

```
app/
├── main.py            # montaje de routers, CORS
├── core/              # config, security, logging (filtro anti-PHI)
├── domains/
│   ├── incidents/     # router.py, service.py, data.py, models.py, schemas.py
│   ├── clinics/       # las 12 clínicas, país, ubicación
│   ├── patients/      # dominio más sensible (PHI/GDPR)
│   └── analytics/     # agregados para el dashboard
└── db/                # session, base
```

Cada dominio es una carpeta autocontenida: quien toque incidentes sabe dónde ir.

## Rutas por dominio

Versionadas desde el inicio (`/api/v1`) para evolucionar sin romper el frontend:

| Dominio | Prefijo |
|---|---|
| Incidents | `/api/v1/incidents` |
| Clinics | `/api/v1/clinics` |
| Patients | `/api/v1/patients` |
| Analytics | `/api/v1/analytics` |

`analytics` devuelve solo agregados — nunca filas individuales.

## Decisiones técnicas

- **FastAPI** — documentación OpenAPI automática (= contrato) y validación nativa.
- **Pydantic** — esquemas de entrada/salida **separados de los modelos de datos**. Un esquema de salida que no incluye `patient_id` garantiza que nunca salga por la API, aunque esté guardado.
- **Persistencia** — la capa de acceso a datos aísla al sistema del motor de base de datos, así que ese motor se puede decidir después **sin tocar la arquitectura**: es uno de los beneficios del patrón por capas.

## Riesgos

1. **Exposición de PHI** (crítico): un `patient_id` en log, error o respuesta invalida la salida. Mitigación: separación modelo/esquema, filtro de logging, tests que fallen si aparece un campo sensible.
2. **Validación país/clínica dispersa**: la regla debe vivir una sola vez, en servicios.
3. **Lógica de negocio en los routers**: routers delgados, servicios gordos.
4. **CORS/auth definidos tarde**: fijarlos en el primer sprint.