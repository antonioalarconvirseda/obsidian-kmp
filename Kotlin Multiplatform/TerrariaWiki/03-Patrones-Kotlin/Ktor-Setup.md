# Ktor — setup del cliente HTTP

## Contexto
TerrariaWiki consume una API REST JSON. Necesitamos un cliente HTTP que sea:
- **Idiomático en Kotlin** (suspend, Flow nativo).
- **Multiplatform** (futuro KMP para iOS/Desktop).
- **Sin codegen** (no queremos KAPT ni KSP por simplicidad).
- **Configurable** (timeouts, logging, headers, JSON parser).

## Decisión
Adoptar **Ktor 2.3.x** con el engine **OkHttp** en Android, plugin `ContentNegotiation` + `kotlinx.serialization` para JSON, `HttpTimeout` para cortar requests colgados, y `Logging` para depurar en debug.

El `HttpClient` se crea con `createHttpClient()` en `core/network/HttpClientFactory.kt` y se inyecta vía Koin en el resto de la app.

### Configuración clave

| Aspecto | Decisión | Por qué |
|---|---|---|
| Engine | `OkHttp` | Maduro, transparente, soporte HTTP/2, mismo motor de Retrofit |
| JSON | `kotlinx.serialization` con `ignoreUnknownKeys = true` y `explicitNulls = false` | La API de MediaWiki devuelve campos variables; queremos tolerar cambios |
| `expectSuccess = true` | Lanza excepción en 4xx/5xx | Obliga a manejar errores explícitamente; lo gestionamos en el Repository |
| `HttpTimeout` | request 15s, connect 10s, socket 15s | La wiki puede ser lenta; evitamos esperas infinitas |
| `User-Agent` | Descriptivo, con contacto | MediaWiki lo exige y bloquea a UAs genéricos |

## Implementación Kotlin

```kotlin
// core/network/HttpClientFactory.kt
fun createHttpClient(): HttpClient = HttpClient(OkHttp) {
    expectSuccess = true

    install(ContentNegotiation) {
        json(Json {
            ignoreUnknownKeys = true
            isLenient = true
            explicitNulls = false
        })
    }

    install(HttpTimeout) {
        requestTimeoutMillis = 15_000
        connectTimeoutMillis = 10_000
        socketTimeoutMillis = 15_000
    }

    install(Logging) {
        level = LogLevel.INFO
        logger = object : Logger {
            override fun log(message: String) {
                android.util.Log.d("TerrariaHttp", message)
            }
        }
    }

    defaultRequest {
        url {
            protocol = URLProtocol.HTTPS
            host = "terraria.wiki.gg"
        }
        header("User-Agent", "TerrariaWikiApp/0.1.0 (contacto)")
    }
}
```

## Alternativas descartadas
- **Retrofit + OkHttp + Moshi:** estándar histórico de Android. Funciona, pero requiere converter factories, no es KMP-ready en su totalidad, y mete codegen si se usan anotaciones complejas.
- **HttpURLConnection directa:** primitivo, sin JSON parser, sin plugins. Reescribiríamos lo que Ktor ya da.
- **Ktor con engine `CIO` o `Android`:** `CIO` es Kotlin puro pero menos maduro; `Android` engine está deprecado desde Ktor 2.x. `OkHttp` es la mejor opción hoy.
- **Apollo GraphQL:** la wiki no expone GraphQL; sería scrapear.

## Riesgos & mitigación
- **Riesgo:** el logging plugin en release mete strings sensibles en logcat. **Mitigación:** bajar a `LogLevel.NONE` en release (pendiente cuando se cree `BuildConfig.DEBUG` check o flavor).
- **Riesgo:** `expectSuccess = true` lanza excepciones que se propagan hasta el ViewModel. **Mitigación:** el Repository las captura con `runCatching` y las mapea a un `Result` sellado (ver [[Error-Handling-Coroutines]]).
- **Riesgo:** Ktor 3.x introduce cambios en algunos APIs. **Mitigación:** fijado a 2.3.12 en `libs.versions.toml`; upgrade consciente cuando se decida.
