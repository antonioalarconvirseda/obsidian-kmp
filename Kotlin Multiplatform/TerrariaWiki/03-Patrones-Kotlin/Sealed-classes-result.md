# Sealed classes / `kotlin.Result` — modelado de estados y resultados

## Contexto
En TerrariaWiki nos encontramos con dos momentos distintos donde representar "este valor puede ser X o Y" o "esta operación puede haber funcionado o haber fallado":
1. **En el Repository / UseCase** — el resultado de una operación de red (refresh, search, getByName) puede ser éxito o fallo.
2. **En la capa de UI** — el estado de una pantalla (ViewModel) puede ser Loading, Ready, Empty o Error.

Las alternativas naïve (data class con nullables, enum simple, try/catch suelto) tienen problemas concretos:
- Data class con `isLoading: Boolean, items: List<Item>?, error: String?` permite combinaciones inválidas (`isLoading=true` con `items != null`).
- `enum class` no permite asociar datos a la variante (`Error("mensaje")`).
- `try/catch` se repite en cada `viewModelScope.launch` y se olvida capturar en algún sitio.

## Decisión
Aplicar dos técnicas, cada una en su capa:

### 1. `kotlin.Result<T>` en la capa de Repository / UseCase
Para operaciones one-shot (refresh, search, getByName) que pueden fallar.

```kotlin
// data/ItemsRepository.kt
override suspend fun refresh(): Result<Unit> = runCatching {
    val response = api.queryItems(fields = DEFAULT_FIELDS, limit = 50)
    val items = response.cargoquery.map { it.title.toDomain() }
    cacheMutex.withLock { _items.value = items }
}

override suspend fun search(query: String): Result<List<Item>> = runCatching { ... }
override suspend fun getByName(name: String): Result<Item?> = runCatching { ... }
```

El consumidor (UseCase, ViewModel) usa `result.fold(onSuccess, onFailure)` para reaccionar.

### 2. `sealed interface UiState` en la capa de ViewModel
Para el estado observable de una pantalla.

```kotlin
class ItemsViewModel(...) : ViewModel() {
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

En Compose, `when` exhaustivo garantiza que cubrimos todas las variantes:

```kotlin
when (val state = uiState) {
    is ItemsViewModel.UiState.Loading -> LoadingState()
    is ItemsViewModel.UiState.Error   -> ErrorState(state.message, onRetry = ...)
    is ItemsViewModel.UiState.Empty   -> EmptyState()
    is ItemsViewModel.UiState.Ready   -> ItemsList(state.items)
}
```

## Diferencia clave entre las dos técnicas

| Aspecto | `kotlin.Result<T>` | `sealed interface UiState` |
|---|---|---|
| Uso típico | Repository, UseCase | ViewModel, UI |
| Tipo de operación | one-shot (suspend fun) | reactiva (StateFlow) |
| Datos asociados | `T` (el valor) o `Throwable` | variantes con data class propia |
| Consumidor | `fold(onSuccess, onFailure)` | `when` exhaustivo |

## Implementación Kotlin (extracto real de TerrariaWiki)

```kotlin
// domain/UseCase — pasa el Result tal cual
class GetItemsUseCase(private val repository: ItemsRepository) {
    suspend fun refresh(): Result<Unit> = repository.refresh()
}

// ui/ItemsViewModel — mapea Result<Unit> a UiState
fun refresh() {
    viewModelScope.launch {
        _uiState.value = UiState.Loading
        getItems.refresh().fold(
            onSuccess = {
                val items = getItems().first()
                _uiState.value = if (items.isEmpty()) UiState.Empty
                else UiState.Ready(items)
            },
            onFailure = { error ->
                _uiState.value = UiState.Error(error.message ?: "Error desconocido")
            }
        )
    }
}
```

## Alternativas descartadas

- **`sealed class` en lugar de `sealed interface`:** funciona, pero `sealed interface` permite herencia múltiple si en el futuro un estado necesita extender algo más.
- **Data class con booleanos/nullables:** permite combinaciones imposibles (Loading+Ready, etc.).
- **Enum simple:** no permite asociar datos a la variante.
- **Excepciones custom selladas (`NetworkError`, `ParseError`, …):** sobreingeniería para MVP; `kotlin.Result` envuelve la `Throwable` y el `message` es suficiente.
- **Lanzar la excepción sin capturar:** deja `viewModelScope` con uncaught exception, comportamiento indefinido.
- **Crear un `Result` custom:** duplica `kotlin.Result`. Ya existe en stdlib.

## Riesgos & mitigación

- **Riesgo:** variante de UiState "huérfana" sin documentar qué lleva. **Mitigación:** documentar en el comentario de la `sealed interface` qué contiene cada variante y por qué.
- **Riesgo:** `kotlin.Result` (Kotlin 1.3+) es **inline class**; no se puede usar directamente con `suspend` en algunas APIs viejas. **Mitigación:** en Kotlin 1.5+ no hay problema; TerrariaWiki usa 2.0.20.
- **Riesgo:** `runCatching` capturaría `CancellationException` y rompería cancelación estructurada. **Mitigación:** desde Kotlin 1.5 `runCatching` NO captura `CancellationException`; sigue siendo transparente a coroutines.
- **Riesgo:** mensajes de error técnicos llegan al usuario. **Mitigación:** mapear `Throwable.message` a un mensaje amigable en el ViewModel; centralizar con un helper `getErrorMessage(throwable)` en el futuro.
- **Riesgo:** `when` no exhaustivo en Compose por usar import wildcard o `when {}` sin sujeto. **Mitigación:** usar `when (val state = uiState)` para forzar exhaustividad; el compilador avisa.
