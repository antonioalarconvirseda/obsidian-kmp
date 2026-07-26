# Iteración 15 — Paleta más viva, borde de hierba, widgets de Home (2026-07-26)

El grid de categorías con sprites reales (Iteración 14) convenció más, pero el usuario pidió seguir acercándose a la referencia: colores más vivos ("ese azul chulo" de la wiki, no pastel), el detalle de borde de hierba encima de las tarjetas de categoría, y más contenido en Home (mensaje de bienvenida + últimas versiones, como los widgets de dashboard de la wiki). Se validaron los 3 puntos con preguntas antes de implementar.

| Cambio | Archivo(s) | Descripción |
|---|---|---|
| feat | `Color.kt` | Saturación de la paleta base + acentos de bioma subida ~+22% en HSL (mismo hue); `StoneGray` excluido a propósito (es el único token neutro) |
| feat | `InventorySlotCard.kt` | Parámetro `topAccent: Boolean` + `GrassTopAccent` (Canvas, franja verde con borde zigzag) |
| feat | `HomeScreen.kt` | `CategoryCard` activa `topAccent = true`; nuevos `WelcomeCard`/`LatestVersionsCard` como items de ancho completo (`GridItemSpan(maxLineSpan)`) antes del grid de categorías |

### Documentación actualizada
- [[05-UI-Design-System]] — tabla de paleta con los nuevos hex, `GrassTopAccent` y los widgets de Home documentados.

### Hand-off
- Build/tests: sin cambios de lógica de negocio, 51/51 se mantiene.
- **Nota:** `LatestVersionsCard` tiene contenido hardcodeado (no hay endpoint de versión en la API Cargo) — recordar actualizarlo a mano si Terraria libera una versión nueva.
- **Pendiente:** validación visual en dispositivo — este entorno sigue sin `adb`.

---

[[00-Index]]
