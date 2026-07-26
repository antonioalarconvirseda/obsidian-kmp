# Compose — LazyColumn

## Contexto
TerrariaWiki muestra listas de items (potencialmente miles). Renderizar miles de Composables a la vez peta el rendimiento. Compose ofrece `LazyColumn` (equivalente a RecyclerView pero declarativo) que solo compone los ítems visibles.

## Decisión
- Usar `LazyColumn` para todas las listas.
- Proporcionar `key` estable (e.g. `item.internalName`) para que Compose reuse los slots al reordenar.
- `contentPadding` para padding respecto al top/bottom de la lista.
- `verticalArrangement` para el espaciado entre items.
- `Modifier.animateItem()` (Compose 1.7+) o `animateItemPlacement()` para animación al reordenar.

## Implementación Kotlin

```kotlin
LazyColumn(
    contentPadding = PaddingValues(horizontal = 16.dp, vertical = 8.dp),
    verticalArrangement = Arrangement.spacedBy(8.dp),
    modifier = Modifier.fillMaxSize()
) {
    itemsIndexed(
        items = items,
        key = { index, item -> "${item.internalName ?: item.name}_$index" }
    ) { _, item ->
        ItemCard(
            item = item,
            onClick = { onItemClick(item.name) }
        )
    }
    item {
        Spacer(modifier = Modifier.height(16.dp))
        Surface(
            color = MaterialTheme.colorScheme.surfaceVariant,
            shape = MaterialTheme.shapes.small
        ) {
            Text(
                text = "Mostrando ${items.size} items",
                style = MaterialTheme.typography.labelSmall,
                modifier = Modifier.padding(8.dp)
            )
        }
    }
}
```

## Alternativas descartadas
- **`Column` con `forEach`:** renderea TODO a la vez; fatal para listas grandes.
- **`LazyVerticalGrid`:** orientado a rejillas 2D; no aporta para listas de una columna.
- **RecyclerView + ViewBinding:** legacy, requiere XML, no encaja con Compose declarativo.

## Riesgos & mitigación
- **Riesgo:** keys inestables (p.ej. posición en la lista) → Compose recrea el item al reordenar. **Mitigación:** siempre usar un ID único y estable.
- **Riesgo real (2026-07-26): keys duplicadas crashean la LazyColumn.** `key = { it.internalName ?: it.name }` asume `internalName` único. La categoría Armor tenía 21+ filas "set" con `internalName == "None"` (ver [[Dto-Mapper]] — quirk de Cargo), todas colisionando en la misma key → `IllegalArgumentException: Key "None" was already used`, crash solo en esa categoría. Fix en dos capas: (1) root cause en el mapper (filtrar `"None"` antes de que llegue a la UI), (2) defensa en la propia lista — `itemsIndexed` con key compuesta `"${internalName ?: name}_$index"`, el índice como desempate garantiza unicidad aunque un futuro campo repita valores. Aplicado en `ItemsScreen.kt` e `ItemsByCategoryScreen.kt`.
- **Mismo riesgo recurrió en `BossListScreen.kt` (2026-07-26, feature Bosses):** `key = { it.name }` crasheó en dispositivo real con `IllegalArgumentException: Key "Dark Mage" was already used` — la tabla Cargo `NPCs` lista "Dark Mage" dos veces (tiers T1 y T3 del boss, mismo `nameraw`). Solo se detectó probando en el móvil (los tests unitarios con listas de mock no lo cubren, igual que la vez anterior). Fix idéntico: `itemsIndexed` con `"${boss.name}_$index"`. **Lección consolidada: cualquier `key` de `LazyColumn`/`LazyRow` derivada de un campo de la API debe asumirse no-única por defecto y llevar índice de desempate desde el primer commit, no como parche posterior — la wiki repite nombres con más frecuencia de la esperada (variantes, tiers, sets).**
- **Riesgo:** la `LazyColumn` está dentro de un `Column` → `fillMaxSize` puede no aplicarse bien. **Mitigación:** envolverla en un `Box` con `fillMaxSize`.
- **Riesgo:** imágenes en `AsyncImage` dentro de items hacen scroll janky. **Mitigación:** Coil tiene caché + placeholders; asegurar que el `model` se cachea por URL.
