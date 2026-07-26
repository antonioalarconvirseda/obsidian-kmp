# Iteración 17 — Features Bosses + Events, primera ampliación más allá de Items (2026-07-26)

El usuario pidió añadir dos categorías nuevas separadas visualmente de las principales: **Jefes** (con foto) y **Eventos** (con foto), pidiendo explícitamente verificar primero si la wiki podía proveer esa información antes de implementar. Se siguió el flujo `brainstorming` → `writing-plans` → implementación, con investigación real contra la API (`curl`, no suposiciones) antes de diseñar.

**Investigación previa (decisiva para el diseño):**
- `action=cargotables` reveló que Cargo solo tiene **12 tablas totales** (`Drops, Equipinfo, Exclusive, History, Imageinfo, Items, Modifiers, NPCs, Recipes, Weapon_source, _fileData, _pageData`) — no existe `Events`.
- `NPCs` con `type HOLDS 'boss'` sí trae bosses reales (Betsy, Brain of Cthulhu, Deerclops, Duke Fishron, Empress of Light...) con imagen y stats — mismo patrón que `Items`.
- Para Events se probaron 2 heurísticas de imagen y ambas fallaron en ~40% de casos (ver [[03-Patrones-Kotlin/Static-Domain-Catalog]] para el detalle): parsear `{{flavor text|...|image=}}`/primer `[[File:...]]` del wikitext, y la convención `"<PageName> Icon.png"`. Se resolvió con preguntas al usuario en 3 puntos (fuente de Events, resolución de imagen, profundidad de detalle de Bosses/Events) antes de tocar código.

| Cambio | Archivo(s) | Descripción |
|---|---|---|
| feat | `features/bosses/` (nuevo, 11 archivos) | Feature completa `data/domain/di/ui` mirroreando `items/` — `BossesApiImpl` consulta `NPCs` con `type HOLDS 'boss'`, `BossesMapper` limpia HTML + wikitext `[[...]]` de los stats multi-modo, lista + detalle |
| feat | `features/events/` (nuevo, 3 archivos) | Sin `data/`/`di/` — `EventCatalog` estático en `domain/` con 14 eventos y filenames de imagen verificados uno a uno contra la API real; tap abre la wiki en el navegador (`Intent.ACTION_VIEW`), sin pantalla de detalle in-app |
| feat | `HomeScreen.kt` | Nueva sección "Más allá de los items" (`SectionHeader` + 2 `EntryCard`) separada del grid de Items vía `item(span = maxLineSpan)`, mismo mecanismo ya usado para `WelcomeCard`/`LatestVersionsCard` |
| feat | `MainActivity.kt` | Rutas nuevas `bosses`, `boss/{name}`, `events` |
| feat | `TerrariaWikiApp.kt` | `bossesModule` registrado en Koin |

### Bugs corregidos
- **Crash real en dispositivo** (no detectado por tests): `BossListScreen` usaba `key = { it.name }` en su `LazyColumn`, y "Dark Mage" aparece dos veces en la tabla `NPCs` (tiers T1/T3) → `IllegalArgumentException: Key "Dark Mage" was already used`. Es la misma clase de bug que ya había ocurrido con `internalname == "None"` en Armor (iteración previa, ver [[03-Patrones-Kotlin/Compose-LazyColumn]]) — confirma que es un riesgo recurrente de la API, no un caso aislado. Fix: `itemsIndexed` con key compuesta `"${boss.name}_$index"`.

### Documentación nueva
- [[03-Patrones-Kotlin/Static-Domain-Catalog]] — patrón nuevo: cuándo usar un `object` estático en `domain/` en vez del patrón `data/di` completo, con la investigación de Events como caso real.
- [[03-Patrones-Kotlin/Compose-LazyColumn]] — riesgo de key duplicada actualizado con la recurrencia en Bosses, lección consolidada.
- [[04-API-Terraria/Endpoint-Lista]] — lista completa de las 12 tablas Cargo, tabla `NPCs` documentada con sus campos y el quirk de `nameraw` no-único.
- [[04-API-Terraria/Query-Ejemplos]] — 2 ejemplos nuevos: query de bosses y `cargotables`.

### Hand-off
- Tests: **67/67** passing (suite previa + `BossesMapperTest` 9 casos + `BossListViewModelTest`/`BossDetailViewModelTest` 6 casos).
- Build: `./gradlew :app:assembleDebug :app:testDebugUnitTest` en verde.
- **Validado en dispositivo físico real** (no solo build): Home con sección separada, lista de Jefes con imágenes y stats, detalle de Betsy con vida/defensa/daño/retroceso legibles, tap en Eventos abre correctamente Blood Moon / Martian Madness en el navegador. El crash de `LazyColumn` se detectó y arregló en esta misma verificación — confirma otra vez que probar en dispositivo real encuentra bugs que la suite de tests no cubre.
- Roadmap: la decisión de "NPCs" pendiente en iteraciones anteriores queda parcialmente resuelta (Bosses reusa la tabla `NPCs`), pero **NPCs no-boss (enemigos regulares) sigue sin implementar** — sigue abierta la decisión entre eso, Room cache, o migración KMP.

---

[[00-Index]]
