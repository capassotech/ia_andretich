# Bambuk — arquitectura y guía de desarrollo

Contexto para desarrollar funcionalidades en **bambuk** (frontend) y **bambuk-api**
(backend). Ambos repos declaran explícitamente **Clean Architecture** en sus propios
`README.md` — este documento resume esa arquitectura, los patrones reales usados en el
código, y cómo agregar una feature nueva sin romper las convenciones existentes.

Antes de tocar código, resolver las rutas locales vía `.claude/repos.config`
(`BAMBUK_FRONT`, `BAMBUK_API`) — ver [../README.md](../README.md).

---

## bambuk (frontend)

**Stack**: React 19 + React Router 7, TypeScript, Vite, Tailwind CSS v4 + shadcn/ui,
sin Redux/Zustand (estado con Context API a mano), sin axios/react-query (fetch nativo
envuelto en un cliente propio). PWA (vite-plugin-pwa), SignalR para notificaciones
en tiempo real.

### Capas (Clean Architecture adaptada a frontend)

El repo organiza cada dominio de negocio (`compatibilidades`, `productos`, `usuarios`,
`marcas`, etc.) en **tres capas paralelas**, replicadas en tres carpetas distintas:

1. **Infraestructura de datos** — `src/api/{dominio}/`
   Funciones que llaman al backend usando `apiFetch` (`src/api/client.ts`), que inyecta
   el Bearer token y hace refresh automático en 401. Los tipos de request/response viven
   en `src/api/dtos/{dominio}.dto.ts`. `src/api/url.ts` resuelve la base URL.

2. **Aplicación / casos de uso** — `src/service/{dominio}/{dominio}Service.ts`
   Orquesta las llamadas de `api/*`, traduce errores técnicos (`ApiHttpError`, `TypeError`)
   a una excepción de negocio propia por dominio (ej. `CompatibilidadServiceError`) con
   mensajes en español pensados para mostrar en UI.

3. **Presentación** — `src/pages/{dominio}/` (vistas por ruta) y
   `src/components/{dominio}/` (componentes de UI del dominio: sheets, forms, tablas).
   **No debe llamar `fetch` directo** — siempre pasa por `service/*`.

Estado de UI compartido va en `src/context/` (trío por contexto: `*-context.ts` puro,
`*.types.ts`, `Provider.tsx`, `use*.ts`). Hooks transversales en `src/hooks/`. Mappers
DTO→modelo de UI unificado (útil cuando hay varias fuentes externas tipo Estrada/Exin)
en `src/data/`. Helpers de parsing de rutas/querystring (sin JSX) en `src/routes/`.

### Cómo agregar una feature nueva (ej. tomando `compatibilidades` como referencia)

1. DTOs en `src/api/dtos/{dominio}.dto.ts`.
2. Funciones de API en `src/api/{dominio}/{dominio}.ts` usando `apiFetch` + `buildApiUrl`.
3. Service en `src/service/{dominio}/{dominio}Service.ts`, con su clase de error propia.
4. Si hay navegación con tabs/filtros por querystring: helper en `src/routes/`.
5. Página(s) en `src/pages/{dominio}/`.
6. Componentes específicos en `src/components/{dominio}/` (+ `types.ts`/`constants.ts`
   locales al dominio si hacen falta).
7. Skeleton de loading en `src/components/skeletons/` si aplica.
8. Registrar la ruta en `src/App.tsx` si es una ruta top-level nueva (lazy con
   `React.lazy` + `Suspense` para rutas secundarias).

No hay scaffolding automatizado: se replica copiando la estructura de un dominio
existente similar.

### Convenciones

- Componentes: `PascalCase.tsx`. Servicios: `xxxService.ts`. DTOs: `xxx.dto.ts` en
  español (dominio de negocio en español, términos técnicos en inglés).
- Alias de import `@/*` → `./src/*` (configurado en `tsconfig.json`, `vite.config.ts` y
  `components.json`).
- Estilos con `cn()` (`src/lib/utils.ts`, wrapper de `clsx` + `tailwind-merge`) y
  variantes con `class-variance-authority`.

### Gaps a tener en cuenta

- **No hay testing automatizado** (sin Jest/Vitest, sin archivos `*.test.*`). No asumir
  que existe cobertura — si se agrega una feature crítica, señalar esto en vez de asumir
  que hay tests que la protegen.
- No hay Prettier configurado, solo ESLint (flat config con presets recomendados de TS
  y React Hooks/Refresh).

### Correr en local

```bash
npm install
npm run dev      # Vite dev server, hot reload
npm run build    # tsc -b && vite build
npm run lint      # eslint .
npm run preview   # sirve el build
```

Backend por defecto según rama (`main` → prod, resto → QA), override con
`VITE_API_BASE_URL` en `.env`. No hay script `test`.

---

## bambuk-api (backend)

**Stack**: .NET 10 / ASP.NET Core Web API, Entity Framework Core 10 + SQL Server,
MediatR (CQRS), AutoMapper, JWT Bearer, Serilog, Swagger en `/docs`, SignalR, OData.

### Capas (Clean Architecture, proyectos separados en la solución)

```
bambu-api/            → Presentación (Controllers, Program.cs = composición raíz)
Application/           → Casos de uso: CQRS con MediatR (Commands/Queries + Handlers)
Domain/                → Entidades EF, DTOs, enums, perfil AutoMapper central
Infrastructure/        → EF Core (DbContext, migraciones, config), servicios externos, DI
```

Dependencias: `Domain` no depende de nada; `Application` depende solo de `Domain` y
define interfaces (ej. `IAppDbContext`) que `Infrastructure` implementa;
`Infrastructure` depende de `Application` + `Domain`; `bambu-api` (presentación)
depende de ambos y despacha todo vía `IMediator`.

No hay capa de "Repository" genérica: los handlers inyectan `IAppDbContext` y acceden a
los `DbSet` directamente (excepto algunos repositorios específicos de búsqueda en
`Infrastructure/Services`, ej. `ICatalogoBusquedaRepository`).

### Patrón CQRS — estructura de una operación

Cada entidad de negocio tiene su carpeta bajo `Application/UseCases/V{n}/{Entidad}Operation/`,
con subcarpetas `Commands/` y `Queries/`. Cada acción vive en su propia carpeta:

```
Application/UseCases/V1/MarcaOperation/Commands/CrearMarca/
  CrearMarcaCommand.cs         → record : IRequest<TResponse>
  CrearMarcaCommandHandler.cs  → IRequestHandler<CrearMarcaCommand, TResponse>
```

Flujo completo de un endpoint (ejemplo real `POST /api/v1/marca`):

1. **Controller** (`bambu-api/Controllers/V1/MarcaController.cs`) recibe el DTO,
   `mediator.Send(new CrearMarcaCommand(request))`.
2. **Request DTO** (`Domain/Dtos/Marcas/CrearEditarMarcaRequestDto.cs`) — validado con
   **DataAnnotations** (`[Required]`, `[MaxLength]`), no FluentValidation.
3. **Command** (`Application/.../CrearMarcaCommand.cs`).
4. **Handler** — inyecta `IAppDbContext`, arma la entidad, `context.Set.Add(...)`,
   `SaveChangesAsync`.
5. **Entidad** (`Domain/Entities/Productos/MarcaProducto.cs`) — POCO simple.
6. **DbContext** (`Infrastructure/Persistence/AppDbContext.cs`) expone el `DbSet`;
   configuración fina de EF (si aplica) en `Infrastructure/Persistence/Configurations/`
   (patrón `IEntityTypeConfiguration<T>`).
7. **DI** (`Infrastructure/Bootstrap/DependencyInjection.cs`) registra `AppDbContext`,
   `IAppDbContext` y escanea el assembly de `Application` para MediatR.
8. Controller responde `201 Created` / `NotFound` / `NoContent` según lo que devuelva
   el handler (no hay middleware de mapeo de excepciones de dominio → HTTP).

### Cómo agregar un endpoint nuevo (orden sugerido)

1. Entidad en `Domain/Entities/{Modulo}/` (si es tabla nueva).
2. DTOs de request/response en `Domain/Dtos/{Modulo}/`.
3. Command o Query en `Application/UseCases/V{n}/{Entidad}Operation/Commands|Queries/{Accion}/`.
4. Handler en la misma carpeta (inyecta `IAppDbContext` u otras interfaces de
   `Application/Common/Interfaces/`).
5. Mapeo en `Domain/Common/DomainAutoMapping.cs` si hace falta AutoMapper.
6. Si hay `DbSet` nuevo: exponerlo en `AppDbContext.cs` + `IEntityTypeConfiguration` en
   `Infrastructure/Persistence/Configurations/` si necesita config fina.
7. Migración EF: `dotnet ef migrations add NombreMigracion --project Infrastructure --startup-project bambu-api`.
8. Endpoint en `bambu-api/Controllers/V{n}/{Entidad}Controller.cs`, inyectando
   `IMediator`.
9. Si hace falta un servicio externo nuevo: interfaz en `Application/Common/Interfaces/`,
   implementación en `Infrastructure/Services/{Modulo}/`, registro en
   `Infrastructure/Bootstrap/DependencyInjection.cs`.

### Convenciones

- Controllers: `{Entidad}Controller.cs`, ruta `api/v{n}/{entidad}`.
- Commands/Queries: `{Accion}Command.cs` + `{Accion}CommandHandler.cs` (o Query/QueryHandler).
- DTOs: `Domain/Dtos/{Modulo}/`, sufijo `Dto` o `RequestDto`.
- Entidades: `Domain/Entities/{Modulo}/`, singular sin sufijo.
- Servicios: prefijo `I` + sufijo `Service`/`Repository`, implementación en
  `Infrastructure/Services/{Modulo}/`.
- Dominio de negocio en español (Marca, Cuenta, Sincronizacion), términos técnicos en
  inglés (Command, Query, Dto, Handler).

### Gotchas importantes

- **FluentValidation está en las dependencias pero no se usa realmente** —
  `Application/Common/Behaviours/ValidationBehaviour.cs` está vacío y no registrado en
  DI. La validación real de request es con DataAnnotations. No sugerir "agregar un
  validator de FluentValidation" sin antes confirmar si el equipo piensa terminar de
  conectarlo.
- Manejo de errores global es básico (`UseExceptionHandler` devuelve un 500 genérico).
  Los handlers comunican fallos devolviendo `null`/`bool`, y el controller traduce a
  `NotFound`/`NoContent` — no hay excepciones de dominio tipadas mapeadas a HTTP.
- El esquema de base de datos no vive 100% en EF Migrations: hay scripts SQL manuales en
  `scripts/`, `scripts-sql/`, `migration.sql`, `migration_prod.sql`. Antes de asumir que
  una migración EF alcanza para reflejar un cambio de schema, revisar si hace falta un
  script manual también.
- `appsettings.json` tiene credenciales reales versionadas (connection strings de
  producción, SMTP, API keys externas) — tenerlo presente al compartir contenido de ese
  archivo o al armar ejemplos.
- Proyecto de tests (`bambu.Application.Tests`, xUnit + Moq + FluentAssertions) existe
  pero está vacío — no hay cobertura real hoy.
- Selección de connection string es dinámica vía la clave `ActiveConnection` en
  appsettings (`BambuDesa`, `BambuProd`, `BambuLocal`).

### Correr y testear en local

```bash
dotnet restore bambu-api.sln
dotnet build bambu-api.sln
dotnet run --project bambu-api        # https://localhost:7070, Swagger en /docs
dotnet ef database update --project Infrastructure --startup-project bambu-api
dotnet ef migrations add NombreMigracion --project Infrastructure --startup-project bambu-api
dotnet test bambu.Application.Tests/bambu.Application.Tests.csproj
```

`appsettings.Development.json` usa `ActiveConnection: BambuLocal` (SQL Server local,
ej. Docker en `localhost,1433`) y `DisableAuth: true` para saltear JWT en desarrollo.

---

## Cómo interactúan bambuk y bambuk-api

- El frontend resuelve la URL base del backend según la rama de deploy (`VITE_API_BASE_URL`,
  ver `vite.config.ts` en bambuk) — para debuggear un flujo end-to-end conviene confirmar
  contra qué ambiente (prod/QA/local) está apuntando el front antes de buscar el bug en
  el backend.
- El contrato de datos entre ambos son los DTOs: `src/api/dtos/*.dto.ts` en bambuk debe
  reflejar los `Domain/Dtos/{Modulo}/*Dto.cs` de bambuk-api — si un campo cambia de un
  lado, revisar el otro.
- Autenticación: JWT emitido por bambuk-api, guardado por el front en `localStorage`
  (`src/auth/authToken.ts`), refrescado automáticamente por `apiFetch` en 401.
- Notificaciones en tiempo real van por SignalR: hub en
  `Infrastructure/Hubs/NotificacionHub.cs` (backend) ↔ `NotificacionesProvider.tsx` /
  `NotificacionesPanel.tsx` (frontend).
