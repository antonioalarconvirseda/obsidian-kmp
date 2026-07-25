# Navigation Compose

## Contexto
TerrariaWiki tiene dos pantallas: lista de items y detalle. Necesitamos un mecanismo de navegación que:
- Soporte argumentos (el `name` del item).
- Mantenga un backstack correcto.
- Sea declarativo.
- Encaje con Compose.

## Decisión
Adoptar **Navigation Compose 2.7.x** con rutas string tipadas y `NavType.StringType` para argumentos.

### Convenciones
- Rutas como constantes en un object:
  ```kotlin
  object ItemsRoutes {
      const val LIST = "items"
      const val DETAIL = "item/{name}"
      fun detail(name: String) = "item/${URLEncoder.encode(name, "UTF-8")}"
  }
  ```
- Argumentos URL-encoded para soportar nombres con espacios o caracteres especiales (e.g. "Wood Plank").
- `composable(route, arguments) { backStackEntry -> ... }` por destino.
- `navController.navigate("...")` para ir, `navController.popBackStack()` para volver.
- `LaunchedEffect(itemId)` en la pantalla de detalle para disparar `viewModel.load(...)` cuando llega el argumento.

## Implementación Kotlin

```kotlin
@Composable
private fun TerrariaWikiNavHost() {
    val navController = rememberNavController()
    NavHost(
        navController = navController,
        startDestination = "items"
    ) {
        composable("items") {
            ItemsScreen(
                onItemClick = { name ->
                    navController.navigate("item/${URLEncoder.encode(name, "UTF-8")}")
                }
            )
        }
        composable(
            route = "item/{name}",
            arguments = listOf(navArgument("name") { type = NavType.StringType })
        ) { backStackEntry ->
            val encoded = backStackEntry.arguments?.getString("name").orEmpty()
            val name = URLDecoder.decode(encoded, "UTF-8")
            ItemDetailScreen(
                name = name,
                onBack = { navController.popBackStack() }
            )
        }
    }
}
```

## Alternativas descartadas
- **Fragments + Navigation Component clásico:** pre-Compose, requiere XML, no encaja con el resto.
- **Implementación manual de navegación con `mutableStateOf`:** reinventar la rueda sin backstack ni deep links.
- **Voyager / Decompose:** buenas librerías KMP, pero añaden una dependencia extra. Para Android puro, Navigation Compose es suficiente.

## Riesgos & mitigación
- **Riesgo:** argumento con caracteres especiales no se preserva. **Mitigación:** URL-encode al navegar, URL-decode al recibir.
- **Riesgo:** `popBackStack()` en la pantalla inicial cierra la app. **Mitigación:** `popBackStack()` retorna `false` si no hay más; el sistema lo maneja con finish().
- **Riesgo:** perder el state al rotar. **Mitigación:** el ViewModel sobrevive a rotaciones por diseño; `collectAsStateWithLifecycle()` re-suscribe automáticamente.
