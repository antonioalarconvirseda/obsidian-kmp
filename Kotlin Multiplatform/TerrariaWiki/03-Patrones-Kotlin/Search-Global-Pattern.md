# Search Global Pattern — búsqueda en toda la wiki desde Home

## Contexto
MVP de iteración 4 introdujo una Home con grid de 10 categorías. Cada categoría abre una lista paginada de items de ese tipo. Pero el usuario a menudo no sabe en qué categoría está un item concreto. La wiki de terraria.wiki.gg expone un endpoint de búsqueda global `action=query&list=search` que devuelve páginas por nombre/snippet en cualquier namespace.

## Decisión
Añadir búsqueda global accesible desde la Home con icono de lupa en el TopAppBar. La búsqueda usa el endpoint con `srnamespace=0` (solo páginas principales) y muestra resultados en tiempo real con debounce 250ms. Al tap en un resultado, intentamos cargar el item con `getByName(title)`; si no es un item, se muestra un Snackbar informativo.

## Implementación

### Endpoint

```bash
GET https://terraria.wiki.gg/api.php
  ?action=query
  &list=search
  &srsearch={query}
  &srlimit=25
  &srnamespace=0
  &format=json
```

Devuelve:
```json
{
  "query": {
    "searchinfo": { "totalhits": 136 },
    "search": [
      { "ns": 0, "title": "Terra Blade", "pageid": 5976, "snippet": "..." },
      ...
    ]
  }
}
```

El `snippet` viene con HTML (`<span class="searchmatch">...`) que se limpia con `stripHtml()` en el mapper.

### Domain

```kotlin
data class SearchResult(
    val title: String,
    val pageId: Int,
    val snippet: String
)
```

### Repository

```kotlin
suspend fun searchAll(query: String, limit: Int = 25): Result<List<SearchResult>>
```

### SearchViewModel

```kotlin
class SearchViewModel(repository: ItemsRepository) : ViewModel() {
    sealed interface UiState {
        data object Idle : UiState
        data object Loading : UiState
        data class Ready(val results: List<SearchResult>) : UiState
        data object Empty : UiState
        data class Error(val message: String) : UiState
    }

    @OptIn(FlowPreview::class)
    val uiState: StateFlow<UiState> = _query
        .debounce(250)
        .distinctUntilChanged()
        .mapLatest { q ->
            if (q.isBlank()) UiState.Idle
            else repository.searchAll(q).fold(
                onSuccess = { results -> if (results.isEmpty()) UiState.Empty else UiState.Ready(results) },
                onFailure = { error -> UiState.Error(error.message ?: "Error") }
            )
        }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), UiState.Idle)
}
```

### SearchScreen

TopAppBar con `OutlinedTextField` como title (input directo, no como cuerpo), icono de lupa leading, botón clear trailing, botón back navigation. Debajo según `UiState`:
- `Idle`: texto explicativo "Escribe para buscar en toda la wiki de Terraria…".
- `Loading`: spinner.
- `Empty`: EmptyState con "Sin resultados para «$query»".
- `Error`: ErrorState con retry (re-trigger typing).
- `Ready`: `LazyColumn` con `ResultCard` (title + snippet, max 2 líneas, ellipsis).

### Navegación y tap

- Ruta `search` en el NavHost.
- Tap en resultado → `navController.navigate("item/{title}")`.
- El `ItemDetailViewModel` ya hace `getByName(name)` que es el mismo lookup. Si el resultado no es un item, el detalle muestra un error "No se encontró «$title»" (redirigir a un Snackbar global queda como mejora futura).

### Koin module

```kotlin
viewModel { SearchViewModel(get()) }
```

## Alternativas descartadas

- **Webview**: abrir el HTML de la página en un WebView. Más completo pero rompe la identidad de la app y añade ~10 MB al APK.
- **Cache local de toda la wiki**: pre-cargar todos los títulos de la wiki al inicio. Cuesta startup time y memoria, no escala con la wiki (5000+ items).
- **Búsqueda solo sobre items (filtrar listcat)**: la API `cargoquery` no soporta `LIKE` sobre el campo `listcat` (mismo problema que con `ingredients` de Recipes). Inutilizable.
- **Búsqueda en cliente filtrando la lista por nombre**: solo funcionaría sobre los items cargados en memoria, no sobre toda la wiki.

## Tests añadidos (5)

- `initial state is Idle`.
- `search with empty query stays Idle`.
- `search with results emits Ready` (con Turbine y advanceTimeBy(300) para el debounce).
- `search with no results emits Empty`.
- `search failure emits Error`.

## Limitaciones conocidas

- **Mixto de items y páginas**: el resultado puede incluir páginas conceptuales (`Swords`, `Paintings`, `Attack speed`). Al tap, `ItemDetailViewModel.load` muestra el error "No se encontró" — aceptable, el usuario entiende.
- **Snippet corto**: solo 2 líneas con ellipsis. El usuario puede abrir la página de la wiki para más detalle (futuro).
- **Sin highlight de match**: el `snippet` de MediaWiki contiene `<span class="searchmatch">` para el término buscado. Por simplicidad, lo eliminamos con `stripHtml()`. Mejora futura: renderizar el highlight con AnnotatedString en Compose.

## Riesgos & mitigación

- **Riesgo:** rate-limit si se busca mucho (1 query por 250ms de debounce). **Mitigación:** el debounce ya limita la frecuencia. Además, `srnamespace=0` acota el scope a páginas principales.
- **Riesgo:** `mapLatest` cancela búsquedas en curso si llega una nueva query. **Mitigación:** intencional — solo la última query importa para el usuario.
- **Riesgo:** `stripHtml` elimina el highlight del snippet. **Mitigación:** aceptable para MVP; mejora futura con AnnotatedString.
- **Riesgo:** tap en página no-item muestra error en `ItemDetailViewModel`, no en SearchScreen. **Mitigación:** aceptable por ahora. Mejora futura: snackbar global via `SavedStateHandle` en el backstack de Navigation.
