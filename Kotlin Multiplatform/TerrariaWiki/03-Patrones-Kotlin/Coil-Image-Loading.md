# Coil — carga de imágenes

## Contexto
TerrariaWiki muestra imágenes de items obtenidas de la wiki (campo `image` con `[[File:Wood.png|caption]]`). Necesitamos un cargador de imágenes que:
- Funcione dentro de Compose.
- Maneje placeholders y errores amigables.
- Haga caché automática.
- Soporte URLs con redirects (la wiki devuelve 301 a `Special:Redirect/file/...`).

## Decisión
Adoptar **Coil 2.6.0** con su integración `coil-compose`. Es:
- Compose-first (`AsyncImage` composable).
- Kotlin coroutines.
- Caché en memoria + disco automático.
- Sigue redirects HTTP transparentemente.

## Implementación Kotlin

```kotlin
// En ItemCard.kt
@Composable
fun ItemThumbnail(item: Item, size: Dp = 56.dp) {
    val imageModel = item.imageFilename?.let { filename ->
        "${TerrariaApiConfig.HOST_IMAGE_BASE}/$filename"
    }
    Box(
        modifier = Modifier
            .size(size)
            .clip(RoundedCornerShape(8.dp))
            .background(MaterialTheme.colorScheme.surfaceVariant),
        contentAlignment = Alignment.Center
    ) {
        if (imageModel != null) {
            AsyncImage(
                model = "https://$imageModel",
                contentDescription = item.name,
                modifier = Modifier.size(size),
                placeholder = painterResource(R.drawable.ic_item_placeholder),
                error = painterResource(R.drawable.ic_item_error)
            )
        } else {
            Image(
                painter = painterResource(R.drawable.ic_item_placeholder),
                contentDescription = item.name,
                modifier = Modifier.size(size)
            )
        }
    }
}
```

## Particularidades descubiertas

1. **URL construida:** `https://terraria.wiki.gg/wiki/Special:Redirect/file/Wood.png` devuelve 301 → URL final en `static.wikia.com/.../Wood.png`. Coil sigue el redirect sin más.
2. **Placeholder vs Error:** distinguir "cargando" (placeholder) de "no se pudo cargar" (error). Lo hacemos con `placeholder` y `error` separados.
3. **Mapeo de filename:** la API devuelve `[[File:Wood.png|caption]]`; el mapper extrae `Wood.png` con regex y compone la URL.

## Alternativas descartadas
- **Glide:** veterano, sin integración Compose oficial (necesitas wrappers).
- **Picasso:** sin mantenimiento activo.
- **Carga manual con Ktor:** tienes que reimplementar cache, decodificación, etc.
- **Coil 3.x (KMP completo):** ideal para futuro multiplatform, pero requiere migración del OkHttp engine. Lo dejamos para cuando se haga la migración.

## Riesgos & mitigación
- **Riesgo:** imágenes no se cachean y refetch en cada scroll. **Mitigación:** Coil tiene caché en memoria por defecto; el segundo scroll ya no hace red.
- **Riesgo:** URLs de imagen rotas (item eliminado de la wiki) → imagen rota en UI. **Mitigación:** `error = painterResource(R.drawable.ic_item_error)` muestra un placeholder.
- **Riesgo:** R8/ProGuard elimina clases internas de Coil. **Mitigación:** reglas ProGuard:
  ```
  -keep class coil.** { *; }
  ```
  (A añadir en `proguard-rules.pro` cuando se active minify en release.)
- **Riesgo:** rate-limit 429 de CloudFlare al pedir muchas imágenes simultáneas a `Special:Redirect/file/...` (iteración 5). **Mitigación (iteración 6):** saltamos `Special:Redirect` y construimos directamente la URL final `https://terraria.wiki.gg/images/<filename>`. Esa URL está cacheada en CF con `cache-control: public, max-age=31536000, immutable` y responde `cf-cache-status: HIT` → no hay rate-limit. Función `buildItemImageUrl(filename)` en `core/network/HttpClientFactory.kt` centraliza el encoding (`%20` para espacios, `%27` para apostrofes).
