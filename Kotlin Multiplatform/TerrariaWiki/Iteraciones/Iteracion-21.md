# Iteración 21 — Auditoría UX honesta + consolidación de componentes UI (2026-07-27)

## Contexto

El usuario pidió una revisión honesta, "como profesional de UX", de si la interfaz actual (que ya le gustaba) alcanzaba un estándar serio o si había margen de mejora real. Se hizo una auditoría de código completa (tema, tipografía, layout de cada feature, navegación, drawables) sin tocar nada todavía, con veredicto: base sólida y con voz visual propia (Silkscreen + `InventorySlotCard`/`GrassTopAccent` + paleta de bioma), pero con puntos concretos por debajo de "wiki pulida": componentes duplicados por feature, `Spacing` tokens usados a medias, pantallas de detalle demasiado "settings" (cajas apiladas), estados vacío/error con iconos Material genéricos, y el drawable de error de la Iteración 20 leído como una regresión de identidad visual.

El usuario validó el diagnóstico y pidió implementar primero los fixes de "alto impacto/bajo esfuerzo", y después consolidar la deuda de "medio plazo" antes de seguir sumando features (NPCs/Enemies en roadmap repetirían la misma duplicación si no se atajaba ahora).

## Decisiones

**1. Título dinámico en pantallas de detalle.** `ItemDetailScreen`/`BossDetailScreen` mostraban "Detalle" fijo en el `TopAppBar`; ahora muestran el nombre del item/boss una vez cargado (`(uiState as? UiState.Ready)?.item?.name ?: "Detalle"`), fallback solo mientras carga.

**2. `EventCard` sobre el mismo `InventorySlotCard` que Items/Bosses.** Antes usaba un `OutlinedCard` propio — visualmente distinto pese a ser el mismo concepto de tile. Esto forzó a extraer `InventorySlotCard` (antes duplicado en `items/ui/components/` y `bosses/ui/components/`) a `core/ui/components/`, porque `events` no puede importar de `items` (regla de features aislados).

**3. `ErrorState`/`EmptyState` tematizados, no iconos Material sueltos.** Antes: `Icons.Filled.Error`/`SearchOff` flotando sin contexto — rompía la inmersión que sí lograba el resto de la app. Ahora ambos van dentro de un `StateSlot`: una caja bordeada 72dp con el mismo lenguaje visual que `InventorySlotCard` (esquinas `shapes.medium`, borde `InventorySlotBorderColor`), como si fuera un slot de inventario vacío o roto.

**4. Reversión parcial de la Iteración 20: `ic_item_error.xml` vuelve a ser temático.** En la Iteración 20 se cambió el fallback de error de un hexágono rojo con círculo blanco (estilo escudo/slime) a un documento genérico gris, para que se leyera claramente como "sin imagen". El audit de esta sesión concluyó que ese cambio, aunque resolvía la confusión, era un paso atrás en identidad visual: quedaba indistinguible del placeholder normal. Se mantiene el hexágono rojo (`slime_red`) — temático y distintivo — pero ahora con un "!" blanco explícito en vez del círculo ambiguo original, para que sea inequívocamente un estado de error y no un icono de categoría.

**5. Consolidación: componentes duplicados por feature → `core/ui/components/`.** Se confirmó duplicación real entre `items/ui/components/` y `bosses/ui/components/`: `InventorySlotCard` (idéntico), `StateScreens` (`LoadingState`/`ErrorState`/`EmptyState`, solo cambiaban textos por defecto), y los thumbnails (`ItemThumbnail`/`BossThumbnail`, mismo `Box` 56dp + `AsyncImage` + placeholder/error, solo cambiaba el tipo de dominio). Se extrajeron a `core/ui/components/`: `InventorySlotCard.kt`, `StateScreens.kt` (ahora `LoadingState` exige `message` explícito, sin default hardcodeado a "items"), `WikiThumbnail.kt` (nuevo, desacoplado de `Item`/`Boss` — recibe `imageFilename`/`contentDescription`). `ItemThumbnail`/`BossThumbnail` quedan como wrappers de una línea sobre `WikiThumbnail`. Ver [[03-Patrones-Kotlin/Shared-UI-Components-Core]] para la decisión de arquitectura completa (features siguen sin compartir domain/data, pero sí UI de presentación pura vía `core`).

**6. `Spacing` tokens forzados donde ya existía un valor de la escala hardcodeado.** `ItemsByCategoryScreen`, `ItemDetailScreen`, `BossDetailScreen`, `SearchScreen`, `ItemCard`, `BossCard`: literales `8dp/12dp/16dp/20dp/24dp` reemplazados por `Spacing.sm/md/lg/xl/xxl`. No se tocaron tamaños de icono/thumbnail (48/56/64/72/128dp) ni micro-ajustes puntuales (2dp/6dp/10dp) — no son parte de la escala de espaciado.

**7. `DetailSection` rediseñado: ya no es una card con borde.** Antes cada sección de detalle (Descripción/Receta/Estadísticas/Categorías/Economía/Información) era un `InventorySlotCard` propio → un item con muchos campos apilaba 5-6 cajas idénticas, más "pantalla de ajustes" que artículo de wiki. Ahora `DetailSection` (movido a `core/ui/components/DetailSection.kt`, compartido con Bosses) es un `Column` simple: título + `HorizontalDivider` + contenido, sin borde, separado del resto solo por espaciado (`Spacing.lg`). El único elemento con borde que queda en la pantalla es el `DetailHeader` (imagen + nombre), que hace de portada. Ver actualización en [[03-Patrones-Kotlin/UX-Detail-Screen-Decisions]].

## Cambios

| Cambio | Archivo(s) | Descripción |
|---|---|---|
| fix | `ItemDetailScreen.kt`, `BossDetailScreen.kt` | Título del `TopAppBar` dinámico (nombre del item/boss) |
| fix | `EventListScreen.kt` | `EventCard` usa `InventorySlotCard` compartido en vez de `OutlinedCard` propio |
| fix | `core/ui/components/StateScreens.kt` (nuevo) | `ErrorState`/`EmptyState` con `StateSlot` bordeado en vez de icono suelto |
| fix | `res/drawable/ic_item_error.xml` | De documento genérico (Iteración 20) a hexágono rojo con "!" — temático pero inequívoco |
| refactor | `core/ui/components/InventorySlotCard.kt` (nuevo) | Extraído de `items/ui/components/` y `bosses/ui/components/` (duplicados, borrados) |
| refactor | `core/ui/components/WikiThumbnail.kt` (nuevo) | `ItemThumbnail`/`BossThumbnail` ahora delegan aquí |
| refactor | `core/ui/components/DetailSection.kt` (nuevo) | `DetailSection`/`StatRow` compartidos, sin borde (antes duplicados y con `InventorySlotCard`) |
| refactor | `ItemsByCategoryScreen.kt`, `ItemDetailScreen.kt`, `BossDetailScreen.kt`, `SearchScreen.kt`, `ItemCard.kt`, `BossCard.kt` | Literales de espaciado → `Spacing.sm/md/lg/xl/xxl` |

## Hand-off

- Verificado con `./gradlew :app:assembleDebug` (build limpio) y `./gradlew :app:testDebugUnitTest` (sin regresiones) tras cada bloque de cambios.
- No se instaló en dispositivo/emulador en esta sesión — pendiente de verificación visual manual la próxima vez que se abra Android Studio.
- Pendiente de decisión de roadmap (sin cambios): NPCs, Enemigos, cache offline con Room, o migración KMP real.
- Quedan abiertos del audit original, no abordados en esta iteración (decisión de producto, no solo visual): navegación lateral (bottom bar) antes de sumar NPCs/Enemies/Biomes; uso de `iconAsset`/`colorHex` de `ItemCategory` (hoy sin usar) como acento visual de categoría; inconsistencia de Eventos (sale a navegador externo en vez de detalle in-app).
- Commits: `fix: theme error/empty states and add dynamic detail titles`, `refactor: consolidate duplicated UI components into core/ui/components` — directos a `main`.

---

[[00-Index]]
