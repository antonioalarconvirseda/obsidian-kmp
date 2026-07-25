# Cargo `HOLDS` filter — filtrado de campos tipo List

## Contexto
La extensión **Cargo** de MediaWiki devuelve campos "List" como strings con delimitador `^` (e.g. `"block^crafting material"`). Al filtrar en `cargoquery`, `where=type='weapon'` falla con `internal_api_error_MWException` porque el operador `=` solo funciona con campos escalares. Para listas hay que usar un operador específico de Cargo.

## Decisión
Adoptar el operador **`HOLDS 'value'`** que comprueba si un valor está contenido en una lista de Cargo (independientemente de la posición).

```sql
-- funciona
where = "type HOLDS 'weapon'"

-- falla con MWException
where = "type = 'weapon'"
```

`HOLDS` está documentado en la [wiki de Cargo](https://www.mediawiki.org/wiki/Extension:Cargo/Querying_data) como el operador para "el valor es uno de los elementos de la lista".

## Implementación Kotlin

```kotlin
override suspend fun queryByCategory(
    category: ItemCategory,
    fields: String = ItemsApiImpl.DEFAULT_FIELDS,
    limit: Int = 50,
    offset: Int = 0
): CargoResponse<ItemDto> = client.get(TerrariaApiConfig.BASE_PATH) {
    parameter("action", "cargoquery")
    parameter("tables", "Items")
    parameter("fields", fields)
    parameter("where", "type HOLDS '${category.apiFilter}'")
    parameter("limit", limit)
    parameter("offset", offset)
    parameter("format", "json")
}.body()
```

`ItemCategory` es un enum con el valor Cargo real:
```kotlin
enum class ItemCategory(
    val displayName: String,
    val apiFilter: String,   // valor para "type HOLDS '...'"
    val iconAsset: String,
    val colorHex: Long
) {
    WEAPONS("Armas", "weapon", "gavel", 0xFFE94B4B),
    ARMOR("Armaduras", "armor", "shield", 0xFF4A93B0),
    POTIONS("Pociones", "potion", "local_drink", 0xFFF2C94C),
    // ... 10 categorías totales
}
```

## Verificación con curl (iteración 4)

```bash
# falla con MWException
curl "...cargoquery&tables=Items&fields=name&where=type='weapon'&format=json"
# → { "error": { "code": "internal_api_error_MWException" } }

# funciona
curl "...cargoquery&tables=Items&fields=name&where=type HOLDS 'weapon'&format=json"
# → 483 items
```

Counts verificados:
- `weapon` → 483
- `accessory` → 480
- `armor` → 333
- `potion` → 159
- `block` → 256
- `consumable` → 43
- `mechanism` → 200
- `furniture` → 500
- `vanity` → 500
- `tool` → varios

## Alternativas descartadas

- **`type = 'weapon'`**: falla con MWException (verificado).
- **`type LIKE '%weapon%'`**: funciona en algunos casos pero es SQL, no semántico, y matchea "weaponry" también.
- **Filtrado en cliente (Kotlin)**: pedir todos los items y filtrar localmente → 500+ items por request, varios MB descargados, muy lento.
- **Tabla `Items` con un campo `category` explícito**: la wiki no tiene ese campo, habría que añadirlo en la wiki.
- **Dos queries: una con `type='weapon'` y otra con `type='weapon^X'`**: workaround hacky, no escala.

## Riesgos & mitigación

- **Riesgo:** algunas categorías devuelven 500 (máximo por query) → no es total real, hay que paginar. **Mitigación:** `Repository.refreshByCategory` y `loadMoreByCategory` con `offset = size actual`. `hasMoreFor` se pone a `false` si la respuesta trae < 50 items.
- **Riesgo:** `type` es el único campo List que filtramos; un item que no tenga `type` no aparece en ninguna categoría. **Mitigación:** la categoría `MISC` usa `furniture` como fallback para items sin tipo específico. Si un item no encaja en ningún filtro, queda en Misceláneo.
- **Riesgo:** la wiki podría cambiar el valor de `type` (e.g. `weapon` → `weapons` con 's'). **Mitigación:** los counts de probe eran precisos el 2026-07-25; documentar en `Query-Ejemplos.md`. Si cambian, ajustar `ItemCategory.apiFilter`.
- **Riesgo:** rate-limit (~60 req/min). **Mitigación:** una query por tap de categoría, no prefetch agresivo. El `hasMore` ya evita re-fetch en el mismo scrolleo.
