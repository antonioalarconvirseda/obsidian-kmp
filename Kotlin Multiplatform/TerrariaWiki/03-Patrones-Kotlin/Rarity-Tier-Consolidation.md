---
plantilla: patrón
secciones: [Contexto, Decisión, Implementación Kotlin, Alternativas descartadas, Riesgos & mitigación]
---

# Rarity Tier Consolidation

## Contexto
La lógica de color/etiqueta de rareza vivía duplicada en dos sitios que podían desincronizarse: `Color.kt::rarityColor(level: Int)` devolvía blanco plano tanto para Master (`-1`) como para Común (`0`), mientras que `RarityChip.kt` ignoraba ese resultado para esos dos niveles y aplicaba su propio `when` (`-1` → negro casi puro, `0` → gris claro), además de mantener su propio mapa de labels en español (`rarityLabel()`) que no existía en ningún otro sitio. El resultado visible en pantalla dependía del segundo mapa, no del primero — `rarityColor()` era código muerto en la práctica para esos dos casos, y cualquier cambio futuro de paleta requería tocar dos archivos y recordar cuál ganaba.

## Decisión
Un único `RarityTier` enum en `features/items/domain/RarityTier.kt`, con `level: Int`, `colorHex: Long`, `textColorHex: Long` y `label: String`, más un factory `fromLevel(level): RarityTier` con fallback seguro a `COMMON` para valores no mapeados. `colorHex`/`textColorHex` son `Long` (no `Color` de Compose) para mantener el dominio libre de dependencias de UI — mismo criterio que `ItemCategory.colorHex`. `RarityChip` pasa a ser un consumidor simple: `RarityTier.fromLevel(rarity)` y lee `.colorHex`/`.textColorHex`/`.label`.

## Implementación Kotlin

```kotlin
// features/items/domain/RarityTier.kt
enum class RarityTier(
    val level: Int,
    val colorHex: Long,
    val textColorHex: Long,
    val label: String
) {
    MASTER(-1, 0xFF1A1A1A, 0xFFFFFFFF, "Master"),
    COMMON(0, 0xFFE5E5E5, 0xFF333333, "Común"),
    BLUE(1, 0xFF1A8FBF, 0xFFFFFFFF, "Azul"),
    // ... hasta EXPERT(11, ...)
    ;

    companion object {
        fun fromLevel(level: Int): RarityTier =
            entries.find { it.level == level } ?: COMMON
    }
}
```

```kotlin
// features/items/ui/components/RarityChip.kt (antes/después)
// Antes: val backgroundColor = when (rarity) { -1 -> ...; 0 -> ...; else -> rarityColor(rarity) }
//        val textColor = when (rarity) { -1 -> ...; 0 -> ...; else -> Color.White }
//        text = rarityLabel(rarity)
// Después:
val tier = RarityTier.fromLevel(rarity)
Box(
    modifier = modifier
        .background(Color(tier.colorHex))
        .border(InventorySlotBorderWidth, Color.Black.copy(alpha = 0.25f), shape)
) {
    Text(text = tier.label, color = Color(tier.textColorHex))
}
```

Se conservó el comportamiento del override de `RarityChip` (el que realmente se veía en pantalla: Master oscuro con texto blanco, Común gris claro con texto oscuro), no el de `rarityColor()`, porque era la elección más deliberada — distingue visualmente ambos niveles en vez de dejarlos blancos e indistinguibles.

## Alternativas descartadas
- **Sealed class en vez de enum:** no hay comportamiento por tier más allá de datos planos; el enum es más simple y da exhaustividad gratis en cualquier `when` futuro sobre `RarityTier`.
- **Mantener dos mapas sincronizados manualmente (constante compartida):** no resuelve la duplicación, solo la traslada — el objetivo era una única fuente de verdad, no dos fuentes que citan a una tercera.

## Riesgos & mitigación
- **Riesgo:** una futura actualización de Terraria añade una tier de rareza nueva (fuera del rango -1..11 actual) y requiere una nueva entrada en el enum + recompilar. **Mitigación:** `fromLevel()` cae a `COMMON` para cualquier nivel no reconocido, así que la app no crashea ni muestra un color roto mientras se prepara el release con la tier nueva.
