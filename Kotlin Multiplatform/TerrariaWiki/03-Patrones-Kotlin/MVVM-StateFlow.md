# MVVM con StateFlow

## Contexto
La UI de TerrariaWiki es declarativa (Jetpack Compose). Necesitamos un patrón que mantenga la pantalla sincronizada con los datos del dominio y sobreviva a recomposiciones y cambios de configuración. Las opciones históricas eran MVP (verboso) y MVVM con LiveData (atado a Android).

## Decisión
Adoptar **MVVM clásico** con:
- `ViewModel` que expone un único `StateFlow<UiState>`.
- `UiState` como `sealed interface` con todas las variantes posibles (Loading, Ready, Empty, Error).
- Compose consume el state con `collectAsStateWithLifecycle()` (respeta el ciclo de vida: no consume CPU cuando la pantalla está en background).

### Por qué StateFlow sobre LiveData
- **Multiplatform:** StateFlow es Kotlin puro; LiveData es `androidx.lifecycle`.
- **Compose-friendly:** `collectAsStateWithLifecycle()` se integra sin glue code.
- **Operadores:** `map`, `filter`, `combine`, `stateIn`, `debounce`… todo de la stdlib de coroutines.

## Implementación Kotlin

```kotlin
class ItemsViewModel(
    private val getItems: GetItemsUseCase,
    private val searchItems: SearchItemsUseCase
) : ViewModel() {

    sealed interface UiState {
        data object Loading : UiState
        data class Ready(val items: List<Item>) : UiState
        data object Empty : UiState
        data class Error(val message: String) : UiState
    }

    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    init { refresh() }

    fun refresh() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            getItems.refresh().fold(
                onSuccess = { /* ... emit Ready/Empty ... */ },
                onFailure = { e -> _uiState.value = UiState.Error(e.message ?: "Error") }
            )
        }
    }
}

// En Compose:
@Composable
fun ItemsScreen(viewModel: ItemsViewModel = koinViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    when (val state = uiState) {
        is ItemsViewModel.UiState.Loading -> LoadingState()
        is ItemsViewModel.UiState.Error -> ErrorState(state.message, onRetry = viewModel::refresh)
        is ItemsViewModel.UiState.Empty -> EmptyState()
        is ItemsViewModel.UiState.Ready -> ItemsList(state.items)
    }
}
```

### Búsqueda reactiva

```kotlin
private val _searchState = MutableStateFlow(SearchState())

@OptIn(FlowPreview::class)
val filteredItems: StateFlow<List<Item>> = _searchState
    .debounce(250)  // espera 250ms desde la última tecla
    .distinctUntilChanged { old, new -> old.query == new.query }
    .combine(getItems()) { search, items ->
        if (search.query.isBlank()) items
        else items.filter { it.name.contains(search.query, ignoreCase = true) }
    }
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList())
```

## Alternativas descartadas
- **MVI (Model-View-Intent):** el estado se gestiona con intents explícitos y un reducer. Más predecible para apps muy reactivas, pero más boilerplate. Lo descartamos por simplicidad.
- **MVP (Model-View-Presenter):** verboso, no encaja con Compose.
- **LiveData + Observer:** legacy, anclado a Android, no KMP-ready.
- **Múltiples StateFlows (uno por campo):** atomiza el estado, pierde coherencia atómica de pantalla.

## Riesgos & mitigación
- **Riesgo:** un `StateFlow` por fragmento de estado y perdemos atomicidad. **Mitigación:** un único `UiState` por pantalla, con `data class` que agrupa todos los campos.
- **Riesgo:** `collectAsStateWithLifecycle()` con lifecycle desfasado. **Mitigación:** usar la versión `lifecycle-runtime-compose` (2.8.x), que respeta el estado STOPPED.
- **Riesgo:** el VM se mantiene vivo entre pantallas y consume memoria. **Mitigación:** `viewModelScope` se cancela con el VM; coroutines colgadas no sobreviven al `onCleared()`.
