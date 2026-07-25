# UI Design System — TerrariaWiki

Paleta visual y directrices de motion inspiradas en **Terraria 1.4+** (estilo "journey" / zen del juego actual). Estilo amigable, redondeado, sin pixel-art duro.

---

## Paleta de colores (tokens)

| Token | Hex | Rol |
|---|---|---|
| `SkyTeal` | `#4A93B0` | Primary / fondo principal / top app bar |
| `JungleGreen` | `#3B7C36` | Secondary / acentos de "naturaleza" |
| `GoldGem` | `#F2C94C` | Rareza legendaria / highlight / FAB hover |
| `SlimeRed` | `#E94B4B` | Error / HP / rareza alta |
| `HellOrange` | `#FF8A3D` | Acento / FAB / alertas de boss |
| `CaveDark` | `#1E1E2A` | Textos / futuro dark mode (Underworld) |
| `CloudWhite` | `#F4F1E6` | Fondo de cards / superficies |
| `StoneGray` | `#6B7280` | Texto secundario / divisores |

### Justificación de la paleta

- **SkyTeal** evoca el cielo de Terraria 1.4.
- **JungleGreen** conecta con el bioma selvático del juego.
- **GoldGem** es universal como "rare/legendario" en la UI del propio juego.
- **SlimeRed** y **HellOrange** son acentos enérgicos (Slime King, Eye of Cthulhu, Underworld).
- **CaveDark** permite futuro dark mode "Underworld" sin reescribir la paleta.

---

## Mapeo de rareza → color

Terraria tiene 12 niveles de rareza (0 a 11). El juego asigna un color por nivel; replicamos la convención:

```kotlin
fun rarityColor(level: Int): Color = when (level) {
    -1, 0 -> Color(0xFFFFFFFF)  // Blanco (sin rareza / master)
    1     -> Color(0xFF1A8FBF)  // Azul
    2     -> Color(0xFF3B7C36)  // Verde
    3     -> Color(0xFFF2C94C)  // Amarillo
    4     -> Color(0xFFFF8A3D)  // Naranja
    5     -> Color(0xFFE94B4B)  // Rojo claro
    6     -> Color(0xFFE94B8F)  // Rosa
    7     -> Color(0xFFB14FCF)  // Lila
    8     -> Color(0xFF8A3DCF)  // Violeta
    9     -> Color(0xFF6B7280)  // Gris (ambiguous)
    10    -> Color(0xFF4A93B0)  // SkyTeal (raro)
    11    -> Color(0xFF1ED4D4)  // Turquesa (expert/master)
    else  -> Color(0xFF6B7280)
}
```

## Tipografía

- **Fuente:** sistema (`FontFamily.Default` con `FontFamily.SansSerif` redondeada).
- **Pesos:** Regular 400 para cuerpo, Medium 500 para títulos, Bold 700 solo para stats destacadas.
- **Tamaño base:** 16sp, escalas Material 3 (headline 24sp, title 20sp, body 16sp, label 12sp).

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

### `RarityChip(level: Int)`
Pill con fondo translúcido y texto en el color de la rareza. Usado en lista (opcional) y en detalle (siempre).

### `ItemCard(item: Item, onClick: () -> Unit)`
Card Material 3 con:
- Thumbnail izquierda (64×64 con `AsyncImage`)
- Nombre (titleMedium)
- Tipos como chips pequeños (labelSmall)
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

private val TerrariaDarkColors = darkColorScheme(
    primary = SkyTeal,
    onPrimary = CaveDark,
    secondary = JungleGreen,
    onSecondary = CloudWhite,
    tertiary = GoldGem,
    onTertiary = CaveDark,
    error = SlimeRed,
    background = CaveDark,
    onBackground = CloudWhite,
    surface = CaveDark,
    onSurface = CloudWhite,
)
```

## Iconografía

- Iconos de Material Symbols (filled): `search`, `arrow_back`, `favorite`, `shopping_cart` (sell), `construction` (damage), `shield` (defense).
- Sin assets personalizados en MVP (los iconos de items vienen del wiki vía Coil).
