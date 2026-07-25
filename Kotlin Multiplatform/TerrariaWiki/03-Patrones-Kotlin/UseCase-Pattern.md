# UseCase Pattern

## Contexto
En Clean Architecture, la capa `domain` representa las reglas de negocio puras, sin dependencias de UI ni de datos. Pero en una app Android moderna, "negocio" a menudo se reduce a "leer datos y mostrarlos". ¿Necesitamos una capa de UseCases cuando la lógica es tan simple?

## Decisión
Sí, **usamos UseCases** pero con moderación. Razones:
- **Testabilidad:** un UseCase se testea sin ViewModel, sin Compose, sin Android.
- **SDL (Single Dependency Layer):** el ViewModel depende solo de UseCases, no del Repository. Si mañana cambiamos el Repository (Room en vez de Ktor), solo cambiamos la implementación del UseCase.
- **Lenguaje del dominio:** el nombre del UseCase describe QUÉ hace el negocio (`SearchItemsUseCase`), no CÓMO se hace (HTTP a wiki.gg).
- **Reutilización:** si varias pantallas necesitan los mismos datos, el UseCase evita duplicar la llamada al Repository.

## Implementación Kotlin

```kotlin
// domain/GetItemsUseCase.kt
class GetItemsUseCase(
    private val repository: ItemsRepository
) {
    operator fun invoke(): Flow<List<Item>> = repository.observeItems()
    suspend fun refresh(): Result<Unit> = repository.refresh()
}

// domain/SearchItemsUseCase.kt
class SearchItemsUseCase(
    private val repository: ItemsRepository
) {
    suspend operator fun invoke(query: String): Result<List<Item>> =
        repository.search(query)
}

// domain/GetItemByNameUseCase.kt
class GetItemByNameUseCase(
    private val repository: ItemsRepository
) {
    suspend operator fun invoke(name: String): Result<Item?> =
        repository.getByName(name)
}
```

### Convenciones
- **`operator fun invoke(...)`**: permite usar el UseCase como función: `getItems()` en vez de `getItems.execute()`.
- **Devolver `Result<T>` o `Flow<T>`:** `Flow` para datos reactivos (lista que se observa); `Result<T>` para one-shots (refresh, search, getByName).
- **Un UseCase por acción**, no un UseCase "manager" con 10 métodos.

## Alternativas descartadas
- **ViewModel llama directo al Repository:** aceptable para apps muy simples, pero rompe SDL y complica el testing.
- **UseCase "manager" con muchos métodos:** anti-patrón. Cada UseCase debe tener una sola responsabilidad.
- **UseCase para `formatRarity(level: Int)`:** eso es lógica de UI, no de dominio. Va en el mapper o en el composable.

## Riesgos & mitigación
- **Riesgo:** inflación de UseCases que solo delegan. **Mitigación:** regla práctica: si el UseCase añade 0 valor sobre el Repository, no lo creamos. (En TerrariaWiki los creamos porque documentan intención y serán testables.)
- **Riesgo:** los UseCase se vuelven "god classes". **Mitigación:** un UseCase = una acción del usuario o del sistema.
