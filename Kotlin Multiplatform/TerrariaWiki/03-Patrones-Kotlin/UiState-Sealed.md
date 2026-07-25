# UiState sellado (sealed interface)

## Contexto
Una pantalla de TerrariaWiki puede estar: cargando, lista con items, vacía, o con error. Si el state es un `data class UiState(val isLoading: Boolean, val items: List<Item>?, val error: String?)`, terminamos con combinaciones inválidas (`isLoading=true` y `items=non-null` al mismo tiempo).

## Decisión
Usar una `sealed interface UiState` con una variante por estado lógico:

```kotlin
sealed interface UiState {
    data object Loading : UiState
    data class Ready(val items: List<Item>) : UiState
    data object Empty : UiState
    data class Error(val message: String) : UiState
}
```

En Compose usamos `when` exhaustivo:
```kotlin
when (val state = uiState) {
    is ItemsViewModel.UiState.Loading -> LoadingState()
    is ItemsViewModel.UiState.Error -> ErrorState(state.message, onRetry = ...)
    is ItemsViewModel.UiState.Empty -> EmptyState()
    is ItemsViewModel.UiState.Ready -> ItemsList(state.items)
}
```

El compilador exige cubrir todas las variantes; si añadimos una nueva (p.ej. `Refreshing`), el IDE marca las omisiones.

## Implementación Kotlin

```kotlin
class ItemsViewModel : ViewModel() {
    sealed interface UiState {
        data object Loading : UiState
        data class Ready(val items: List<Item>) : UiState
        data object Empty : UiState
        data class Error(val message: String) : UiState
    }
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
}
```

## Alternativas descartadas
- **Data class con booleanos/nullables:** permite estados imposibles.
- **`sealed class` en lugar de `sealed interface`:** funciona también, pero `sealed interface` permite herencia múltiple si el día de mañana queremos un `Ready` que extienda algo más.
- **Enum simple:** pierde los datos asociados (lista de items, mensaje de error).

## Riesgos & mitigación
- **Riesgo:** variantes "huérfanas" (p.ej. `Empty` sin mensaje). **Mitigación:** documentar explícitamente qué lleva cada variante en la `sealed interface`.
- **Riesgo:** `when` no exhaustivo por usar un import wildcard. **Mitigación:** usar `when` con valor (no `when {}`) para forzar exhaustividad.
