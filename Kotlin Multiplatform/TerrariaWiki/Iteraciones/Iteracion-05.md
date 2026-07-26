# Iteración 5 — Icono Tree of Life real + apostrofe fix + Recetas (2026-07-25)

Tras la tercera validación en móvil, el usuario reportó 3 cosas: el icono de la app se notaba "generado por AI", muchas imágenes de items no salían (especialmente las que tienen apostrofes como Abigail's Flower), y se necesitaba mostrar los crafteos en la ficha de detalle.

GLM-5.2 (plan) auditó y Minimax M3 (build) ejecutó 7 commits en `terrariawiki`:

| # | Commit | Tipo | Cambio |
|---|---|---|---|
| 1 | `d79c1ca` | feat | Icono real Tree of Life (PNG 512×512 provisto por el usuario) |
| 2 | `f6518ef` | fix | URL-encoding del apostrofe en filenames de items |
| 3 | (siguiente) | feat | Recipes API + mapper + repository (capa de datos) |
| 4 | (siguiente) | feat | Sección "Receta" en ItemDetailScreen |
| 5 | (siguiente) | test | RecipesMapperTest con 7 casos |
| 6 | (siguiente) | fix | Build errors (Koin, Ingredient type, IC launcher background) |
| 7 | (siguiente) | test | Fix ItemDetailViewModel tests con repository mock |

### Bugs corregidos

| Bug | Causa | Síntoma |
|---|---|---|
| Icono parecía generado | Vector drawable hecho a mano, no del juego | Identidad de marca poco profesional |
| Imágenes con apostrofe no salían | `android.net.Uri.encode` no codifica `'` (RFC 3986 unreserved) | `Abigail's Flower.png` → URL rota, icono de error en Coil |

### Features nuevas

- **Sección "Receta"** en ficha de detalle (entre Descripción y Estadísticas).
- Muestra: "Se craftea N × Item", "Estación: X", "• Ingrediente ×Cantidad".
- Solo se renderiza si `recipes.isNotEmpty()`.
- Soporta múltiples recipes para el mismo item (renderiza "Receta 1", "Receta 2").

### Documentación nueva

- [[03-Patrones-Kotlin/Recipes-API-Pattern]] — parser del campo `ings` con delimitador `¦`, decisión de NO mostrar query inversa, alternativas descartadas.

### Hand-off

- Tests: **31/31** passing (14 ItemsMapper + 6 ItemsViewModel + 7 RecipesMapper + 4 CategoryViewModel).
- APK: 19 MB con el nuevo icono.
- Repo público: https://github.com/antonioalarconvirseda/terrariawiki con 19 commits.
- Vault: https://github.com/antonioalarconvirseda/obsidian-kmp con 5 commits.
- **Home con 10 categorías** + infinite scroll por categoría, icono Tree of Life, chip de rareza legible, paginación completa de la wiki.

---

[[00-Index]]
