---
estado: COMPLETADO
fecha: 2026-07-25
build: ✅ assembleDebug + lintDebug + 15/15 tests passed
repo: https://github.com/antonioalarconvirseda/terrariawiki
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
