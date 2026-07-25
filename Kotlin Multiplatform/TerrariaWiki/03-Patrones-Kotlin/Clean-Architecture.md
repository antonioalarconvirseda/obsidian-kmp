# Clean Architecture (capas data / domain / ui)

## Contexto
TerrariaWiki consume una API externa (terraria.wiki.gg) y muestra los datos en Compose. Sin una separación clara entre "cómo se obtiene la información" y "cómo se muestra", el código se vuelve un monolito donde cualquier cambio de UI toca la red y viceversa. Además, la migración futura a Kotlin Multiplatform exige que la lógica de negocio no dependa de Android.

## Decisión
Adoptar Clean Architecture con **tres capas** y una **regla de dependencias estricta**:

```
ui  ──►  domain  ◄──  data
```

- `ui` (Compose, ViewModels) → conoce `domain` (modelos, UseCases).
- `data` (Ktor API, DTOs, Mappers, Repository) → conoce `domain` (para mapear DTOs a modelos de dominio).
- `domain` → no conoce a nadie (ni Ktor ni Compose ni nada de Android).

El flujo: `ui` invoca un UseCase, el UseCase invoca el Repository, el Repository llama a la API y mapea DTO → modelo de dominio, el resultado vuelve a `ui`.

## Implementación Kotlin

```kotlin
// domain/Item.kt
data class Item(
    val name: String,
    val types: List<String>,
    val rarity: Int,
    val tooltip: String?,
    val damage: Int?,
    val defense: Int?,
    // ...
)

// domain/GetItemsUseCase.kt
class GetItemsUseCase(private val repository: ItemsRepository) {
    operator fun invoke(): Flow<List<Item>> = repository.observeItems()
}

// data/ItemsRepositoryImpl.kt
class ItemsRepositoryImpl(
    private val api: ItemsApi
) : ItemsRepository {
    override fun observeItems(): Flow<List<Item>> = flow {
        val dtos = api.queryItems(fields = "name,type,rare,...").cargoquery
        emit(dtos.map { it.title.toDomain() })
    }
}

// ui/ItemsViewModel.kt
class ItemsViewModel(
    private val getItems: GetItemsUseCase
) : ViewModel() {
    val uiState: StateFlow<UiState> = getItems()
        .map<List<Item>, UiState> { UiState.Ready(it) }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), UiState.Loading)
}
```

## Alternativas descartadas
- **Sin capa de dominio, ViewModel llama directo al Repository:** más simple, pero ata la UI a decisiones de datos. Cuando migremos a KMP, el ViewModel no se podrá mover a `commonMain` (porque conoce el Repository que conoce Ktor).
- **Clean Architecture estricta con use cases para todo (incluso para `getById`):** sobredimensionado. Usamos UseCase cuando aporta (orquestación, reglas de negocio, cache local).
- **MVI (Model-View-Intent):** el estado se gestiona con intents explícitos y un reducer. Más predecible pero más ceremony. Lo descartamos por simplicidad para MVP de aprendizaje; podemos adoptarlo más adelante si la app crece.

## Riesgos & mitigación
- **Riesgo:** la regla de dependencias se rompe accidentalmente (p.ej. un mapper de data importa un composable). **Mitigación:** convenciones de revisión; opcionalmente Detekt en el futuro para arch-tests.
- **Riesgo:** sobreingeniería — crear UseCases que simplemente delegan sin lógica. **Mitigación:** añadir UseCase solo cuando hay valor real (combinación de repos, transformación, reglas).
