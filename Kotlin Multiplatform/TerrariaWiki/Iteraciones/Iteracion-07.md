# Iteración 7 — Fix underscore en CDN + búsqueda global desde Home (2026-07-25)

Tras la quinta validación en móvil, el usuario reportó algo raro: "iconos que se veían antes no se ven, pero otros que no se veían ahora se ven". GLM-5.2 (plan) auditó con probes curl y descubrió que el directorio `/images/` de terraria.wiki.gg requiere **underscores `_`** para espacios, no `%20` que producía `buildItemImageUrl`. Antes funcionaba porque `Special:Redirect` convertía automáticamente `%20` → `_` en el redirect final; ahora, usando la URL directa `/images/`, esa conversión no ocurre. **Fix:** reemplazar `' '` con `'_'` directamente en el encoder.

Aprovechando la iteración, se implementó la **búsqueda global** desde Home (que estaba pendiente de iteración 7 según la iteración 6). Icono de lupa en el TopAppBar del Home abre un `SearchScreen` con `OutlinedTextField` y debounce 250ms contra `action=query&list=search&srnamespace=0`. Al tap en un resultado intenta `getByName(title)`; si no es item, el `ItemDetailViewModel` muestra el error "No se encontró".

Minimax M3 (build) ejecutó 5 commits:

| # | Commit | Tipo | Cambio |
|---|---|---|---|
| 1 | `ddd161d` | fix | `buildItemImageUrl` reemplaza espacios con `_` (no `%20`) |
| 2 | `18ab275` | test | `ImageUrlTest` cubriendo underscore, apostrofes y paréntesis |
| 3 | `8bd7a81` (ya en repo) | feat | Search: domain, mapper, repository, ViewModel, Screen, nav, Koin |
| 4 | `d121f56` | test | SearchViewModelTest con Turbine |
| 5 | `5441b82` | fix | Build fixes (IconButton import) y refactor de tests con Turbine |

### Bugs corregidos

| Bug | Causa | Síntoma |
|---|---|---|
| Items con espacios no se ven en iteración 6 | `/images/` espera `_` para espacios, no `%20` | Terra Blade, Copper Broadsword, etc. no cargaban imagen |
| Búsqueda global no existía | Solo navegación por categorías | No había forma rápida de encontrar un item concreto |

### Features nuevas

- **Búsqueda global** desde Home con icono de lupa en el TopAppBar.
- Debounce 250ms para no saturar la API.
- Resultados mixtos (items + páginas conceptuales): al tap en item, abre detalle; al tap en página no-item, error en el detalle.
- Limit=25 por query (rápido).

### Documentación nueva

- [[03-Patrones-Kotlin/Search-Global-Pattern]] — patrón completo de búsqueda, debate de tap en no-items, alternativas descartadas (webview, cache local, LIKE en listcat).

### Hand-off

- Tests: **40/40** passing (14 ItemsMapper + 5 SearchViewModel + 6 ItemsViewModel + 7 RecipesMapper + 4 ImageUrl + 4 CategoryViewModel).
- APK: 19 MB.
- Repo público: https://github.com/antonioalarconvirseda/terrariawiki con 24 commits.

---

[[00-Index]]
