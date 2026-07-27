---
plantilla: patrón
secciones: [Contexto, Decisión, Implementación Kotlin, Alternativas descartadas, Riesgos & mitigación]
---

# Componentes de UI compartidos vía `core/ui/components/`

## Contexto
Desde la Iteración 17 (features Bosses/Events), la regla "features no se importan entre sí" (ver [[03-Patrones-Kotlin/Feature-based-structure]]) se venía aplicando también a componentes de presentación puros: `bosses/ui/components/` tenía copias casi idénticas de `InventorySlotCard`, `StateScreens` (`LoadingState`/`ErrorState`/`EmptyState`) y el thumbnail bordeado (`ItemThumbnail`/`BossThumbnail`), solo porque `bosses` no podía importar de `items`. En la Iteración 21, al querer que `EventCard` (feature `events`) usara el mismo `InventorySlotCard` que Items/Bosses, esa regla se volvió insostenible: o `events` importaba de `items` (viola la regla), o se duplicaba una tercera vez.

## Decisión
La regla "features no se importan entre sí" sigue aplicando a `domain`/`data` (dominio y acceso a datos siguen sin compartirse — cada feature es dueña de su modelo). Para **componentes de UI de presentación pura, sin lógica ni tipos de dominio de una feature específica**, se centralizan en `core/ui/components/` y cualquier feature los importa libremente, igual que ya hacían con `core/ui/theme/`.

Regla práctica para decidir si un componente va a `core/ui/components/`: si el componente no referencia ningún tipo de `features.*.domain` en su firma (recibe `String`/`Dp`/lambdas en vez de `Item`/`Boss`), es candidato a `core`. Si necesita el tipo de dominio para variar su contenido de forma no genérica (ej. `ItemCard` muestra `RarityChip`, `BossCard` muestra "Vida: N" — cosas distintas por feature), el wrapper específico se queda en la feature y solo delega la parte genérica (el thumbnail, la card contenedora) a `core`.

## Implementación Kotlin

`core/ui/components/WikiThumbnail.kt` — desacoplado de `Item`/`Boss`, recibe primitivos:
```kotlin
@Composable
fun WikiThumbnail(
    imageFilename: String?,
    contentDescription: String,
    modifier: Modifier = Modifier,
    size: Dp = 56.dp
) {
    val imageModel = imageFilename?.let { buildItemImageUrl(it) }
    Box(
        modifier = modifier
            .size(size)
            .clip(RoundedCornerShape(8.dp))
            .background(MaterialTheme.colorScheme.surfaceVariant)
            .border(InventorySlotBorderWidth, InventorySlotBorderColor, RoundedCornerShape(8.dp)),
        contentAlignment = Alignment.Center
    ) {
        if (imageModel != null) {
            AsyncImage(model = imageModel, contentDescription = contentDescription, /* ... */)
        } else {
            Image(painter = painterResource(R.drawable.ic_item_placeholder), contentDescription = contentDescription, /* ... */)
        }
    }
}
```

`features/items/ui/components/ItemCard.kt` mantiene el wrapper específico, que sí conoce `Item`:
```kotlin
@Composable
fun ItemThumbnail(item: Item, size: Dp = 56.dp, modifier: Modifier = Modifier) {
    WikiThumbnail(
        imageFilename = item.imageFilename,
        contentDescription = item.name,
        modifier = modifier,
        size = size
    )
}
```

Mismo patrón para `InventorySlotCard.kt`, `StateScreens.kt` y `DetailSection.kt` — todos se movieron a `core/ui/components/` porque ninguno referenciaba `Item`/`Boss` en su firma (recibían `Composable () -> Unit`/`String` como contenido).

## Alternativas descartadas
- **Mantener la duplicación por feature**: es lo que había hasta la Iteración 20; se descartó porque cada feature nueva (NPCs, Enemies en roadmap) la repetiría, y ya causaba desvío visual real (`EventCard` con su propio `OutlinedCard`, distinto a `InventorySlotCard`).
- **Permitir que features se importen entre sí para UI** (ej. `events` importa `items.ui.components.InventorySlotCard`): se descartó porque diluye la regla de aislamiento sin ganar nada frente a subir el componente a `core` — y complica el futuro split KMP (`core` es más fácil de mover a `commonMain`/`androidMain` compartido que un grafo de imports cruzados entre features).
- **Un único `core/ui/components/Card.kt` genérico con slots para todo** (thumbnail + stats + chips configurables): se descartó por sobre-ingeniería — `ItemCard`/`BossCard` siguen siendo distintos a propósito (rareza vs vida), forzarlos a una única función paramétrica los haría menos legibles sin ahorrar líneas reales.

## Riesgos & mitigación
- **Riesgo:** `core/ui/components/` se convierte en cajón de sastre si no se aplica la regla práctica (firma sin tipos de dominio) con disciplina. **Mitigación:** antes de añadir algo a `core`, comprobar que no importa ningún tipo de `features.*.domain`; si lo necesita, se queda en la feature.
- **Riesgo:** una feature futura (NPCs, Enemies) con una UI genuinamente distinta (no un simple thumbnail+card) fuerza a `core` a volverse más genérico/parametrizado de lo necesario. **Mitigación:** no anticipar — solo se sube a `core` lo que ya está duplicado en ≥2 features reales, no lo que "podría" reutilizarse.
