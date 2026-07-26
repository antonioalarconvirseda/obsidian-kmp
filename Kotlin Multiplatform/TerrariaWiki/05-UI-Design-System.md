# UI Design System — TerrariaWiki

Paleta visual y directrices de motion inspiradas en **Terraria 1.4+**. Desde el rediseño de Iteración 12: identidad más marcada del juego (fuente pixel en headers, look "slot de inventario" en cards) sin sacrificar accesibilidad/contraste — amigable primero, fiel al juego en los detalles.

---

## Paleta de colores (tokens)

### Base (modo claro — misma hue desde el MVP, saturación subida ~+22% en Iteración 15)

| Token | Hex | Rol |
|---|---|---|
| `SkyTeal` | `#2F9FCC` | Primary / fondo principal / top app bar — el "azul chulo" que el usuario señaló en la referencia de la wiki |
| `JungleGreen` | `#2A9022` | Secondary / acentos de "naturaleza" / borde de hierba (`GrassTopAccent`) |
| `GoldGem` | `#FFD03F` | Rareza legendaria / highlight / FAB hover |
| `SlimeRed` | `#FF3535` | Error / HP / rareza alta |
| `HellOrange` | `#FF8A3D` | Acento / FAB / alertas de boss (sin cambio) |
| `CaveDark` | `#1E1E2A` | Textos en modo claro (sin cambio) |
| `CloudWhite` | `#F4F1E6` | Fondo de cards / superficies (sin cambio) |
| `StoneGray` | `#6B7280` | Texto secundario / divisores — deliberadamente sin saturar, es un neutro |

Iteración 15: tras feedback de "colores más cálidos... ese azul chulo", se recalcularon estos hex subiendo saturación en HSL (~+22%) manteniendo el hue exacto de cada token — más vivos, menos "pastel", sin inventar una paleta nueva. `StoneGray` se excluyó a propósito: es el único token pensado como neutro (texto secundario, divisores) y saturarlo le habría dado un tinte, rompiendo esa función.

### Acentos de bioma (Iteración 12, recalculados en Iteración 15 — categorías, etiquetas)

| Token | Hex | Uso actual |
|---|---|---|
| `Corruption` | `#6926B9` | Categoría "Estética" (Vanity) |
| `Crystal` | `#37EAFE` | Categoría "Accesorios" |
| `Desert` | `#EDBA56` | Categoría "Bloques" |
| `Abyss` | `#0E3969` | Categoría "Mecanismos" |

### Paleta "Cielo Nocturno" (dark mode, Iteración 16 — reemplaza "Underworld" de Iteración 12)

La paleta "Underworld" (lava/ceniza/obsidiana, naranja/marrón dominante) no convenció al usuario tras probarla en el móvil. Se pivotó a un dark mode que **comparte los acentos vivos del modo claro** (`SkyTeal`/`JungleGreen`/`GoldGem`/`SlimeRed`, Iteración 15) en vez de tener una paleta de color propia — solo aporta los neutros de fondo/superficie, en azul índigo:

| Token | Hex | Rol |
|---|---|---|
| `NightBackground` | `#131A33` | Background |
| `NightSurface` | `#1C2444` | Surface |
| `NightSurfaceAlt` | `#262F58` | SurfaceVariant / contenedores |
| `NightOutline` | `#3A4570` | Outline / borde "slot de inventario" en dark |
| `StarlightWhite` | `#EDEFFA` | onBackground / onSurface |

Primary/secondary/tertiary/error en dark mode son literalmente los mismos vals que en modo claro (`SkyTeal`, `JungleGreen`, `GoldGem`, `SlimeRed`) — marca consistente entre ambos temas, sin un set de colores "solo para dark" que pueda desviarse con el tiempo. Contraste verificado manualmente: `StarlightWhite` sobre `NightBackground`/`NightSurface` y `SkyTeal` sobre `NightBackground` cumplen WCAG AA (4.5:1); `onTertiary` usa `NightBackground` (oscuro) en vez de blanco porque `GoldGem` es un amarillo claro.

### Justificación de la paleta

- **SkyTeal** evoca el cielo de Terraria 1.4.
- **JungleGreen** conecta con el bioma selvático del juego.
- **GoldGem** es universal como "rare/legendario" en la UI del propio juego.
- **SlimeRed** y **HellOrange** son acentos enérgicos (Slime King, Eye of Cthulhu, Underworld).
- **Corruption/Crystal/Desert/Abyss** dan variedad temática a categorías sin inventar hues nuevos de la nada — cada uno referencia un bioma real del juego.
- **Underworld dark mode** usa lava/ceniza/obsidiana en vez de gris genérico porque es, literalmente, el bioma que le da nombre.

---

## Rareza → color/label

Consolidado en el enum de dominio `RarityTier` (`features/items/domain/RarityTier.kt`) — ver [[03-Patrones-Kotlin/Rarity-Tier-Consolidation]] para el contexto completo de por qué existía duplicación antes y cómo se resolvió. Uso: `RarityTier.fromLevel(item.rarity).let { it.colorHex; it.textColorHex; it.label }`.

## Tipografía

- **Cuerpo/labels:** `FontFamily.SansSerif` del sistema (`TerrariaBodyFont` en `Type.kt`) — `titleMedium` hacia abajo, incluyendo los `titleSmall`/`bodySmall` que antes faltaban y caían a defaults de Material3 sin querer.
- **Títulos/headers grandes:** fuente pixel **Silkscreen** (`TerrariaDisplayFont`), empaquetada offline en `res/font/silkscreen_{regular,bold}.ttf` — aplicada solo a `displayLarge`, `headlineLarge`, `headlineMedium`, `titleLarge`. Se descartó Pixelify Sans (más pesos disponibles pero look más "cute mobile game" que fiel a la UI in-game de Terraria) y se descartó la variante descargable de Google Fonts (requiere red + Play Services la primera vez; esta es una wiki de consulta, offline-first era la prioridad).
- **Pesos:** Silkscreen solo tiene Regular/Bold, suficiente para uso exclusivo en headers.
- **Tamaño base:** 16sp, escalas Material 3 (headline 24-28sp, title 14-20sp, body 12-16sp, label 11-14sp).

## Forma y espaciado (Iteración 12)

- **`Shapes.kt`:** esquinas moderadas — `small`/`extraSmall` 8dp, `medium` 10dp, `large`/`extraLarge` 12dp. Colapsa el 8/12/16dp disperso que había antes en cada archivo.
- **Look "slot de inventario":** borde sólido de 2dp (`InventorySlotBorderWidth` + `InventorySlotBorderColor`) sobre las cards y contenedores de icono, evocando los slots del inventario del juego. Aplicado vía el componente compartido `InventorySlotCard` (`features/items/ui/components/InventorySlotCard.kt`) en vez de que cada pantalla reimplemente su propia `Card`.
- **`Spacing.kt`:** escala `xs`(4dp)/`sm`(8dp)/`md`(12dp)/`lg`(16dp)/`xl`(20dp)/`xxl`(24dp) — mismos valores que ya estaban en uso ad hoc, ahora con nombre. Aplicado en los archivos tocados por el rediseño; un barrido completo del resto del código queda como pendiente futuro, no era necesario para este trabajo.

## Motion / Animaciones

| Elemento | Animación | Duración |
|---|---|---|
| Lista de items (entrada) | `Modifier.animateItem()` con fade | 200ms |
| Búsqueda (cambio) | Crossfade entre estados | 250ms |
| Lista → Detalle | `SharedTransitionLayout` (Compose 1.7+) — fallback `AnimatedContent` | 300ms |
| Tap en card | `Ripple` por defecto de Material 3 | — |
| Skeleton de carga | `shimmer` o `Modifier.alpha` pulsante | 1200ms loop |

### SharedTransitionLayout (Compose 1.7+)

Si Compose BOM 2024.09+ está disponible, usar la API oficial `SharedTransitionLayout` con `AnimatedContent` y `with(sharedTransitionScope)` para transicionar la imagen del item entre lista y detalle. Si no, fallback a `Crossfade` simple.

## Componentes clave

### `InventorySlotCard` (Iteración 12)
Card compartida con el look "slot de inventario": `Shapes.medium` + borde sólido 2dp. Reemplaza las cuatro implementaciones privadas duplicadas que había antes (`HomeScreen.CategoryCard`, `ItemDetailScreen`'s hero/section cards, `SearchScreen.ResultCard`) — todas convergen en este único componente.

### `GrassTopAccent` (Iteración 15, retextura en Iteración 16)
Composable privado en `InventorySlotCard.kt`. La primera versión (zigzag simétrico de triángulos sólidos) se sintió "cutre" al usuario. Rediseñado como una franja de 16dp por columnas (24 columnas finas), cada una con una fracción de altura verde (`JungleGreen`) distinta pero **fija** (`GRASS_COLUMN_HEIGHTS`, no aleatoria en runtime — resultado reproducible) sobre una base marrón (`DirtBrown`), más resaltes de `GrassHighlight` en columnas puntuales — imita mejor el filo tupido e irregular del césped real que un borde geométrico limpio. Expuesto vía el parámetro `topAccent: Boolean = false` de `InventorySlotCard` — se inserta como primer hijo de la `Column` antes del `content()`, así el recorte a las esquinas redondeadas lo sigue gestionando la propia card. Solo `HomeScreen.CategoryCard` lo activa (`topAccent = true`); el resto de usos de `InventorySlotCard` (`ItemCard`, detalle, búsqueda, los widgets de Home) no lo llevan.

### Widgets de Home (Iteración 15)
`HomeScreen` añadió 2 `InventorySlotCard` de ancho completo antes del grid de categorías (vía `item(span = { GridItemSpan(maxLineSpan) })` en el mismo `LazyVerticalGrid`, evitando anidar un scroll dentro de otro):
- **`WelcomeCard`**: texto fijo de bienvenida a la wiki.
- **`LatestVersionsCard`**: lista estática `plataforma → versión` (PC/Consolas/Móvil/Switch). **Contenido hardcodeado** — la API Cargo que consume la app no expone la versión del juego, así que requiere edición manual si Terraria libera una versión nueva.

### `HomeScreen.CategoryCard` — grid 3 columnas, sprite real (Iteración 14)
Tras dos rondas de feedback ("se ve simplón" → Iteración 13 con `PixelIcon`/layout banner → "no me convence nada, sobretodo los iconos"), la tarjeta se rehizo siguiendo capturas reales de la wiki Fandom que compartió el usuario: grid de `GridCells.Fixed(3)` (antes 2), tarjeta cuadrada (`aspectRatio(1f)`) con `AsyncImage` del sprite real (48dp) + nombre debajo dentro de la misma `Column`, fondo **neutro** (sin tinte de color por categoría — se quita el `containerColor` custom, usa el default de `InventorySlotCard`). El color solo lo aporta el sprite, igual que en la referencia; el borde "slot" (`InventorySlotCard`) se mantiene sin cambios.

### `RarityChip(level: Int)`
Pill con fondo sólido (color de `RarityTier`) y borde tipo slot-inventario. Usado en lista (opcional) y en detalle (siempre).

### `ItemCard(item: Item, onClick: () -> Unit)`
`InventorySlotCard` clickeable con:
- Thumbnail izquierda (56dp con `AsyncImage`, también con borde slot-inventario)
- Nombre (titleMedium)
- Tipos/tooltip (bodySmall)
- `RarityChip` a la derecha

### `StatSection(label: String, value: String?)`
Sección reutilizable para el detalle: label en StoneGray, valor en CaveDark. Si `value == null`, oculta la fila.

---

## Material 3 mapping

```kotlin
private val TerrariaLightColors = lightColorScheme(
    primary = SkyTeal,
    onPrimary = CloudWhite,
    secondary = JungleGreen,
    onSecondary = CloudWhite,
    tertiary = GoldGem,
    onTertiary = CaveDark,
    error = SlimeRed,
    background = CloudWhite,
    onBackground = CaveDark,
    surface = CloudWhite,
    onSurface = CaveDark,
)

// "Underworld" — paleta propia, no inversión del claro (Iteración 12)
private val TerrariaDarkColors = darkColorScheme(
    primary = LavaOrange,
    onPrimary = ObsidianBlack,
    secondary = EmberRed,
    onSecondary = CinderWhite,
    tertiary = HellstoneOrange,
    onTertiary = ObsidianBlack,
    error = SlimeRed,
    onError = CinderWhite,
    background = ObsidianBlack,
    onBackground = CinderWhite,
    surface = AshSurface,
    onSurface = CinderWhite,
    surfaceVariant = AshSurfaceAlt,
    outline = MagmaCrust,
)
```

## Iconografía

- **Categorías de Home (Iteración 14):** cada `ItemCategory` lleva un `representativeImageFile: String` (ej. `"Terra Blade.png"` para Armas, `"Wood.png"` para Bloques) — la tarjeta muestra la imagen real del item vía el mismo pipeline Coil/CDN que usa `ItemCard`/`ItemThumbnail` (`buildItemImageUrl` + placeholder/error), no un icono dibujado. Decisión tomada tras ver capturas de la wiki Fandom de Terraria (el usuario las compartió porque `WebFetch`/`curl` no pueden acceder al sitio, bloqueado por Cloudflare) — esa referencia usa sprites reales del juego en un grid denso, no formas abstractas ni Material Icons genéricos.
  - Iteración 13 (intento previo: `PixelIcon` con patrones de grid dibujados a mano) quedó descartada — ver [[03-Patrones-Kotlin/Pixel-Icon-Rendering]] para el porqué, conservada como historial.
- Resto de iconografía (TopAppBar, estados): Material Symbols (filled): `search`, `arrow_back`, `clear`, `error`, `search_off`.

## Icono de la app

Tras la validación en móvil (iteración 4) el usuario pidió reemplazar el placeholder por el **Tree of Life** (Living Wood Tree), el árbol icónico de la portada oficial de Terraria 1.4.

### Iteración 4 (vector generado)
Composición del icono (vector drawable, viewport 108×108): degradado teal→verde, tronco marrón, copa de 3 capas verdes, hojas azules, castillo al fondo.

### Iteración 5 (PNG real del juego)
El usuario proveyó un PNG real (512×512 RGBA) del árbol de Terraria. El icono actual es:
- **Foreground:** `@drawable/tree_of_life` (el PNG provisto).
- **Background:** `@drawable/ic_launcher_background` (degradado teal→verde, layer-list con shape).
- **Adaptive icon:** `@mipmap-anydpi-v26/ic_launcher.xml` y `ic_launcher_round.xml` referencian ambos drawables.

**Ventajas del PNG real:**
- Identidad de marca oficial del juego (no generado).
- Mejor calidad visual a cualquier densidad.
- Más rápido de renderizar que un vector complejo.

**Trade-off:** requiere actualización manual del PNG si Terraria cambia de identidad de marca.
