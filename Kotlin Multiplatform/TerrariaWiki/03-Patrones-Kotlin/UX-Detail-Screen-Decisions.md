# UX — Decisiones de la ficha de detalle

## Contexto
Tras la validación en móvil (iteración 3), el usuario reportó que la ficha de detalle de un item era **poco amigable**: labels crípticos ("ID interno", "ID wiki", "ticks"), falta de stats relevantes para jugadores de Terraria (critical, velocity, autoswing), secciones sin orden claro, chip de rareza translúcido poco visible.

Como referencia se consultó [terraria.fandom.com/es/wiki/Wiki_Terraria](https://terraria.fandom.com/es/wiki/Wiki_Terraria) para identificar qué información muestra una wiki bien organizada sobre un item:
- Cabecera con imagen + nombre + tipo.
- Descripción/tooltip corto.
- Estadísticas relevantes al tipo de item.
- Categoría (para navegación).
- Precio de compra / venta.
- Identificadores técnicos al final.

## Decisión
Rediseñar `ItemDetailScreen` con:

1. **Orden de secciones fijo y lógico:** Cabecera → Descripción → Estadísticas → Inventario → Categorías → Economía → Información. Cada sección solo se muestra si tiene datos.
2. **Labels traducidos a lenguaje del juego:**
   - "ID interno" → "Nombre en el juego" (en monospace, ej. `TerraBlade`)
   - "ID wiki" → "ID de item" (con prefijo `#`, ej. `#757`)
   - "Velocidad de uso 22 ticks" → "Velocidad de uso" + valor numérico (sin "ticks")
   - "Retroceso" se mantiene (jerga reconocible en Terraria)
3. **Stats condicionales según tipo de item:**
   - Para armas (cuando `isWeapon` o `listcat` contiene "weapon"): mostrar "Probabilidad de crítico: N%" y "Velocidad de proyectil: N".
   - "Ataque automático" se muestra solo si `autoSwing == "1"`.
4. **Secciones condicionales:** Inventario solo si `stack > 1`. Categorías solo si `listcat` no está vacío. Economía solo si hay `buy` o `sell`.
5. **Chip de rareza con color sólido:** fondo `rarityColor(level)` con texto blanco semi-bold, con `RarityChip(large = true)` en la ficha de detalle. Reemplaza el chip translúcido anterior.
6. **Hardmode badge:** pill naranja "Modo difícil" cuando `hardmode == "1"`. Solo aparece en items del modo difícil.
7. **CategoryChip vs ItemTypeChips:** los tipos (`type`) son funcionales (block, weapon, etc.) → chips verdes; las categorías de gameplay (`listcat`) son descriptivas (broadswords, craftable items) → chips en fondo secundario con borde.

> **Actualización (Iteración 21):** `DetailSection`/`StatRow` se movieron a `core/ui/components/DetailSection.kt` (compartidos con `BossDetailScreen`, ver [[03-Patrones-Kotlin/Shared-UI-Components-Core]]) y perdieron el borde de `InventorySlotCard` que tenían originalmente — cada sección es ahora un `Column` simple con título + `HorizontalDivider`, sin card propia. Un item con muchos campos poblados (receta+stats+categorías+economía+info) se veía como 5-6 cajas idénticas apiladas, más "pantalla de ajustes" que artículo de wiki; el único elemento con borde que queda en la pantalla es `DetailHeader`. El snippet de abajo refleja la versión **original** (con card por sección) — ver el código actual en `ItemDetailScreen.kt`/`core/ui/components/DetailSection.kt` para la versión vigente.

## Implementación Kotlin

```kotlin
@Composable
private fun ItemDetailContent(item: Item) {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .verticalScroll(rememberScrollState())
            .padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        DetailHeader(item = item)

        if (!item.tooltip.isNullOrBlank()) {
            DetailSection(title = "Descripción") { ... }
        }

        if (item.hasStats) {
            DetailSection(title = "Estadísticas") {
                StatRow("Daño", item.damage?.toString())
                StatRow("Defensa", item.defense?.toString())
                StatRow("Retroceso", item.knockback?.let { "%.2f".format(it) })
                StatRow("Velocidad de uso", item.useTime?.toString())
                if (item.isWeapon) {
                    StatRow("Probabilidad de crítico", item.critical?.let { "$it%" })
                    StatRow("Velocidad de proyectil", item.velocity?.let { "%.1f".format(it) })
                }
                if (item.autoSwing) StatRow("Ataque automático", "Sí")
            }
        }

        val stackDisplay = item.stack?.takeIf { it.isNotBlank() && it != "1" }?.let { "Hasta $it" }
        if (stackDisplay != null) {
            DetailSection(title = "Inventario") {
                StatRow("Apilable", stackDisplay)
            }
        }

        if (item.listCategories.isNotEmpty()) {
            DetailSection(title = "Categorías") {
                Row(horizontalArrangement = Arrangement.spacedBy(6.dp)) {
                    item.listCategories.take(4).forEach { CategoryChip(it) }
                }
            }
        }

        val priceRows = listOfNotNull(
            item.buyRaw?.let { "Precio de compra" to it },
            item.sellRaw?.let { "Precio de venta" to it }
        )
        if (priceRows.isNotEmpty()) {
            DetailSection(title = "Economía") {
                priceRows.forEach { (l, v) -> StatRow(l, v) }
            }
        }

        val infoRows = listOfNotNull(
            item.internalName?.let { "Nombre en el juego" to it },
            item.wikiId?.let { "ID de item" to "#$it" }
        )
        if (infoRows.isNotEmpty()) {
            DetailSection(title = "Información") {
                infoRows.forEach { (l, v) -> InfoRow(l, v) }
            }
        }
    }
}
```

`RarityChip` con dos variantes:
```kotlin
@Composable
fun RarityChip(rarity: Int, large: Boolean = false, modifier: Modifier = Modifier) {
    val color = rarityColor(rarity)
    Box(
        modifier = modifier
            .clip(RoundedCornerShape(if (large) 12.dp else 8.dp))
            .background(color)
            .border(width = 1.dp, color = Color.Black.copy(alpha = 0.15f), shape = ...)
            .padding(
                horizontal = if (large) 14.dp else 8.dp,
                vertical = if (large) 5.dp else 2.dp
            )
    ) {
        Text(
            text = rarityLabel(rarity),
            style = if (large) MaterialTheme.typography.labelLarge else MaterialTheme.typography.labelSmall,
            color = Color.White,
            fontWeight = FontWeight.SemiBold
        )
    }
}
```

## Alternativas descartadas

- **Tabbed layout** (tabs "Stats", "Info", "Historia"): oculta info importante tras un click extra; el móvil permite scroll largo bien.
- **Colapsar/expandir secciones** (accordion): más clicks para info que el usuario quiere ver de un vistazo.
- **Mostrar TODAS las stats siempre** (incluyendo nulls): ruido visual para items triviales como statues.
- **Hardcoded translation per rarity level** (ej. "Rare" en lugar de "Raro"): ya se tiene `rarityLabel(rarity)` con texto en español.
- **Usar `type` en lugar de `listcat` para "Categorías"**: `type` es más técnico (incluye "block^crafting material" en Wood), `listcat` son categorías de gameplay (broadswords, Melee weapons, craftable items) más legibles.

## Riesgos & mitigación

- **Riesgo:** el orden fijo puede no ser ideal para todos los items (consumibles vs armas vs armaduras). **Mitigación:** las stats condicionales por tipo (`isWeapon`) ya cubren las variantes principales. Si crece, se puede pasar a un layout por tipo en el futuro.
- **Riesgo:** `listcat` puede tener 4+ categorías para items complejos → chips saturados. **Mitigación:** `take(4)` en el render. Si hay más, se podrían unir con un "+N más".
- **Riesgo:** el "Nombre en el juego" en monospace puede no funcionar para todos los caracteres del juego. **Mitigación:** solo se muestra para items que tienen `internalName` (no null); si falla, no se renderiza.
- **Riesgo:** secciones muy cortas (1 stat) quedan visualmente pobres. **Mitigación:** usar el espaciado consistente del `DetailSection` con divider; aceptado para items minimalistas.
