# Material 3 — Theme tokens

## Contexto
Compose usa un `MaterialTheme` con un `ColorScheme` que mapea roles semánticos (`primary`, `secondary`, `error`, …) a colores concretos. La alternativa naïve es usar `Color(0xFF...)` directamente en cada composable, lo cual es frágil y no escala. En TerrariaWiki además tenemos una **paleta de marca** (Terraria 1.4) y una **función de mapeo** (rareza → color) que se usa en muchas pantallas.

## Decisión
- Definir los **tokens de paleta** como `val Color` en `core/ui/theme/Color.kt` — incluyendo 4 acentos de bioma adicionales (`Corruption`, `Crystal`, `Desert`, `Abyss`, Iteración 12) y los neutros de dark mode "Cielo Nocturno" (`NightBackground`/`NightSurface`/`NightSurfaceAlt`/`NightOutline`/`StarlightWhite`, Iteración 16 — ver riesgos).
- Construir dos `ColorScheme` (light/dark) usando esos tokens en `Theme.kt`. El dark scheme comparte `primary`/`secondary`/`tertiary`/`error` con el modo claro (mismos vals `SkyTeal`/`JungleGreen`/`GoldGem`/`SlimeRed`), solo cambian los neutros de fondo/superficie.
- El mapeo semántico de rareza (nivel → color/label) se movió a un enum de dominio dedicado — ver [[Rarity-Tier-Consolidation]] — en vez de vivir como función suelta en este archivo.
- Nuevos tokens de forma y espaciado: `Shapes.kt` (esquinas 8-12dp + borde "slot de inventario" de 2dp) y `Spacing.kt` (escala 4/8/12/16/20/24dp), ambos wireados en `TerrariaWikiTheme`.
- Exponer todo vía `TerrariaWikiTheme` composable que envuelve `MaterialTheme`.
- Tipografía en `Type.kt`: sans-serif del sistema para cuerpo/labels, más una fuente pixel (**Silkscreen**, empaquetada offline en `res/font/`) reservada a `displayLarge`/`headlineLarge`/`headlineMedium`/`titleLarge`.

## Implementación Kotlin

```kotlin
// core/ui/theme/Color.kt — saturación subida ~+22% en Iteración 15, hex actuales
val SkyTeal = Color(0xFF2F9FCC)
val JungleGreen = Color(0xFF2A9022)
val GoldGem = Color(0xFFFFD03F)
val SlimeRed = Color(0xFFFF3535)
val HellOrange = Color(0xFFFF8A3D)
val CaveDark = Color(0xFF1E1E2A)
val CloudWhite = Color(0xFFF4F1E6)
val StoneGray = Color(0xFF6B7280) // neutro, sin saturar

// Acentos de bioma (categorías, etiquetas)
val Corruption = Color(0xFF6926B9)
val Crystal = Color(0xFF37EAFE)
val Desert = Color(0xFFEDBA56)
val Abyss = Color(0xFF0E3969)

// Paleta "Cielo Nocturno" (dark mode, Iteración 16) — solo neutros, los acentos son los mismos del claro
val NightBackground = Color(0xFF131A33)
val NightSurface = Color(0xFF1C2444)
val NightSurfaceAlt = Color(0xFF262F58)
val NightOutline = Color(0xFF3A4570)
val StarlightWhite = Color(0xFFEDEFFA)

// core/ui/theme/Theme.kt
private val TerrariaLightColors = lightColorScheme(
    primary = SkyTeal,
    onPrimary = CloudWhite,
    secondary = JungleGreen,
    tertiary = GoldGem,
    error = SlimeRed,
    background = CloudWhite,
    onBackground = CaveDark,
    surface = CloudWhite,
    onSurface = CaveDark
)

private val TerrariaDarkColors = darkColorScheme(
    primary = SkyTeal,
    onPrimary = StarlightWhite,
    secondary = JungleGreen,
    tertiary = GoldGem,
    onTertiary = NightBackground,
    error = SlimeRed,
    background = NightBackground,
    onBackground = StarlightWhite,
    surface = NightSurface,
    onSurface = StarlightWhite,
    outline = NightOutline
)

@Composable
fun TerrariaWikiTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit
) {
    val colorScheme = if (darkTheme) TerrariaDarkColors else TerrariaLightColors
    MaterialTheme(
        colorScheme = colorScheme,
        typography = TerrariaTypography,
        shapes = TerrariaShapes,
        content = content
    )
}
```

Uso en composables:
```kotlin
Text(
    text = item.name,
    color = MaterialTheme.colorScheme.onSurface
)
RarityChip(rarity = item.rarity) // usa RarityTier.fromLevel(item.rarity) internamente
```

## Alternativas descartadas
- **Hardcodear colores en cada composable:** imposible de mantener, no hay dark mode, no hay consistencia de marca.
- **XML `colors.xml` y temas Android viejos:** es lo que hacíamos pre-Compose. No encaja con el modelo declarativo de Compose.
- **Librería externa (e.g. `material-color-utilities`):** útil para generar paletas desde un color semilla, pero sobreingeniería para una paleta ya definida.

## Riesgos & mitigación
- **Riesgo:** `lightColorScheme` y `darkColorScheme` tienen ~30+ slots, es fácil olvidar uno y acabar con defaults grises. **Mitigación:** definir los dos schemes de forma simétrica, revisar visualmente con previews.
- ~~**Riesgo:** el dark mode "Underworld" definitivo se pospone.~~ **Resuelto en Iteración 12, revertido en Iteración 16:** la paleta lava/ceniza/obsidiana de Iteración 12 no convenció al usuario en el móvil (se veía naranja/marrón). Se reemplazó por "Cielo Nocturno": el dark mode ahora comparte `primary`/`secondary`/`tertiary`/`error` con el modo claro y solo aporta neutros de fondo/superficie en azul índigo — menos superficie de mantenimiento (una paleta de acento menos que puede desviarse) y consistencia de marca entre ambos temas.
- **Riesgo (nuevo):** la fuente Silkscreen se empaquetó como `.ttf` en `res/font/` en vez de vía Google Fonts descargable, precisamente para evitar depender de red/Play Services en runtime — decisión explícita del usuario dado que la app es una wiki de consulta offline-friendly. **Mitigación:** ya no hay riesgo de runtime, el trade-off es solo el peso del `.ttf` commiteado al repo (~60KB para 2 pesos).
