---
placeholder: se completará durante Fase 4
actualizado: tras implementar ItemsRepository
---

# Cargo envelopes — estructura de respuesta

## Contexto
La extensión **Cargo** de MediaWiki devuelve los datos tabulares envueltos en una estructura fija que hay que parsear correctamente. En TerrariaWiki necesitamos entender el envelope para deserializarlo a data classes Kotlin.

## Estructura envelope

Para `action=cargoquery`, la respuesta es:

```json
{
  "cargoquery": [
    {
      "title": {
        "name": "Wood",
        "type": "block^crafting material",
        "rare": "0",
        "sell": "<span class=\"coin\" ...>60 CC</span>",
        "damage": "",
        "defense": "",
        "itemid": "2702"
      }
    },
    ...
  ]
}
```

Es decir:
- Array `cargoquery` (puede faltar si no hay resultados).
- Cada elemento tiene un campo `title` (a pesar del nombre, contiene TODA la fila).
- Dentro de `title` van los campos solicitados en `fields=`.

## Implementación Kotlin

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
    val sell: String? = null,
    // ... el resto
)
```

Uso:
```kotlin
val response: CargoResponse<ItemDto> = client.get(URL) { ... }.body()
val items: List<Item> = response.cargoquery.map { it.title.toDomain() }
```

## Tipos de campos especiales

| Tipo | Formato en la respuesta | Manejo |
|---|---|---|
| `Integer` | String con el número (`"2702"`) | `toIntOrNull() ?: null` |
| `Boolean` | `"1"` / `"0"` o string vacío | Mapear a Boolean |
| `String` (con default) | String plano | directo |
| `Wikitext` | Puede contener `[[File:X.png\|caption]]` | regex para extraer |
| `String isList delimiter "^"` | `"block^crafting material"` | split por `"^"` |

## Limitaciones conocidas (probes del 2026-07-25)

1. **`imagefile` produce `internal_api_error_MWException`** cuando se incluye en joins multi-campo. Workaround: usar `image` (wikitext) y parsear con regex.
2. **`rare` como String:** se parsea con `toIntOrNull() ?: 0`.
3. **Stats vacías como `""`:** tratar como `null` en el mapper.
4. **`sell` con HTML:** limpiar con `stripHtml()` + regex `data-sort-value="(\d+)"`.

## Búsqueda (`action=query&list=search`)

```json
{
  "query": {
    "searchinfo": { "totalhits": 837 },
    "search": [
      { "title": "Slimes", "pageid": 9609, "snippet": "..." }
    ]
  }
}
```

Implementado en `SearchResponse` / `SearchQuery` / `SearchHit` / `SearchInfo`.

## Alternativas descartadas
- Usar el campo `parse` (HTML completo): innecesario para listas de items; parsear wikitext es más costoso.
- Ignorar el envelope y leer el JSON como `Map<String, Any>`: viable pero pierde type safety.
