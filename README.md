# Bóveda Obsidian — Kotlin Multiplatform

Bóveda de notas de aprendizaje sobre **Kotlin**, **Android**, y (objetivo a medio plazo) **Kotlin Multiplatform**. Documenta decisiones de arquitectura, patrones de código, y particularidades de cada proyecto.

## Cómo abrirla en Obsidian

1. Clona este repo: `git clone https://github.com/antonioalarconvirseda/obsidian-kmp.git`
2. Abre Obsidian → **Open another vault** → **Open folder as vault** → selecciona la carpeta clonada.
3. Si Obsidian te pregunta por confiar en la carpeta, di **Trust**.

Una vez abierta, las notas viven bajo `Kotlin Multiplatform/`. El [[Kotlin Multiplatform/Bienvenido]] es el punto de partida.

## Estructura

```
Kotlin Multiplatform/
├── Bienvenido.md                  (nota de bienvenida por defecto de Obsidian)
├── TerrariaWiki/                  (proyecto 1: app Android de wiki de Terraria)
│   ├── 00-Index.md                (mapa del proyecto)
│   ├── 01-Decisiones-de-Arquitectura.md
│   ├── 02-Stack-Tecnologico.md
│   ├── 03-Patrones-Kotlin/        (16 notas, una por patrón introducido)
│   ├── 04-API-Terraria/           (endpoints de la MediaWiki API)
│   ├── 05-UI-Design-System.md
│   ├── 06-Setup-Entorno.md
│   └── 07-GitHub-y-Publicacion.md
└── (futuros proyectos aquí)
```

Cada proyecto nuevo se documenta como una **carpeta hermana** dentro de `Kotlin Multiplatform/`, con la misma plantilla de notas: `00-Index`, `01-Decisiones`, `02-Stack`, `03-Patrones/`, notas específicas, etc.

## Convenciones de documentación

- Cada nota sigue la [[Kotlin Multiplatform/TerrariaWiki/_template-patron|plantilla de 4 secciones]]: Contexto · Decisión · Implementación Kotlin real · Alternativas descartadas y riesgos.
- Las notas se crean **just-in-time** (en el mismo momento de implementar el patrón), no antes. Los snippets son reales.
- Los enlaces `[[…]]` se resuelven dentro de la propia bóveda (gracias a Obsidian). El [[Kotlin Multiplatform/TerrariaWiki/00-Index]] es la raíz de navegación del proyecto TerrariaWiki.

## `.gitignore` del vault

Excluimos:
- `Kotlin Multiplatform/.obsidian/workspace.json` y `workspace-mobile.json` — son state local (posición de paneles, cursor) que cambia constantemente y no aporta.
- `Kotlin Multiplatform/.trash/` — papelera interna de Obsidian.

Mantenemos:
- `Kotlin Multiplatform/.obsidian/app.json`, `appearance.json`, `core-plugins.json`, `graph.json` — config replicable entre máquinas, útil para reproducir el setup.

## Proyectos

- **TerrariaWiki** — app Android nativa (Kotlin + Compose) que consume la MediaWiki API de [terraria.wiki.gg](https://terraria.wiki.gg). Repo: [antonioalarconvirseda/terrariawiki](https://github.com/antonioalarconvirseda/terrariawiki). Documentación: `Kotlin Multiplatform/TerrariaWiki/`.

## Licencia

[MIT](./Kotlin%20Multiplatform/TerrariaWiki/LICENSE) — Copyright (c) 2026 Antonio Alarcón Virseda. Las notas son de aprendizaje; el código de los proyectos se licencia en cada repo.
