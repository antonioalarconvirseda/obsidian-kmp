# Iteración 13 — Ajuste Home: iconos pixel custom + layout banner (2026-07-26)

Tras probar la Iteración 12 en móvil, el usuario confirmó que le gustó el resultado global (fuentes y descripciones ampliadas "encantaron"), pero señaló que las tarjetas de categoría de Home seguían viéndose "simplonas": iconos genéricos de Material Icons sin relación con el juego, y grid uniforme poco editorial. Compartió como referencia visual la wiki Fandom de Terraria (no se pudo cargar directamente, `WebFetch` devolvió 402), así que el criterio se resolvió por preguntas directas de alcance.

Decisiones confirmadas: (1) set de iconos vectoriales pixel/fantasy propio, no Material Icons ni imágenes reales de items; (2) grid 2 columnas con tarjetas tipo banner (icono grande + nombre superpuesto con gradiente), no grid simétrico icono+label.

| Cambio | Archivo(s) | Descripción |
|---|---|---|
| feat | `PixelCategoryIcons.kt` (nuevo) | `PixelIcon` composable (`Canvas`/`drawRect` por celda) + 10 patrones 12×12, uno por `ItemCategory` |
| refactor | `HomeScreen.kt` | `CategoryCard` reescrita a layout banner (`Box` + `PixelIcon` + gradiente + label), altura 120dp→160dp; eliminada `iconForCategory()` y los 9 imports de Material Icons que solo esa función usaba |

### Documentación nueva
- [[03-Patrones-Kotlin/Pixel-Icon-Rendering]] — nueva nota de patrón para el mecanismo de render pixel-art vía `Canvas`.
- [[05-UI-Design-System]] — sección "Iconografía" y componentes actualizados.

### Hand-off
- Build/tests: sin cambios de lógica de negocio, 51/51 se mantiene.
- **Pendiente:** validación visual en dispositivo — este entorno sigue sin `adb`. Los 10 patrones pixel son texto plano fácil de ajustar si alguna silueta no se lee bien a 160dp, o si el contraste del texto blanco falla sobre algún acento claro (`Crystal`, `Desert`).

---

[[00-Index]]
