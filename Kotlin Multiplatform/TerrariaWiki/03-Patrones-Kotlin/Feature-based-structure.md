# Estructura por feature (vs. por capa)

## Contexto
Cuando un proyecto Android crece, hay dos formas de organizar los paquetes:
- **Por capa:** todos los `data/` juntos, todos los `domain/` juntos, todos los `ui/` juntos.
- **Por feature:** cada feature tiene su propio `data/`, `domain/`, `ui/`, `di/`.

En TerrariaWiki tenemos planeadas múltiples features (Items, NPCs, Enemigos, Bosses, …) y un objetivo a medio plazo de **migrar a Kotlin Multiplatform**. La pregunta era: ¿cómo organizamos los paquetes para que ambas cosas escalen bien?

## Decisión
**Paquetes por feature.** Cada feature agrupa todo lo que necesita:

```
features/items/
├── data/      (ItemsApi, ItemsDto, ItemsMapper, ItemsRepository)
├── domain/    (Item model, GetItemsUseCase, SearchItemsUseCase)
├── di/        (Koin module de la feature)
└── ui/        (ItemsScreen, ItemDetailScreen, ViewModels, navigation)
```

Existe además una carpeta `core/` transversal con cosas compartidas (network, theme, DI raíz, util).

## Implementación Kotlin
Ver `app/src/main/java/com/terrariawiki/` en el proyecto. La estructura actual:

```
com.terrariawiki/
├── TerrariaWikiApp.kt        Application + arranque Koin
├── MainActivity.kt           setContent { AppRoot() }
├── core/
│   ├── network/              HttpClientFactory (Ktor)
│   ├── di/                   networkModule
│   ├── ui/theme/             Color.kt, Theme.kt, Type.kt
│   └── util/                 helpers
└── features/items/
    ├── data/                 ItemsApi, ItemsDto, ItemsRepository
    ├── domain/               Item, GetItemsUseCase
    ├── di/                   itemsModule
    └── ui/                   ItemsScreen, ItemsViewModel, navigation
```

## Alternativas descartadas
- **Por capa (data/domain/ui):** cuando un proyecto crece, los ficheros relacionados con la misma feature quedan dispersos. Para KMP, además, no se puede convertir fácilmente cada feature en un módulo Gradle.
- **Por feature pero con `core/` mezclado dentro de cada feature:** tentación de duplicar `network/` o `theme/`. Lo evitamos centralizando en `core/`.

## Riesgos & mitigación
- **Riesgo:** un `common` que crece demasiado y se vuelve god-package. **Mitigación:** revisar periódicamente que `core/` solo tenga transversal real (network, theme, util, DI raíz).
- **Riesgo:** al migrar a KMP, las dependencias entre features (si las hubiera) se complican. **Mitigación:** mantener las features independientes, sin que `items` importe nada de `npcs` por ejemplo.
