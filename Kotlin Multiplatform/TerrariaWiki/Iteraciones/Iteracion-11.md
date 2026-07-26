# Iteración 11 — Fix crash en categoría Armaduras + entorno persistente (2026-07-26)

El usuario reportó: "cuando le doy a la sección de armaduras la aplicación crashea, solo ocurre con eso". Investigado con agente Explore de solo lectura + query directa a la API real de Cargo (`where=type HOLDS 'armor'`, 100 filas), siguiendo la regla de [[Debug-Lecciones-User-Agent]] de nunca iterar sin evidencia real.

**Causa raíz:** Cargo devuelve el string literal `"None"` (no `null` JSON) para campos escalares sin valor. Armor incluye, además de las piezas individuales, una fila "set" agregada por cada armadura completa (`"Adamantite armor"`, `"Bee armor"`...) con `internalname = "None"`. El mapper (`?.takeIf { it.isNotBlank() }`) no filtraba ese string, así que 21+ items terminaban con `internalName == "None"`, y la key de `LazyColumn` (`key = { it.internalName ?: it.name }`) exige unicidad → `IllegalArgumentException: Key "None" was already used`, crash solo en Armor (la única categoría con filas "set").

Claude Code (Sonnet 5, plan + build en la misma sesión) ejecutó el fix vía TDD:

| Cambio | Archivo | Descripción |
|---|---|---|
| fix | `ItemsMapper.kt` | Helper `cargoValueOrNull()` filtra blank y `"None"` literal; aplicado a `internalname`, `imagefile`, `stack`, `tooltip` |
| test | `ItemsMapperTest.kt` | Caso `internalname/imagefile/stack = "None"` → deben mapear a `null` (falló antes del fix, pasó después) |
| fix | `ItemsScreen.kt`, `ItemsByCategoryScreen.kt` | `items` → `itemsIndexed`, key compuesta `"${internalName ?: name}_$index"` como defensa adicional ante duplicados futuros |

### Documentación actualizada
- [[03-Patrones-Kotlin/Dto-Mapper]] — nueva particularidad "Cargo `\"None\"` literal" + `cargoValueOrNull()` en el snippet.
- [[03-Patrones-Kotlin/Compose-LazyColumn]] — nuevo riesgo real documentado (keys duplicadas) + snippet actualizado a `itemsIndexed`.

### Entorno de desarrollo (fuera de este repo de código)
Se detectaron dos problemas de persistencia de entorno en el Mac del usuario (no relacionados con el bug de Armor): `JAVA_HOME` y `adb` solo estaban disponibles dentro de la sesión de la herramienta, no en `~/.zshrc`, así que cada terminal nueva fallaba con "Unable to locate a Java Runtime" y luego "adb: command not found". Fix: añadidas 2 secciones a `~/.zshrc` — `JAVA_HOME` apuntando al JBR embebido de Android Studio (`/Applications/Android Studio.app/Contents/jbr/Contents/Home`), y `ANDROID_HOME` + `platform-tools` en `PATH`. Verificado con `zsh -ic` (nota: `zsh -lc` NO sourcea `.zshrc`, solo shells interactivos lo hacen).

### Hand-off
- Tests: mapper suite en verde tras el fix (caso `"None"` añadido).
- Pendiente: confirmación del usuario en dispositivo físico tras `adb install -r ...` abriendo la sección Armaduras.
- Decisión de siguiente feature aún abierta: NPCs, Room cache, Dark mode, o migración KMP.

---

[[00-Index]]
