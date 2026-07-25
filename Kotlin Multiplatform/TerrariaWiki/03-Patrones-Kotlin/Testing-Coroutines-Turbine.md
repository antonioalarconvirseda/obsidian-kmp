# Testing — Coroutines y Turbine

## Contexto
Los ViewModels exponen `StateFlow<UiState>`, los UseCases son `suspend fun` o devuelven `Flow`. Necesitamos testearlos sin:
- Llamadas reales a la API (lentas, frágiles, pueden rate-limitear).
- Dependencias de Android (no podemos instanciar `HttpClient` en JVM puro sin Robolectric).
- Coroutines que se ejecutan "cuando quieren" (necesitamos control determinista).

## Decisión
Stack de testing:
- **JUnit 4** — runner estándar.
- **MockK** — mocking idiomático en Kotlin (soporta `coEvery` para `suspend fun`).
- **kotlinx-coroutines-test** — `runTest`, `TestDispatcher`, `UnconfinedTestDispatcher`, `setMain` / `resetMain`.
- **Turbine** — aserciones legibles sobre `Flow` / `StateFlow`.

### Reglas
- `Dispatchers.setMain(testDispatcher)` en `@Before` para que `viewModelScope` use el test dispatcher.
- `Dispatchers.resetMain()` en `@After`.
- Usar `UnconfinedTestDispatcher` para que las coroutines se ejecuten eagerly sin `advanceUntilIdle()` explícito en muchos casos.
- `runTest(testDispatcher) { ... }` envuelve cada test.

## Implementación Kotlin

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class ItemsViewModelTest {
    private val testDispatcher = UnconfinedTestDispatcher()
    private lateinit var getItems: GetItemsUseCase
    private lateinit var searchItems: SearchItemsUseCase

    private val sampleItems = listOf(
        Item("Wood", listOf("block"), 0, null, ...),
        Item("Terra Blade", listOf("weapon", "melee"), 5, ...)
    )

    @Before fun setup() {
        Dispatchers.setMain(testDispatcher)
        getItems = mockk()
        searchItems = mockk()
    }

    @After fun tearDown() { Dispatchers.resetMain() }

    @Test
    fun `refresh emits Ready with items on success`() = runTest(testDispatcher) {
        coEvery { getItems.invoke() } returns MutableStateFlow(sampleItems)
        coEvery { getItems.refresh() } returns Result.success(Unit)

        val vm = ItemsViewModel(getItems, searchItems)

        vm.uiState.test {
            var state = awaitItem()
            while (state !is ItemsViewModel.UiState.Ready) state = awaitItem()
            assertEquals(3, (state as ItemsViewModel.UiState.Ready).items.size)
        }
    }
}
```

### Testing del mapper (puro, sin coroutines)

```kotlin
@Test
fun `parses rarity as integer`() {
    val dto = ItemDto(name = "Terra Blade", type = "weapon^melee", rare = "5")
    val item = dto.toDomain()
    assertEquals(5, item.rarity)
}
```

### Aserciones con Turbine

```kotlin
vm.uiState.test {
    assertEquals(UiState.Loading, awaitItem())
    assertTrue(awaitItem() is UiState.Ready)
    cancelAndIgnoreRemainingEvents()
}
```

`awaitItem()` espera la siguiente emisión del `Flow`. `cancelAndIgnoreRemainingEvents()` evita leaks de la suscripción.

## Alternativas descartadas
- **Espresso / UI Automator:** para UI testing, fuera del alcance del MVP. Aquí testeamos VM, no pantallas.
- **Robolectric:** ejecuta Android en JVM. Útil para tests de `Context`, pero aquí no lo necesitamos (los tests son JVM puro).
- **Mockito:** menos idiomático para Kotlin (necesita `kotlin-runner`); MockK maneja `suspend` y `object` out-of-the-box.
- **Truth / AssertJ:** ambas válidas. Mantenemos JUnit `assertEquals`/`assertTrue` por simplicidad.

## Riesgos & mitigación
- **Riesgo:** `UnconfinedTestDispatcher` ejecuta las coroutines eager; si hay race conditions, los tests pasan pero prod crashea. **Mitigación:** tests críticos añadir `advanceUntilIdle()` explícito; considerar `StandardTestDispatcher` para más realismo.
- **Riesgo:** mockear `Result.success(Unit)` y olvidar mockear el `getItems.invoke()` que devuelve el flow. **Mitigación:** test bien organizado; cada test stub lo que necesita.
- **Riesgo:** tests flaky por tiempo. **Mitigación:** nada de `Thread.sleep`; solo `runTest` + `advanceUntilIdle()`.
