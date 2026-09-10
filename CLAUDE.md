# Agente Andretich — gestor multi-proyecto

Este proyecto es un agente de Claude Code cuyo rol es dar soporte transversal sobre el
ecosistema del cliente Andretich: gestión de tickets en Jira, consultas a la base de datos
SQL Server, y lectura/edición de los repos de código del cliente. El objetivo es tener
contexto completo del cliente para poder responder preguntas, investigar bugs y gestionar
tickets sin perder de vista cómo interactúan los distintos proyectos entre sí.

## Proyectos del cliente

| Proyecto | Rol |
|---|---|
| flok-front | Frontend de Flok |
| flok-back | Backend de Flok |
| bambuk | Frontend de Bambuk |
| bambuk-api | Backend/API de Bambuk |
| api-servicios | API de servicios compartida |

Las rutas a estos repos se configuran en `.claude/repos.config` (copiado desde
[.claude/repos.config.example](.claude/repos.config.example)), que **no se versiona** —
cada persona del equipo lo completa con sus propias rutas locales, porque cada uno clona
los repos en una estructura de carpetas distinta. **Antes de buscar o editar código de
un proyecto del cliente, resolver la ruta leyendo `.claude/repos.config`** (las variables
son `BAMBUK_FRONT`, `BAMBUK_API`, `CATALOGOYAPI`, `FLOK_FRONT`, `FLOK_BACK`,
`SERVICIOS_ANDRETICH`). **Si `.claude/repos.config` no existe, o si alguna variable
puntual está vacía, usar como valor por defecto la carpeta hermana correspondiente**
(mismo padre que `ia_andretich/`): `../bambuk` (`BAMBUK_FRONT`), `../bambuk-api`
(`BAMBUK_API`), `../flok-front` (`FLOK_FRONT`), `../flok-back` (`FLOK_BACK`),
`../api-servicios` (`SERVICIOS_ANDRETICH`, `CATALOGOYAPI`). Estos defaults son solo un
fallback de último recurso — si la ruta resuelta (por `repos.config` o por default) no
existe en disco, avisar al usuario en vez de asumir o inventar una ruta. **`CATALOGOYAPI`,
`FLOK_FRONT`, `FLOK_BACK` y `SERVICIOS_ANDRETICH` pueden apuntar a la misma ruta local**:
en algunos setups flok-front, flok-back y api-servicios viven juntos en un único repo
(`CatalogoYApi`) en vez de en carpetas separadas — `repos.config` es la fuente de verdad
sobre esto, no asumir un layout sin revisarlo primero. **`CatalogoYApi` es un monorepo
que adentro contiene el frontend de Flok, el backend de Flok y la API de servicios de
Andretich como subcarpetas** (no repos separados) — antes de buscar o editar algo de
`flok-front`, `flok-back` o `api-servicios` cuando `repos.config` apunta a
`CatalogoYApi`, explorar la raíz del repo (y revisar `ESTRUCTURA_COMPONENTES.md` /
`ORGANIZACION_COMPONENTES.md` si existen ahí) para identificar qué subcarpeta corresponde
a cada proyecto, en vez de asumir un nombre de carpeta.

El acceso de lectura/escritura a estos repos se hace con las herramientas nativas de
archivos (Read, Edit, Glob, Grep), no con un MCP de filesystem. Para que esas herramientas
puedan salir de la carpeta de `ia_andretich/` hace falta declarar las rutas de los repos
en `permissions.additionalDirectories`, dentro de `.claude/settings.json` (rutas relativas
por defecto, ej. `../bambuk`) y/o `.claude/settings.local.json` (rutas absolutas, para
cuando el layout local de una persona difiere del default — por ejemplo, si
`flok-back`/`api-servicios` viven dentro de un `CatalogoYApi` que no es carpeta hermana
directa). Si al leer o editar un archivo de alguno de estos repos la herramienta lo
rechaza por estar fuera del working directory, el problema casi siempre es que su ruta
falta en `additionalDirectories` — agregarla ahí, no volver a un MCP de filesystem. Ver
[README.md](README.md) para el setup.

## MCPs configurados

Ver [.mcp.json.example](.mcp.json.example) (plantilla) — el `.mcp.json` real de cada
usuario no está en el repo:

- **jira** — Atlassian Remote MCP oficial (OAuth). Da acceso a tickets/proyectos de Jira
  y Confluence. Primera vez que se use, Claude Code va a pedir autenticarse.
- **sqlserver** — MCP de base de datos (`@executeautomation/database-server`) apuntando a
  la SQL Server del cliente (database principal). El flag correcto de esta herramienta es
  `--sqlserver` (no `--mssql`), y el server/puerto van en flags separados (`--server`,
  `--port`), no como `host,puerto` estilo connection string ADO.NET.
- **sqlserver-carrito** — mismo tipo de MCP, misma instancia de SQL Server, pero apunta a
  la database `DB_CARRITO`. Usar este cuando la consulta sea sobre datos de carrito.

El acceso a los repos del cliente ya no pasa por un MCP de filesystem — se hace con las
herramientas nativas (Read, Edit, Glob, Grep) más `permissions.additionalDirectories` en
`.claude/settings.json` / `settings.local.json` (ver sección de arriba).

## Cómo trabajar

- Antes de tocar código de flok o bambuk, revisar primero si el ticket de Jira tiene
  contexto adicional (comentarios, adjuntos, tickets vinculados).
- Para bugs que cruzan proyectos (ej: un dato mal guardado que viene de bambuk-api y se
  muestra mal en el front de bambuk), usar el MCP de SQL Server para verificar el estado
  real de los datos antes de asumir dónde está el bug.
- Ver los skills en [.claude/skills/](.claude/skills/) para flujos específicos:
  - `jira-tickets`: cómo buscar, leer, actualizar y comentar tickets.
  - `multi-project-triage`: cómo diagnosticar un problema que puede originarse en
    cualquiera de los proyectos del cliente.
- No commitear nunca `.mcp.json` (tiene credenciales) — solo `.mcp.json.example`.
