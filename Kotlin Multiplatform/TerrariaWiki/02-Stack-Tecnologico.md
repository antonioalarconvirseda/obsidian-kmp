# Stack Tecnológico — TerrariaWiki

Una subsección por librería. Para cada una: qué es, por qué, versión, cómo se añade al `build.gradle.kts`, snippet de uso y dónde se usa en el proyecto.

---

## Ktor 2.3.x (HTTP client)

**Qué es:** Framework HTTP en Kotlin idiomático, creado por JetBrains. Multiplatform por diseño (Android, iOS, JVM, JS, Native).

**Por qué lo elegimos:**
- API suspend-native (no necesitamos `Call`/callbacks).
- Sin codegen (a diferencia de Retrofit) → no requiere KAPT/KSP.
- Plugins intercambiables (ContentNegotiation, Logging, Auth…).
- Diseñado para KMP: cuando migremos a iOS, solo cambiamos el engine.

**Engine en Android:** `OkHttp` (estable, maduro, soporta HTTP/2, transparente en tamaño de APK).

**Cómo se añade (en `gradle/libs.versions.toml`):**
```toml
[versions]
ktor = "2.3.12"

[libraries]
ktor-client-core = { module = "io.ktor:ktor-client-core", version.ref = "ktor" }
ktor-client-okhttp = { module = "io.ktor:ktor-client-okhttp", version.ref = "ktor" }
ktor-client-content-negotiation = { module = "io.ktor:ktor-client-content-negotiation", version.ref = "ktor" }
ktor-serialization-kotlinx-json = { module = "io.ktor:ktor-serialization-kotlinx-json", version.ref = "ktor" }
ktor-client-logging = { module = "io.ktor:ktor-client-logging", version.ref = "ktor" }
```

**Snippet básico:**
```kotlin
val client = HttpClient(OkHttp) {
    install(ContentNegotiation) { json(Json { ignoreUnknownKeys = true }) }
    install(Logging) { level = LogLevel.INFO }
}
val resp: CargoResponse<ItemDto> = client.get("https://terraria.wiki.gg/api.php") {
    parameter("action", "cargoquery")
    parameter("tables", "Items")
    parameter("fields", "name,type,rare")
    parameter("format", "json")
}.body()
```

**Dónde se usa:** `core/network/HttpClientFactory.kt`, `features/items/data/ItemsApi.kt`.

---

## kotlinx.serialization 1.7.x

**Qué es:** Librería oficial de JetBrains para serializar/deserializar Kotlin a JSON, Protobuf, etc. Funciona con anotaciones o sin ellas.

**Por qué:**
- Multiplatform nativo.
- Sin reflexión en runtime (compilador genera serializers).
- `@Serializable` data class → 0 boilerplate.

**Cómo se añade:**
```toml
[versions]
kotlinx-serialization = "1.7.3"

[plugins]
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
```
Y en `app/build.gradle.kts`: `alias(libs.plugins.kotlin.serialization)`.

**Snippet:**
```kotlin
@Serializable
data class CargoResponse<T>(
    val cargoquery: List<CargoItem<T>> = emptyList()
)
@Serializable
data class CargoItem<T>(val title: T)
@Serializable
data class ItemDto(
    val name: String = "",
    val type: String = "",
    val rare: String = "0",
    @SerialName("tooltip") val tooltip: String? = null,
    // ...
)
```

**Dónde se usa:** `features/items/data/ItemsDto.kt`.

---

## Koin 3.5.x (DI)

**Qué es:** Framework de inyección de dependencias pragmático, Kotlin puro, sin KAPT/KSP.

**Por qué:**
- DSL declarativo, fácil de leer.
- `viewModel { }` integrado con `androidx.lifecycle`.
- KMP-ready (mismas APIs en commonMain).
- Resolución en runtime (no en compile-time), tradeoff conocido.

**Cómo se añade:**
```toml
[versions]
koin = "3.5.6"

[libraries]
koin-android = { module = "io.insert-koin:koin-android", version.ref = "koin" }
koin-androidx-compose = { module = "io.insert-koin:koin-androidx-compose", version.ref = "koin" }
koin-core = { module = "io.insert-koin:koin-core", version.ref = "koin" }
```

**Snippet:**
```kotlin
val networkModule = module {
    single { createHttpClient() }
    single { ItemsApi(get()) }
}

class TerrariaWikiApp : Application() {
    override fun onCreate() {
        super.onCreate()
        startKoin {
            androidContext(this@TerrariaWikiApp)
            modules(networkModule, itemsModule)
        }
    }
}
```

**Dónde se usa:** `core/di/`, `features/items/di/`, `TerrariaWikiApp.kt`, `MainActivity.kt`.

---

## Coil 2.6.x (carga de imágenes)

**Qué es:** Compositor de imágenes para Android/KMP. Compose-first, Kotlin coroutines.

**Por qué:**
- API `AsyncImage` directamente en Compose.
- Cache en memoria + disco automático.
- Multiplatform (Coil 3.x es full KMP, Coil 2.x es Android-only pero suficiente para esta fase).

**Cómo se añade:**
```toml
[versions]
coil = "2.6.0"

[libraries]
coil-compose = { module = "io.coil-kt:coil-compose", version.ref = "coil" }
```

**Snippet:**
```kotlin
AsyncImage(
    model = "https://terraria.wiki.gg/wiki/Special:Redirect/file/Wood.png",
    contentDescription = "Wood",
    modifier = Modifier.size(64.dp),
    placeholder = painterResource(R.drawable.placeholder_item),
    error = painterResource(R.drawable.error_item)
)
```

**Dónde se usa:** `features/items/ui/ItemDetailScreen.kt`, cards de `ItemsScreen.kt`.

---

## Navigation Compose 2.7.x

**Qué es:** API declarativa de navegación para Jetpack Compose. Parte de `androidx.navigation`.

**Por qué:**
- Define rutas como strings tipadas.
- Argumentos tipados (`NavType.StringType`).
- Backstack gestionado automáticamente.

**Cómo se añade:**
```toml
[versions]
navigation = "2.7.7"

[libraries]
navigation-compose = { module = "androidx.navigation:navigation-compose", version.ref = "navigation" }
```

**Snippet:**
```kotlin
NavHost(navController, startDestination = "items") {
    composable("items") { ItemsScreen(onItemClick = { navController.navigate("item/${it.name}") }) }
    composable(
        route = "item/{name}",
        arguments = listOf(navArgument("name") { type = NavType.StringType })
    ) { backStackEntry ->
        ItemDetailScreen(name = backStackEntry.arguments?.getString("name").orEmpty())
    }
}
```

**Dónde se usa:** `features/items/ui/navigation/ItemsNavigation.kt`, `MainActivity.kt`.

---

## Lifecycle ViewModel Compose 2.8.x

**Qué es:** Integración `ViewModel` + Compose: `viewModel()` composable que obtiene o crea un ViewModel scoped a la nav graph.

**Por qué:**
- ViewModel sobrevive a recomposiciones y cambios de configuración.
- API `viewModel()` devuelve la instancia por `KoinViewModelFactory`.

**Cómo se añade:**
```toml
[versions]
lifecycle = "2.8.4"

[libraries]
lifecycle-viewmodel-compose = { module = "androidx.lifecycle:lifecycle-viewmodel-compose", version.ref = "lifecycle" }
lifecycle-runtime-compose = { module = "androidx.lifecycle:lifecycle-runtime-compose", version.ref = "lifecycle" }
```

**Dónde se usa:** `ItemsViewModel.kt`, `ItemDetailViewModel.kt`.

---

## Coroutines + Flow 1.8.x

**Qué es:** Librería estándar de JetBrains para concurrencia estructurada en Kotlin.

**Por qué:**
- `suspend fun` y `Flow` son la base de todo (red, UI reactiva).
- `viewModelScope` para coroutines atadas al ViewModel.
- KMP nativo.

**Cómo se añade:**
```toml
[versions]
coroutines = "1.8.1"

[libraries]
kotlinx-coroutines-android = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-android", version.ref = "coroutines" }
```

**Dónde se usa:** Repository (`Flow<List<Item>>`), ViewModel (`StateFlow`), UseCases (`suspend`).

---

## Futuro: Room (cache offline)

Pendiente. Cuando se introduzca, se documentará aquí con su porqué (acceso offline, queries SQL sobre miles de items, …) y el wrapper KMP-friendly (`androidx.room` es Android-only; la alternativa KMP sería `SQLDelight`).

## Matiz KMP-readiness (auditoría iteración 2)

Las librerías elegidas son **KMP-friendly**, pero la **integración actual** no es 100% portable. Anclajes Android identificados:

| Componente | Estado actual | Refactor KMP necesario |
|---|---|---|
| `HttpClient(OkHttp)` | Android/JVM | `expect/actual`: OkHttp en androidMain, Darwin engine en iosMain |
| `android.util.Log` | Android | `expect/actual` o lib KMP (Napier/Kermit) |
| `koin-android` + `koin-androidx-compose` | Android | `koin-core` en commonMain + binding Android |
| `androidx.lifecycle.ViewModel` | Android | `viewmodel-savedstate` o `lifecycle-viewmodel-compose` ya tiene variante KMP |
| Compose Android | Android | Compose Multiplatform (KMP oficial) |

El **dominio** (`Item`, UseCases) y la **capa de datos pura** (DTO, Mapper, Repository interface) **sí son KMP-puros** y se pueden mover a `commonMain` sin tocar una línea.

Para una migración real hace falta:
1. Split Gradle: módulo `:shared` (con commonMain + androidMain + iosMain) + `:android-app` delgado.
2. Refactor de `HttpClientFactory.kt` a `expect/actual`.
3. Sustituir `koin-android` por `koin-core` con binding Android por target.

Documentación completa de KMP en [[01-Decisiones-de-Arquitectura]] §3.
