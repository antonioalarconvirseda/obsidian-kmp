# Iteración 18 — Audit arquitectura + fix Dependency Inversion en Repository (2026-07-26)

El usuario pidió una auditoría del proyecto contra sus propias directrices (`CLAUDE.md`), la escalabilidad hacia una migración KMP real, y si el proyecto seguía la arquitectura correcta. En paralelo, decisión sobre si integrar esta bóveda Obsidian dentro del repo de código o mantenerla separada.

**Auditoría (agente Explore, contraste literal contra `CLAUDE.md`):**
- Compliant: DI Koin (`single`/`factory`/`viewModel`), `Result`/`runCatching` + `UiState` sealed, testing (MockK/Turbine, cero Mockito/`Thread.sleep`), anclajes Android confinados a `HttpClientFactory.kt`/`CoilImageLoaderFactory.kt` sin fugas a `domain`/`data`, módulo único `:app` sin `commonMain` (como documentado).
- **Hallazgo real:** `domain` importaba la interfaz `Repository` desde `data` (`GetItemsUseCase.kt`, `GetBossesUseCase.kt`, etc.) — contradecía la regla literal "`domain` nunca importa de `data`". No rompía portabilidad KMP (la interfaz es Kotlin puro) pero invertía la Dependency Inversion clásica: el *port* debe vivir en `domain`, `data` lo implementa.
- **Hallazgo secundario:** roadmap y árbol de paquetes de `CLAUDE.md` desactualizados — no mencionaban `features/bosses`/`features/events`, ya implementados desde la iteración 17.

**Decisión vault Obsidian:** mantenerla como repo separado (`obsidian-kmp`, remote propio). Razones: ya es fuente de verdad externa referenciada por path absoluto desde `CLAUDE.md` (convención establecida y funcional); Obsidian necesita su propia estructura (`.obsidian/`, graph, backlinks) que no pertenece a un repo de código Android; historiales de commits con propósitos distintos (iteraciones de doc vs. commits de código) no deben mezclarse; un futuro split KMP (`:shared`+`:android-app`) no afecta a un repo de docs ya independiente.

| Cambio | Archivo(s) | Descripción |
|---|---|---|
| refactor | `features/items/domain/ItemsRepository.kt` (nuevo), `features/items/data/ItemsRepositoryImpl.kt` (renombrado desde `ItemsRepository.kt`) | Interfaz movida a `domain/`, Impl se queda en `data/` |
| refactor | `features/bosses/domain/BossesRepository.kt` (nuevo), `features/bosses/data/BossesRepositoryImpl.kt` (renombrado) | Mismo fix, mirror de items |
| refactor | UseCases, ViewModels, `ItemsModule.kt`, `BossesModule.kt`, tests | Imports actualizados a `...domain.ItemsRepository`/`...domain.BossesRepository` |
| docs | `CLAUDE.md` | Árbol de paquetes con `features/bosses`/`features/events`, convención Repository corregida ("interfaz en `domain/`, Impl en `data/`"), roadmap sin orden fundacional obsoleto |

### Hand-off
- Tests: **67/67** passing (sin tests nuevos — refactor puro de ubicación de interfaz, mismo comportamiento).
- Build: `./gradlew :app:testDebugUnitTest` en verde tras el refactor.
- Commits: `refactor: move Repository interfaces from data to domain layer`, `docs: sync CLAUDE.md with actual package tree and roadmap state` — directo a `main`.
- Roadmap sigue abierto entre NPCs no-boss, Enemigos, cache Room, o migración KMP real; con el fix de este iteración, el contrato `Repository` ya queda 100% listo para `commonMain` sin retrabajo adicional cuando se acometa la migración.

---

[[00-Index]]
