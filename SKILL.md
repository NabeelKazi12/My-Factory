---
name: repos-grandes-ahorro-tokens
description: Diagnostica y configura un repositorio grande o monorepo para que Claude Code consuma menos tokens. Úsala cuando el usuario pida optimizar, configurar o preparar Claude Code para un repositorio de gran tamaño, reducir el consumo de tokens o de contexto en un repo, reorganizar CLAUDE.md o .claude/settings.json en un monorepo, o evaluar herramientas de indexación de código como Graphify o Repowise.
---

# Optimizar un repositorio grande para Claude Code

Tu objetivo es dejar el repositorio configurado para que Claude cargue solo lo que cada tarea necesita. Prioriza la configuración que Claude Code aplica de forma determinista (settings, reglas de permisos, CLAUDE.md por capas) frente a recomendaciones que dependen de hábitos, porque la configuración funciona en todas las sesiones y para todo el equipo sin depender de que nadie la recuerde.

Sigue los pasos en orden. No saltes el paso 3: el usuario debe aprobar los cambios antes de que escribas nada.

## Paso 1. Diagnosticar con el script

Desde la raíz del repositorio, ejecuta:

```bash
bash <ruta-de-esta-skill>/scripts/diagnostico.sh
```

Usa su salida como fuente para el diagnóstico. No recorras el repositorio leyendo archivos a mano para hacer el inventario: el script existe precisamente para que el diagnóstico cueste pocos tokens. Solo abre un archivo concreto si el script señala un problema en él y necesitas su contenido para proponer el cambio (por ejemplo, un CLAUDE.md demasiado largo que hay que dividir).

Si el directorio no es la raíz de un repositorio git, pregunta al usuario cuál es la raíz antes de continuar.

## Paso 2. Decidir qué proponer

Aplica estos criterios a la salida del diagnóstico:

| Señal del diagnóstico | Propuesta | Referencia |
|---|---|---|
| CLAUDE.md raíz con más de 200 líneas | Dividir en CLAUDE.md raíz breve + uno por subsistema; mover procedimientos a skills | `references/configuracion.md` §1 |
| Reglas en `.claude/rules/` sin `paths:` | Añadir frontmatter `paths:` | `references/configuracion.md` §2 |
| Directorios generados, build o vendor versionados | Reglas `Read` en `permissions.deny` | `references/configuracion.md` §3 |
| Subsistemas o paquetes claramente separados | CLAUDE.md por subsistema, `claudeMdExcludes`, `worktree.sparsePaths` | `references/configuracion.md` §1, §4, §5 |
| Lenguajes tipados detectados | Plugins de code intelligence (LSP) | `references/configuracion.md` §6 |
| Más de ~5.000 archivos de código, o varios repositorios relacionados | Evaluar una capa de inteligencia del código | `references/capa-inteligencia.md` |
| El usuario menciona salidas largas de tests o logs | Hook de filtrado o `repowise distill` | `references/hooks.md` |
| Posibles secretos versionados | No abrirlos; avisar al usuario y proponer reglas `Read` en `permissions.deny` | `references/privacidad.md` |

Lee solo las referencias que necesites según las señales detectadas. Antes de recomendar cualquier herramienta externa (Graphify, Repowise u otra), lee `references/privacidad.md`, porque algunas envían datos fuera de la infraestructura del usuario y eso puede ser incompatible con su política de datos.

Si hay que elegir entre Graphify y Repowise, recomienda **solo una**. Dos capas de indexación activas a la vez duplican herramientas MCP en el contexto y hacen dudar al agente sobre cuál usar.

## Paso 3. Presentar la propuesta y esperar aprobación

Muestra una tabla con estas columnas:

| Cambio | Archivo | Efecto esperado | Lo aplica |
|---|---|---|---|

En la columna "Lo aplica" indica **Claude** o **Usuario**. Marca como "Usuario" todo lo que requiera comandos integrados de Claude Code (`/plugin install`, `/context`, `/usage`, `/mcp`), porque no puedes ejecutarlos tú.

Después pregunta qué cambios aprueba y **detente**. No escribas ningún archivo hasta tener la respuesta.

## Paso 4. Aplicar solo lo aprobado

- **Settings JSON**: si `.claude/settings.json` o `.claude/settings.local.json` ya existen, fusiona las claves nuevas con las existentes. Nunca sobrescribas el archivo entero. Valida el resultado con `python3 -m json.tool <archivo>` (o `jq . <archivo>`).
- **Dónde va cada ajuste**: lo que debe aplicarse a todo el equipo va en `.claude/settings.json` (se versiona). Las preferencias personales van en `.claude/settings.local.json` (no se versiona).
- **CLAUDE.md**: al dividir un archivo existente, conserva todas las instrucciones; solo muévelas al subsistema que corresponda o a una skill.
- **Instalaciones y hooks**: no instales paquetes (`pip`, `uv`, `npm`) ni crees hooks sin confirmación explícita del usuario para esa acción concreta, porque ejecutan código en su máquina.
- **Git**: no hagas commit. Deja los cambios preparados para que el usuario los revise.
- **Secretos**: no escribas credenciales, claves API ni datos personales en ningún archivo. Usa placeholders como `TU_CLAVE_AQUI`.

## Paso 5. Cerrar con los pasos del usuario

Termina con:

1. Un resumen breve de los archivos creados o modificados.
2. La lista de acciones que debe ejecutar el usuario (instalar plugins LSP, activar o desactivar servidores MCP, etc.).
3. La recomendación de medir: ejecutar `/context` en una sesión nueva y comparar con la situación anterior, y revisar `/usage` tras unos días de trabajo.
4. Si el equipo aún no la tiene, menciona que existe una guía de hábitos de sesión para personas (`GUIA-EQUIPO.md`), que complementa esta configuración.

## Referencias

- `scripts/diagnostico.sh`: inventario del repositorio con salida compacta.
- `references/configuracion.md`: plantillas de CLAUDE.md, reglas, permisos, worktrees, acceso entre directorios y plugins LSP.
- `references/capa-inteligencia.md`: Graphify y Repowise, cuándo elegir cada uno y cómo instalarlos.
- `references/hooks.md`: hooks para filtrar salidas de tests y logs, y `repowise distill`.
- `references/privacidad.md`: qué datos salen de la infraestructura con cada herramienta.
