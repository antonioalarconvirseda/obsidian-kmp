# Lección: User-Agent ausente causaba 429 en CloudFlare

## Contexto
En TerrariaWiki, 3 iteraciones consecutivas intentando arreglar imágenes intermitentes. El síntoma reportado por el usuario:

> "Las 10 primeras imágenes cargan muy bien, el resto no. Si vuelvo a hacer scroll, a veces cargan al rato."

### La trampa
Asumimos, sin evidencia, que el problema era **throughput** (rate-limit por volumen de peticiones). Iteramos 2 veces con fixes de rate-limit (`RateLimitInterceptor` con `Thread.sleep` backoff, luego `TokenBucketInterceptor` con `AtomicLong.compareAndSet`). Ninguna resolvió el problema de fondo.

### La evidencia que lo resolvió
El usuario ejecutó por fin `adb logcat -s CoilHttp:D` y compartió el archivo `/tmp/coil_diag.txt` (44 líneas). El log reveló:

```
07-26 01:18:09.280  D CoilHttp: 429 https://terraria.wiki.gg/images/Bouncy_Bomb.png
07-26 01:18:10.098  D CoilHttp: 429 https://terraria.wiki.gg/images/Bouncy_Dynamite.png
07-26 01:18:10.196  D CoilHttp: 429 https://terraria.wiki.gg/images/Bouncy_Grenade.png
... (13 peticiones, TODAS 429, espaciadas ~100ms exactos)
07-26 01:18:11.461  D CoilHttp: 429 https://terraria.wiki.gg/images/Cannon.png
... (pausa de 14s)
07-26 01:18:25.596  D CoilHttp: 429 https://terraria.wiki.gg/images/Cannonball.png
... (21 más, TODAS 429)
07-26 01:18:28.829  D TerrariaHttp: REQUEST: https://terraria.wiki.gg/api.php?action=cargoquery
```

**Hallazgos clave:**
1. Las peticiones estaban **espaciadas ~100ms exactos** → el `TokenBucketInterceptor` (10 req/s) funcionaba.
2. **TODAS las imágenes devolvían 429**, desde la primera petición, sin un solo 200.
3. La API `cargoquery` (Ktor, `api.php`) **sí funcionaba** — petición exitosa a la 28.829s.
4. El primer request de la sesión ya era 429.

La diferencia entre las dos APIs que compartían host: Ktor enviaba `User-Agent`, Coil/OkHttp no.

## Causa raíz
El `OkHttpClient` configurado para Coil en `CoilImageLoaderFactory.kt:82-91` **NO añadía la cabecera `User-Agent`**. CloudFlare (CDN delante de `terraria.wiki.gg`) interpreta clientes anónimos como bots/scrapers y les responde `429 Too Many Requests` a casi todo, especialmente en `/images/`.

Ktor, en cambio, sí añade `User-Agent` por defecto vía `defaultRequest { header("User-Agent", TerrariaApiConfig.USER_AGENT) }` en `HttpClientFactory.kt:69`. Por eso las llamadas a `api.php` funcionaban y las imágenes no.

## Decisión
Añadir un `UserAgentInterceptor` (OkHttp `Interceptor` síncrono) que:
1. Pone `User-Agent: TerrariaWikiApp/0.1.0 (https://github.com/...; ...)` a cada request de imagen si no existe.
2. Reintenta **1 vez** con `Thread.sleep(500)` cap, parseando `Retry-After` si CF lo manda. Es la recuperación reactiva mínima que complementaba al token bucket preventivo.

Orden de interceptores en el `OkHttpClient`:
```
Request → TokenBucketInterceptor (throttle 10/s) → UserAgentInterceptor (añade UA + retry 429) → Server
```

## Implementación Kotlin

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

Wiring en `CoilImageLoaderFactory.kt:87-91`:
```kotlin
val okHttpClient = OkHttpClient.Builder()
    .dispatcher(dispatcher)
    .addInterceptor(TokenBucketInterceptor())
    .addInterceptor(UserAgentInterceptor())
    .build()
```

## Lecciones generalizables

### 1. OkHttp no añade `User-Agent` por defecto
A diferencia de Ktor (`defaultRequest` aplica headers globales), `OkHttpClient.Builder()` sale sin `User-Agent`. Esto es responsabilidad del cliente. Si en cualquier proyecto futuro aparece comportamiento extraño con un CDN/API que funciona con `curl` o navegador, **comprobar headers antes de asumir rate-limit o IP bloqueada**.

Cómo verificar en logcat o wireshark:
```bash
# quick test
curl -I https://terraria.wiki.gg/images/Terra_Blade.png
# vs
# nuestro cliente Coil NO manda User-Agent, solo Host y Accept
```

### 2. Si una API funciona y otra no en la misma app, **comprueba headers comunes**
Caso típico: dos librerías HTTP en el mismo proyecto, una configurada correctamente, otra no. La que falla se diagnostica por **diferencia**, no por la sintomatología. En este proyecto: Ktor vs OkHttp, dos clientes, una API ok y la otra no. La diferencia fue 1 cabecera.

### 3. Siempre pedir `adb logcat` antes de iterar
Llevamos 2 iteraciones perdidas (8 y 9) asumiendo. La 3ª iteración (10) la resolvimos en menos de 1 commit porque teníamos evidencia real. La regla:

> Antes de empezar a tocar código de red, ejecuta el comando que captura los códigos HTTP y timestamps, y **lee al menos 30 líneas** del log. El siguiente paso se deduce solo.

Comando a memorizar:
```bash
~/Library/Android/sdk/platform-tools/adb logcat -c
# acción en la app
~/Library/Android/sdk/platform-tools/adb logcat -d -s CoilHttp:D TerrariaHttp:D
```

## Alternativas descartadas

- **Thumbnail URLs** `https://terraria.wiki.gg/images/thumb/<file>/32px-<file>`: reduce payload 50-100KB → 2-5KB, mejor cache edge hit. **Queda como plan B** si 429 persiste tras el fix de UA. Útil para listas donde solo se ven thumbnails pequeños.
- **`Special:Redirect/file/<filename>`**: en iteración 7 vimos que este endpoint también devuelve 429, peor que el path directo a `/images/`. Descartado.
- **Cambiar a un cliente HTTP que añada UA por defecto**: descartado. La solución es configurar OkHttp correctamente, no reemplazar la librería.
- **Llamar a la `imageinfo` API para resolver la URL real antes de fetch**: añade 1 request extra por imagen, multiplica el problema de rate-limit. Descartado.

## Verificación post-fix

```bash
~/Library/Android/sdk/platform-tools/adb logcat -c
# scroll 20s por categoría Weapons
~/Library/Android/sdk/platform-tools/adb logcat -d -s CoilHttp:D
```

Salida esperada:
```
D CoilHttp: 200 https://terraria.wiki.gg/images/Bouncy_Bomb.png
D CoilHttp: 200 https://terraria.wiki.gg/images/Bouncy_Dynamite.png
... (10 req/s constante, sin 429, sin retry)
```

Si aparece algún `429` aislado:
```
D CoilHttp: 429 https://terraria.wiki.gg/images/Something.png
W CoilHttp: 429 rate-limited, retrying in 500ms ...
D CoilHttp: retry -> 200 https://terraria.wiki.gg/images/Something.png
```

✅ **Verificado en móvil el 2026-07-26**: el fix funciona. Las imágenes cargan correctamente en scroll continuo. La causa raíz era el `User-Agent` ausente en el `OkHttpClient` de Coil.
