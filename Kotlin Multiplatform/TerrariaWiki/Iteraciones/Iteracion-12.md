# Iteración 12 — Rediseño visual/UX (Underworld dark mode, Silkscreen, RarityTier, shapes/spacing) (2026-07-26)

El usuario pidió un rediseño completo de la interfaz con criterio UX profesional: amigable/accesible pero fiel a la identidad visual de Terraria. Tras un brainstorming con preguntas de alcance (navegación fuera de este trabajo, dark mode Underworld sí se implementa, código muerto se elimina, fuente pixel solo en headers y empaquetada offline, paleta se amplía con acentos de bioma aplicados ya a categorías, shapes en look "slot de inventario"), Claude Code (Sonnet 5, plan + build en la misma sesión) ejecutó el rediseño.

| Cambio | Archivo(s) | Descripción |
|---|---|---|
| chore | `ItemsScreen.kt`, `navigation/ItemsNavigation.kt` | Eliminado código muerto (no referenciado desde `MainActivity` desde la reorganización Home de iteración 4) |
| feat | `Color.kt` | 4 acentos de bioma (`Corruption`/`Crystal`/`Desert`/`Abyss`) + paleta Underworld dark (`ObsidianBlack`/`AshSurface`/`LavaOrange`/`EmberRed`/`MagmaCrust`/`CinderWhite`) |
| feat | `Theme.kt` | `TerrariaDarkColors` reconstruido con paleta Underworld real (antes reusaba tonos del claro); `shapes = TerrariaShapes` wireado |
| feat | `Shapes.kt` (nuevo) | Esquinas 8-12dp + `InventorySlotBorderWidth`/`Color` para el borde "slot de inventario" |
| feat | `Spacing.kt` (nuevo) | Escala `xs..xxl` (4-24dp) nombrando los valores ad hoc ya en uso |
| feat | `Type.kt` | Fuente pixel Silkscreen (`res/font/silkscreen_{regular,bold}.ttf`, empaquetada offline) en headers; `titleSmall`/`bodySmall` añadidos (faltaban, caían a default M3) |
| refactor | `RarityTier.kt` (nuevo, domain) | Enum único de rareza (color+label+level), reemplaza `Color.kt::rarityColor()` + overrides locales de `RarityChip.kt` |
| feat | `InventorySlotCard.kt` (nuevo) | Card compartida con shape+borde, reemplaza 4 implementaciones privadas duplicadas (`HomeScreen`, `ItemDetailScreen` x2, `SearchScreen`) |
| feat | `ItemCategory.kt` | Recolor de 4 categorías a los nuevos acentos de bioma (Vanity→Corruption, Accesorios→Crystal, Bloques→Desert, Mecanismos→Abyss) |
| test | `RarityTierTest.kt` (nuevo) | 4 casos: `fromLevel(-1)`, `fromLevel(0)`, `fromLevel(11)`, fallback a `COMMON` para nivel no mapeado |

### Bugs corregidos
- Duplicación/inconsistencia de color y label de rareza entre `Color.kt` y `RarityChip.kt`.
- Nombre engañoso `RoundedFont` (era solo `FontFamily.SansSerif`, no una fuente redondeada real).
- `titleSmall`/`bodySmall` ausentes en `TerrariaTypography`, caían silenciosamente a defaults de Material3.
- Dark mode "Underworld" era solo una inversión fondo/texto del modo claro, no una paleta propia.

### Features nuevas
- Dark mode "Underworld" con paleta lava/ceniza/obsidiana real.
- Fuente pixel Silkscreen en títulos/headers, empaquetada offline (sin dependencia de red/Play Services).
- Look "slot de inventario" (borde sólido + esquinas moderadas) vía componente compartido `InventorySlotCard`.
- 4 acentos de bioma nuevos, aplicados ya a colores de categoría en Home.

### Documentación nueva
- [[03-Patrones-Kotlin/Rarity-Tier-Consolidation]] — nueva nota de patrón para el enum `RarityTier`.
- [[03-Patrones-Kotlin/Material-Theme-Tokens]] actualizado — paleta ampliada, Shapes/Spacing, riesgo "Underworld pospuesto" cerrado como resuelto.
- [[05-UI-Design-System]] reescrito — paleta completa, tipografía, nueva sección "Forma y espaciado", Material3 mapping actualizado, rareza apunta a la nota nueva.

### Hand-off
- Tests: **51/51** passing (suite previa + 4 `RarityTierTest` nuevos).
- Build: `./gradlew :app:assembleDebug` y `:app:testDebugUnitTest` en verde.
- **Pendiente:** validación visual en dispositivo físico — este entorno no tiene `adb` disponible, no se pudo instalar el APK ni recorrer las pantallas a ojo. Recomendado antes de dar el rediseño por cerrado.
- Pendiente (fuera de alcance, anotado): barrido completo de `.dp` literales al nuevo `Spacing` en archivos no tocados por este trabajo.
- Decisión de siguiente feature sigue abierta: NPCs, Room cache, o migración KMP (dark mode ya no está en esta lista, se resolvió aquí).

---

[[00-Index]]
