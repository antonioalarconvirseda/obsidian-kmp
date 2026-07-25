---
estado: COMPLETADO
fecha: 2026-07-25
build: ✅ assembleDebug + lintDebug + 17/17 tests passed
repo: https://github.com/antonioalarconvirseda/terrariawiki
iteracion-2: 3 bugs corregidos (a7d9016, a347cad, 632d836)
vault: https://github.com/antonioalarconvirseda/obsidian-kmp
---

# Publicación en GitHub — TerrariaWiki

## Repositorio destino

- **URL:** https://github.com/antonioalarconvirseda/terrariawiki
- **Visibilidad:** público ✅
- **Default branch:** `main`
- **Descripción:** "Super Wiki de Terraria — Android app en Kotlin con Jetpack Compose. KMP-ready. Datos desde terraria.wiki.gg (MediaWiki + Cargo)."

## Comandos ejecutados (real)

```bash
# 1. init
cd /Users/aav/gits/terrariawiki
git init -b main
git add .
git commit -m "feat: initial MVP — Terraria Wiki Items (Kotlin + Compose, KMP-ready) ..."

# 2. creación del repo + push
gh repo create antonioalarconvirseda/terrariawiki \
  --public \
  --description "Super Wiki de Terraria — Android app en Kotlin con Jetpack Compose. KMP-ready. Datos desde terraria.wiki.gg (MediaWiki + Cargo)." \
  --source=. \
  --remote=origin \
  --push
```

`gh repo create --source` automáticamente: inicializa el remote, configura `origin`, y pushea el commit inicial. Sin necesidad del fallback manual.

## Licencia aplicada

**MIT** — ver `LICENSE` en raíz. Copyright (c) 2026 Antonio Alarcón Virseda.

## README

Sí, en raíz del repo. Secciones:
- Estado y stack
- Arquitectura (árbol de paquetes)
- Cómo compilar y cómo instalar en dispositivo
- Tests
- Créditos a terraria.wiki.gg y aviso de no afiliación a Re-Logic
- Licencia MIT

## Branch strategy

- `main` directa (proyecto personal sin equipo).
- Conventional Commits: `feat:`, `fix:`, `docs:`, `chore:`, `refactor:`, `test:`, `build:`.
- El primer commit usa `feat:` por convención (scaffold inicial = feature inicial).

## Secretos

Verificado:
- `local.properties` está en `.gitignore` (no se subió).
- `build/` y `.gradle/` están en `.gitignore`.
- No hay keystores, API keys, ni tokens.
- El repo público no expone nada sensible.

## Verificación post-push

```bash
$ git ls-remote --heads origin
0fa8ca3... refs/heads/main
```

Un solo commit en `main`, con el scaffold completo del MVP.

## Política de commits futura

Recomendado para próximas iteraciones (las planificará GLM-5.2):

- `feat:` nueva funcionalidad (e.g. `feat: add NPCs section`)
- `fix:` corrección (e.g. `fix: handle null tooltip in ItemsScreen`)
- `docs:` documentación (e.g. `docs: add Koin-DI pattern note`)
- `refactor:` reorganización sin cambio funcional
- `test:` tests sin cambio de código
- `chore:` tareas menores (e.g. bump dep version)
- `build:` cambios en Gradle/AGP

Mensajes en inglés, max 72 chars en subject, body opcional descriptivo.

## Vault Obsidian aparte (auditoría iteración 2)

Durante la auditoría de iteración 2, GLM-5.2 preguntó cómo enlazar la bóveda Obsidian con el proyecto. Se eligió la **Opción A** del análisis (repo aparte) por la naturaleza multi-proyecto del vault.

### Repo del vault

- **URL:** https://github.com/antonioalarconvirseda/obsidian-kmp
- **Visibilidad:** público
- **Default branch:** `main`
- **Descripción:** "Bóveda Obsidian — documentación de aprendizaje Kotlin / KMP (proyecto TerrariaWiki y futuros)"

### Por qué repo aparte (no submódulo, no `docs/`)

- El vault es **multi-proyecto** por diseño: contiene `TerrariaWiki/` y tendrá más proyectos en el futuro. Meterlo dentro de `terrariawiki/docs/` lo fragmentaría y obligaría a tener bóvedas duplicadas por proyecto.
- Un submódulo Git (`git submodule add`) añade fricción al clonar (`git submodule update --init`) sin beneficio real para este caso de uso (la bóveda se lee directamente en GitHub sin clonar el proyecto).
- Mantenerlo en repo aparte permite que la bóveda evolucione independientemente de los proyectos.

### `.gitignore` del vault

```gitignore
Kotlin Multiplatform/.obsidian/workspace.json
Kotlin Multiplatform/.obsidian/workspace-mobile.json
Kotlin Multiplatform/.trash/
.DS_Store
```

- **Excluidos:** `workspace.json` y `workspace-mobile.json` (state local, cambian constantemente con cada movimiento del cursor); `.trash/` (papelera interna de Obsidian).
- **NO excluidos (sí se suben):** `app.json`, `appearance.json`, `core-plugins.json`, `graph.json`. Son configuración replicable entre máquinas, útil para reproducir el setup del vault.

### Enlace bidireccional

- `terrariawiki/README.md` → enlace al vault (sección "Documentación del proyecto").
- `obsidian-kmp/README.md` → enlace al proyecto `terrariawiki` y a su repo.

### Comandos de creación ejecutados (iteración 2)

```bash
# 1. init local
cd /Users/aav/gits/obsidian-kmp
git init -b main
git add .
git commit -m "docs: initial Obsidian vault — TerrariaWiki MVP documentation (26 notes)"

# 2. crear repo público + push
gh repo create antonioalarconvirseda/obsidian-kmp \
  --public \
  --description "Bóveda Obsidian — documentación de aprendizaje Kotlin / KMP (proyecto TerrariaWiki y futuros)" \
  --source=. \
  --remote=origin \
  --push
```

## Resumen de iteración 2 — bugs corregidos

3 bugs detectados por la auditoría de GLM-5.2 (modo plan) y corregidos con commits `fix:` en `terrariawiki`:

| Commit | Bug | Severidad |
|---|---|---|
| `a7d9016` | Doble `https://` en URL de imagen — Coil no podía cargar ninguna imagen | **Crítica** |
| `a347cad` | `extractSellValue` definido pero no usado; "Venta" mostraba HTML crudo | Media (UX rota) |
| `632d836` | `Flow.collect` infinito en `refresh()` — coroutine leaked | Leve (UI funcional pero leak) |

Tests: 15 → **17** (2 nuevos en `ItemsMapperTest` para cubrir la limpieza de `sell`).

Tras los parches: `./gradlew :app:assembleDebug :app:lintDebug :app:testDebugUnitTest` → SUCCESSFUL.

## Mejoras de documentación en la iteración 2

- **Nota nueva creada:** `Sealed-classes-result.md` — el plan original la había prometido pero se omitió. Ahora cubre `kotlin.Result<T>` (Repository) vs `sealed interface UiState` (ViewModel).
- **`01-Decisiones-de-Arquitectura` §3** — añadido párrafo "Matiz KMP-readiness (auditoría post-MVP)" listando los 3 anclajes Android pendientes.
- **`02-Stack-Tecnologico`** — añadido "Matiz KMP-readiness" con tabla de refactors necesarios y referencia a [[01-Decisiones-de-Arquitectura]] §3.
- **`00-Index.md`** — conteos corregidos (26 notas totales, 17 patrones), nueva sección "Iteración 2 — Auditoría de GLM-5.2 y parches" con tabla de commits.
