# Home + Navigation Pattern — rediseño UX de la wiki

## Contexto
La iteración 2 dejó la app con una `ItemsScreen` que mostraba una lista plana de 50 items fetched del API sin filtrar. Tras la primera validación en móvil el usuario reportó que la wiki no era "user-friendly" ni estaba bien organizada. Se tomó como referencia la web de [terraria.fandom.com/es](https://terraria.fandom.com/es/wiki/Wiki_Terraria) que organiza todo por categorías (Armas, Armaduras, Accesorios, Pociones, Bloques, ...) y subcategorías.

## Decisión

Adoptar el patrón **"Home con grid de categorías + drill-down"**:

1. **Pantalla `HomeScreen`**: grid de 2 columnas con 10 categorías (cards con icono + nombre). Tap → navega a la categoría.
2. **Pantalla `ItemsByCategoryScreen`**: LazyColumn infinite-scroll con 50 items por página. `derivedStateOf` detecta final de scroll → `loadMore()` carga el siguiente lote.
3. **Pantalla `ItemDetailScreen`**: igual que antes, accesible desde la lista de categoría.
4. **Navegación Compose**: rutas `"home"` (start) → `"category/{id}"` → `"item/{name}"`.

## Implementación Kotlin

### HomeScreen — grid 2x5

```kotlin
LazyVerticalGrid(
    columns = GridCells.Fixed(2),
    contentPadding = PaddingValues(16.dp),
    verticalArrangement = Arrangement.spacedBy(12.dp),
    horizontalArrangement = Arrangement.spacedBy(12.dp)
) {
    items(ItemCategory.entries.toList()) { category ->
        CategoryCard(category = category, onClick = { onCategoryClick(category) })
    }
}

@Composable
private fun CategoryCard(category: ItemCategory, onClick: () -> Unit) {
    val color = Color(category.colorHex)
    Card(
        onClick = onClick,
        shape = RoundedCornerShape(16.dp),
        colors = CardDefaults.cardColors(
            containerColor = color.copy(alpha = 0.12f)
        ),
        modifier = Modifier.fillMaxWidth().height(120.dp)
    ) {
        Column(
            horizontalAlignment = Alignment.CenterHorizontally,
            verticalArrangement = Arrangement.Center,
            modifier = Modifier.fillMaxSize().padding(16.dp)
        ) {
            Icon(
                imageVector = iconForCategory(category),
                contentDescription = null,
                tint = color,
                modifier = Modifier.size(40.dp)
            )
            Spacer(Modifier.height(8.dp))
            Text(
                text = category.displayName,
                style = MaterialTheme.typography.titleMedium,
                fontWeight = FontWeight.SemiBold
            )
        }
    }
}
```

### Infinite scroll — derivación reactiva

```kotlin
val shouldLoadMore by remember {
    derivedStateOf {
        val lastVisible = listState.layoutInfo.visibleItemsInfo.lastOrNull()?.index ?: 0
        val totalItems = (uiState as? UiState.Ready)?.items?.size ?: 0
        lastVisible >= totalItems - 5 && hasMore && !isLoadingMore && totalItems > 0
    }
}
LaunchedEffect(shouldLoadMore) {
    if (shouldLoadMore) viewModel.loadMore()
}
```

- `lastVisible >= totalItems - 5`: precarga cuando faltan 5 items para el final.
- `hasMore` viene del Repository (`true` si la última query devolvió 50 items).
- `isLoadingMore` previene requests duplicadas si la UI aún no se actualizó.

### Navegación con Koin + parámetro

```kotlin
viewModel: CategoryViewModel = koinViewModel(parameters = { parametersOf(category) })
```

Y en el módulo Koin:
```kotlin
viewModel { params -> CategoryViewModel(params.get<ItemCategory>(), get()) }
```

## Alternativas descartadas

- **BottomNavigation (tabs)** con todas las categorías: complica la navegación en móvil con 10+ tabs; en Android las tabs son 3-5 máximo.
- **NavigationDrawer (hamburguesa lateral)**: válido pero añade un tap extra (abrir drawer) y oculta parte de la UI. La Fandom lo usa y es un patrón válido, pero para 10 items un grid es más visual.
- **Paginación con botón "Cargar más" al final del LazyColumn**: más explícito, menos moderno, requiere un click extra.
- **Lista plana con filtro por categoría en top tabs (Material 3)**: factible pero combina 10 tabs en 10 chips pequeños — menos legible.
- **Drawer lateral con secciones expandibles**: más navegación, fricción alta.
- **Tabs verticales en lateral con scroll**: no es idiomático en Android.

## Riesgos & mitigación

- **Riesgo:** scroll rápido dispara muchos `loadMore()`. **Mitigación:** `isLoadingMore.value` se chequea antes de lanzar coroutine; ignora reentradas.
- **Riesgo:** la categoría `MISC` solapa con `FURNITURE` (mismo `apiFilter`). **Mitigación:** por ahora documentado como overlap intencional. Si hay feedback, se cambia el `apiFilter` de MISC a `miscellaneous` (no existe en la wiki, quedaría vacío). Mejor: MISC = filtro `OR` con client-side sobre la lista de furniture.
- **Riesgo:** si el usuario scrollea una categoría durante un `refresh`, `_uiState` cambia y la lista se "resetea". **Mitigación:** la toolbar de backbutton siempre permite volver al home; documentar este comportamiento.
- **Riesgo:** `parametersOf(category)` en Koin requiere `params.get<ItemCategory>()` y fallo si el tipo no encaja. **Mitigación:** `koinViewModel(parameters = ...)` propaga el error y el composable simplemente no carga; un test cubre el caso de éxito.
- **Riesgo:** rate-limit de MediaWiki con muchos scrolls rápidos. **Mitigación:** una query por `loadMore()` (50 items), no 1 query por scroll. Con throttling nativo del propio `_isLoadingMore`, en la práctica solo se dispara 1-2 veces por categoría (la mayoría de categorías devuelven < 500 items en 10-12 scrolls).

## Resultado

- **Antes:** lista plana 50 items, sin organización.
- **Después:** Home con grid 2x5 categorías. Cada categoría lista con paginación completa (10-12 páginas de 50). Total items visibles: ~5000 (10 categorías × ~500 items). Tap → detalle → back. UX muy alineada con la web Fandom de referencia.
