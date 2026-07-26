# Iteración 2 — Auditoría de GLM-5.2 y parches (2026-07-25)

GLM-5.2 (modo plan) auditó el MVP inicial (`0fa8ca3`) y reportó **3 bugs** que los tests unitarios no capturaban (porque no ejercitan Compose ni red real). Se aplicaron 3 commits `fix:` en `terrariawiki`:

| # | Commit | Bug | Impacto |
|---|---|---|---|
| 1 | `a7d9016` | Doble `https://` en la URL de imágenes (`ItemCard.kt:112`) | **Crítico:** ninguna imagen se cargaba en la app |
| 2 | `a347cad` | `sellRaw` mostraba HTML crudo en "Venta" (función `extractSellValue` ya existía pero no se llamaba) | UX rota: el usuario veía `<span class="coin">60 CC</span>` |
| 3 | `632d836` | `Flow.collect` infinito en `ItemsViewModel.refresh()` causaba coroutine leaked | Leve: la UI funcionaba, pero la coroutine quedaba viva para siempre |

Tras los 3 parches: `./gradlew :app:assembleDebug :app:lintDebug :app:testDebugUnitTest` → SUCCESSFUL con **17/17 tests passed** (2 tests nuevos en `ItemsMapperTest` para cubrir la limpieza de HTML del `sell`).

### Mejoras de documentación aplicadas en esta iteración

- **Nota nueva:** [[03-Patrones-Kotlin/Sealed-classes-result]] — pattern que se había prometido en el plan original pero se había omitido. Cubre `kotlin.Result` (Repository) vs `sealed interface UiState` (ViewModel) y por qué cada uno.
- **`01-Decisiones-de-Arquitectura` §3** y **`02-Stack-Tecnologico`** — añadido matiz "KMP-readiness honesto": librerías KMP-ready, integración con 3 anclajes Android pendientes (OkHttp engine, `android.util.Log`, `koin-android`). El dominio es portable tal cual a `commonMain`.
- **Conteos corregidos** en este `00-Index.md`: 26 notas totales, 17 patrones (antes decía 16).

### Hand-off (redactado al cierre de esta iteración, antes de que existieran las iteraciones 4-18)

El usuario conectará su móvil Android 12+ cuando quiera probar. Mientras tanto, la siguiente replanificación debería abordar una de estas opciones (en orden de recomendación):

1. **Pruebas en dispositivo físico:** una vez el usuario conecte el móvil y ejecute `adb install -r ...`, recoger feedback visual y posibles bugs de layout/UX. Documentar en una nota `08-Iteracion-1-Feedback-Movil.md`.

2. **Sección NPCs** (siguiente feature): reusar todos los patrones (Repository, UseCase, ViewModel, Compose, Koin). Documentar las micro-decisiones en sus notas correspondientes. Probablemente la API de NPCs use la tabla Cargo `NPCs`.

3. **Room como cache offline** (mejora técnica): introducir la segunda fuente de datos en `ItemsRepository`, refrescar estrategia y tests adicionales. Documentar en `03-Patrones-Kotlin/Repository-with-Multiple-Sources.md`.

4. **Dark mode "Underworld"** (mejora UX): el `TerrariaDarkColors` ya está skeleton; falta poblarlo con tonos definitivos y probar con un palette generator.

5. **Migración a KMP** (objetivo mayor): requiere refactor de Gradle a multi-module, mover `core/` y `features/items/{data,domain,di}/` a `commonMain`. Documentar el proceso en una nueva nota dedicada.

---

[[00-Index]]
