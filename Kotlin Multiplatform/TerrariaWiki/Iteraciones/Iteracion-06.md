# Iteración 6 — Fix imágenes vía CDN directo (2026-07-25)

Tras la cuarta validación en móvil, el usuario reportó que "las imágenes aún no cargan". GLM-5.2 (plan) auditó con probes curl y descubrió que el problema **no era el encoding de URLs**, sino el **rate-limit (429) de CloudFlare sobre `Special:Redirect`** cuando el móvil pide 50 imágenes casi simultáneamente al hacer scroll de una lista.

**El fix:** saltarse `Special:Redirect` y construir directamente la URL final `https://terraria.wiki.gg/images/<filename>`. Esa URL está cacheada en CloudFlare con `cache-control: public, max-age=31536000, immutable` y responde `cf-cache-status: HIT` instantáneamente.

Minimax M3 (build) ejecutó 2 commits:

| # | Commit | Tipo | Cambio |
|---|---|---|---|
| 1 | `471dcfe` | fix | `buildItemImageUrl` usa CDN directo, sin Special:Redirect |
| 2 | `18ab275` | test | `ImageUrlTest` cubre encoding de espacios (%20) y apostrofes (%27) |

### Hand-off

- Tests: **34/34** passing (14 ItemsMapper + 6 ItemsViewModel + 7 RecipesMapper + 4 CategoryViewModel + 3 ImageUrl).
- APK: 19 MB.
- Repo público: https://github.com/antonioalarconvirseda/terrariawiki con 21 commits.

---

[[00-Index]]
