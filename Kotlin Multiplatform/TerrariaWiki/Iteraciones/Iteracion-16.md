# Iteración 16 — Dark mode "Cielo Nocturno" (reemplaza Underworld) + hierba más fiel (2026-07-26)

Tras probar Iteración 15, el usuario reportó dos puntos: la franja de hierba se veía "cutre" (quería algo más fiel a la textura real del juego/la wiki), y el dark mode seguía predominando en naranja/marrón (paleta "Underworld" de Iteración 12) — pidió explícitamente que se eligieran colores agradables, delegando la decisión. Se confirmó por pregunta el pivote de identidad del dark mode antes de implementar.

| Cambio | Archivo(s) | Descripción |
|---|---|---|
| revert | `Color.kt` | Se eliminan los tokens "Underworld" (`ObsidianBlack`, `AshSurface`, `AshSurfaceAlt`, `LavaOrange`, `EmberRed`, `HellstoneOrange`, `MagmaCrust`, `BrimstoneYellow`, `CinderWhite`) |
| feat | `Color.kt` | Nuevos tokens "Cielo Nocturno" (`NightBackground`, `NightSurface`, `NightSurfaceAlt`, `NightOutline`, `StarlightWhite`) + `DirtBrown`/`GrassHighlight` para la hierba |
| refactor | `Theme.kt` | `TerrariaDarkColors` reconstruido: `primary`/`secondary`/`tertiary`/`error` ahora son los MISMOS vals que el modo claro (`SkyTeal`/`JungleGreen`/`GoldGem`/`SlimeRed`), solo los neutros de fondo/superficie cambian |
| refactor | `InventorySlotCard.kt` | `GrassTopAccent` rediseñado: 24 columnas finas con altura de verde variable pero fija (`GRASS_COLUMN_HEIGHTS`, reproducible) sobre base marrón, más resaltes en columnas puntuales — reemplaza el zigzag simétrico anterior |

### Documentación actualizada
- [[05-UI-Design-System]] — paleta dark reescrita ("Cielo Nocturno"), `GrassTopAccent` actualizado.
- [[03-Patrones-Kotlin/Material-Theme-Tokens]] — riesgo de Underworld marcado como revertido, snippet actualizado.

### Hand-off
- Build/tests: sin cambios de lógica de negocio, 51/51 se mantiene.
- **Pendiente:** validación visual en dispositivo — este entorno sigue sin `adb`.
- Lección repetida: el dark mode "Underworld" ya había pasado por una ronda de diseño propio (Iteración 12) antes de que el usuario lo viera en pantalla real y lo rechazara — ratifica la lección de Iteración 14 sobre validar con referencias/capturas antes de invertir en una identidad visual completa.

---

[[00-Index]]
