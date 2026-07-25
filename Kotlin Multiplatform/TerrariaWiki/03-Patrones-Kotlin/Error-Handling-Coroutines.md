# Manejo de errores en coroutines

## Contexto
Ktor (con `expectSuccess = true`) lanza excepciones en 4xx/5xx. Sin captura, la coroutine muere silenciosamente. Si capturamos todo con `try/catch` en cada `viewModelScope.launch`, repetimos código y olvidamos capturar en algún sitio.

## Decisión
Patrón **"capturar en el Repository, propagar como `Result<T>`"**:

1. **Repository:** cada método `suspend` envuelve el bloque en `runCatching { ... }`. Devuelve `Result<T>` (`kotlin.Result`).
2. **UseCase:** pasa el `Result` tal cual (es identidad).
3. **ViewModel:** usa `result.fold(onSuccess, onFailure)` para mapear a `UiState.Error`.
4. **UI:** muestra un `ErrorState` con botón retry que llama de nuevo al ViewModel.

### Por qué no `try/catch` en `viewModelScope.launch`
- Es verboso y repetitivo.
- Si olvidas uno, la excepción se propaga al `CoroutineExceptionHandler` global (que es muy grosero para UI específica).
- `Result<T>` fuerza al consumidor a decidir qué hacer.

### Por qué no sealed `Result` propio
- `kotlin.Result` ya existe y es idiomático. Crear uno custom duplica el lenguaje.

## Implementación Kotlin

```kotlin
// data/ItemsRepository.kt
override suspend fun refresh(): Result<Unit> = runCatching {
    val response = api.queryItems(fields = DEFAULT_FIELDS, limit = 50)
    val items = response.cargoquery.map { it.title.toDomain() }
    cacheMutex.withLock { _items.value = items }
}

// ui/ItemsViewModel.kt
fun refresh() {
    viewModelScope.launch {
        _uiState.value = UiState.Loading
        getItems.refresh().fold(
            onSuccess = { ... emit Ready/Empty ... },
            onFailure = { error ->
                _uiState.value = UiState.Error(
                    error.message ?: "Error desconocido al cargar items"
                )
            }
        )
    }
}

// ui/components/StateScreens.kt
@Composable
fun ErrorState(message: String, onRetry: () -> Unit) { ... }
```

## Estados de UI completos

```kotlin
sealed interface UiState {
    data object Loading : UiState
    data class Ready(val items: List<Item>) : UiState
    data object Empty : UiState
    data class Error(val message: String) : UiState
}
```

`Empty` y `Error` se muestran con composables dedicados (`EmptyState`, `ErrorState`) que tienen su icono y mensaje amigable.

## Alternativas descartadas
- **`CoroutineExceptionHandler` global:** útil para崩溃 reporting, no para UX.
- **`Flow<Result<T>>` para one-shots:** convierte un one-shot en reactivo sin necesidad; confunde al consumidor.
- **Excepciones custom selladas (`NetworkError`, `ParseError`, ...):** sobreingeniería para MVP; `Throwable.message` es suficiente para mostrar.

## Riesgos & mitigación
- **Riesgo:** `runCatching` captura `CancellationException` y rompe la cancelación estructurada. **Mitigación:** Kotlin 1.5+ ya no captura `CancellationException` dentro de `runCatching` (es transparente), pero si ves comportamiento raro, considera `runCatching { ... }.onFailure { if (it is CancellationException) throw it }`.
- **Riesgo:** mensajes de error técnicos llegan al usuario ("JSON parse error at line 12"). **Mitigación:** mapear excepciones a mensajes amigables en el ViewModel (`e.message ?: "Error al cargar"`); en futuro, `getErrorMessage(throwable)` con localization.
- **Riesgo:** el usuario presiona retry y vuelve a fallar. **Mitigación:** el botón `Retry` siempre está disponible; el backoff exponencial (esperar 1s, 2s, 4s entre reintentos) se puede añadir con `retryWhen` de Flow si hace falta.
