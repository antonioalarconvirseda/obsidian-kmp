# Recipes API Pattern — crafteos en TerrariaWiki

## Contexto
La wiki de terraria.wiki.gg tiene una tabla Cargo `Recipes` independiente de `Items` que describe cómo se craftean los items. Necesitamos esto para mostrar en la ficha de detalle qué se necesita para craftear el item y en qué estación.

## Decisión
**Mostrar solo la "Receta" directa**: dado un item X, mostrar sus ingredientes y la estación requerida. **No** mostrar "qué se craftea con X" (query inversa) porque MediaWiki Cargo no soporta filtros `LIKE` ni `HOLDS` sobre el campo `ings` (string con delimitadores especiales `¦`).

### Estructura del API

```bash
GET https://terraria.wiki.gg/api.php
  ?action=cargoquery
  &tables=Recipes
  &fields=result,amount,station,ings
  &where=result='Wooden%20Sword'
  &format=json
```

### Estructura del campo `ings`

El campo viene como string con formato especial:
- Cada ingrediente delimitado por `^`.
- Cada ingrediente envuelto por `¦` (U+00A6, BROKEN BAR) con la cantidad al final.

Ejemplo Terra Blade:
```
¦Broken Hero Sword¦1^¦True Excalibur¦1^¦True Night's Edge¦1
```

Parseo en `RecipesMapper.kt`:
```kotlin
private const val RECIPES_LIST_DELIMITER = "^"
private const val FIELD_WRAP = "\u00a6"  // ¦

fun RecipeDto.toDomain(): Recipe {
    val ingredients = ings
        .split(RECIPES_LIST_DELIMITER)
        .filter { it.isNotBlank() }
        .map { entry ->
            val parts = entry.split(FIELD_WRAP)
            Ingredient(
                name = parts.getOrNull(1).orEmpty(),
                quantity = parts.getOrNull(2)?.toIntOrNull() ?: 1
            )
        }
    return Recipe(result, amount.toIntOrNull() ?: 1, station, ingredients)
}
```

## Implementación Kotlin

### Capa de datos (`Recipe` en domain)
```kotlin
data class Recipe(
    val result: String,
    val amount: Int,
    val station: String,
    val ingredients: List<Ingredient>
)
data class Ingredient(val name: String, val quantity: Int)
```

### Repository
```kotlin
override suspend fun getRecipes(name: String): Result<List<Recipe>> = runCatching {
    val response = api.getRecipesForItem(name)
    response.cargoquery.map { it.title.toDomain() }
}
```

### UI (sección "Receta" en ItemDetailScreen)
Renderiza solo si `recipes.isNotEmpty()`. Si hay varias recipes para el mismo item, las enumera como "Receta 1", "Receta 2", ...

```kotlin
if (recipes.isNotEmpty()) {
    DetailSection(title = "Receta") {
        recipes.forEach { recipe ->
            if (recipes.size > 1) {
                Text("Receta ${recipes.indexOf(recipe) + 1}", style = titleSmall)
            }
            Text("Se craftea ${recipe.amount} × ${recipe.result}", style = bodyMedium, fontWeight = Medium)
            if (recipe.station.isNotBlank()) {
                Text("Estación: ${recipe.station}", style = bodySmall, onSurfaceVariant)
            }
            recipe.ingredients.forEach { (name, qty) ->
                Text("  • $name × $qty", style = bodySmall, onSurfaceVariant)
            }
        }
    }
}
```

## Posición en la ficha de detalle

1. Cabecera (imagen + nombre + rareza + tipos)
2. Descripción (si tooltip)
3. **Receta** (NUEVO — si craftable)
4. Estadísticas
5. Inventario, Categorías, Economía, Información

## Tests añadidos (7 casos en `RecipesMapperTest.kt`)

- Receta con 1 ingrediente.
- Receta con cantidad explícita (`¦Wood¦7`).
- Receta multi-ingrediente (2 ingredientes).
- Receta multi-ingrediente con cantidades variadas.
- Fallback a amount=1 cuando campo amount está vacío o no es numérico.
- Lista vacía cuando `ings` es vacío.
- Receta con apostrofe en el nombre del ingrediente (Terra Blade).

## Limitaciones conocidas

- **No se puede buscar "qué se craftea con X"** (query inversa) por la limitación de Cargo. Si se necesita, opciones:
  - **A.** Cache client-side: cargar 500-1000 recipes y filtrar en Kotlin. Memory ~200 KB, factible.
  - **B.** Scraping HTML de la página de X en wiki.gg y parsear las recetas listadas en la infobox. Frágil, depende del markup.
  - **C.** Aceptar la limitación y dejar solo "Receta" (decisión actual).

## Alternativas descartadas

- **Mostrar la query inversa (se usa para):** no funcional con Cargo.
- **Web scraping del infobox de cada item:** demasiado frágil y rompe el principio de usar la API estructurada.
- **Agrupar varias recipes en una sola sección:** ya implementado con numeración "Receta 1", "Receta 2".
- **Pre-cargar todas las recipes en cache:** viable pero fuera del MVP. Memory + startup time innecesarios.
