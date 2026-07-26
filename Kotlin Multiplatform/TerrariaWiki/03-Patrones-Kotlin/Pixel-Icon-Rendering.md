---
plantilla: patrón
secciones: [Contexto, Decisión, Implementación Kotlin, Alternativas descartadas, Riesgos & mitigación]
---

> **⚠️ Revertido en Iteración 14.** El usuario probó esta solución en móvil y no le convenció ("no me convence absolutamente nada sobretodo los iconos"). Compartió capturas de la wiki Fandom de Terraria como referencia: usa sprites reales del juego, no formas abstractas. Se sustituyó `PixelIcon` por imágenes reales de items vía el mismo pipeline Coil/CDN que ya usaba `ItemCard` — ver [[03-Patrones-Kotlin/Coil-Image-Loading]] para ese pipeline, y [[Iteraciones/Iteracion-14]] para el detalle completo de esa iteración. Esta nota se conserva como historial de la decisión descartada y su porqué, no como el estado vigente del código.

# Pixel Icon Rendering

## Contexto
Tras la Iteración 12 (rediseño visual completo), el usuario probó la app en móvil y valoró bien la paleta/tipografía/dark mode, pero señaló que las tarjetas de categoría de Home seguían viéndose "simplonas": usaban iconos genéricos de Material Icons (`Icons.Filled.Gavel`, `Shield`, `Watch`...) sin ninguna relación visual con Terraria. Se necesitaba un set de iconos con identidad propia del juego, sin recurrir a imágenes reales de items (eso habría requerido elegir y cachear una imagen del CDN por categoría, con más superficie de fallo de red que un icono estático).

## Decisión
Un composable único `PixelIcon` que renderiza un patrón de grid (12×12, `List<String>` con `'X'` = celda rellena) vía `Canvas`/`drawRect` por celda, en vez de 10 `VectorDrawable`s XML hechos a mano. Cada `ItemCategory` tiene su patrón definido como constante de texto en `PixelCategoryIcons.kt`, seleccionado por `pixelPatternFor(category)`. El color de cada icono es el mismo `colorHex` de la categoría (acentos de bioma de Iteración 12), manteniendo consistencia con el resto del sistema de color.

## Implementación Kotlin

```kotlin
// features/items/ui/components/PixelCategoryIcons.kt
@Composable
fun PixelIcon(pattern: List<String>, tint: Color, modifier: Modifier = Modifier) {
    Canvas(modifier = modifier) {
        val cell = size.minDimension / pattern.size
        val offsetX = (size.width - cell * pattern.first().length) / 2f
        val offsetY = (size.height - cell * pattern.size) / 2f
        pattern.forEachIndexed { row, line ->
            line.forEachIndexed { col, ch ->
                if (ch == 'X') drawRect(
                    color = tint,
                    topLeft = Offset(offsetX + col * cell, offsetY + row * cell),
                    size = Size(cell, cell)
                )
            }
        }
    }
}

private val SWORD_PATTERN = listOf(
    ".........XX.",
    "........XX..",
    // ... resto del patrón, silueta de espada diagonal + guarda
)
```

Uso en `HomeScreen.kt::CategoryCard`:
```kotlin
PixelIcon(
    pattern = pixelPatternFor(category),
    tint = Color(category.colorHex),
    modifier = Modifier.align(Alignment.Center).fillMaxSize(0.55f)
)
```

## Alternativas descartadas
- **10 `VectorDrawable` XML hechos a mano:** más "estándar" en Android, pero cada ajuste de silueta requiere editar XML de paths a mano; el patrón de texto plano (`"X"`/`"."`) es mucho más rápido de iterar y de leer en review.
- **Imagen real de un item representativo por categoría (vía CDN/Coil):** máxima fidelidad al juego, pero introduce dependencia de red para un elemento puramente decorativo de navegación (la Home debería renderizar instantáneo, sin esperar a Coil), y obliga a elegir y mantener 10 mapeos categoría→item concretos que podrían quedar desactualizados.
- **Librería de iconos de terceros (ej. game-icons.net vía SVG):** añade una dependencia externa y assets no versionados por el propio proyecto, para un problema que un `Canvas` nativo resuelve sin dependencias nuevas.

## Riesgos & mitigación
- **Riesgo:** los patrones 12×12 son de baja resolución; a tamaños de render muy pequeños algunas siluetas (engranaje, máscara) pueden perder legibilidad. **Mitigación:** las tarjetas de categoría en Home son grandes (160dp de alto, icono al 55% del tamaño de la tarjeta), tamaño mínimo donde el patrón es legible; si se reutiliza `PixelIcon` en un contexto más pequeño en el futuro, subir la resolución del grid (ej. 16×16) antes de reducir el tamaño de render.
- **Riesgo:** diseñar patrones a mano es subjetivo — la legibilidad real solo se confirma viendo el render en pantalla. **Mitigación:** los patrones son texto plano fácil de ajustar sin tocar Compose; pendiente de validación visual en dispositivo (este entorno no tiene `adb`).
