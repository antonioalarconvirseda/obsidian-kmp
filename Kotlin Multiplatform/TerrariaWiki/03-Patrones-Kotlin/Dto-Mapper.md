# DTO vs. Modelo de dominio (Mapper)

## Contexto
La MediaWiki API devuelve un JSON con una estructura fija (`cargoquery[].title.*`). Si el modelo de dominio de TerrariaWiki (`Item`) fuera directamente ese DTO, cualquier cambio en la API rompería la app. Además, el modelo de dominio tiene tipos más ricos (`Int` en lugar de `String` para `rarity`, `List<String>` en lugar de un string con `^`).

## Decisión
- **DTO** en `data/ItemsDto.kt`: refleja 1:1 la respuesta JSON. Todos los campos son opcionales y tipos "permissive" (String, aunque sepamos que es número).
- **Modelo de dominio** en `domain/Item.kt`: tipos fuertes (`Int?`, `Float?`, `List<String>`), con campos ya limpios.
- **Mapper** `ItemDto.toDomain(): Item` en `data/ItemsMapper.kt`: concentra toda la lógica de limpieza.

Regla: **el dominio no importa nada del paquete `data`**. Solo `data` conoce a `domain` (vía el mapper).

## Implementación Kotlin

```kotlin
// data/ItemsDto.kt
@Serializable
data class ItemDto(
    val name: String = "",
    val type: String = "",
    val rare: String = "0",
    val sell: String? = null,
    val damage: String? = null,
    // ... todos String? con default
)

// domain/Item.kt
data class Item(
    val name: String,
    val types: List<String>,
    val rarity: Int,
    val tooltip: String?,
    val damage: Int?,
    val defense: Int?,
    val knockback: Float?,
    val useTime: Int?,
    val sellRaw: String?,
    val internalName: String?,
    val wikiId: Int?,
    val imageFilename: String?
)

// data/ItemsMapper.kt
private const val LIST_DELIMITER = "^"

fun ItemDto.toDomain(): Item = Item(
    name = name,
    types = type.split(LIST_DELIMITER).filter { it.isNotBlank() },
    rarity = rare.toIntOrNull() ?: 0,
    tooltip = tooltip?.takeIf { it.isNotBlank() }?.stripHtml(),
    damage = damage?.toIntOrNull(),
    defense = defense?.toIntOrNull(),
    knockback = knockback?.toFloatOrNull(),
    useTime = usetime?.toIntOrNull(),
    sellRaw = sell?.takeIf { it.isNotBlank() },
    internalName = internalname?.takeIf { it.isNotBlank() },
    wikiId = itemid?.toIntOrNull(),
    imageFilename = image?.extractFileName()
)
```

## Particularidades de MediaWiki descubiertas en probes
- **`rare` viene como String `"0"`, no Int.** Hay que parsear.
- **Listas con `^`.** El campo `type` puede ser `"block^crafting material"`. Aplica también a `listcat`.
- **`sell` y `buy` vienen con HTML** (`<span class="coin" title="20 Gold Coins" data-sort-value="200000">`). Parsear el `title` con regex `title="(\d+)\s+(\w+)\s+Coins` para obtener cantidad legible y moneda. **NO usar `data-sort-value` directamente** (devuelve el valor en centavos, produce "200000 GC" en vez de "20 GC").
- **Stats vacías como `""`** (no `null`). `toIntOrNull() ?: null` lo deja en `null` cuando están vacías.
- **`imagefile` SÍ funciona** (corregido en iteración 3). Da el filename directo, sin regex. Usar como prioridad; `image` (wikitext) solo como fallback.
- **URL-encoding de filenames** (iteración 3): la mayoría de filenames tienen espacios (`Terra Blade.png`, `Wood Plank.png`, `'0' Statue.png`...). `URLEncoder.encode` produce `+` para espacios → 404 en MediaWiki. Usar `android.net.Uri.encode` (que produce `%20`) o `URLEncoder.encode(...).replace("+", "%20")` para KMP.
- **Booleans como `"1"` / `""`**: `autoswing` y `hardmode` llegan como string. Comparar `== "1"` para `true`.

## Alternativas descartadas
- **Reutilizar el mismo DTO como modelo de dominio:** cambios de la API rompen la app; tipos permisivos en todo el código.
- **MapStruct o generación automática de mappers:** sobreingeniería para 13 campos.
- **Anotaciones de Moshi/kotlinx en el modelo de dominio:** acopla el dominio a la serialización. Mal para KMP.

## Riesgos & mitigación
- **Riesgo:** una columna nueva en la API no se mapea y se pierde en silencio. **Mitigación:** `ignoreUnknownKeys = true` en el JSON parser + tests unitarios del mapper que cubran cada campo.
- **Riesgo:** la regex de extracción de imagen falla con formatos no anticipados. **Mitigación:** cuando falle, el `imageFilename` queda `null` y se muestra placeholder; aceptable.
