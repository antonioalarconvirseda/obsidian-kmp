# Iteración 19 — Configuración de graphify, grafo de conocimiento del código (2026-07-26)

El usuario instaló `graphifyy` (paquete PyPI, repo `Graphify-Labs/graphify`) y pidió configurarlo para este proyecto. Es una herramienta que construye un grafo de dependencias/relaciones del código (extracción AST vía tree-sitter, con soporte Kotlin) y se integra como skill/hook en asistentes de IA, exponiendo consultas tipo `graphify query`, `graphify path`, `graphify explain`, `graphify god-nodes` sobre un `graph.json`.

**Decisión: extracción code-only, sin backend LLM.** El proyecto no necesita labeling semántico de comunidades para un grafo de dependencias Kotlin — el AST vía `tree-sitter-kotlin` ya captura clases, funciones y relaciones de llamada/import. Evita coste de API y dependencia de una clave configurada. Contra: los nombres de comunidad quedan como placeholders (`Community N`) en vez de etiquetas semánticas; aceptable, el propósito aquí es navegación de dependencias, no documentación generada.

**Decisión: git hooks + PreToolUse hook, no solo comando manual.** `graphify hook install` registra `post-commit`/`post-checkout` para que el grafo no quede obsoleto silenciosamente. Además, `graphify claude install` registra un hook `PreToolUse` en `.claude/settings.json` que fuerza consultar el grafo antes de `Read`/`Grep`/`Bash` de exploración — coherente con la regla ya existente en `CLAUDE.md` de "explorar antes de codificar", ahora con un recurso más barato en tokens que grep/lectura cruda cuando la pregunta es sobre dependencias o arquitectura.

**Regla añadida a `CLAUDE.md`:** tras cualquier cambio que toque arquitectura o dependencias (nueva dependencia, nuevo feature/módulo, cambio de capa domain/data/ui, nueva relación entre packages) hay que correr `graphify update .` de inmediato, sin esperar al hook de post-commit — el grafo debe reflejar el estado real antes de que otra pregunta de arquitectura lo consulte en la misma sesión.

| Cambio | Archivo(s) | Descripción |
|---|---|---|
| chore | `.claude/settings.json` (nuevo) | Hooks `PreToolUse` (Bash/Grep, Read/Glob) que fuerzan consultar el grafo antes de explorar código |
| chore | `.gitattributes` (nuevo) | Merge driver `graphify` para `graphify-out/graph.json` (union-merge, evita conflictos triviales) |
| chore | `.gitignore` | `graphify-out/` (artefacto generado), `.claude/settings.local.json` y `.claude/scheduled_tasks.lock` (estado local de la sesión, no compartido) |
| docs | `CLAUDE.md` | Sección `## graphify` con reglas de cuándo consultar el grafo y cuándo forzar `graphify update .` |
| infra | `.git/hooks/post-commit`, `.git/hooks/post-checkout` | Instalados por `graphify hook install`, rebuild automático del grafo (AST-only, sin coste) |

### Hand-off
- Grafo inicial: 476 nodos / 711 edges / 29 comunidades (`graphify extract . --code-only`); tras un `graphify update .` de verificación subió a 501 nodos / 733 edges / 34 comunidades (nuevos archivos `.claude/` detectados).
- Verificado con `graphify god-nodes --top 10` (top: `ItemCategory`, `Item`, `ItemDto`, `ItemsRepositoryImpl`...) y `graphify query "¿qué usa ItemsRepositoryImpl?"` — ambos devuelven subgrafo coherente con la arquitectura Clean/MVVM ya documentada.
- Visualización force-directed en `graphify-out/graph.html` (no versionada, generada localmente); alternativa jerárquica disponible vía `graphify tree`.
- Binario `graphify` no quedaba en `PATH` tras `pip3 install` (fue a `Python.framework/.../bin`); se symlinkeó a `/usr/local/bin` manualmente (requiere `sudo`, lo ejecutó el usuario).
- Commit: `chore: configure graphify code-graph tool for this project` — directo a `main`.
- Sin cambios de comportamiento en la app — esta iteración es tooling/documentación pura, no toca `app/src`.

---

[[00-Index]]
