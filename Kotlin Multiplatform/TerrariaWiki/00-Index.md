# TerrariaWiki — Índice del proyecto

> Wiki de Terraria en app Android nativa (Kotlin + Jetpack Compose), con datos en vivo desde [terraria.wiki.gg](https://terraria.wiki.gg) (MediaWiki API + extensión Cargo).

## Estado actual ✅

**MVP: Items + Búsqueda + Detalle — COMPLETADO.**

- **Build:** `./gradlew :app:assembleDebug` → SUCCESSFUL (APK 19 MB en `app/build/outputs/apk/debug/app-debug.apk`).
- **Lint:** `./gradlew :app:lintDebug` → SUCCESSFUL.
- **Tests:** `./gradlew :app:testDebugUnitTest` → **15/15 passed** (9 ItemsMapperTest + 6 ItemsViewModelTest).
- **Repo público:** https://github.com/antonioalarconvirseda/terrariawiki (1 commit inicial en `main`, MIT license).
- **Documentación:** 29 notas en esta bóveda Obsidian (fundación + patrones + API).
- **Hand-off:** listo para que GLM-5.2 planifique la siguiente iteración.

## Cómo usar la app

1. Instalar APK en dispositivo Android 8.0+ (ver `README.md`).
2. Al abrir, la app hace `cargoquery` a terraria.wiki.gg y muestra 50 items.
3. Buscar en el campo superior (filtra en local con debounce 250ms).
4. Tap en un item → ficha de detalle con imagen, rareza, tipos, stats.
5. Botón atrás en la top bar para volver a la lista.

## Navegación de la bóveda

### Decisiones y arquitectura
- [[01-Decisiones-de-Arquitectura]] — las 7 decisiones de fondo, con su porqué
- [[02-Stack-Tecnologico]] — librerías elegidas, una por sección
- [[05-UI-Design-System]] — paleta Terraria 1.4, tokens, motion
- [[06-Setup-Entorno]] — Android Studio, JDK, SDK, adb, conexión del móvil
- [[07-GitHub-y-Publicacion]] — repo público, gh CLI, LICENSE, README

### Patrones Kotlin (20 notas, todos implementados en código)
- [[03-Patrones-Kotlin/Feature-based-structure]]
- [[03-Patrones-Kotlin/Clean-Architecture]]
- [[03-Patrones-Kotlin/Material-Theme-Tokens]]
- [[03-Patrones-Kotlin/Ktor-Setup]]
- [[03-Patrones-Kotlin/Repository-Pattern]]
- [[03-Patrones-Kotlin/Dto-Mapper]]
- [[03-Patrones-Kotlin/kotlinx-serialization]]
- [[03-Patrones-Kotlin/UseCase-Pattern]]
- [[03-Patrones-Kotlin/Sealed-classes-result]]
- [[03-Patrones-Kotlin/MVVM-StateFlow]]
- [[03-Patrones-Kotlin/UiState-Sealed]]
- [[03-Patrones-Kotlin/Compose-LazyColumn]]
- [[03-Patrones-Kotlin/Coil-Image-Loading]]
- [[03-Patrones-Kotlin/Koin-DI]]
- [[03-Patrones-Kotlin/Error-Handling-Coroutines]]
- [[03-Patrones-Kotlin/Testing-Coroutines-Turbine]]
- [[03-Patrones-Kotlin/Navigation-Compose]]
- [[03-Patrones-Kotlin/UX-Detail-Screen-Decisions]]
- [[03-Patrones-Kotlin/Cargo-HOLDS-filter]]
- [[03-Patrones-Kotlin/Home-Navigation-Pattern]]

### API de Terraria (MediaWiki + Cargo)
- [[04-API-Terraria/Endpoint-Lista]]
- [[04-API-Terraria/Query-Ejemplos]]
- [[04-API-Terraria/Cargo-envelopes]]

### Plantilla
- [[_template-patron]] — plantilla de 4 secciones para nuevos patrones

## Tree del proyecto (estado final)

```
terrariawiki/
├── .gitignore
├── LICENSE                                    (MIT)
├── README.md
├── build.gradle.kts                           (root)
├── settings.gradle.kts
├── gradle.properties
├── gradlew, gradlew.bat
├── gradle/
│   ├── libs.versions.toml                     (version catalog)
│   └── wrapper/gradle-wrapper.{jar,properties}
└── app/
    ├── build.gradle.kts
    ├── proguard-rules.pro
    └── src/
        ├── main/
        │   ├── AndroidManifest.xml
        │   ├── java/com/terrariawiki/
        │   │   ├── TerrariaWikiApp.kt         (startKoin)
        │   │   ├── MainActivity.kt            (NavHost)
        │   │   ├── core/
        │   │   │   ├── network/HttpClientFactory.kt
        │   │   │   ├── di/NetworkModule.kt
        │   │   │   ├── ui/theme/{Color,Theme,Type}.kt
        │   │   │   └── util/
        │   │   └── features/items/
        │   │       ├── data/  ({ItemsApi,ItemsApiImpl,ItemsDto,ItemsMapper,ItemsRepository}.kt)
        │   │       ├── domain/({Item,GetItemsUseCase,SearchItemsUseCase,GetItemByNameUseCase}.kt)
        │   │       ├── di/ItemsModule.kt
        │   │       └── ui/
        │   │           ├── ItemsScreen.kt
        │   │           ├── ItemsViewModel.kt
        │   │           ├── ItemDetailScreen.kt
        │   │           ├── ItemDetailViewModel.kt
        │   │           ├── components/ ({RarityChip,ItemCard,StateScreens}.kt)
        │   │           └── navigation/ItemsNavigation.kt
        │   └── res/  (themes, colors, strings, drawable, mipmap)
        └── test/  (15 unit tests, JUnit4 + MockK + Turbine)
```

## Glosario

| Término | Significado |
|---|---|
| MVP | Minimum Viable Product (primera versión funcional) |
| KMP | Kotlin Multiplatform (objetivo migratorio futuro) |
| MVVM | Model-View-ViewModel (patrón de UI) |
| DI | Dependency Injection (inyección de dependencias) |
| UseCase | Capa de casos de uso (lógica de aplicación) |
| Repository | Patrón que abstrae el origen de datos |
| StateFlow | Flujo de estado frío/caliente de Kotlin Coroutines |
| UiState | Estado inmutable de una pantalla (sealed interface) |
| MediaWiki | Motor de wiki que usa terraria.wiki.gg |
| Cargo | Extensión de MediaWiki para datos tabulares estructurados |
| DTO | Data Transfer Object (representación de capa de red) |
| Ktor | Cliente/servidor HTTP en Kotlin idiomático |

## Iteración 2 — Auditoría de GLM-5.2 y parches (2026-07-25)

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

---

## Hand-off a GLM-5.2 (próxima iteración)

El usuario conectará su móvil Android 12+ cuando quiera probar. Mientras tanto, la siguiente replanificación debería abordar una de estas opciones (en orden de recomendación):

1. **Pruebas en dispositivo físico:** una vez el usuario conecte el móvil y ejecute `adb install -r ...`, recoger feedback visual y posibles bugs de layout/UX. Documentar en una nota `08-Iteracion-1-Feedback-Movil.md`.

2. **Sección NPCs** (siguiente feature): reusar todos los patrones (Repository, UseCase, ViewModel, Compose, Koin). Documentar las micro-decisiones en sus notas correspondientes. Probablemente la API de NPCs use la tabla Cargo `NPCs`.

3. **Room como cache offline** (mejora técnica): introducir la segunda fuente de datos en `ItemsRepository`, refrescar estrategia y tests adicionales. Documentar en `03-Patrones-Kotlin/Repository-with-Multiple-Sources.md`.

4. **Dark mode "Underworld"** (mejora UX): el `TerrariaDarkColors` ya está skeleton; falta poblarlo con tonos definitivos y probar con un palette generator.

5. **Migración a KMP** (objetivo mayor): requiere refactor de Gradle a multi-module, mover `core/` y `features/items/{data,domain,di}/` a `commonMain`. Documentar el proceso en una nueva nota dedicada.

## Cómo se va actualizando esta bóveda

Cada vez que se introduce un nuevo patrón, librería o decisión, se crea (o actualiza) la nota correspondiente siguiendo [[_template-patron]]. Esto se hace en el mismo momento de la implementación ("just-in-time"), no antes, para que los snippets y referencias Kotlin sean reales y no teóricos.
