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

## Iteración 5 — Icono Tree of Life real + apostrofe fix + Recetas (2026-07-25)

Tras la tercera validación en móvil, el usuario reportó 3 cosas: el icono de la app se notaba "generado por AI", muchas imágenes de items no salían (especialmente las que tienen apostrofes como Abigail's Flower), y se necesitaba mostrar los crafteos en la ficha de detalle.

GLM-5.2 (plan) auditó y Minimax M3 (build) ejecutó 7 commits en `terrariawiki`:

| # | Commit | Tipo | Cambio |
|---|---|---|---|
| 1 | `d79c1ca` | feat | Icono real Tree of Life (PNG 512×512 provisto por el usuario) |
| 2 | `f6518ef` | fix | URL-encoding del apostrofe en filenames de items |
| 3 | (siguiente) | feat | Recipes API + mapper + repository (capa de datos) |
| 4 | (siguiente) | feat | Sección "Receta" en ItemDetailScreen |
| 5 | (siguiente) | test | RecipesMapperTest con 7 casos |
| 6 | (siguiente) | fix | Build errors (Koin, Ingredient type, IC launcher background) |
| 7 | (siguiente) | test | Fix ItemDetailViewModel tests con repository mock |

### Bugs corregidos

| Bug | Causa | Síntoma |
|---|---|---|
| Icono parecía generado | Vector drawable hecho a mano, no del juego | Identidad de marca poco profesional |
| Imágenes con apostrofe no salían | `android.net.Uri.encode` no codifica `'` (RFC 3986 unreserved) | `Abigail's Flower.png` → URL rota, icono de error en Coil |

### Features nuevas

- **Sección "Receta"** en ficha de detalle (entre Descripción y Estadísticas).
- Muestra: "Se craftea N × Item", "Estación: X", "• Ingrediente ×Cantidad".
- Solo se renderiza si `recipes.isNotEmpty()`.
- Soporta múltiples recipes para el mismo item (renderiza "Receta 1", "Receta 2").

### Documentación nueva

- [[03-Patrones-Kotlin/Recipes-API-Pattern]] — parser del campo `ings` con delimitador `¦`, decisión de NO mostrar query inversa, alternativas descartadas.

### Hand-off

- Tests: **31/31** passing (14 ItemsMapper + 6 ItemsViewModel + 7 RecipesMapper + 4 CategoryViewModel).
- APK: 19 MB con el nuevo icono.
- Repo público: https://github.com/antonioalarconvirseda/terrariawiki con 19 commits.
- Vault: https://github.com/antonioalarconvirseda/obsidian-kmp con 5 commits.
- **Home con 10 categorías** + infinite scroll por categoría, icono Tree of Life, chip de rareza legible, paginación completa de la wiki.

## Iteración 4 — Rediseño UX + Home categorías + icono + fixes visuales (2026-07-25)

Tras la segunda validación en móvil, el usuario reportó 5 problemas críticos: imágenes solo en statues, label blanco invisible, icono placeholder feo, solo 50 items visibles, y necesidad de reorganización por categorías como en la wiki Fandom de referencia.

GLM-5.2 (plan) auditoría y Minimax M3 (build) ejecutaron 7 commits `fix:` / `feat:` en `terrariawiki`:

| # | Commit | Tipo | Cambio |
|---|---|---|---|
| 1 | `9f6afca` | fix | Chip rareza blanco invisible para items `rare=0` (gris claro con texto oscuro) |
| 2 | `761d57d` | feat | Icono app: Tree of Life sobre degradado teal→verde |
| 3 | `695ff72` | feat | Modelo `ItemCategory` + `queryByCategory` con `HOLDS` + paginación |
| 4 | (siguiente) | feat | HomeScreen con grid 2x5 + ItemsByCategoryScreen infinite scroll |
| 5 | (siguiente) | feat | Koin: register `CategoryViewModel` con parámetro |
| 6 | (siguiente) | test | `CategoryViewModelTest` con 4 tests |
| 7 | (siguiente) | fix | `observeByCategoryDirect` para que el VM lea state actual |

### Cambios de UX principales

- **Pantalla Home (nuevo start destination):** grid 2x5 de categorías con icono Material Symbols + color de la paleta Terraria.
- **Lista por categoría:** topbar con nombre español + back, `LazyColumn` con infinite scroll via `derivedStateOf` (precarga al faltar 5 items para el final).
- **Paginación de 50 en 50:** ~10 categorías × ~500 items = ~5000 items accesibles, vs. 50 antes.
- **Sin rate-limit:** una query por `loadMore()`, throttling por `_isLoadingMore.value` evita duplicados.

### Bugs corregidos

| Bug | Causa | Síntoma |
|---|---|---|
| Rarity chip blanco invisible | `rare=0` tenía fondo blanco y texto blanco | Statues y furniture (la mayoría de los 50 primeros items) no se distinguían |
| Solo statues con icono | Las imágenes con filenames cortos sí cargaban; las demás fallaban por la URL sin encoding | El usuario percibía "solo statues funcionan", el resto no |
| 50 items máximo | `limit=50` hardcoded sin paginación | No se podía explorar más allá de los primeros 50 |

### Documentación nueva

- [[03-Patrones-Kotlin/Cargo-HOLDS-filter]] — operador `HOLDS` para campos tipo List (alternativa a `=` que falla con MWException).
- [[03-Patrones-Kotlin/Home-Navigation-Pattern]] — rationale de la reorganización Home + drill-down con infinite scroll.
- [[05-UI-Design-System]] actualizado con sección "Icono de la app" (Tree of Life).

### Hand-off

- Tests: **24/24** passing (14 ItemsMapper + 6 ItemsViewModel + 4 CategoryViewModel).
- APK: 19 MB.
- Repo público: https://github.com/antonioalarconvirseda/terrariawiki con 12 commits.
- Vault: https://github.com/antonioalarconvirseda/obsidian-kmp con 4 commits.

Pendiente para próxima sesión GLM-5.2:
- Validación en móvil tras reinstalar APK.
- Decisión de siguiente feature: NPCs (reusar patrón), Room cache, Dark mode "Underworld", o migración real a KMP.
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

### Patrones Kotlin (24 notas, todos implementados en código)
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

## Iteración 6 — Fix imágenes vía CDN directo (2026-07-25)

Tras la cuarta validación en móvil, el usuario reportó que "las imágenes aún no cargan". GLM-5.2 (plan) auditó con probes curl y descubrió que el problema **no era el encoding de URLs**, sino el **rate-limit (429) de CloudFlare sobre `Special:Redirect`** cuando el móvil pide 50 imágenes casi simultáneamente al hacer scroll de una lista.

**El fix:** saltarse `Special:Redirect` y construir directamente la URL final `https://terraria.wiki.gg/images/<filename>`. Esa URL está cacheada en CloudFlare con `cache-control: public, max-age=31536000, immutable` y responde `cf-cache-status: HIT` instantáneamente.

Minimax M3 (build) ejecutó 2 commits:

| # | Commit | Tipo | Cambio |
|---|---|---|---|
| 1 | `471dcfe` | fix | `buildItemImageUrl` usa CDN directo, sin Special:Redirect |
| 2 | `18ab275` | test | `ImageUrlTest` cubre encoding de espacios (%20) y apostrofes (%27) |

### Hand-off

- Tests: **34/34** passing (14 ItemsMapper + 6 ItemsViewModel + 7 RecipesMapper + 4 CategoryViewModel + 3 ImageUrl).
- APK: 19 MB.
- Repo público: https://github.com/antonioalarconvirseda/terrariawiki con 21 commits.

## Iteración 7 — Fix underscore en CDN + búsqueda global desde Home (2026-07-25)

Tras la quinta validación en móvil, el usuario reportó algo raro: "iconos que se veían antes no se ven, pero otros que no se veían ahora se ven". GLM-5.2 (plan) auditó con probes curl y descubrió que el directorio `/images/` de terraria.wiki.gg requiere **underscores `_`** para espacios, no `%20` que producía `buildItemImageUrl`. Antes funcionaba porque `Special:Redirect` convertía automáticamente `%20` → `_` en el redirect final; ahora, usando la URL directa `/images/`, esa conversión no ocurre. **Fix:** reemplazar `' '` con `'_'` directamente en el encoder.

Aprovechando la iteración, se implementó la **búsqueda global** desde Home (que estaba pendiente de iteración 7 según la iteración 6). Icono de lupa en el TopAppBar del Home abre un `SearchScreen` con `OutlinedTextField` y debounce 250ms contra `action=query&list=search&srnamespace=0`. Al tap en un resultado intenta `getByName(title)`; si no es item, el `ItemDetailViewModel` muestra el error "No se encontró".

Minimax M3 (build) ejecutó 5 commits:

| # | Commit | Tipo | Cambio |
|---|---|---|---|
| 1 | `ddd161d` | fix | `buildItemImageUrl` reemplaza espacios con `_` (no `%20`) |
| 2 | `18ab275` | test | `ImageUrlTest` cubriendo underscore, apostrofes y paréntesis |
| 3 | `8bd7a81` (ya en repo) | feat | Search: domain, mapper, repository, ViewModel, Screen, nav, Koin |
| 4 | `d121f56` | test | SearchViewModelTest con Turbine |
| 5 | `5441b82` | fix | Build fixes (IconButton import) y refactor de tests con Turbine |

### Bugs corregidos

| Bug | Causa | Síntoma |
|---|---|---|
| Items con espacios no se ven en iteración 6 | `/images/` espera `_` para espacios, no `%20` | Terra Blade, Copper Broadsword, etc. no cargaban imagen |
| Búsqueda global no existía | Solo navegación por categorías | No había forma rápida de encontrar un item concreto |

### Features nuevas

- **Búsqueda global** desde Home con icono de lupa en el TopAppBar.
- Debounce 250ms para no saturar la API.
- Resultados mixtos (items + páginas conceptuales): al tap en item, abre detalle; al tap en página no-item, error en el detalle.
- Limit=25 por query (rápido).

### Documentación nueva

- [[03-Patrones-Kotlin/Search-Global-Pattern]] — patrón completo de búsqueda, debate de tap en no-items, alternativas descartadas (webview, cache local, LIKE en listcat).

### Hand-off

- Tests: **40/40** passing (14 ItemsMapper + 5 SearchViewModel + 6 ItemsViewModel + 7 RecipesMapper + 4 ImageUrl + 4 CategoryViewModel).
- APK: 19 MB.
- Repo público: https://github.com/antonioalarconvirseda/terrariawiki con 24 commits.

## Iteración 10 — User-Agent era la causa raíz (2026-07-26)

Tras iteración 9, el usuario reportó "a partir de las 11 imágenes solo cargan al rato". **Esta vez ejecutó `adb logcat -s CoilHttp:D` y compartió los logs**. El archivo `/tmp/coil_diag.txt` (44 líneas) reveló evidencia rotunda:

- 13 peticiones espaciadas ~100ms exactos → token bucket funcionaba.
- **TODAS 429 desde la primera** → no era throughput, era identificación.
- `cargoquery` por Ktor (api.php) sí funcionaba.
- La diferencia: Ktor añade `User-Agent` (`HttpClientFactory.kt:69`), Coil no.

Causa raíz: el `OkHttpClient` de Coil no mandaba `User-Agent`, así que CloudFlare bloqueaba TODAS las imágenes como si fueran bots. El token bucket había funcionado perfectamente; era otra capa la que fallaba.

Minimax M3 (build) ejecutó:

| # | Commit | Tipo | Cambio |
|---|---|---|---|
| 1 | `dadc37d` | feat | `UserAgentInterceptor` con `User-Agent: TerrariaWikiApp/0.1.0` + retry 1 intento con `Retry-After` parseo. `CoilConfig.MAX_RETRIES_429=1`, `RETRY_SLEEP_MS=500`. TokenBucket movido a throttle-only (logging va al UserAgent). |

### Hand-off

- Tests: **46/46** passing (2 UserAgent + 4 TokenBucket + 14 ItemsMapper + 5 SearchVM + 6 ItemsVM + 7 RecipesMapper + 4 ImageUrl + 4 CategoryVM).
- APK: 19 MB.
- Repos públicos:
  - Código: https://github.com/antonioalarconvirseda/terrariawiki — 28 commits.
  - Docs: https://github.com/antonioalarconvirseda/obsidian-kmp — 12 commits.

### Lección

**No iterar sin logs reales.** Iteraciones 8 y 9 asumimos "rate-limit por throughput" sin evidencia. Los logs reales mostraron que era un problema de identificación. **Siempre pedir logs antes de iterar**.

### ✅ Validación en móvil (2026-07-26)

El fix **funciona correctamente**. Verificado en el dispositivo del usuario:

- `adb logcat -s CoilHttp:D` muestra **mayoría `200`** desde la primera request. Algunos `429` aislados se recuperan con `retry -> 200`.
- Scroll continuo carga imágenes progresivamente. **El síntoma "10 primeras cargan, el resto no" ha desaparecido**.
- No hace falta Plan B (thumbnail URLs). La causa raíz era únicamente el `User-Agent` ausente.

**Notas actualizadas**:
- `[[Coil-Rate-Limit-Interceptor]]` → sección "Iteración 10" con epílogo VERIFICADO.
- `[[Debug-Lecciones-User-Agent]]` → nueva nota standalone con el patrón aprendido (3 lecciones generalizables: OkHttp sin UA, comparar headers si dos APIs se comportan distinto, pedir logs antes de iterar).

Pendiente para próxima sesión GLM-5.2:
- Decisión de siguiente feature: NPCs, Room cache, Dark mode, o migración KMP.
