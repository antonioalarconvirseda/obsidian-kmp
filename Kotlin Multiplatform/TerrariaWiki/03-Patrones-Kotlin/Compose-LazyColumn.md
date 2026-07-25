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
    items(
        items = items,
        key = { it.internalName ?: it.name }
    ) { item ->
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
- **Riesgo:** la `LazyColumn` está dentro de un `Column` → `fillMaxSize` puede no aplicarse bien. **Mitigación:** envolverla en un `Box` con `fillMaxSize`.
- **Riesgo:** imágenes en `AsyncImage` dentro de items hacen scroll janky. **Mitigación:** Coil tiene caché + placeholders; asegurar que el `model` se cachea por URL.
