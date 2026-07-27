# Iteración 20 — Fix icono categoría Jefes + rediseño fallback genérico (2026-07-27)

Bug reportado: la tarjeta "Jefes" en Home no mostraba icono propio, caía siempre al drawable de error. Causa raíz: `EntryCard` (`HomeScreen.kt`) tenía hardcodeado `imageFile = "Animated Betsy.gif"` — nombre de archivo inventado/inexistente en terraria.wiki.gg. A diferencia de `ItemCategory.representativeImageFile` (verificado contra la wiki real), este valor nunca se validó, así que Coil recibía 404 en cada carga y renderizaba `ic_item_error`.

**Decisión: usar King Slime como icono de la categoría, verificado contra la wiki real.** Es el primer jefe del juego (encaja semánticamente con "entrar a la categoría Jefes") y su filename real (`King_Slime.gif`) se confirmó vía WebFetch a `terraria.wiki.gg/wiki/King_Slime` antes de hardcodearlo — mismo patrón que ya usa `ItemCategory` (`"Item Name.ext"` con espacios, `buildItemImageUrl` convierte a underscore al construir la URL).

**Decisión: rediseñar `ic_item_error.xml`, no solo arreglar el filename.** El usuario pidió explícitamente que el fallback deje de ser el icono actual (un escudo/hexágono rojo estilo slime, pensado en su día como "error visual" pero que se lee como icono genérico equivocado) y pase a ser un icono neutro de documento genérico. Se rediseñó como documento con esquina doblada + líneas de texto, en `stone_gray`/`cloud_white` (mismos colores que `ic_item_placeholder`, que ya era razonablemente neutro) para que ambos fallbacks (loading y error) se lean como "sin imagen", no como un elemento de juego fuera de contexto.

| Cambio | Archivo(s) | Descripción |
|---|---|---|
| fix | `features/items/ui/HomeScreen.kt` | `EntryCard` de Jefes: `imageFile` de `"Animated Betsy.gif"` (inexistente) a `"King Slime.gif"` (verificado) |
| fix | `res/drawable/ic_item_error.xml` | Rediseño: de escudo rojo con círculo blanco a documento genérico con esquina doblada + líneas (`stone_gray`/`cloud_white`) |

### Hand-off
- Verificado con `./gradlew :app:assembleDebug` → build limpio.
- No se tocó `ic_item_placeholder.xml` (loading state) — ya era un icono de documento razonablemente neutro, no necesitaba cambio.
- `ic_item_error` se comparte entre `HomeScreen.kt` (Jefes + categorías Items), `BossCard.kt`, `ItemCard.kt` y `EventListScreen.kt` — el rediseño aplica a los cuatro sitios automáticamente (un solo drawable).
- Pendiente de decisión de roadmap (sin cambios): NPCs, Enemigos, cache offline con Room, o migración KMP real — ninguno tocado en esta iteración.
- Commit: `fix: correct Bosses category icon and redesign error fallback drawable` — directo a `main`.

---

[[00-Index]]
