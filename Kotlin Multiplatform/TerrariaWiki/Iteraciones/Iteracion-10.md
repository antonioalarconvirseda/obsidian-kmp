# Iteración 10 — User-Agent era la causa raíz (2026-07-26)

Tras iteración 9, el usuario reportó "a partir de las 11 imágenes solo cargan al rato". **Esta vez ejecutó `adb logcat -s CoilHttp:D` y compartió los logs**. El archivo `/tmp/coil_diag.txt` (44 líneas) reveló evidencia rotunda:

- 13 peticiones espaciadas ~100ms exactos → token bucket funcionaba.
- **TODAS 429 desde la primera** → no era throughput, era identificación.
- `cargoquery` por Ktor (api.php) sí funcionaba.
- La diferencia: Ktor añade `User-Agent` (`HttpClientFactory.kt:69`), Coil no.

Causa raíz: el `OkHttpClient` de Coil no mandaba `User-Agent`, así que CloudFlare bloqueaba TODAS las imágenes como si fueran bots. El token bucket había funcionado perfectamente; era otra capa la que fallaba.

Minimax M3 (build) ejecutó:

| # | Commit | Tipo | Cambio |
|---|---|---|---|
| 1 | `dadc37d` | feat | `UserAgentInterceptor` con `User-Agent: TerrariaWikiApp/0.1.0` + retry 1 intento con `Retry-After` parseo. `CoilConfig.MAX_RETRIES_429=1`, `RETRY_SLEEP_MS=500`. TokenBucket movido a throttle-only (logging va al UserAgent). |

### Hand-off

- Tests: **46/46** passing (2 UserAgent + 4 TokenBucket + 14 ItemsMapper + 5 SearchVM + 6 ItemsVM + 7 RecipesMapper + 4 ImageUrl + 4 CategoryVM).
- APK: 19 MB.
- Repos públicos:
  - Código: https://github.com/antonioalarconvirseda/terrariawiki — 28 commits.
  - Docs: https://github.com/antonioalarconvirseda/obsidian-kmp — 12 commits.

### Lección

**No iterar sin logs reales.** Iteraciones 8 y 9 asumimos "rate-limit por throughput" sin evidencia. Los logs reales mostraron que era un problema de identificación. **Siempre pedir logs antes de iterar**.

### ✅ Validación en móvil (2026-07-26)

El fix **funciona correctamente**. Verificado en el dispositivo del usuario:

- `adb logcat -s CoilHttp:D` muestra **mayoría `200`** desde la primera request. Algunos `429` aislados se recuperan con `retry -> 200`.
- Scroll continuo carga imágenes progresivamente. **El síntoma "10 primeras cargan, el resto no" ha desaparecido**.
- No hace falta Plan B (thumbnail URLs). La causa raíz era únicamente el `User-Agent` ausente.

**Notas actualizadas**:
- `[[Coil-Rate-Limit-Interceptor]]` → sección "Iteración 10" con epílogo VERIFICADO.
- `[[Debug-Lecciones-User-Agent]]` → nueva nota standalone con el patrón aprendido (3 lecciones generalizables: OkHttp sin UA, comparar headers si dos APIs se comportan distinto, pedir logs antes de iterar).

Pendiente para próxima sesión GLM-5.2:
- Decisión de siguiente feature: NPCs, Room cache, Dark mode, o migración KMP.

---

[[00-Index]]
