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

## Iteración 18 — Audit arquitectura + fix Dependency Inversion en Repository (2026-07-26)

El usuario pidió una auditoría del proyecto contra sus propias directrices (`CLAUDE.md`), la escalabilidad hacia una migración KMP real, y si el proyecto seguía la arquitectura correcta. En paralelo, decisión sobre si integrar esta bóveda Obsidian dentro del repo de código o mantenerla separada.

**Auditoría (agente Explore, contraste literal contra `CLAUDE.md`):**
- Compliant: DI Koin (`single`/`factory`/`viewModel`), `Result`/`runCatching` + `UiState` sealed, testing (MockK/Turbine, cero Mockito/`Thread.sleep`), anclajes Android confinados a `HttpClientFactory.kt`/`CoilImageLoaderFactory.kt` sin fugas a `domain`/`data`, módulo único `:app` sin `commonMain` (como documentado).
- **Hallazgo real:** `domain` importaba la interfaz `Repository` desde `data` (`GetItemsUseCase.kt`, `GetBossesUseCase.kt`, etc.) — contradecía la regla literal "`domain` nunca importa de `data`". No rompía portabilidad KMP (la interfaz es Kotlin puro) pero invertía la Dependency Inversion clásica: el *port* debe vivir en `domain`, `data` lo implementa.
- **Hallazgo secundario:** roadmap y árbol de paquetes de `CLAUDE.md` desactualizados — no mencionaban `features/bosses`/`features/events`, ya implementados desde la iteración 17.

**Decisión vault Obsidian:** mantenerla como repo separado (`obsidian-kmp`, remote propio). Razones: ya es fuente de verdad externa referenciada por path absoluto desde `CLAUDE.md` (convención establecida y funcional); Obsidian necesita su propia estructura (`.obsidian/`, graph, backlinks) que no pertenece a un repo de código Android; historiales de commits con propósitos distintos (iteraciones de doc vs. commits de código) no deben mezclarse; un futuro split KMP (`:shared`+`:android-app`) no afecta a un repo de docs ya independiente.

| Cambio | Archivo(s) | Descripción |
|---|---|---|
| refactor | `features/items/domain/ItemsRepository.kt` (nuevo), `features/items/data/ItemsRepositoryImpl.kt` (renombrado desde `ItemsRepository.kt`) | Interfaz movida a `domain/`, Impl se queda en `data/` |
| refactor | `features/bosses/domain/BossesRepository.kt` (nuevo), `features/bosses/data/BossesRepositoryImpl.kt` (renombrado) | Mismo fix, mirror de items |
| refactor | UseCases, ViewModels, `ItemsModule.kt`, `BossesModule.kt`, tests | Imports actualizados a `...domain.ItemsRepository`/`...domain.BossesRepository` |
| docs | `CLAUDE.md` | Árbol de paquetes con `features/bosses`/`features/events`, convención Repository corregida ("interfaz en `domain/`, Impl en `data/`"), roadmap sin orden fundacional obsoleto |

### Hand-off
- Tests: **67/67** passing (sin tests nuevos — refactor puro de ubicación de interfaz, mismo comportamiento).
- Build: `./gradlew :app:testDebugUnitTest` en verde tras el refactor.
- Commits: `refactor: move Repository interfaces from data to domain layer`, `docs: sync CLAUDE.md with actual package tree and roadmap state` — directo a `main`.
- Roadmap sigue abierto entre NPCs no-boss, Enemigos, cache Room, o migración KMP real; con el fix de este iteración, el contrato `Repository` ya queda 100% listo para `commonMain` sin retrabajo adicional cuando se acometa la migración.

## Iteración 17 — Features Bosses + Events, primera ampliación más allá de Items (2026-07-26)

El usuario pidió añadir dos categorías nuevas separadas visualmente de las principales: **Jefes** (con foto) y **Eventos** (con foto), pidiendo explícitamente verificar primero si la wiki podía proveer esa información antes de implementar. Se siguió el flujo `brainstorming` → `writing-plans` → implementación, con investigación real contra la API (`curl`, no suposiciones) antes de diseñar.

**Investigación previa (decisiva para el diseño):**
- `action=cargotables` reveló que Cargo solo tiene **12 tablas totales** (`Drops, Equipinfo, Exclusive, History, Imageinfo, Items, Modifiers, NPCs, Recipes, Weapon_source, _fileData, _pageData`) — no existe `Events`.
- `NPCs` con `type HOLDS 'boss'` sí trae bosses reales (Betsy, Brain of Cthulhu, Deerclops, Duke Fishron, Empress of Light...) con imagen y stats — mismo patrón que `Items`.
- Para Events se probaron 2 heurísticas de imagen y ambas fallaron en ~40% de casos (ver [[03-Patrones-Kotlin/Static-Domain-Catalog]] para el detalle): parsear `{{flavor text|...|image=}}`/primer `[[File:...]]` del wikitext, y la convención `"<PageName> Icon.png"`. Se resolvió con preguntas al usuario en 3 puntos (fuente de Events, resolución de imagen, profundidad de detalle de Bosses/Events) antes de tocar código.

| Cambio | Archivo(s) | Descripción |
|---|---|---|
| feat | `features/bosses/` (nuevo, 11 archivos) | Feature completa `data/domain/di/ui` mirroreando `items/` — `BossesApiImpl` consulta `NPCs` con `type HOLDS 'boss'`, `BossesMapper` limpia HTML + wikitext `[[...]]` de los stats multi-modo, lista + detalle |
| feat | `features/events/` (nuevo, 3 archivos) | Sin `data/`/`di/` — `EventCatalog` estático en `domain/` con 14 eventos y filenames de imagen verificados uno a uno contra la API real; tap abre la wiki en el navegador (`Intent.ACTION_VIEW`), sin pantalla de detalle in-app |
| feat | `HomeScreen.kt` | Nueva sección "Más allá de los items" (`SectionHeader` + 2 `EntryCard`) separada del grid de Items vía `item(span = maxLineSpan)`, mismo mecanismo ya usado para `WelcomeCard`/`LatestVersionsCard` |
| feat | `MainActivity.kt` | Rutas nuevas `bosses`, `boss/{name}`, `events` |
| feat | `TerrariaWikiApp.kt` | `bossesModule` registrado en Koin |

### Bugs corregidos
- **Crash real en dispositivo** (no detectado por tests): `BossListScreen` usaba `key = { it.name }` en su `LazyColumn`, y "Dark Mage" aparece dos veces en la tabla `NPCs` (tiers T1/T3) → `IllegalArgumentException: Key "Dark Mage" was already used`. Es la misma clase de bug que ya había ocurrido con `internalname == "None"` en Armor (iteración previa, ver [[03-Patrones-Kotlin/Compose-LazyColumn]]) — confirma que es un riesgo recurrente de la API, no un caso aislado. Fix: `itemsIndexed` con key compuesta `"${boss.name}_$index"`.

### Documentación nueva
- [[03-Patrones-Kotlin/Static-Domain-Catalog]] — patrón nuevo: cuándo usar un `object` estático en `domain/` en vez del patrón `data/di` completo, con la investigación de Events como caso real.
- [[03-Patrones-Kotlin/Compose-LazyColumn]] — riesgo de key duplicada actualizado con la recurrencia en Bosses, lección consolidada.
- [[04-API-Terraria/Endpoint-Lista]] — lista completa de las 12 tablas Cargo, tabla `NPCs` documentada con sus campos y el quirk de `nameraw` no-único.
- [[04-API-Terraria/Query-Ejemplos]] — 2 ejemplos nuevos: query de bosses y `cargotables`.

### Hand-off
- Tests: **67/67** passing (suite previa + `BossesMapperTest` 9 casos + `BossListViewModelTest`/`BossDetailViewModelTest` 6 casos).
- Build: `./gradlew :app:assembleDebug :app:testDebugUnitTest` en verde.
- **Validado en dispositivo físico real** (no solo build): Home con sección separada, lista de Jefes con imágenes y stats, detalle de Betsy con vida/defensa/daño/retroceso legibles, tap en Eventos abre correctamente Blood Moon / Martian Madness en el navegador. El crash de `LazyColumn` se detectó y arregló en esta misma verificación — confirma otra vez que probar en dispositivo real encuentra bugs que la suite de tests no cubre.
- Roadmap: la decisión de "NPCs" pendiente en iteraciones anteriores queda parcialmente resuelta (Bosses reusa la tabla `NPCs`), pero **NPCs no-boss (enemigos regulares) sigue sin implementar** — sigue abierta la decisión entre eso, Room cache, o migración KMP.

## Iteración 16 — Dark mode "Cielo Nocturno" (reemplaza Underworld) + hierba más fiel (2026-07-26)

Tras probar Iteración 15, el usuario reportó dos puntos: la franja de hierba se veía "cutre" (quería algo más fiel a la textura real del juego/la wiki), y el dark mode seguía predominando en naranja/marrón (paleta "Underworld" de Iteración 12) — pidió explícitamente que se eligieran colores agradables, delegando la decisión. Se confirmó por pregunta el pivote de identidad del dark mode antes de implementar.

| Cambio | Archivo(s) | Descripción |
|---|---|---|
| revert | `Color.kt` | Se eliminan los tokens "Underworld" (`ObsidianBlack`, `AshSurface`, `AshSurfaceAlt`, `LavaOrange`, `EmberRed`, `HellstoneOrange`, `MagmaCrust`, `BrimstoneYellow`, `CinderWhite`) |
| feat | `Color.kt` | Nuevos tokens "Cielo Nocturno" (`NightBackground`, `NightSurface`, `NightSurfaceAlt`, `NightOutline`, `StarlightWhite`) + `DirtBrown`/`GrassHighlight` para la hierba |
| refactor | `Theme.kt` | `TerrariaDarkColors` reconstruido: `primary`/`secondary`/`tertiary`/`error` ahora son los MISMOS vals que el modo claro (`SkyTeal`/`JungleGreen`/`GoldGem`/`SlimeRed`), solo los neutros de fondo/superficie cambian |
| refactor | `InventorySlotCard.kt` | `GrassTopAccent` rediseñado: 24 columnas finas con altura de verde variable pero fija (`GRASS_COLUMN_HEIGHTS`, reproducible) sobre base marrón, más resaltes en columnas puntuales — reemplaza el zigzag simétrico anterior |

### Documentación actualizada
- [[05-UI-Design-System]] — paleta dark reescrita ("Cielo Nocturno"), `GrassTopAccent` actualizado.
- [[03-Patrones-Kotlin/Material-Theme-Tokens]] — riesgo de Underworld marcado como revertido, snippet actualizado.

### Hand-off
- Build/tests: sin cambios de lógica de negocio, 51/51 se mantiene.
- **Pendiente:** validación visual en dispositivo — este entorno sigue sin `adb`.
- Lección repetida: el dark mode "Underworld" ya había pasado por una ronda de diseño propio (Iteración 12) antes de que el usuario lo viera en pantalla real y lo rechazara — ratifica la lección de Iteración 14 sobre validar con referencias/capturas antes de invertir en una identidad visual completa.

## Iteración 15 — Paleta más viva, borde de hierba, widgets de Home (2026-07-26)

El grid de categorías con sprites reales (Iteración 14) convenció más, pero el usuario pidió seguir acercándose a la referencia: colores más vivos ("ese azul chulo" de la wiki, no pastel), el detalle de borde de hierba encima de las tarjetas de categoría, y más contenido en Home (mensaje de bienvenida + últimas versiones, como los widgets de dashboard de la wiki). Se validaron los 3 puntos con preguntas antes de implementar.

| Cambio | Archivo(s) | Descripción |
|---|---|---|
| feat | `Color.kt` | Saturación de la paleta base + acentos de bioma subida ~+22% en HSL (mismo hue); `StoneGray` excluido a propósito (es el único token neutro) |
| feat | `InventorySlotCard.kt` | Parámetro `topAccent: Boolean` + `GrassTopAccent` (Canvas, franja verde con borde zigzag) |
| feat | `HomeScreen.kt` | `CategoryCard` activa `topAccent = true`; nuevos `WelcomeCard`/`LatestVersionsCard` como items de ancho completo (`GridItemSpan(maxLineSpan)`) antes del grid de categorías |

### Documentación actualizada
- [[05-UI-Design-System]] — tabla de paleta con los nuevos hex, `GrassTopAccent` y los widgets de Home documentados.

### Hand-off
- Build/tests: sin cambios de lógica de negocio, 51/51 se mantiene.
- **Nota:** `LatestVersionsCard` tiene contenido hardcodeado (no hay endpoint de versión en la API Cargo) — recordar actualizarlo a mano si Terraria libera una versión nueva.
- **Pendiente:** validación visual en dispositivo — este entorno sigue sin `adb`.

## Iteración 14 — Ajuste Home v2: sprites reales + grid 3 columnas (revierte Iteración 13) (2026-07-26)

El usuario probó la Iteración 13 (iconos `PixelIcon` + layout banner) en móvil y no le convenció en absoluto, especialmente los iconos abstractos. Pidió mirar la wiki Fandom de Terraria (`terraria.fandom.com/es/wiki/Wiki_Terraria`) como referencia directa. `WebFetch` devolvió 402 y `curl` directo (con varios User-Agent) devolvió 403 — Cloudflare bloquea el scraping del sitio — así que el usuario compartió 2 capturas de pantalla, que sí se pudieron analizar visualmente.

La referencia mostraba: grid denso de 6 columnas con tarjetas pequeñas cuadradas, **sprites reales del juego** (no formas dibujadas) arriba + nombre debajo, fondo neutro oscuro igual para todas las categorías (el color viene del propio sprite, no de un tinte por categoría). Se validaron las decisiones con el usuario ANTES de implementar esta vez (lección aprendida de Iteración 13, donde se implementó sin confirmar visualmente primero).

| Cambio | Archivo(s) | Descripción |
|---|---|---|
| revert | `PixelCategoryIcons.kt` (eliminado) | Se descarta el enfoque de iconos dibujados a mano de Iteración 13 |
| feat | `ItemCategory.kt` | Nuevo campo `representativeImageFile: String` — un item real por categoría (`"Terra Blade.png"`, `"Wood.png"`, etc.) |
| refactor | `HomeScreen.kt` | Grid `Fixed(2)`→`Fixed(3)`; `CategoryCard` vuelve a `Column` compacta (cuadrada, `aspectRatio(1f)`) con `AsyncImage` real vía Coil/CDN en vez de icono dibujado; fondo de tarjeta vuelve a neutro (sin tinte de bioma) |

### Documentación actualizada
- [[03-Patrones-Kotlin/Pixel-Icon-Rendering]] — marcada como revertida al inicio, con el motivo, se conserva como historial.
- [[05-UI-Design-System]] — iconografía y `CategoryCard` actualizados de nuevo.

### Hand-off
- Build/tests: sin cambios de lógica de negocio, 51/51 se mantiene.
- **Riesgo abierto:** solo `"Terra Blade.png"` y `"Wood.png"` son filenames verificados (ya usados en tests existentes); el resto de `representativeImageFile` son nombres de items conocidos pero no verificados contra el CDN real desde este entorno (sin acceso de red a imágenes). Si alguna categoría muestra el icono de error en el móvil, es un ajuste de una línea en `ItemCategory.kt` una vez el usuario confirme el nombre exacto.
- **Pendiente:** validación visual en dispositivo — este entorno sigue sin `adb`.
- Lección para iteraciones futuras de UI: cuando el usuario tenga una referencia visual concreta (captura, URL), pedirla y validarla ANTES de implementar, no después — evita una ronda de trabajo descartado.

## Iteración 13 — Ajuste Home: iconos pixel custom + layout banner (2026-07-26)

Tras probar la Iteración 12 en móvil, el usuario confirmó que le gustó el resultado global (fuentes y descripciones ampliadas "encantaron"), pero señaló que las tarjetas de categoría de Home seguían viéndose "simplonas": iconos genéricos de Material Icons sin relación con el juego, y grid uniforme poco editorial. Compartió como referencia visual la wiki Fandom de Terraria (no se pudo cargar directamente, `WebFetch` devolvió 402), así que el criterio se resolvió por preguntas directas de alcance.

Decisiones confirmadas: (1) set de iconos vectoriales pixel/fantasy propio, no Material Icons ni imágenes reales de items; (2) grid 2 columnas con tarjetas tipo banner (icono grande + nombre superpuesto con gradiente), no grid simétrico icono+label.

| Cambio | Archivo(s) | Descripción |
|---|---|---|
| feat | `PixelCategoryIcons.kt` (nuevo) | `PixelIcon` composable (`Canvas`/`drawRect` por celda) + 10 patrones 12×12, uno por `ItemCategory` |
| refactor | `HomeScreen.kt` | `CategoryCard` reescrita a layout banner (`Box` + `PixelIcon` + gradiente + label), altura 120dp→160dp; eliminada `iconForCategory()` y los 9 imports de Material Icons que solo esa función usaba |

### Documentación nueva
- [[03-Patrones-Kotlin/Pixel-Icon-Rendering]] — nueva nota de patrón para el mecanismo de render pixel-art vía `Canvas`.
- [[05-UI-Design-System]] — sección "Iconografía" y componentes actualizados.

### Hand-off
- Build/tests: sin cambios de lógica de negocio, 51/51 se mantiene.
- **Pendiente:** validación visual en dispositivo — este entorno sigue sin `adb`. Los 10 patrones pixel son texto plano fácil de ajustar si alguna silueta no se lee bien a 160dp, o si el contraste del texto blanco falla sobre algún acento claro (`Crystal`, `Desert`).

## Iteración 12 — Rediseño visual/UX (Underworld dark mode, Silkscreen, RarityTier, shapes/spacing) (2026-07-26)

El usuario pidió un rediseño completo de la interfaz con criterio UX profesional: amigable/accesible pero fiel a la identidad visual de Terraria. Tras un brainstorming con preguntas de alcance (navegación fuera de este trabajo, dark mode Underworld sí se implementa, código muerto se elimina, fuente pixel solo en headers y empaquetada offline, paleta se amplía con acentos de bioma aplicados ya a categorías, shapes en look "slot de inventario"), Claude Code (Sonnet 5, plan + build en la misma sesión) ejecutó el rediseño.

| Cambio | Archivo(s) | Descripción |
|---|---|---|
| chore | `ItemsScreen.kt`, `navigation/ItemsNavigation.kt` | Eliminado código muerto (no referenciado desde `MainActivity` desde la reorganización Home de iteración 4) |
| feat | `Color.kt` | 4 acentos de bioma (`Corruption`/`Crystal`/`Desert`/`Abyss`) + paleta Underworld dark (`ObsidianBlack`/`AshSurface`/`LavaOrange`/`EmberRed`/`MagmaCrust`/`CinderWhite`) |
| feat | `Theme.kt` | `TerrariaDarkColors` reconstruido con paleta Underworld real (antes reusaba tonos del claro); `shapes = TerrariaShapes` wireado |
| feat | `Shapes.kt` (nuevo) | Esquinas 8-12dp + `InventorySlotBorderWidth`/`Color` para el borde "slot de inventario" |
| feat | `Spacing.kt` (nuevo) | Escala `xs..xxl` (4-24dp) nombrando los valores ad hoc ya en uso |
| feat | `Type.kt` | Fuente pixel Silkscreen (`res/font/silkscreen_{regular,bold}.ttf`, empaquetada offline) en headers; `titleSmall`/`bodySmall` añadidos (faltaban, caían a default M3) |
| refactor | `RarityTier.kt` (nuevo, domain) | Enum único de rareza (color+label+level), reemplaza `Color.kt::rarityColor()` + overrides locales de `RarityChip.kt` |
| feat | `InventorySlotCard.kt` (nuevo) | Card compartida con shape+borde, reemplaza 4 implementaciones privadas duplicadas (`HomeScreen`, `ItemDetailScreen` x2, `SearchScreen`) |
| feat | `ItemCategory.kt` | Recolor de 4 categorías a los nuevos acentos de bioma (Vanity→Corruption, Accesorios→Crystal, Bloques→Desert, Mecanismos→Abyss) |
| test | `RarityTierTest.kt` (nuevo) | 4 casos: `fromLevel(-1)`, `fromLevel(0)`, `fromLevel(11)`, fallback a `COMMON` para nivel no mapeado |

### Bugs corregidos
- Duplicación/inconsistencia de color y label de rareza entre `Color.kt` y `RarityChip.kt`.
- Nombre engañoso `RoundedFont` (era solo `FontFamily.SansSerif`, no una fuente redondeada real).
- `titleSmall`/`bodySmall` ausentes en `TerrariaTypography`, caían silenciosamente a defaults de Material3.
- Dark mode "Underworld" era solo una inversión fondo/texto del modo claro, no una paleta propia.

### Features nuevas
- Dark mode "Underworld" con paleta lava/ceniza/obsidiana real.
- Fuente pixel Silkscreen en títulos/headers, empaquetada offline (sin dependencia de red/Play Services).
- Look "slot de inventario" (borde sólido + esquinas moderadas) vía componente compartido `InventorySlotCard`.
- 4 acentos de bioma nuevos, aplicados ya a colores de categoría en Home.

### Documentación nueva
- [[03-Patrones-Kotlin/Rarity-Tier-Consolidation]] — nueva nota de patrón para el enum `RarityTier`.
- [[03-Patrones-Kotlin/Material-Theme-Tokens]] actualizado — paleta ampliada, Shapes/Spacing, riesgo "Underworld pospuesto" cerrado como resuelto.
- [[05-UI-Design-System]] reescrito — paleta completa, tipografía, nueva sección "Forma y espaciado", Material3 mapping actualizado, rareza apunta a la nota nueva.

### Hand-off
- Tests: **51/51** passing (suite previa + 4 `RarityTierTest` nuevos).
- Build: `./gradlew :app:assembleDebug` y `:app:testDebugUnitTest` en verde.
- **Pendiente:** validación visual en dispositivo físico — este entorno no tiene `adb` disponible, no se pudo instalar el APK ni recorrer las pantallas a ojo. Recomendado antes de dar el rediseño por cerrado.
- Pendiente (fuera de alcance, anotado): barrido completo de `.dp` literales al nuevo `Spacing` en archivos no tocados por este trabajo.
- Decisión de siguiente feature sigue abierta: NPCs, Room cache, o migración KMP (dark mode ya no está en esta lista, se resolvió aquí).

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

## Iteración 11 — Fix crash en categoría Armaduras + entorno persistente (2026-07-26)

El usuario reportó: "cuando le doy a la sección de armaduras la aplicación crashea, solo ocurre con eso". Investigado con agente Explore de solo lectura + query directa a la API real de Cargo (`where=type HOLDS 'armor'`, 100 filas), siguiendo la regla de [[Debug-Lecciones-User-Agent]] de nunca iterar sin evidencia real.

**Causa raíz:** Cargo devuelve el string literal `"None"` (no `null` JSON) para campos escalares sin valor. Armor incluye, además de las piezas individuales, una fila "set" agregada por cada armadura completa (`"Adamantite armor"`, `"Bee armor"`...) con `internalname = "None"`. El mapper (`?.takeIf { it.isNotBlank() }`) no filtraba ese string, así que 21+ items terminaban con `internalName == "None"`, y la key de `LazyColumn` (`key = { it.internalName ?: it.name }`) exige unicidad → `IllegalArgumentException: Key "None" was already used`, crash solo en Armor (la única categoría con filas "set").

Claude Code (Sonnet 5, plan + build en la misma sesión) ejecutó el fix vía TDD:

| Cambio | Archivo | Descripción |
|---|---|---|
| fix | `ItemsMapper.kt` | Helper `cargoValueOrNull()` filtra blank y `"None"` literal; aplicado a `internalname`, `imagefile`, `stack`, `tooltip` |
| test | `ItemsMapperTest.kt` | Caso `internalname/imagefile/stack = "None"` → deben mapear a `null` (falló antes del fix, pasó después) |
| fix | `ItemsScreen.kt`, `ItemsByCategoryScreen.kt` | `items` → `itemsIndexed`, key compuesta `"${internalName ?: name}_$index"` como defensa adicional ante duplicados futuros |

### Documentación actualizada
- [[03-Patrones-Kotlin/Dto-Mapper]] — nueva particularidad "Cargo `\"None\"` literal" + `cargoValueOrNull()` en el snippet.
- [[03-Patrones-Kotlin/Compose-LazyColumn]] — nuevo riesgo real documentado (keys duplicadas) + snippet actualizado a `itemsIndexed`.

### Entorno de desarrollo (fuera de este repo de código)
Se detectaron dos problemas de persistencia de entorno en el Mac del usuario (no relacionados con el bug de Armor): `JAVA_HOME` y `adb` solo estaban disponibles dentro de la sesión de la herramienta, no en `~/.zshrc`, así que cada terminal nueva fallaba con "Unable to locate a Java Runtime" y luego "adb: command not found". Fix: añadidas 2 secciones a `~/.zshrc` — `JAVA_HOME` apuntando al JBR embebido de Android Studio (`/Applications/Android Studio.app/Contents/jbr/Contents/Home`), y `ANDROID_HOME` + `platform-tools` en `PATH`. Verificado con `zsh -ic` (nota: `zsh -lc` NO sourcea `.zshrc`, solo shells interactivos lo hacen).

### Hand-off
- Tests: mapper suite en verde tras el fix (caso `"None"` añadido).
- Pendiente: confirmación del usuario en dispositivo físico tras `adb install -r ...` abriendo la sección Armaduras.
- Decisión de siguiente feature aún abierta: NPCs, Room cache, Dark mode, o migración KMP.
