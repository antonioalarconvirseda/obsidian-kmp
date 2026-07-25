# Material 3 — Theme tokens

## Contexto
Compose usa un `MaterialTheme` con un `ColorScheme` que mapea roles semánticos (`primary`, `secondary`, `error`, …) a colores concretos. La alternativa naïve es usar `Color(0xFF...)` directamente en cada composable, lo cual es frágil y no escala. En TerrariaWiki además tenemos una **paleta de marca** (Terraria 1.4) y una **función de mapeo** (rareza → color) que se usa en muchas pantallas.

## Decisión
- Definir los **tokens de paleta** como `val Color` en `core/ui/theme/Color.kt`.
- Construir dos `ColorScheme` (light/dark) usando esos tokens en `Theme.kt`.
- Centralizar la **función de mapeo semántico** (rareza → color) en el mismo archivo.
- Exponer todo vía `TerrariaWikiTheme` composable que envuelve `MaterialTheme`.
- Tipografía custom también en `Type.kt` con fuente redondeada del sistema.

## Implementación Kotlin

```kotlin
// core/ui/theme/Color.kt
val SkyTeal = Color(0xFF4A93B0)
val JungleGreen = Color(0xFF3B7C36)
val GoldGem = Color(0xFFF2C94C)
val SlimeRed = Color(0xFFE94B4B)
val HellOrange = Color(0xFFFF8A3D)
val CaveDark = Color(0xFF1E1E2A)
val CloudWhite = Color(0xFFF4F1E6)
val StoneGray = Color(0xFF6B7280)

fun rarityColor(level: Int): Color = when (level) {
    -1, 0 -> Color(0xFFFFFFFF)
    1 -> Color(0xFF1A8FBF)
    2 -> Color(0xFF3B7C36)
    // ... -1..11
    else -> StoneGray
}

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

@Composable
fun TerrariaWikiTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit
) {
    val colorScheme = if (darkTheme) TerrariaDarkColors else TerrariaLightColors
    MaterialTheme(
        colorScheme = colorScheme,
        typography = TerrariaTypography,
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
RarityChip(rarity = item.rarity) // usa rarityColor(item.rarity) internamente
```

## Alternativas descartadas
- **Hardcodear colores en cada composable:** imposible de mantener, no hay dark mode, no hay consistencia de marca.
- **XML `colors.xml` y temas Android viejos:** es lo que hacíamos pre-Compose. No encaja con el modelo declarativo de Compose.
- **Librería externa (e.g. `material-color-utilities`):** útil para generar paletas desde un color semilla, pero sobreingeniería para una paleta ya definida.

## Riesgos & mitigación
- **Riesgo:** `lightColorScheme` y `darkColorScheme` tienen ~30+ slots, es fácil olvidar uno y acabar con defaults grises. **Mitigación:** definir los dos schemes de forma simétrica, revisar visualmente con previews.
- **Riesgo:** el dark mode "Underworld" definitivo se pospone. **Mitigación:** dejar el `TerrariaDarkColors` ya con tonos de cueva como placeholder honesto; no se nota raro en dispositivos forzados a dark.
