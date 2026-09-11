# ia_andretich

Agente de Claude Code para gestión multi-proyecto del cliente Andretich (Flok, Bambuk,
API de servicios) con acceso a Jira y SQL Server.

## Setup

1. Copiar `.mcp.json.example` a `.mcp.json`:

   ```
   cp .mcp.json.example .mcp.json
   ```

2. Completar en `.mcp.json` las credenciales reales de SQL Server (`server`, `port`,
   `database`, `user`, `password` en los `args` de los servidores `sqlserver` y
   `sqlserver-carrito`).

   `.mcp.json` está en `.gitignore` y nunca se sube al repo (tiene credenciales). Solo se
   versiona `.mcp.json.example` como plantilla.

3. Copiar `.claude/repos.config.example` a `.claude/repos.config`:

   ```
   cp .claude/repos.config.example .claude/repos.config
   ```

   y completar cada variable con la ruta local real del repo correspondiente
   (`BAMBUK_FRONT`, `BAMBUK_API`, `CATALOGOYAPI`, `FLOK_FRONT`, `FLOK_BACK`,
   `SERVICIOS_ANDRETICH`). Ojo: `CATALOGOYAPI`, `FLOK_FRONT`, `FLOK_BACK` y
   `SERVICIOS_ANDRETICH` van a la **misma ruta** si en tu setup flok-front, flok-back y
   api-servicios están juntos en un solo repo (`CatalogoYApi`) en vez de en carpetas
   separadas. Tampoco se versiona.

   El acceso a estos repos lo dan las herramientas nativas de archivos (Read, Edit, Glob,
   Grep) — no hay ningún MCP de filesystem que configurar. Si clonaste los repos como
   carpetas hermanas de `ia_andretich/` (`../bambuk`, `../bambuk-api`, `../flok-front`,
   `../flok-back`, `../api-servicios`), no necesitás hacer nada más: ese caso ya está
   cubierto por el `.claude/settings.json` del repo.

   Si tu layout es distinto (por ejemplo, tenés todo clonado en otro lado, o
   `flok-back`/`api-servicios` viven dentro de un `CatalogoYApi` que no es hermano
   directo), copiá la plantilla y completá tus rutas absolutas:

   ```
   cp .claude/settings.local.json.example .claude/settings.local.json
   ```

   Editá `additionalDirectories` con las rutas que necesites (borrá las que no apliquen
   a tu setup) y dejá `enabledMcpjsonServers` como está — sin ese campo, Claude Code te
   va a pedir aprobar manualmente cada MCP (`jira`, `sqlserver`, `sqlserver-carrito`) la
   primera vez que se use en la sesión.

   `settings.local.json` está gitignoreado (nunca se sube, es config personal). Sin la
   ruta correspondiente en `additionalDirectories`, cualquier intento de leer o editar
   código fuera de `ia_andretich/` va a fallar aunque el path exista en disco — este es
   el paso que reemplaza al viejo MCP `clientes-fs`.

4. Abrir Claude Code en esta carpeta. Al usar por primera vez el MCP `jira`, se va a
   pedir autenticación OAuth contra Atlassian (Jira + Confluence).

5. Verificar que los MCPs levantan correctamente (`claude mcp list` o `/mcp` en una
   sesión interactiva).

Ver [CLAUDE.md](CLAUDE.md) para el contexto completo y [.claude/skills/](.claude/skills/)
para los flujos de trabajo (tickets Jira, triage multi-proyecto).
