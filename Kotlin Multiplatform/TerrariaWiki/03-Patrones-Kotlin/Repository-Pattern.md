# Repository Pattern

## Contexto
En Clean Architecture, la capa `data` es la única que sabe cómo obtener datos (Ktor, Room, SharedPreferences, etc.). La capa `domain` necesita listas de `Item` pero no debe saber de dónde vienen. El ViewModel (en `ui`) tampoco debe conocer los detalles.

## Decisión
Aplicar el patrón **Repository**:
- Interfaz `ItemsRepository` en `features/items/data/` (expuesta al domain).
- Implementación `ItemsRepositoryImpl` con lógica real (Ktor + cache en memoria).
- Los ViewModels reciben `ItemsRepository` por inyección (Koin), nunca `ItemsApi`.

El repository además:
- Expone un `Flow<List<Item>>` (estado observable).
- Centraliza la lógica de "primera vez carga, después usar cache".
- Mapea excepciones a `Result<T>` (Kotlin stdlib) para que el ViewModel pueda mostrar Error state.

## Implementación Kotlin

```kotlin
interface ItemsRepository {
    fun observeItems(): Flow<List<Item>>
    suspend fun refresh(): Result<Unit>
    suspend fun search(query: String): Result<List<Item>>
    suspend fun getByName(name: String): Result<Item?>
}

class ItemsRepositoryImpl(
    private val api: ItemsApi
) : ItemsRepository {
    private val _items = MutableStateFlow<List<Item>>(emptyList())
    private val cacheMutex = Mutex()

    override fun observeItems(): Flow<List<Item>> = _items.asStateFlow()

    override suspend fun refresh(): Result<Unit> = runCatching {
        val response = api.queryItems(fields = DEFAULT_FIELDS, limit = 50)
        val items = response.cargoquery.map { it.title.toDomain() }
        cacheMutex.withLock { _items.value = items }
    }

    override suspend fun search(query: String): Result<List<Item>> = runCatching {
        val trimmed = query.trim()
        if (trimmed.isEmpty()) _items.value
        else _items.value.filter { it.name.contains(trimmed, ignoreCase = true) }
    }

    override suspend fun getByName(name: String): Result<Item?> = runCatching {
        _items.value.firstOrNull { it.name.equals(name, ignoreCase = true) }
            ?: api.getByName(name).cargoquery.firstOrNull()?.title?.toDomain()
    }
}
```

## Alternativas descartadas
- **Active Record:** el `Item` se conecta directamente a la API. Acopla el modelo a Ktor.
- **DAO directo (sin repository):** el ViewModel conoce la API; rompe la regla de Clean Architecture.
- **Multiple repositories fragmentados** (uno para remote, otro para cache): útil en apps grandes, pero aquí es YAGNI.

## Riesgos & mitigación
- **Riesgo:** el cache en memoria `MutableStateFlow` se pierde al cerrar la app. **Mitigación:** cuando se añada Room (futuro), el repository lo introduce como segunda fuente de datos sin cambiar la interfaz.
- **Riesgo:** `Mutex` con `withLock` añade overhead. **Mitigación:** el cache write es infrecuente (solo en `refresh()`); negligible.
- **Riesgo:** `Result<T>` no es `Flow`; perdemos reactividad. **Mitigación:** las operaciones suspend devuelven `Result` (one-shot), pero la **observación** de la lista es `Flow` (reactivo). Es la convención habitual.
