---
plantilla: patrón
secciones: [Contexto, Decisión, Implementación Kotlin, Alternativas descartadas, Riesgos & mitigación]
---

# Catálogo estático en domain — cuando no hay API que lo respalde

## Contexto
Al añadir la categoría **Events** (Blood Moon, Goblin Army, Frost Moon...) se comprobó contra la API real de terraria.wiki.gg que **Cargo no tiene tabla `Events`** — solo existen 12 tablas (`Drops, Equipinfo, Exclusive, History, Imageinfo, Items, Modifiers, NPCs, Recipes, Weapon_source, _fileData, _pageData`), confirmado con `action=cargotables`. Los eventos solo existen como páginas normales de MediaWiki agrupadas en categorías (`Category:Random events`, `Category:Seasonal events`, `Category:Summoned events`).

Se probó extraer una imagen representativa por evento parseando el wikitext de cada página (`action=parse`, regex sobre `{{flavor text|...|image=}}` y sobre el primer `[[File:...]]`) y **no es fiable**: en "Goblin Army" el primer `[[File:...]]` de la página es un emote random, no el icono del evento. Se probó también la convención `"<PageName> Icon.png"`: existe para eventos "invocables" (Goblin Army, Frost Moon, Pirate Invasion, Martian Madness) pero NO para eventos climáticos (Blood Moon, Solar Eclipse, Sandstorm, Rain), que no tienen icono de item dedicado.

## Decisión
Cuando no hay tabla Cargo ni regla de extracción fiable, **no se fuerza el patrón `data/ + di/` completo**. En vez de eso: un `object` en `domain/` con una lista hardcodeada, cada filename de imagen **verificado a mano** contra la wiki real (`action=query&titles=File:...` comprobando que `missing` no aparece) antes de comitear. Terraria saca eventos muy raramente, así que el coste de mantenimiento de una lista estática es bajo — se documenta explícitamente en el propio código por qué es estática, para que no se intente "arreglar" hacia un fetch dinámico sin releer este contexto.

Consecuencia de diseño: sin repository/API, **no hay pantalla de detalle in-app** para eventos — el tap abre la página real de la wiki en el navegador (`Intent.ACTION_VIEW`) en vez de duplicar contenido que no se puede obtener con garantías.

## Implementación Kotlin

```kotlin
// features/events/domain/EventCatalog.kt
object EventCatalog {
    val all: List<Event> = listOf(
        Event("Blood Moon", "Bestiary Blood Moon.png", EventCategory.RANDOM, "Blood Moon"),
        Event("Goblin Army", "Goblin Army Icon.png", EventCategory.SUMMONED, "Goblin Army"),
        // ... resto verificado uno a uno contra terraria.wiki.gg
    )
}
```

```kotlin
// MainActivity.kt — navegación sin pantalla de detalle propia
composable("events") {
    val context = LocalContext.current
    EventListScreen(
        onEventClick = { event ->
            val url = "https://terraria.wiki.gg/wiki/${Uri.encode(event.wikiPageTitle)}"
            context.startActivity(Intent(Intent.ACTION_VIEW, Uri.parse(url)))
        },
        onBack = { navController.popBackStack() }
    )
}
```

Contrasta con **Bosses**, que sí sigue el patrón completo `data/domain/di/ui` porque la tabla Cargo `NPCs` con `type HOLDS 'boss'` sí existe y trae datos reales (vida, defensa, daño) — ver [[Cargo-HOLDS-filter]].

## Alternativas descartadas
- **Parsear wikitext por evento (`action=parse` + regex de imagen):** descartado, verificado no-fiable (ver Contexto).
- **Repository con `Result<List<Event>>` envolviendo la lista estática:** over-engineering — no hay error posible que capturar (no hay red), `runCatching` no aporta nada sobre una constante en memoria.
- **Detalle in-app con solo nombre+imagen (sin descripción):** descartado por el usuario; prefiere la página real de la wiki (contenido completo) antes que una ficha vacía.

## Riesgos & mitigación
- **Riesgo:** un filename de imagen queda obsoleto si la wiki renombra el archivo. **Mitigación:** cada filename se verificó contra la API real en el momento de escribir la lista (2026-07-26); si Coil muestra el icono de error para algún evento, es un ajuste de una línea en `EventCatalog.kt`.
- **Riesgo:** Terraria añade un evento nuevo y la lista queda incompleta. **Mitigación:** aceptado — bajo coste de mantenimiento, se añade a mano cuando se detecte (o cuando el usuario lo pida).
- **Riesgo:** que este patrón se use como excusa para evitar `data/` en features que sí tienen API real. **Mitigación:** el patrón solo aplica cuando se ha *verificado* con `cargotables`/`cargofields` que no existe fuente estructurada — no es un atajo por pereza.
