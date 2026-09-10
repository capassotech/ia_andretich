---
name: multi-project-triage
description: Diagnosticar un bug o comportamiento inesperado cuyo origen no está claro entre los proyectos del cliente Andretich (flok-front, flok-back, bambuk, bambuk-api, api-servicios) y la base de datos. Usar cuando el usuario reporte un problema sin especificar en qué proyecto está, o cuando el síntoma se ve en un lugar pero la causa podría estar en otro.
---

# Triage multi-proyecto

El cliente tiene varios proyectos que interactúan entre sí (front/back de Flok, front/back
de Bambuk, y una API de servicios compartida) contra una misma base de datos SQL Server.
Un síntoma visible en un front puede originarse en cualquier capa de atrás.

## Orden de diagnóstico recomendado

1. **Reproducir el síntoma** y anotar en qué proyecto/pantalla se observa.
2. **Revisar los datos reales** con el MCP `sqlserver` — muchas veces el problema es un
   dato mal guardado o inconsistente, no un bug de lógica.
3. **Revisar la capa de API** correspondiente (`bambuk-api`, `flok-back` o
   `api-servicios`) para ver si el dato que llega al front ya viene mal formado.
4. **Revisar el front** (`flok-front` o `bambuk`) solo si los pasos anteriores confirman
   que el dato llega bien pero se muestra/procesa mal.
5. Si el problema toca `api-servicios`, verificar si otros proyectos consumen el mismo
   endpoint — un fix ahí puede impactar a más de un cliente/frontend.

## Reglas

- No asumir que el bug está en el proyecto donde se reportó el síntoma sin antes revisar
  la capa de datos y de API.
- Si el repo relevante (`flok-back` o `api-servicios`) todavía no está accesible via el
  MCP `clientes-fs` (ver [CLAUDE.md](../../../CLAUDE.md)), avisar al usuario en vez de
  adivinar el comportamiento del backend.
- Documentar el diagnóstico en el ticket de Jira correspondiente (ver skill
  `jira-tickets`) una vez identificada la causa raíz.
