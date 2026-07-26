# Iteración 4 — Rediseño UX + Home categorías + icono + fixes visuales (2026-07-25)

Tras la segunda validación en móvil, el usuario reportó 5 problemas críticos: imágenes solo en statues, label blanco invisible, icono placeholder feo, solo 50 items visibles, y necesidad de reorganización por categorías como en la wiki Fandom de referencia.

GLM-5.2 (plan) auditoría y Minimax M3 (build) ejecutaron 7 commits `fix:` / `feat:` en `terrariawiki`:

| # | Commit | Tipo | Cambio |
|---|---|---|---|
| 1 | `9f6afca` | fix | Chip rareza blanco invisible para items `rare=0` (gris claro con texto oscuro) |
| 2 | `761d57d` | feat | Icono app: Tree of Life sobre degradado teal→verde |
| 3 | `695ff72` | feat | Modelo `ItemCategory` + `queryByCategory` con `HOLDS` + paginación |
| 4 | (siguiente) | feat | HomeScreen con grid 2x5 + ItemsByCategoryScreen infinite scroll |
| 5 | (siguiente) | feat | Koin: register `CategoryViewModel` con parámetro |
| 6 | (siguiente) | test | `CategoryViewModelTest` con 4 tests |
| 7 | (siguiente) | fix | `observeByCategoryDirect` para que el VM lea state actual |

### Cambios de UX principales

- **Pantalla Home (nuevo start destination):** grid 2x5 de categorías con icono Material Symbols + color de la paleta Terraria.
- **Lista por categoría:** topbar con nombre español + back, `LazyColumn` con infinite scroll via `derivedStateOf` (precarga al faltar 5 items para el final).
- **Paginación de 50 en 50:** ~10 categorías × ~500 items = ~5000 items accesibles, vs. 50 antes.
- **Sin rate-limit:** una query por `loadMore()`, throttling por `_isLoadingMore.value` evita duplicados.

### Bugs corregidos

| Bug | Causa | Síntoma |
|---|---|---|
| Rarity chip blanco invisible | `rare=0` tenía fondo blanco y texto blanco | Statues y furniture (la mayoría de los 50 primeros items) no se distinguían |
| Solo statues con icono | Las imágenes con filenames cortos sí cargaban; las demás fallaban por la URL sin encoding | El usuario percibía "solo statues funcionan", el resto no |
| 50 items máximo | `limit=50` hardcoded sin paginación | No se podía explorar más allá de los primeros 50 |

### Documentación nueva

- [[03-Patrones-Kotlin/Cargo-HOLDS-filter]] — operador `HOLDS` para campos tipo List (alternativa a `=` que falla con MWException).
- [[03-Patrones-Kotlin/Home-Navigation-Pattern]] — rationale de la reorganización Home + drill-down con infinite scroll.
- [[05-UI-Design-System]] actualizado con sección "Icono de la app" (Tree of Life).

### Hand-off

- Tests: **24/24** passing (14 ItemsMapper + 6 ItemsViewModel + 4 CategoryViewModel).
- APK: 19 MB.
- Repo público: https://github.com/antonioalarconvirseda/terrariawiki con 12 commits.
- Vault: https://github.com/antonioalarconvirseda/obsidian-kmp con 4 commits.

Pendiente para próxima sesión GLM-5.2:
- Validación en móvil tras reinstalar APK.
- Decisión de siguiente feature: NPCs (reusar patrón), Room cache, Dark mode "Underworld", o migración real a KMP.
- **Hand-off:** listo para que GLM-5.2 planifique la siguiente iteración.

---

[[00-Index]]
