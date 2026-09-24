# Roadmap: Optimización del consumo de tokens con Claude Code en un proyecto agéntico

## Contexto

En Claude Code, el gasto de tokens no depende tanto de lo que escribe el usuario como del **contexto que el agente acumula y reenvía en cada paso**: historial de conversación, archivos leídos, salidas de herramientas, CLAUDE.md, definiciones de MCP y tokens de razonamiento. Cada uso de herramientas genera una nueva petición con todo ese contexto.

**Objetivo del roadmap:** mantener el contexto pequeño y estable, asignar el modelo adecuado a cada tarea y establecer mecanismos de control a nivel de equipo.

---

## Fase 0 — Línea base y medición (Semana 1)

Antes de optimizar, hay que saber dónde está el gasto.

- [ ] Revisar el consumo actual con `/usage` (tokens, coste estimado y estadísticas de caché).
- [ ] Usar `/context` para identificar qué ocupa espacio (CLAUDE.md, MCP, historial, salidas de herramientas).
- [ ] Configurar la status line para ver el uso de contexto de forma continua.
- [ ] Ejecutar `/insights` para detectar fricciones y patrones de uso ineficientes.
- [ ] Documentar métricas de partida: coste por sesión, % de tokens desde caché y fallos de caché.

**Entregable:** informe de línea base con los principales focos de consumo.

---

## Fase 1 — Configuración base del proyecto (Semanas 1-2)

Victorias rápidas con alto impacto y bajo esfuerzo.

### 1.1 CLAUDE.md compacto
- [ ] Reducir CLAUDE.md a lo esencial (objetivo: menos de 200 líneas).
- [ ] Incluir solo la arquitectura del sistema agéntico, las convenciones y los comandos clave (build, test, lint).
- [ ] Añadir una sección de **instrucciones de compactación**:

```markdown
# Compact instructions

Al compactar, conservar:
- objetivo de la tarea actual
- archivos modificados
- comandos ya ejecutados
- tests fallidos y errores exactos
- decisiones tomadas
- siguientes pasos

Descartar:
- caminos de exploración antiguos
- logs repetidos
```

### 1.2 Migrar flujos específicos a skills
- [ ] Identificar instrucciones de flujos concretos dentro de CLAUDE.md.
- [ ] Moverlas a skills que se carguen bajo demanda, por ejemplo:
  - `nuevo-agente`: cómo crear y registrar un nuevo agente.
  - `nueva-tool`: cómo definir una herramienta.
  - `evals`: cómo ejecutar evaluaciones.
- [ ] Crear una skill `codebase-overview` con arquitectura, directorios clave y convenciones de nombres, para evitar exploraciones costosas.

**Entregable:** CLAUDE.md reducido y catálogo inicial de skills del proyecto.

---

## Fase 2 — Selección de modelo y razonamiento (Semana 2)

- [ ] Establecer **Sonnet como modelo por defecto** en `/config`.
- [ ] Definir criterios para escalar a **Opus**: decisiones de arquitectura complejas o razonamiento de varios pasos.
- [ ] Asignar **Haiku** a subagentes de tareas mecánicas (`model: haiku` en la configuración del subagente).
- [ ] Ajustar el nivel de esfuerzo con `/effort` en tareas simples, porque los tokens de razonamiento se facturan como salida.
- [ ] Documentar una tabla de referencia para el equipo:

| Tipo de tarea | Modelo | Esfuerzo |
|---|---|---|
| Búsquedas, renombrados, formato | Haiku | Bajo |
| Desarrollo habitual, tests, refactors | Sonnet | Medio |
| Arquitectura multiagente, depuración compleja | Opus | Alto |

**Entregable:** política de modelos documentada y aplicada.

---

## Fase 3 — Herramientas, MCP y preprocesamiento (Semanas 3-4)

### 3.1 Reducir el coste de MCP
- [ ] Auditar los servidores MCP configurados con `/mcp`.
- [ ] Desactivar los que no se usen activamente.
- [ ] Sustituir MCP por CLIs cuando existan (`gh`, `aws`, `gcloud`, etc.).

### 3.2 Hooks de preprocesamiento
- [ ] Crear un hook `PreToolUse` que filtre la salida de tests para mostrar solo los fallos.
- [ ] Crear hooks que filtren logs (por ejemplo, solo líneas con `ERROR`) antes de que Claude los lea.
- [ ] Verificar los hooks con `/hooks`.

Ejemplo de configuración en `settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/filter-test-output.sh"
          }
        ]
      }
    ]
  }
}
```

### 3.3 Inteligencia de código
- [ ] Instalar plugins de inteligencia de código para los lenguajes tipados del proyecto, de modo que la navegación por símbolos sustituya a grep y lecturas múltiples.

**Entregable:** entorno de herramientas optimizado y hooks versionados en el repositorio.

---

## Fase 4 — Arquitectura de trabajo agéntico (Semanas 4-5)

### 4.1 Subagentes
- [ ] Definir subagentes para operaciones verbosas: ejecución de tests, consulta de documentación y procesamiento de logs.
- [ ] Asegurar que devuelven **solo un resumen** a la conversación principal.
- [ ] Asignarles el modelo mínimo necesario.

### 4.2 Agent teams (uso controlado)
- [ ] Usarlos solo cuando el paralelismo aporte valor real, ya que multiplican el consumo (cada compañero tiene su propia ventana de contexto).
- [ ] Reglas de uso:
  - Sonnet para los compañeros.
  - Equipos pequeños.
  - Prompts de arranque concisos.
  - Cerrar cada compañero al terminar su trabajo.

### 4.3 Tareas en segundo plano
- [ ] Revisar tareas programadas (`/loop`) y check-ins, que reenvían el contexto completo aunque la sesión esté inactiva.

**Entregable:** catálogo de subagentes y guía de uso de agent teams.

---

## Fase 5 — Buenas prácticas del equipo (continuo)

### Gestión del contexto en sesión
- [ ] `/clear` al cambiar a una tarea no relacionada (usar `/rename` antes y `/resume` después si se quiere volver).
- [ ] `/compact` con foco solo cuando se necesite continuidad (compactar un contexto grande también cuesta).
- [ ] Tener en cuenta la vida del caché: tras pausas largas, el primer mensaje reprocesa todo el contexto.

### Forma de pedir
- [ ] Prompts específicos: nombrar la función y el archivo en lugar de pedir "mejora el código".
- [ ] Usar **modo plan** (Shift+Tab) antes de implementar cambios complejos.
- [ ] Corregir pronto con Escape y volver atrás con `/rewind`.
- [ ] Dar criterios de verificación (tests, salida esperada).
- [ ] Avanzar de forma incremental: implementar, probar y continuar.

**Entregable:** guía interna de buenas prácticas y sesión de formación al equipo.

---

## Fase 6 — Gobierno y monitorización (Semana 6 y continuo)

- [ ] Configurar **límites de gasto**: por workspace en Claude Console, o por organización, grupo o miembro en Team/Enterprise.
- [ ] Activar la **exportación con OpenTelemetry** para métricas de tokens y coste por usuario en el sistema de observabilidad propio.
- [ ] Crear un dashboard semanal con coste por desarrollador, % de caché y sesiones atípicas.
- [ ] Revisión mensual: comparar con la línea base de la Fase 0 y ajustar políticas.

**Entregable:** panel de seguimiento y proceso de revisión periódica.

---

## Resumen de prioridades

| Prioridad | Estrategia | Impacto | Esfuerzo |
|---|---|---|---|
| 1 | CLAUDE.md compacto + skills | Alto | Bajo |
| 2 | `/clear` entre tareas | Alto | Muy bajo |
| 3 | Sonnet por defecto, Haiku en subagentes | Alto | Bajo |
| 4 | Subagentes para operaciones verbosas | Alto | Medio |
| 5 | Hooks de filtrado de salidas | Medio-Alto | Medio |
| 6 | Auditoría de MCP | Medio | Bajo |
| 7 | Ajuste de esfuerzo de razonamiento | Medio | Bajo |
| 8 | Monitorización y límites de gasto | Medio (control) | Medio |

---

## Referencias

- Gestión de costes en Claude Code: https://code.claude.com/docs/en/costs
- Documentación general de Claude Code: https://docs.claude.com/en/docs/claude-code/overview
