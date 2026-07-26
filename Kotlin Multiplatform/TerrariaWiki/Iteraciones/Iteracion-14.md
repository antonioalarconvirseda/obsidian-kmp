# Iteración 14 — Ajuste Home v2: sprites reales + grid 3 columnas (revierte Iteración 13) (2026-07-26)

El usuario probó la Iteración 13 (iconos `PixelIcon` + layout banner) en móvil y no le convenció en absoluto, especialmente los iconos abstractos. Pidió mirar la wiki Fandom de Terraria (`terraria.fandom.com/es/wiki/Wiki_Terraria`) como referencia directa. `WebFetch` devolvió 402 y `curl` directo (con varios User-Agent) devolvió 403 — Cloudflare bloquea el scraping del sitio — así que el usuario compartió 2 capturas de pantalla, que sí se pudieron analizar visualmente.

La referencia mostraba: grid denso de 6 columnas con tarjetas pequeñas cuadradas, **sprites reales del juego** (no formas dibujadas) arriba + nombre debajo, fondo neutro oscuro igual para todas las categorías (el color viene del propio sprite, no de un tinte por categoría). Se validaron las decisiones con el usuario ANTES de implementar esta vez (lección aprendida de Iteración 13, donde se implementó sin confirmar visualmente primero).

| Cambio | Archivo(s) | Descripción |
|---|---|---|
| revert | `PixelCategoryIcons.kt` (eliminado) | Se descarta el enfoque de iconos dibujados a mano de Iteración 13 |
| feat | `ItemCategory.kt` | Nuevo campo `representativeImageFile: String` — un item real por categoría (`"Terra Blade.png"`, `"Wood.png"`, etc.) |
| refactor | `HomeScreen.kt` | Grid `Fixed(2)`→`Fixed(3)`; `CategoryCard` vuelve a `Column` compacta (cuadrada, `aspectRatio(1f)`) con `AsyncImage` real vía Coil/CDN en vez de icono dibujado; fondo de tarjeta vuelve a neutro (sin tinte de bioma) |

### Documentación actualizada
- [[03-Patrones-Kotlin/Pixel-Icon-Rendering]] — marcada como revertida al inicio, con el motivo, se conserva como historial.
- [[05-UI-Design-System]] — iconografía y `CategoryCard` actualizados de nuevo.

### Hand-off
- Build/tests: sin cambios de lógica de negocio, 51/51 se mantiene.
- **Riesgo abierto:** solo `"Terra Blade.png"` y `"Wood.png"` son filenames verificados (ya usados en tests existentes); el resto de `representativeImageFile` son nombres de items conocidos pero no verificados contra el CDN real desde este entorno (sin acceso de red a imágenes). Si alguna categoría muestra el icono de error en el móvil, es un ajuste de una línea en `ItemCategory.kt` una vez el usuario confirme el nombre exacto.
- **Pendiente:** validación visual en dispositivo — este entorno sigue sin `adb`.
- Lección para iteraciones futuras de UI: cuando el usuario tenga una referencia visual concreta (captura, URL), pedirla y validarla ANTES de implementar, no después — evita una ronda de trabajo descartado.

---

[[00-Index]]
