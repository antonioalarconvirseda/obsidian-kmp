# Decisiones de Arquitectura — TerrariaWiki

Las 7 decisiones de fondo del proyecto, con su porqué y alternativas descartadas.

---

## 1. Fuente de datos: terraria.wiki.gg oficial (MediaWiki + Cargo)

**Decisión:** Consumir la MediaWiki API de `https://terraria.wiki.gg` con la extensión `cargoquery` para datos estructurados de items/NPCs/enemigos.

**Por qué:**
- Es la wiki **oficial** de la comunidad (reemplazó a Fandom en 2023).
- La extensión **Cargo** expone datos tabulares en JSON, listos para mapear a data classes Kotlin sin scraping.
- Permite filtros, paginación y búsqueda en una sola API.

**Alternativas descartadas:**
- **Fandom (terraria.fandom.com):** API propia menos estructurada; ya no es la fuente principal.
- **Scraping HTML de la wiki:** frágil a cambios de markup, no escala, problemático legalmente.

---

## 2. Arquitectura: MVVM + capas (data / domain / ui)

**Decisión:** Patrón MVVM clásico, dividido en tres capas:
- `data/`: Ktor API, DTOs, Mappers, Repositories
- `domain/`: Modelos de dominio puros, UseCases
- `ui/`: Compose Screens, ViewModels, Navigation

**Por qué:**
- Encaja con Compose + StateFlow idiomáticamente.
- Permite que la capa `domain` no conozca ni Ktor ni Compose → reutilizable cuando migremos a KMP.
- Estándar en Android moderno; fácil de razonar para un aprendiz.

**Alternativas descartadas:**
- **MVI (Model-View-Intent):** más estructurado pero overhead para un MVP de aprendizaje.
- **MVP (Model-View-Presenter):** legacy, no encaja con Compose.
- **Clean Architecture estricta con use cases para todo:** sobredimensionado para una app de lectura.

---

## 3. Stack KMP-ready: Ktor + Koin + kotlinx.serialization + Coil + Navigation Compose

**Decisión:** Librerías 100% KMP-compatibles o KMP-friendly.

| Librería | Rol | Por qué KMP-ready |
|---|---|---|
| Ktor 2.3.x | HTTP client | Diseñado para KMP, sin dependencias Android |
| Koin 3.5.x | DI | Kotlin puro, sin KAPT/KSP |
| kotlinx.serialization 1.7.x | JSON | Plugin oficial de JetBrains, KMP nativo |
| Coil 2.6.x | Carga de imágenes | Compose-first, multiplatform |
| Navigation Compose 2.7.x | Navegación | Declarativa, parte de androidx.compose |
| Coroutines + Flow 1.8.x | Async | Estándar Kotlin, KMP nativo |

**Por qué:** cuando se haga la migración a KMP para añadir iOS o Desktop, la capa `data`, `domain` y la DI se mueven tal cual a `commonMain`. Solo `ui` queda por separado por target.

**Alternativas descartadas:**
- **Hilt + Retrofit + OkHttp + LiveData:** anclan el proyecto a Android (KAPT/KSP, `androidx.lifecycle.LiveData`). Migrar a KMP requeriría reescribir medio proyecto.

### Matiz KMP-readiness (auditoría post-MVP, iteración 2)

La elección de librerías (Ktor, Koin, kotlinx.serialization, Coroutines) es **KMP-ready**. Pero la implementación actual tiene 3 anclajes Android concretos que requerirán refactor en una migración real:

1. `HttpClient(OkHttp)` y `android.util.Log` en `core/network/HttpClientFactory.kt` → split con `expect/actual` (OkHttp en `androidMain`, `Darwin` engine en `iosMain`, Napier/Kermit para log).
2. `koin-android` + `koin-androidx-compose` en los módulos DI → migrar a `koin-core` en `commonMain` + binding Android en `androidMain`.
3. Paquete único `app/` sin split Gradle → crear módulo `:shared` (commonMain) + `:android-app` delgado.

El **dominio** (`Item`, UseCases) y la **capa de datos pura** (DTO, Mapper, Repository interface) sí son portables directamente a `commonMain` sin cambios. La migración real será una iteración propia, no un parche.

**Nota (iteración 18, 2026-07-26):** un audit de arquitectura encontró que la interfaz `Repository` vivía en `data/` en vez de `domain/`, contradiciendo esta misma afirmación. Corregido — ver [[03-Patrones-Kotlin/Repository-Pattern]].

---

## 4. StateFlow + UiState sealed (en lugar de LiveData + state suelto)

**Decisión:** Cada ViewModel expone un único `StateFlow<UiState>` donde `UiState` es una `sealed interface` con variantes `Loading`, `Ready`, `Error`, `Empty`.

**Por qué:**
- `StateFlow` es Kotlin nativo → portable a KMP.
- `sealed` permite `when` exhaustivo en Compose: imposible olvidar un estado.
- Reemplaza al antiguo `LiveData` (que depende de `androidx.lifecycle` y no es KMP-ready).

---

## 5. Estructura por feature, no por capa

**Decisión:**
```
features/
├── items/    (data + domain + di + ui, todo junto)
```
en vez de:
```
data/  domain/  ui/   (todas las features mezcladas)
```

**Por qué:**
- Encapsulación: lo de items vive junto, fácil de borrar/mover.
- Migración a KMP-friendly: cada feature puede convertirse en módulo Gradle.
- Mejor navegación cognitiva del proyecto.

---

## 6. MVP: Items + Búsqueda + Detalle

**Decisión:** La primera versión funcional muestra **solo items** con búsqueda y ficha de detalle.

**Por qué:**
- La wiki de Terraria es enorme (items, NPCs, enemigos, bosses, biomas, crafting, logros...). Intentar todo en MVP sería un proyecto de meses.
- **Items** es la sección más consultada y la que mejor ejercita todos los patrones (lista, búsqueda, detalle, imagen, stats, error/empty/loading).
- Los patrones que se introducen (Repository, UseCase, ViewModel, navegación, imagen) **se reusan** para NPCs/Enemigos en siguientes iteraciones.

**Crecimiento planeado:**
1. Items (MVP actual)
2. NPCs
3. Enemigos
4. Bosses
5. Biomas
6. Crafting/Recetas (si la API lo permite con `cargoquery` joins)

---

## 7. Repositorio público en GitHub personal

**Decisión:** El proyecto se publica en `github.com/antonioalarconvirseda/terrariawiki` como **público**, licencia MIT.

**Por qué (decisión del usuario):**
- Portfolio público como parte del aprendizaje.
- Trazabilidad del progreso.
- Licencia MIT permisiva para reutilización.

**Notas:**
- No se sube ningún secreto (no hay claves API en este proyecto porque la MediaWiki API es pública).
- Repo público **no** es afiliación con Re-Logic; se aclara en el README.
