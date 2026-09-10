---
name: jira-tickets
description: Buscar, leer, actualizar, comentar o crear tickets en Jira para los proyectos del cliente Andretich (Flok, Bambuk, API de servicios). Usar cuando el usuario mencione un ticket, una tarea de Jira, un bug reportado, o pida revisar/actualizar el estado de trabajo pendiente.
---

# Gestión de tickets Jira

Este skill guía el uso del MCP `jira` (Atlassian Remote MCP) para gestionar tickets
multi-proyecto del cliente.

## Antes de empezar

Si las herramientas del MCP `jira` no responden o piden autenticación, avisar al usuario
que debe autorizar el acceso (OAuth) la primera vez — no se puede completar ese flujo de
forma no interactiva.

## Flujo para investigar un ticket

1. Buscar el ticket por clave (ej. `FLOK-123`) o por texto libre en el proyecto
   correspondiente.
2. Leer la descripción completa, comentarios y tickets vinculados (linked issues) — el
   contexto real suele estar en los comentarios, no solo en la descripción inicial.
3. Identificar a qué repo(s) corresponde el ticket según el proyecto Jira:
   - Proyecto Flok → `flok-front` y/o `flok-back`.
   - Proyecto Bambuk → `bambuk` (front) y/o `bambuk-api`.
   - Proyecto transversal/API → `api-servicios`.
4. Si el ticket describe un bug de datos, considerar validar en SQL Server (MCP
   `sqlserver`) antes de tocar código, para confirmar la causa real.

## Flujo para actualizar/comentar

- Antes de cambiar el estado de un ticket (ej. a "In Progress" o "Done"), confirmar con
  el usuario si no fue una instrucción explícita.
- Al comentar, ser específico: mencionar commit/PR, archivos tocados, o el motivo del
  cambio de estado.
- No cerrar tickets sin verificar que el fix fue probado o al menos revisado.

## Triage multi-proyecto

Cuando un ticket no aclara en qué proyecto está el problema, o el usuario pide "ver todos
los tickets abiertos del cliente", buscar en los distintos proyectos de Jira del cliente
(Flok, Bambuk, y el proyecto de API/servicios si existe) en vez de asumir uno solo.
