# TerrariaWiki — Índice del proyecto

> Wiki de Terraria en app Android nativa (Kotlin + Jetpack Compose), con datos en vivo desde [terraria.wiki.gg](https://terraria.wiki.gg) (MediaWiki API + extensión Cargo).

## Estado actual ✅

**MVP: Items + Búsqueda + Detalle — COMPLETADO.**

- **Build:** `./gradlew :app:assembleDebug` → SUCCESSFUL (APK 19 MB en `app/build/outputs/apk/debug/app-debug.apk`).
- **Lint:** `./gradlew :app:lintDebug` → SUCCESSFUL.
- **Tests:** `./gradlew :app:testDebugUnitTest` → **15/15 passed** (9 ItemsMapperTest + 6 ItemsViewModelTest).
- **Repo público:** https://github.com/antonioalarconvirseda/terrariawiki (1 commit inicial en `main`, MIT license).
- **Documentación:** 30 notas en esta bóveda Obsidian (fundación + patrones + API).
- Imágenes vía CDN directo cacheado en CF (sin Special:Redirect que rate-limiteaba).
- Icono Tree of Life real (PNG del juego), apostrofes en URLs corregidos, sección "Receta" en detalles.

## Historial de iteraciones

Cada iteración vive en su propia nota dentro de `Iteraciones/`, con el detalle completo de cambios, bugs corregidos y hand-off. Esta lista es solo el índice, ordenado del más reciente al más antiguo.

- **Iteración 19** (2026-07-26) — Configuración de graphify (grafo de dependencias del código) → [[Iteraciones/Iteracion-19]]
- **Iteración 18** (2026-07-26) — Audit arquitectura + fix Dependency Inversion en Repository → [[Iteraciones/Iteracion-18]]
- **Iteración 17** (2026-07-26) — Features Bosses + Events, primera ampliación más allá de Items → [[Iteraciones/Iteracion-17]]
- **Iteración 16** (2026-07-26) — Dark mode "Cielo Nocturno" (reemplaza Underworld) + hierba más fiel → [[Iteraciones/Iteracion-16]]
- **Iteración 15** (2026-07-26) — Paleta más viva, borde de hierba, widgets de Home → [[Iteraciones/Iteracion-15]]
- **Iteración 14** (2026-07-26) — Ajuste Home v2: sprites reales + grid 3 columnas (revierte Iteración 13) → [[Iteraciones/Iteracion-14]]
- **Iteración 13** (2026-07-26) — Ajuste Home: iconos pixel custom + layout banner → [[Iteraciones/Iteracion-13]]
- **Iteración 12** (2026-07-26) — Rediseño visual/UX (Underworld dark mode, Silkscreen, RarityTier, shapes/spacing) → [[Iteraciones/Iteracion-12]]
- **Iteración 11** (2026-07-26) — Fix crash en categoría Armaduras + entorno persistente → [[Iteraciones/Iteracion-11]]
- **Iteración 10** (2026-07-26) — User-Agent era la causa raíz → [[Iteraciones/Iteracion-10]]
- **Iteración 7** (2026-07-25) — Fix underscore en CDN + búsqueda global desde Home → [[Iteraciones/Iteracion-07]]
- **Iteración 6** (2026-07-25) — Fix imágenes vía CDN directo → [[Iteraciones/Iteracion-06]]
- **Iteración 5** (2026-07-25) — Icono Tree of Life real + apostrofe fix + Recetas → [[Iteraciones/Iteracion-05]]
- **Iteración 4** (2026-07-25) — Rediseño UX + Home categorías + icono + fixes visuales → [[Iteraciones/Iteracion-04]]
- **Iteración 2** (2026-07-25) — Auditoría de GLM-5.2 y parches → [[Iteraciones/Iteracion-02]]

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

### Patrones Kotlin (27 notas, todos implementados en código)
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
- [[03-Patrones-Kotlin/Recipes-API-Pattern]]
- [[03-Patrones-Kotlin/Search-Global-Pattern]]
- [[03-Patrones-Kotlin/Debug-Lecciones-User-Agent]]
- [[03-Patrones-Kotlin/Rarity-Tier-Consolidation]]
- [[03-Patrones-Kotlin/Pixel-Icon-Rendering]]
- [[03-Patrones-Kotlin/Static-Domain-Catalog]]

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
        │   │   │   ├── ui/theme/{Color,Theme,Type,Shapes,Spacing}.kt
        │   │   │   └── util/
        │   │   └── features/items/
        │   │       ├── data/  ({ItemsApi,ItemsApiImpl,ItemsDto,ItemsMapper,ItemsRepository}.kt)
        │   │       ├── domain/({Item,ItemCategory,RarityTier,GetItemsUseCase,SearchItemsUseCase,GetItemByNameUseCase}.kt)
        │   │       ├── di/ItemsModule.kt
        │   │       └── ui/
        │   │           ├── HomeScreen.kt
        │   │           ├── ItemsByCategoryScreen.kt
        │   │           ├── ItemDetailScreen.kt
        │   │           ├── ItemDetailViewModel.kt
        │   │           ├── SearchScreen.kt
        │   │           └── components/ ({RarityChip,ItemCard,InventorySlotCard,StateScreens}.kt)
        │   │   ├── features/bosses/          (iteración 17 — mirror de items/, tabla Cargo NPCs)
        │   │       ├── data/  ({BossesApi,BossesApiImpl,BossesDto,BossesMapper,BossesRepository}.kt)
        │   │       ├── domain/({Boss,GetBossesUseCase,GetBossByNameUseCase}.kt)
        │   │       ├── di/BossesModule.kt
        │   │       └── ui/
        │   │           ├── BossListScreen.kt, BossListViewModel.kt
        │   │           ├── BossDetailScreen.kt, BossDetailViewModel.kt
        │   │           └── components/ (duplicados de items/ui/components — features no se importan entre sí)
        │   │   └── features/events/          (iteración 17 — sin data/di, catálogo estático)
        │   │       ├── domain/({Event,EventCatalog}.kt)
        │   │       └── ui/EventListScreen.kt
        │   └── res/  (themes, colors, strings, drawable, mipmap, font/silkscreen_*.ttf)
        └── test/  (67 unit tests, JUnit4 + MockK + Turbine)
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

## Cómo se va actualizando esta bóveda

Cada vez que se introduce un nuevo patrón, librería o decisión, se crea (o actualiza) la nota correspondiente siguiendo [[_template-patron]]. Esto se hace en el mismo momento de la implementación ("just-in-time"), no antes, para que los snippets y referencias Kotlin sean reales y no teóricos.
