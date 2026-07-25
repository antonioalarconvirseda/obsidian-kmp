# Coil Rate-Limit Interceptor — fix de imágenes intermitentes

## Contexto
Tras 7 iteraciones de fixes de encoding, las imágenes seguían fallando "algunas sí, otras no, y al re-scrollear se cargaban poco a poco". El usuario capturó logs y reportó comportamiento. La investigación reveló:

1. **CloudFlare rate-limit (429)**: el CDN `terraria.wiki.gg/images/` limita requests por IP en una ventana deslizante. Peticiones simultáneas se rechazan.
2. **Coil sin control de concurrencia**: cuando la LazyColumn monta 50 cards, lanza ~50 requests simultáneas. CloudFlare bloquea las que exceden su ventana.
3. **Coil sin disk cache persistente**: tras reinstalar o re-abrir, todo se re-descarga desde cero.
4. **Coil sin retry con backoff**: 429 no se reintenta automáticamente, queda con placeholder de error.
5. **Coil sin logging**: los logs de Ktor (que sí captura la API cargoquery) no capturan las imágenes porque Coil usa su propio OkHttp stack independiente.

El comportamiento del usuario ("algunas sí, otras no, y al rato largo otras al rato muy lento") encaja con rate-limit aleatorio: CF decide cuáles acepta según timing dentro de la ventana deslizante.

## Decisión
Configurar un `ImageLoader` custom para Coil con **4 piezas** que en conjunto eliminan el problema:

1. **Dispatcher con concurrencia limitada**: `maxRequestsPerHost = 5` evita saturar CF.
2. **`RateLimitInterceptor`**: detecta 429, lee `Retry-After` (o usa backoff exponencial 1s, 2s, 4s), y reintenta. Máximo 3 reintentos.
3. **DiskCache persistente 50 MB**: sobrevive cierres de app y reinstalaciones. Respeta `Cache-Control: immutable` de CF.
4. **Logging tag `CoilHttp`**: cada request loga su URL + código HTTP para debug futuro.

## Implementación Kotlin

```kotlin
object CoilConfig {
    const val LOG_TAG = "CoilHttp"
    const val MAX_CONCURRENT_PER_HOST = 5
    const val MAX_TOTAL_REQUESTS = 20
    const val CACHE_BYTES = 50L * 1024 * 1024
    const val MAX_RETRIES_429 = 3
    const val INITIAL_BACKOFF_MS = 1_000L
}

class RateLimitInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        val url = request.url.toString()
        var response = chain.proceed(request)
        Log.d(CoilConfig.LOG_TAG, "${response.code} ${url.take(120)}")
        var attempts = 0
        while (response.code == 429 && attempts < CoilConfig.MAX_RETRIES_429) {
            val retryAfter = response.header("Retry-After")?.toLongOrNull()
            val backoffMs = retryAfter?.times(1000)
                ?: (CoilConfig.INITIAL_BACKOFF_MS shl attempts)
            Log.w(CoilConfig.LOG_TAG, "429 rate-limited, retrying in ${backoffMs}ms (attempt ${attempts + 1}/${CoilConfig.MAX_RETRIES_429}) ${url.take(80)}")
            response.close()
            Thread.sleep(backoffMs)
            response = chain.proceed(request)
            Log.d(CoilConfig.LOG_TAG, "retry -> ${response.code} ${url.take(120)}")
            attempts++
        }
        return response
    }
}

fun createCoilImageLoader(context: Context): ImageLoader {
    val dispatcher = Dispatcher().apply {
        maxRequests = CoilConfig.MAX_TOTAL_REQUESTS
        maxRequestsPerHost = CoilConfig.MAX_CONCURRENT_PER_HOST
    }
    val okHttpClient = OkHttpClient.Builder()
        .dispatcher(dispatcher)
        .addInterceptor(RateLimitInterceptor())
        .build()
    return ImageLoader.Builder(context)
        .okHttpClient { okHttpClient }
        .memoryCache {
            MemoryCache.Builder(context).maxSizePercent(0.25).build()
        }
        .diskCache {
            DiskCache.Builder()
                .directory(context.cacheDir.resolve("image_cache"))
                .maxSizeBytes(CoilConfig.CACHE_BYTES)
                .build()
        }
        .crossfade(true)
        .respectCacheHeaders(true)
        .memoryCachePolicy(CachePolicy.ENABLED)
        .diskCachePolicy(CachePolicy.ENABLED)
        .build()
}
```

### Wiring en Application

```kotlin
class TerrariaWikiApp : Application() {
    override fun onCreate() {
        super.onCreate()
        Coil.setImageLoader(createCoilImageLoader(this))   // sustituye singleton
        startKoin { ... }
    }
}
```

`Coil.setImageLoader()` reemplaza el `ImageLoader` por defecto. Todos los `AsyncImage` y `rememberAsyncImagePainter` que usaban el singleton por defecto ahora usan el custom. **No requiere tocar cada composable**.

## Tests añadidos (4)

Usando `MockWebServer` (OkHttp testkit):
- `passes through 200 response without retry` — happy path, sin 429.
- `retries on 429 until success within max attempts` — 3 intentos (429, 429, 200).
- `gives up after max retries and returns last 429` — 4 intentos (429 x 4), devuelve el último 429.
- `does not retry on non-429 errors like 404` — comportamiento correcto, no reintenta en otros códigos.

Para que `android.util.Log` no falle en tests JVM se añade en `app/build.gradle.kts`:
```kotlin
testOptions {
    unitTests.isReturnDefaultValues = true
}
```

## Alternativas descartadas

- **Pre-fetch con WorkManager**: overkill, y no resuelve el rate-limit. Solo ayudaría a tener imágenes listas antes.
- **Bajar la calidad de las imágenes (resize por Coil)**: no ayuda con rate-limit. La calidad de las imágenes actuales ya es ~50-100 KB.
- **Descargar imágenes con Ktor a un mapa en memoria**: añade overhead de decodificación manual y no aprovecha el pipeline de Coil.
- **Cache local persistente con Room**: demasiado pesado para imágenes binarias; el DiskCache de Coil ya las serializa eficiente.
- **Usar API `imageinfo` para resolver la URL real**: ya vimos en iteración 7 que el directorio `/images/` funciona con URL directa (no hay subdirectorios hash). Este enfoque añadiría 1 request por imagen, lo que haría el rate-limit PEOR.

## Riesgos & mitigación

- **Riesgo:** `Thread.sleep` en el interceptor bloquea el thread del dispatcher de OkHttp. Con `maxRequests=20` threads en pool, un sleep de 1-4s no bloquea las otras requests. **Mitigación:** aceptable.
- **Riesgo:** disk cache 50 MB consume espacio del usuario. **Mitigación:** `DiskCache` purga automáticamente los más antiguos cuando se excede el límite. 50 MB es ~1/40 del storage medio de un móvil moderno.
- **Riesgo:** 3 reintentos con backoff 1+2+4 = 7s por imagen. Si muchas imágenes reciben 429 simultáneamente, la pantalla tarda 7s en estar completa. **Mitigación:** `maxRequestsPerHost=5` evita que 50 se envíen simultáneas, así el 429 es raro.

## Verificación en el móvil tras la iteración

Comando debug para ver los requests de imagen:
```bash
~/Library/Android/sdk/platform-tools/adb logcat -s CoilHttp:D
```

Salida esperada:
```
D CoilHttp: 200 https://terraria.wiki.gg/images/Terra_Blade.png
D CoilHttp: 200 https://terraria.wiki.gg/images/Copper_Broadsword.png
D CoilHttp: 200 https://terraria.wiki.gg/images/Magic_Dagger.png
...
```

Si hay 429 verás:
```
D CoilHttp: 429 https://terraria.wiki.gg/images/Something.png
W CoilHttp: 429 rate-limited, retrying in 1000ms (attempt 1/3) https://terraria.wiki.gg/images/Something
D CoilHttp: retry -> 200 https://terraria.wiki.gg/images/Something.png
```
