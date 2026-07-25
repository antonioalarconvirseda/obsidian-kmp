# Koin — Inyección de Dependencias

## Contexto
En Clean Architecture, el ViewModel necesita un Repository, el Repository necesita un API, el API necesita un HttpClient. Sin DI, terminamos con `MainActivity` que hace `val repo = ItemsRepositoryImpl(ItemsApiImpl(createHttpClient()))` y pasa esa cadena al ViewModel. Eso acopla todo, dificulta los tests y rompe la regla de dependencias.

TerrariaWiki además apunta a Kotlin Multiplatform. Necesitamos un framework de DI que:
- Sea Kotlin puro (no KAPT, no KSP, no Android-only).
- Sea simple de leer (es un proyecto de aprendizaje).
- Tenga `viewModel { }` integrado con `androidx.lifecycle`.

## Decisión
Adoptar **Koin 3.5.x** con DSL declarativo.

### Convenciones usadas
- `single { }` para dependencias con estado (HttpClient, Repository).
- `factory { }` para dependencias sin estado que se recrean cada vez (UseCases, mappers simples).
- `viewModel { }` para ViewModels (scope: navGraph).
- `androidContext()` en `startKoin { ... }` para inyectar el `Context` cuando sea necesario (no usado por ahora).
- `KoinAndroidContext { }` envuelve el `setContent` en Compose para que los `koinViewModel()` funcionen.

## Implementación Kotlin

```kotlin
// core/di/NetworkModule.kt
val networkModule = module {
    single { createHttpClient() }
    single<ItemsApi> { ItemsApiImpl(get()) }
}

// features/items/di/ItemsModule.kt
val itemsModule = module {
    single<ItemsRepository> { ItemsRepositoryImpl(get()) }

    factory { GetItemsUseCase(get()) }
    factory { SearchItemsUseCase(get()) }
    factory { GetItemByNameUseCase(get()) }

    viewModel { ItemsViewModel(get(), get()) }
    viewModel { ItemDetailViewModel(get()) }
}

// TerrariaWikiApp.kt
class TerrariaWikiApp : Application() {
    override fun onCreate() {
        super.onCreate()
        startKoin {
            androidLogger(Level.INFO)
            androidContext(this@TerrariaWikiApp)
            modules(networkModule, itemsModule)
        }
    }
}

// MainActivity.kt
setContent {
    KoinAndroidContext {
        TerrariaWikiTheme {
            Surface(modifier = Modifier.fillMaxSize()) {
                TerrariaWikiNavHost()
            }
        }
    }
}
```

En un composable:
```kotlin
@Composable
fun ItemsScreen(
    onItemClick: (String) -> Unit,
    viewModel: ItemsViewModel = koinViewModel()  // Koin lo provee
) { ... }
```

## Alternativas descartadas
- **Hilt (Dagger para Android):** requeriría `kapt` o `ksp`, anotaciones en cada clase, generado en build, anclado a Android. Más "magia" pero menos portable a KMP.
- **Dagger puro:** ceremony alta, curva de aprendizaje empinada. Innecesario para un MVP de 4-5 features.
- **Inyección manual (constructor injection sin framework):** válido pero tedioso. Para tests unitarios sí usamos esto directamente (pasando fakes).
- **Kotlin-Inject (compile-time KSP):** muy buena opción pero suma tooling; Koin es suficiente para MVP.

## Riesgos & mitigación
- **Riesgo:** Koin valida en runtime; una dependencia olvidada peta en producción. **Mitigación:** tests unitarios que arranquen Koin (`startKoin { modules(appModule) }` en setup), `checkModules()` para verificar el grafo completo.
- **Riesgo:** `KoinAndroidContext` olvidado en `setContent` → `koinViewModel()` falla. **Mitigación:** siempre envolver el root de Compose con él; documentar el patrón.
- **Riesgo:** dos `viewModel { }` para el mismo tipo en módulos distintos. **Mitigación:** convención: cada feature expone UN módulo Koin con scope claro; `appModule` agrega todos.
