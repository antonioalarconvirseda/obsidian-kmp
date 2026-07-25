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

---

## Iteración 9: token bucket preventivo (2026-07-26)

### Contexto adicional

El fix de iteración 8 funcionó pero reveló un síntoma nuevo: **"las 10 primeras imágenes cargan muy bien, el resto no, y si vuelvo a hacer scroll a veces cargan"**. El re-scroll "a veces funciona" descartó definitivamente que fuera un problema de encoding o URL malformada. Era un **bug de diseño del propio RateLimitInterceptor**.

### Causa raíz

`Thread.sleep(1000-4000ms)` dentro del interceptor OkHttp **secuestra threads del dispatcher**. Con `maxRequestsPerHost=5`:

1. Las 5 primeras requests ejecutan, reciben 429, entran en `Thread.sleep(1-4s)`. **Los 5 threads quedan bloqueados**.
2. Las requests encoladas (5-15 más) no pueden ejecutarse: el dispatcher de OkHttp no tiene threads libres para el host.
3. El usuario hace scroll → Coil cancela las requests fuera de viewport → algunas requests encoladas se eliminan de la cola OkHttp.
4. El usuario hace scroll atrás → Coil relanza requests → los threads dormidos ya despertaron (pasaron 1-4s) → algunas requests nuevas pasan. **Pero solo si el timing coincide**.

Por eso "10 primeras cargan, las demás no, y a veces al re-scroll sí".

### Decisión

Sustituir `RateLimitInterceptor` por un `TokenBucketInterceptor` preventivo que **limita throughput a 10 req/s antes de `proceed()`**, no reactivo tras un 429. Eliminar completamente el retry con backoff, ya que era la fuente del bug.

### Implementación Kotlin (final)

```kotlin
class TokenBucketInterceptor(
    private val periodMs: Long = CoilConfig.TOKEN_PERIOD_MS  // 100L = 10 req/s
) : Interceptor {
    private val nextSlot = AtomicLong(0L)

    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        val now = System.currentTimeMillis()
        var reserved = 0L
        while (true) {
            val current = nextSlot.get()
            val candidate = maxOf(current, now) + periodMs
            if (nextSlot.compareAndSet(current, candidate)) {
                reserved = candidate
                break
            }
        }
        val waitMs = (reserved - now).coerceAtLeast(0L)
        if (waitMs > 0) Thread.sleep(waitMs)
        val response = chain.proceed(request)
        Log.d(CoilConfig.LOG_TAG, "${response.code} ${request.url.toString().take(120)}")
        return response
    }
}
```

Config:
```kotlin
MAX_CONCURRENT_PER_HOST = 10  // sub 5→10: más threads pasan por la puerta del token
TOKEN_PERIOD_MS = 100L         // 10 req/s sostenido
```

### Por qué funciona

- **Regulación atómica con `compareAndSet`**: el primer thread pone `nextSlot = now+100`. El segundo pone `nextSlot = 200`. Etc. Cada thread reserva su slot único. No hay race condition ni doble reserva.
- **Sleep hasta `reserved`, no hasta `now+periodMs`**: si 10 threads llegan simultáneamente, el 10º reserva `now+1000` y duerme los 1000ms completos. Los 5 primeros duermen <100ms. Total 1s para 10 imágenes, throughput regulado.
- **No hay `Thread.sleep` reactivo**: si llega 429, simplemente se devuelve el 429 (Coil muestra error en esa card puntual). No se reintenta. Se confía en que el throttle de 10 req/s es suficiente para evitar 429.
- **Mayor concurrencia (10 vs 5)**: más threads pasan por el token gate, pero el gate limita el throughput. CF recibe como mucho 10 req/s.

### Tests del TokenBucketInterceptor (4)

`TokenBucketInterceptorTest.kt` reemplaza al anterior `RateLimitInterceptorTest.kt`. Los tests miden el comportamiento real con `MockWebServer`:

1. `single request does not sleep meaningfully` — una request tarda <200ms.
2. `two consecutive requests arrive spaced at least 80ms apart` — 2 requests en paralelo, la 2ª llega al server al menos 80ms después de la 1ª (margen CI).
3. `five parallel requests take at least 400ms total` — 5 requests en paralelo tardan ≥400ms en total (5 × 100ms = 500ms esperado, margen ≥400ms).
4. `passes 200 response through unchanged` — body y response code se preservan.

### Alternativas descartadas en esta iteración

- **"Retry cooperativo con coroutine"**: convertir el interceptor a `suspendCancellableCoroutine` + `delay()` para que la cancelación de Coil propague. Considerada pero descartada porque añade complejidad y no resuelve el problema de raíz (los 429 reactivos son síntoma, no causa).
- **"Quitar el interceptor, subir concurrencia"**: dejar que CF responda 429 y mostrar error. Inestable; dependería de cache edge de CF.
- **"Solo subir maxRequestsPerHost a 10 sin token bucket"**: los 429 seguirían, el sleep seguiría, mismos problemas. La concurrencia sin regulación de throughput es exactamente el escenario que causa rate-limit.
- **"Reintento residual de 1 intento en 429"**: descartado por el usuario tras revisar el plan. Si llegase 429 aislado (improbable con throttle de 10 req/s), se muestra placeholder. Re-scroll natural reintentaría.

### Verificación en el móvil tras la iteración 9

```bash
~/Library/Android/sdk/platform-tools/adb logcat -s CoilHttp:D
```

Salida esperada:
```
D CoilHttp: 200 https://terraria.wiki.gg/images/Terra_Blade.png
D CoilHttp: 200 https://terraria.wiki.gg/images/Copper_Broadsword.png
... (10 req/s constante, sin 429)
```

Comportamiento esperado en la app:
- Scroll continuo por una categoría: las imágenes cargan progresivamente a ~10/s.
- No más "10 primeras cargan, el resto no".
- No más "re-scroll a veces funciona". Ahora carga desde el primer intento.

### Riesgos & mitigación

- **Riesgo: latencia inicial alta en listas grandes.** Una lista de 50 items tarda ~5s en cargar la última imagen (50 × 100ms). Aceptable: mejor 5s garantizados que "nunca carga".
- **Riesgo: si CF cambia su rate-limit a <10 req/s.** Mitigación: subir `TOKEN_PERIOD_MS` a 150-200ms es un cambio de una constante. Documentar en `CoilConfig`.
- **Riesgo: si migramos a KMP, este interceptor OkHttp no es portable.** Deuda técnica documentada; el patrón (atomic CAS sobre Long + sleep hasta slot reservado) es trivialmente portable a Ktor HttpRequestPipeline phase o a un `HttpSend` plugin de Ktor.

---

## Iteración 10: el verdadero culpable era el User-Agent (2026-07-26)

### Contexto

Tras iteración 9, el usuario volvió a reportar: **"a partir de las 11 imágenes solo cargan al rato si empiezas a hacer scroll al rato"**. Y añadió: "no he podido depurarlo si es porque el servidor bloquea las imágenes". Esta vez el usuario ejecutó los comandos `adb logcat` y compartió `/tmp/coil_diag.txt` (44 líneas).

### Lo que los logs revelaron (evidencia rotunda)

El log `/tmp/coil_diag.txt` mostraba:

```
07-26 01:18:09.280  CoilHttp: 429 https://terraria.wiki.gg/images/Bouncy_Bomb.png
07-26 01:18:10.098  CoilHttp: 429 https://terraria.wiki.gg/images/Bouncy_Dynamite.png
07-26 01:18:10.196  CoilHttp: 429 https://terraria.wiki.gg/images/Bouncy_Grenade.png
... (13 peticiones, TODAS 429, espaciadas ~100ms)
07-26 01:18:11.461  CoilHttp: 429 https://terraria.wiki.gg/images/Cannon.png
... (pausa de 14s)
07-26 01:18:25.596  CoilHttp: 429 https://terraria.wiki.gg/images/Cannonball.png
... (21 más, TODAS 429)
07-26 01:18:28.829  TerrariaHttp: REQUEST: https://terraria.wiki.gg/api.php?action=cargoquery&...
```

**Hallazgos:**

1. **El token bucket SÍ funcionaba.** Las peticiones estaban espaciadas ~100ms exactos. El throttle era correcto.
2. **TODAS las imágenes devolvían 429**, no había ni un solo 200.
3. **La API `cargoquery` (Ktor) sí funcionaba.** Era solo `/images/` lo bloqueado.
4. **El primer request de la sesión ya era 429.**

### Causa raíz

Diferencia clave entre Ktor y Coil: Ktor añade `User-Agent` a cada request (`HttpClientFactory.kt:69`: `header("User-Agent", TerrariaApiConfig.USER_AGENT)`). El `OkHttpClient` de Coil **NO mandaba User-Agent**. CF bloquea agresivamente a clientes anónimos, especialmente contra `/images/` (probable fingerprint de bots/scrapers).

Adicionalmente, la IP podía estar marcada por spam de iteraciones 7-8, lo que intensificaba el bloqueo.

### Decisión

Añadir un `UserAgentInterceptor` (OkHttp `Interceptor` síncrono) que ponga `User-Agent` a cada request de imagen si no existe. Aprovechar el mismo interceptor para añadir un retry reactivo de **1 solo intento** con `Thread.sleep(500)` cap, parseando `Retry-After` si CF lo manda. Esto es la recuperación reactiva mínima que complementaba al token bucket preventivo de iteración 9.

### Implementación Kotlin

```kotlin
class UserAgentInterceptor(
    private val userAgent: String = TerrariaApiConfig.USER_AGENT,
    private val maxRetries429: Int = CoilConfig.MAX_RETRIES_429,  // 1
    private val retrySleepMs: Long = CoilConfig.RETRY_SLEEP_MS    // 500L
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val original = chain.request()
        val request = if (original.header("User-Agent") == null) {
            original.newBuilder().header("User-Agent", userAgent).build()
        } else original
        var response = chain.proceed(request)
        Log.d(CoilConfig.LOG_TAG, "${response.code} ${request.url.toString().take(120)}")
        var attempts = 0
        while (response.code == 429 && attempts < maxRetries429) {
            val retryAfter = response.header("Retry-After")?.toLongOrNull()
            val sleepMs = retryAfter?.times(1000)?.coerceAtMost(retrySleepMs) ?: retrySleepMs
            Log.w(CoilConfig.LOG_TAG, "429 rate-limited, retrying in ${sleepMs}ms ${request.url.toString().take(80)}")
            response.close()
            Thread.sleep(sleepMs)
            response = chain.proceed(request)
            Log.d(CoilConfig.LOG_TAG, "retry -> ${response.code} ${request.url.toString().take(120)}")
            attempts++
        }
        return response
    }
}
```

### Wiring en OkHttpClient

```kotlin
val okHttpClient = OkHttpClient.Builder()
    .dispatcher(dispatcher)
    .addInterceptor(TokenBucketInterceptor())   // throttle 10/s
    .addInterceptor(UserAgentInterceptor())     // añade UA + retry 1
    .build()
```

Orden importa: TokenBucket throttlea entrada, UserAgent aplica UA. Si un request ya tiene UA (poco probable con Coil pero posible en tests), se respeta.

### TokenBucket se simplifica

Antes de iteración 10, el `TokenBucketInterceptor` también logueaba. Se movió el logging al `UserAgentInterceptor` para que `TokenBucketInterceptor` sea solo throttle puro. Tests intactos.

### Tests del UserAgentInterceptor (2)

`UserAgentInterceptorTest.kt`:
1. `adds User-Agent header when request has none` — MockWebServer verifica header recibido.
2. `preserves existing User-Agent header` — request con UA pre-existente no se sobrescribe.

### Plan B (si UA no basta)

Si tras añadir UA seguimos recibiendo 429 desde la primera petición, queda como deuda: implementar thumbnail URLs `https://terraria.wiki.gg/images/thumb/<file>/32px-<file>` en lista (reducción 50-100KB → 2-5KB, mejor cache edge hit). En `ItemDetailScreen` se carga la versión full. Aplazado a iteración 11 si hace falta.

### Lección aprendida

**No asumir sin logs reales.** Iteraciones 8 y 9 supusimos que el problema era "rate-limit por throughput" o "thread starvation". Los logs reales demostraron que era algo mucho más simple: el cliente HTTP no se identificaba. Ktor lo hacía por defecto, Coil no. **Siempre pedir logs antes de iterar**.

**Lección de diseño:** OkHttp NO añade `User-Agent` por defecto. Es responsabilidad del cliente. Cualquier nueva dependencia HTTP debe verificar que envía UA identificable.
