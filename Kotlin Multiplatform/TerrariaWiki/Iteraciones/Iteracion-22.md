# Iteración 22 — Auditoría arquitectura ViewModel/UseCase + sync README/CLAUDE.md (2026-07-27)

El usuario pidió una revisión completa de arquitectura: correr graphify, revisar dependencias, verificar si la arquitectura documentada en `CLAUDE.md` se cumple en la práctica, y evaluar si `README.md` había quedado desactualizado.

**Auditoría (graphify + grep dirigido, sin agente Explore):**
- **Hallazgo real — regla violada:** la convención "ViewModels dependen de UseCase, nunca directo de Repository" no se cumple para métodos reactivos/paginados. `CategoryViewModel`, `SearchViewModel` y `BossListViewModel` inyectan el Repository directo (`observeByCategory`, `hasMoreFor`, `refreshByCategory`, `loadMoreByCategory`, `observeBosses`, `refresh`, `searchAll`); `ItemDetailViewModel` es mixto (`GetItemByNameUseCase` + `ItemsRepository` directo para `getRecipes`). Los únicos casos donde la convención sí se cumple son métodos one-shot simples: `getItemByName`, `getBossByName`, `search` (vía `SearchItemsUseCase`, usado solo por `ItemsViewModel`, no por `SearchViewModel`).
- **Hallazgo secundario — código muerto:** `GetBossesUseCase` está creado y registrado en `bossesModule` (`factory { GetBossesUseCase(get()) }`) pero nunca se inyecta en ningún ViewModel — `BossListViewModel` reimplementa la misma lógica (`observeBosses()` + `refresh()`) directo contra el repository.
- **Compliant:** cero imports de `io.ktor`/`androidx.compose`/`android.*` en capas `domain/` (KMP-readiness intacta); cero imports cruzados entre `features/items`, `features/bosses`, `features/events` (aislamiento por feature respetado).
- **Docs desactualizadas:** `README.md` solo documentaba la feature Items (MVP), con roadmap obsoleto ("NPCs, Enemigos, Bosses, Biomas pendientes" cuando Bosses/Events ya estaban implementados desde iteración 17) y conteo de tests incorrecto (decía 15, son 12 clases de test reales). El árbol de paquetes de `CLAUDE.md` no mencionaba `core/ui/components/` (consolidado en iteración 21).

**Decisión de alcance:** el usuario decidió **solo reportar** los hallazgos de arquitectura — no refactorizar los ViewModels ni tocar `GetBossesUseCase` en esta iteración. Queda como deuda técnica conocida para una futura pasada, no bloquea nada del roadmap actual.

| Cambio | Archivo(s) | Descripción |
|---|---|---|
| docs | `README.md` | Estado actualizado (Items+Bosses+Events, dark mode, crafteo), árbol de paquetes expandido (bosses/, events/, core/ui/components/), conteo real de tests (12 clases agrupadas por feature), roadmap corregido |
| docs | `CLAUDE.md` | Árbol de paquetes: `core/ui/theme/` y `core/ui/components/` separados explícitamente |

### Hand-off
- Sin cambios de código — auditoría pura + sync de documentación.
- Build/tests: no se re-corrieron (no hubo cambios de código que los afecten).
- Commits: `docs: sync README and CLAUDE.md with current features and package tree` — directo a `main` en `terrariawiki`.
- Deuda técnica abierta para próxima iteración: decidir si (a) se envuelven los métodos reactivos/paginados en UseCase para cumplir la convención al 100%, o (b) se relaja la regla documentada en `CLAUDE.md` para permitir Repository directo en ViewModels cuando el método es puramente reactivo (StateFlow/paginación) — y en cualquier caso, eliminar o usar `GetBossesUseCase`.
- Roadmap general sigue abierto entre NPCs, Enemigos, cache Room, o migración KMP real.

---

[[00-Index]]
