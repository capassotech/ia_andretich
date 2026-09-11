# CatalogoYApi — arquitectura y guía de desarrollo (Flok + api-servicios)

`CatalogoYApi` es un **monorepo** que en muchos setups locales cubre tres proyectos del
cliente a la vez: **flok-front**, **flok-back** y **api-servicios** (ver tabla de
[../CLAUDE.md](../CLAUDE.md)). No son tres repos separados sino subcarpetas del mismo
repo `.sln` (`Catalogo-API.sln`). Antes de tocar código, resolver la ruta local vía
`.claude/repos.config` (`CATALOGOYAPI`/`FLOK_FRONT`/`FLOK_BACK`/`SERVICIOS_ANDRETICH`,
normalmente todas apuntando a la misma carpeta) — ver [../README.md](../README.md).

**Mapeo de carpetas → proyecto del cliente:**

| Proyecto del cliente | Carpeta real dentro de `CatalogoYApi/` |
|---|---|
| flok-front | `CatalogoWeb.Host/ClientApp/catalogo/` (el React vive ahí, no en `ClientApp/` directo) |
| flok-back | `CatalogoWeb.Host/` (+ `ProyectoWeb.Services`, `ProyectoWeb.Services.Contract`, `MigracionInformacion.Services*`, referenciados por `ProjectReference`) |
| api-servicios | `ServiciosAndretichAPI/` |

El resto de las carpetas del monorepo (`ApiCA`, `ProyectoApiPrediccion*`, `WebApi`,
`ProyectoWeb1`, `ProyectoECommerceTienda`, `WebApplicationWS`, `WebApplicationServP`,
`ProyectoTest`) **no están relacionadas con Flok/api-servicios** — son proyectos
satélite/legacy no referenciados por `CatalogoWeb.Host` ni `ServiciosAndretichAPI`. No
asumir que hay que tocarlos salvo que un ticket lo indique explícitamente.

**Importante — este código es mucho más legacy que bambuk/bambuk-api.** bambuk-api sigue
Clean Architecture estricta con CQRS/MediatR de punta a punta; acá conviven capas
modernas puntuales (el módulo `Auth` de `ServiciosAndretichAPI`) con una base N-Layer
clásica sin DI, sin ORM, con SQL embebido a mano. No copiar patrones de bambuk-api acá
sin verificar primero qué convención sigue el código vecino inmediato — mezclar
paradigmas empeora la deuda existente.

---

## flok-front (`CatalogoWeb.Host/ClientApp/catalogo/`)

**Stack**: React 18 + JavaScript puro (sin TypeScript, sin React Router), Vite 5 con
**17 entry points** distintos, Tailwind CSS 4 + CSS por componente + Bootstrap heredado.
Sin testing, sin ESLint/Prettier.

### Arquitectura: multi-widget sobre vistas Razor, no una SPA

No hay un único árbol de rutas cliente. Es un conjunto de **"widgets" React
independientes**, cada uno montado con su propio `root.render()` sobre un
`<div id="...">` que una vista Razor de `CatalogoWeb.Host` (MVC clásico) inyecta. La
navegación entre pantallas es server-side (Razor), no client-side. `vite.config.js`
declara un entry point por pantalla: `main`, `ProductoDetalle`, `Navbar`,
`CuentaCorriente`, `Pedidos`, `Garantias`, `clientes-main`, `usuarios-main`,
`transportes-main`, `logistica-pedidos-main`, `configuracion-*-main`, `login-main`,
`configurar-password-main`.

Capas informales dentro de `src/`:
- `components/` — presentación, organizada por dominio (`layout/`, `search/`,
  `filters/`, `products/`, `brands/`, `tables/`, `ui/`, más `cart/` que es el módulo más
  grande del proyecto, `clientes/`, `usuarios/`, `pedidos/`, etc. — ver
  `ESTRUCTURA_COMPONENTES.md`/`ORGANIZACION_COMPONENTES.md` en la raíz del monorepo).
- `services/` — funciones `fetch` por dominio, sin capa de mapeo a modelos (devuelven
  el JSON crudo del backend).
- `hooks/` — estado y fetch de datos por feature (`useCatalogProducts.js`,
  `useCatalogFilters.js`).
- `utils/` — funciones puras.
- Pero además hay **componentes "página" gigantes sueltos en la raíz de `src/`**
  (`App.jsx` ~758 líneas, `ProductoDetalle.jsx` ~1068 líneas, y dentro de `cart/`
  archivos como `CartDataShipment.jsx` ~1102 líneas) — no siguen el patrón de carpetas
  por feature, señal de organización orgánica/legacy en evolución, no capas estrictas.

### Comunicación entre widgets

No hay Context ni Redux/Zustand: el estado es local por widget (`useState`/`useEffect`).
Los widgets se sincronizan entre sí leyendo el DOM directamente (ej. `App.jsx` lee
`document.getElementById('searchInput')` para reaccionar al Navbar, que es otro root de
React separado) o vía `localStorage`. Tenerlo presente al debuggear: un bug de estado
puede no estar en el componente que lo muestra sino en otro widget que lo escribe al DOM
o a `localStorage`.

### Conexión al backend

Sin variables de entorno (`.env`)/`import.meta.env`: la config runtime, incluida la
"base URL" de API, se inyecta desde el servidor vía `window.REACT_APP_CONFIG`, seteado
inline en cada vista Razor (`CatalogoWeb.Host/Views/Catalogo/Index.cshtml`, etc.). La
mayoría de los `services/*.js` llaman con rutas relativas same-origin directo a
controllers MVC del propio host (`/Catalogo/Get...`, `/Producto/GetProductosCatalogador`,
`/Usuarios/...`), no a una API externa. `axios` está en `package.json` pero no se usa en
ningún archivo — todo es `fetch` nativo.

### Cómo agregar una pantalla/feature nueva

Ejemplo de referencia (feature "Productos"/catálogo, ver todos los archivos reales en
`App.jsx`, `hooks/useCatalogProducts.js`, `components/products/*`,
`components/filters/*`, `services/catalogFacetApi.js`):

1. Vista Razor en `CatalogoWeb.Host/Views/...` con el `<div id="react-...-root">` y el
   `window.REACT_APP_CONFIG` inline.
2. Entry point en `vite.config.js` (`rollupOptions.input`) si es una pantalla nueva, o
   reutilizar uno existente si la pantalla ya monta desde `main.jsx`.
3. Componente página (en `src/` raíz para features viejas, o `src/<feature>/App.jsx`
   para features nuevas — ver inconsistencia abajo).
4. Hooks de datos/estado en `src/hooks/`.
5. Componentes de UI en `src/components/<dominio>/`.
6. Servicios de fetch en `src/services/`.

**Inconsistencia a tener presente**: features viejas tienen su entry `*-main.jsx` suelto
en la raíz de `src/` (`producto-detalle-main.jsx`); features nuevas meten `main.jsx`
dentro de su propia carpeta (`src/clientes/main.jsx`, `src/usuarios/main.jsx`). Para
código nuevo, seguir el patrón de carpeta propia (más reciente), no el patrón viejo de
archivos sueltos.

### Correr en local

```bash
cd CatalogoWeb.Host/ClientApp/catalogo   # NO en la raíz del monorepo
npm install
npm run dev      # Vite, puerto 3000, proxy /Tienda → localhost:44391
npm run build     # genera dist/ + manifest.json que consume Razor
```

El `package.json` de la raíz del monorepo (`CatalogoYApi/package.json`) es un proyecto
Node totalmente distinto (solo `@tabler/icons`/`sass`, sin scripts de este frontend) —
no confundirlo. Sin script `test` ni `lint`.

---

## flok-back (`CatalogoWeb.Host/` + servicios legacy referenciados)

**Stack**: ASP.NET MVC 5 clásico sobre **.NET Framework 4.7.2** (no .NET moderno),
`packages.config` (NuGet clásico), **sin ORM** (ADO.NET puro con SQL armado a mano,
también acceso directo a Firebird).

### Arquitectura: N-Layer clásico, no Clean Architecture

```
CatalogoWeb.Host          → Controllers MVC + Views Razor (orquesta la request HTTP)
   → ProyectoWeb.Services  → lógica de negocio: XxxServices.cs + XxxHandler.cs por acción
       → ProyectoWeb.Services.Contract → DTOs (Request/Response/Result)
       → HTTP a ServiciosAndretichAPI (fuente real de datos de catálogo, vía BambukHttpClient + Polly)
   → MigracionInformacion.Services → lógica/datos legacy con ADO.NET directo a SQL Server/Firebird
```

No hay DI: los controllers instancian sus dependencias a mano en el constructor
(`_productosServices = new ProductosServices();`), patrón repetido en toda la capa de
servicios. No hay repositorios formales ni AutoMapper — el mapeo es manual dentro de
cada `Handler`.

### Flujo real de un endpoint: `GetProductosCatalogador`

1. `CatalogoWeb.Host/Controllers/ProductoController.cs` — acción que lee `Session`,
   delega a `_productosServices.ObtenerProductosCatalogador(request)`.
2. `ProyectoWeb.Services/Productos/ProductosServices.cs` → delega en
   `ObtenerProductosCatalogadorHandler`.
3. `ObtenerProductosCatalogadorHandler.cs` orquesta: resuelve marca, llama
   `ConsultarCatalogadorHandler` (que hace HTTP contra `ServiciosAndretichAPI` con
   `BambukHttpClient`), normaliza imágenes, calcula precios/descuentos por usuario.
4. DTOs compartidos en `ProyectoWeb.Services.Contract/Productos/`.
5. Del lado de `ServiciosAndretichAPI`, el request llega a un controller .NET 6 que
   delega en handlers de `MigracionInformacion.Services`, que ejecutan SQL/Firebird
   directo.

**Nota de riesgo detectada**: la ruta que arma `ConsultarCatalogadorHandler`
(`api/v1/catalogo/productos`) no coincide literalmente con las rutas de atributo
observadas en los controllers de `ServiciosAndretichAPI` — puede haber un proxy/rewrite
intermedio no explorado, o código desalineado. **Verificar la ruta real antes de tocar
este flujo**, no asumir que coincide con lo que dice el código de un lado solamente.

### Convenciones

- Controllers: `XxxController.cs`, heredan de `BaseController`.
- Servicios: `XxxServices.cs` (fachada) + `<Verbo><Entidad>Handler.cs` por operación
  (patrón "Handler" ad-hoc, sin interfaz común, sin mediador — no confundir con el CQRS
  de bambuk-api).
- DTOs en `ProyectoWeb.Services.Contract/<Dominio>/` con sufijos `Request`/`Response`/
  `Result`.
- Dominio en español (Productos, Marcas, Ventas, Garantías), consistente con el resto
  del ecosistema Andretich.

### Cómo agregar una funcionalidad nueva

1. DTO en `ProyectoWeb.Services.Contract/<Dominio>/`.
2. `XxxHandler.cs` nuevo en `ProyectoWeb.Services/<Dominio>/` (SQL directo o HTTP a
   `ServiciosAndretichAPI` vía `BambukHttpClient`).
3. Exponer el método en `XxxServices.cs`.
4. Acción nueva en el `Controller` de `CatalogoWeb.Host/Controllers/`, con try/catch y
   `Json(...)`/`Content(..., "application/json")`.
5. Ruta especial (si hace falta) en `CatalogoWeb.Host/App_Start/RouteConfig.cs`.

No hay migraciones de EF (no hay EF Code First acá) ni scaffolding — todo se escribe a
mano, y el esquema de BD se asume preexistente.

### Gotchas importantes

- **Connection strings y credenciales de `ServiciosAndretichAPI` en texto plano** en
  `CatalogoWeb.Host/web.config` (`<connectionStrings>`, `<appSettings>`
  `ServiciosAndretichAPI:Username/Password`). No pegar este contenido en lugares
  compartidos sin necesidad.
- Autenticación por `Session` clásica de ASP.NET (`Session["Usuario"]`), no JWT ni
  ASP.NET Identity.
- Logging con `Debug.WriteLine`/`Console.WriteLine`, sin logging estructurado.
- **No hay tests automatizados.** `ProyectoTest.csproj` no es un proyecto de tests pese
  al nombre — es un sitio MVC clásico completo/duplicado.
- Se compila con Visual Studio/MSBuild clásico, no con `dotnet run` (proyecto no
  SDK-style, `packages.config`).

### Correr en local

Abrir `Catalogo-API.sln` en Visual Studio, restaurar NuGet clásico, correr con IIS
Express (puerto SSL configurado: `44391`). No hay comando de terminal equivalente a
`dotnet run` para este proyecto en particular.

---

## api-servicios (`ServiciosAndretichAPI/`)

**Stack**: ASP.NET Core Web API **.NET 6** (minimal hosting, `Program.cs` sin
`Startup.cs`), **sin ORM** (ADO.NET puro contra SQL Server y Firebird), JWT Bearer +
BCrypt, Serilog, Swagger en `/swagger`.

### Qué expone

Es el **backend-for-multiple-frontends / façade** del ERP legado (Firebird "DBSIF" +
SQL Server) para todo el ecosistema Andretich: catálogo/productos/stock/precios,
ventas/pedidos/órdenes de compra, clientes/vendedores/usuarios, integraciones externas
(Nexand, Fulland, WhatsApp Business API para un bot), y **autenticación centralizada**
(JWT + refresh tokens) que el resto de las apps del cliente consume.

### Arquitectura: façade moderna sobre lógica legacy — dos calidades de código conviviendo

No es Clean Architecture ni CQRS. Los controllers (`ServiciosAndretichAPI/Controllers/`)
son delgados y delegan en los mismos proyectos legacy que usa flok-back
(`MigracionInformacion.Services`, `ProyectoWeb.Services`), que acceden a datos con
Handlers `internal static` y SQL/Firebird embebido a mano (Transaction Script, no
Repository).

**Excepción — el módulo `Auth/` es el único bien diseñado**: interfaces (`ITokenService`,
`IInvitacionService`), DI real (`AddScoped` en `Program.cs`), DTOs propios en
`Auth/DTOs/`, logging estructurado. **Para código nuevo en este proyecto, seguir el
patrón de `Auth/` (interfaz + implementación + DI + DTO), no el patrón `new
XxxServices()` del resto de los controllers** — es la única zona del código que
representa la dirección a la que el equipo parece estar migrando.

### Flujo real de un endpoint: `GET /Productos/ObtenerStock/{idProducto}`

1. `ServiciosAndretichAPI/Controllers/ProductosController.cs` instancia
   `new ProductosServices(configuration)` en el constructor.
2. `MigracionInformacion.Services/Productos/ProductosServices.cs` delega a
   `ObtenerStockHandler.Handle(...)`.
3. `ObtenerStockHandler.cs` arma SQL a mano, abre `FbConnection` contra Firebird
   (connection string `"BdPaljet"`, resuelta vía `ConfigurationManager` — **no vive en
   `appsettings.json`**, viene de config legacy heredada; confirmar dónde antes de
   asumir que cambiarla en `appsettings.json` tiene efecto), mapea `DataTable` a
   `List<Stock>` a mano.
4. Controller devuelve `Ok(stock)`.

Nota: otros handlers del mismo controller sí usan `IConfiguration` moderno
correctamente — la inconsistencia es por handler, no por controller entero, así que hay
que revisar caso por caso antes de asumir de dónde sale una connection string.

### Convenciones

- Controllers: `<Entidad>Controller.cs`.
- Servicios legacy: `<Entidad>Services.cs` (fachada) + `<Accion><Entidad>Handler.cs`
  (`internal static`, sin interfaz).
- DTOs en proyectos `*.Contract`, nombres de dominio en español.
- Mezcla de convenciones de nombres de parámetros (español con ñ como `tamañoPagina`,
  abreviaturas como `AuxErr`) — reflejo de código migrado sin normalizar, no un estándar
  a replicar en código nuevo.

### Cómo agregar una funcionalidad nueva

- **Si es parte del dominio legacy** (productos, ventas, clientes, etc.): Handler nuevo
  en `MigracionInformacion.Services/<Dominio>/`, exponerlo en `<Dominio>Services.cs`,
  acción nueva en el controller correspondiente delegando al servicio. Hereda la policy
  global de JWT automáticamente salvo `[AllowAnonymous]`.
- **Si es autenticación o algo nuevo sin arrastre legacy**: replicar el patrón de
  `Auth/` — interfaz en `Auth/Services/`, implementación, registro `AddScoped` en
  `Program.cs`, DTO en `Auth/DTOs/`, acción en un controller de `Auth/Controllers/`.

### Gotchas importantes

- **Secretos committeados en texto plano** en `appsettings.json`/
  `appsettings.Development.json`: JWT `SecretKey`, credenciales SQL Server, connection
  string Firebird (`SYSDBA/masterkey`), credenciales SMTP, credenciales de la API
  PL/ERP. Variables de entorno `JWT_SECRET_KEY`/`JWT_ISSUER`/`JWT_AUDIENCE` las
  sobreescriben si están presentes — preferir env vars sobre editar el appsettings.
- Token y `phoneNumberId` de WhatsApp Business API están **hardcodeados como placeholder**
  en `WhatsappController.cs`, no en configuración — si se toca ese controller, mover a
  config en vez de perpetuar el hardcode.
- `ExceptionHandlingMiddleware` global tiene un bug conocido: serializa el error con
  `.ToString()` sobre un objeto anónimo, así que el body de error 500 no es JSON real
  sino el literal `"{ StatusCode = 500, Message = ... }"` — tenerlo en cuenta si un
  frontend intenta parsear ese body como JSON.
- **No hay tests** para este proyecto ni para los servicios legacy que usa.
- Referencia a un `.dll` de .NET Framework por ruta absoluta de Windows
  (`System.IdentityModel.dll`) — no portable a macOS/Linux si algún día hace falta.

### Correr y testear en local

```bash
cd ServiciosAndretichAPI
dotnet build
dotnet run          # Swagger UI en /swagger, soporta JWT bearer para probar autenticado
```

Sin proyecto de tests vinculado.

---

## Cómo interactúan flok-front, flok-back y api-servicios

- **flok-front → flok-back**: mayormente same-origin, rutas relativas a controllers MVC
  de `CatalogoWeb.Host` (`window.REACT_APP_CONFIG` inyectado por Razor da la base URL
  cuando hace falta apuntar afuera).
- **flok-back → api-servicios**: HTTP vía `ProyectoWeb.Services/Http/BambukHttpClient.cs`
  (con Polly para reintentos), URL configurada en `CatalogoWeb.Host/web.config`
  (`ServiciosAndretichAPI:BaseUrl`). Antes de asumir dónde está un bug de datos, revisar
  si el problema es de este salto HTTP (timeout, credenciales, ruta desalineada — ver
  gotcha de rutas arriba) en vez de asumir que está en el front o en el ERP directamente.
- **Autenticación**: `api-servicios` (`ServiciosAndretichAPI/Auth/`) es la fuente de JWT
  para el ecosistema; `flok-back` la consume vía `BambukHttpClient` con credenciales
  configuradas en `web.config`, mientras que `flok-front` usa sesión clásica ASP.NET
  contra `flok-back` (no JWT directo desde el browser).
- **Base de datos real**: ambos backends terminan leyendo el mismo ERP (Firebird
  `DBSIF`) y/o SQL Server — para bugs de datos mal mostrados, el MCP de SQL Server (ver
  [../CLAUDE.md](../CLAUDE.md)) es más confiable que asumir en qué capa de código está
  el problema.
