# API de Terraria — Endpoints

> Fuente: [terraria.wiki.gg](https://terraria.wiki.gg) — MediaWiki con extensión **Cargo**.

## Endpoint base

```
https://terraria.wiki.gg/api.php
```

Todos los endpoints usan **GET** con `format=json` y devuelven un JSON con envelope estándar de MediaWiki.

---

## Endpoints confirmados (probe 2026-07-25)

### 1. `cargoquery` — datos tabulares estructurados

Devuelve filas de una "tabla Cargo" como objetos JSON.

```
GET /api.php
    ?action=cargoquery
    &tables=Items
    &fields=name,type,rare,sell
    &where=name='Wood'
    &limit=50
    &offset=0
    &format=json
```

**Parámetros:**
- `tables` — nombre de la tabla Cargo (e.g. `Items`, `NPCs`, `Enemies`).
- `fields` — lista separada por comas de campos a devolver.
- `where` (opcional) — cláusula SQL `WHERE` con sintaxis Cargo. Strings con comillas simples: `name='Wood'`.
- `limit` (opcional, default 50, max 500) — número de filas.
- `offset` (opcional) — para paginación.
- `order_by` (opcional) — campo de ordenación.

**Respuesta:**
```json
{
  "cargoquery": [
    {
      "title": {
        "name": "Wood",
        "type": "block^crafting material",
        "rare": "0"
      }
    }
  ]
}
```

### 2. `query&list=search` — búsqueda full-text

```
GET /api.php
    ?action=query
    &list=search
    &srsearch=Slime
    &srlimit=10
    &format=json
```

**Respuesta:**
```json
{
  "query": {
    "search": [
      { "title": "Slimes", "pageid": 9609, "snippet": "..." }
    ],
    "searchinfo": { "totalhits": 837 }
  }
}
```

### 3. `parse` — contenido detallado de una página (HTML + Wikitext)

```
GET /api.php
    ?action=parse
    &page=Wood
    &prop=text|wikitext
    &format=json
```

Devuelve HTML renderizado y Wikitext fuente. Útil para páginas complejas (recetas, biomas con infobox).

### 4. `cargofields` — metadatos de una tabla

```
GET /api.php
    ?action=cargofields
    &table=Items
    &format=json
```

Devuelve los nombres y tipos de cada campo. Útil para introspección.

### 5. `query&prop=imageinfo` — URL real de un archivo de imagen

```
GET /api.php
    ?action=query
    &titles=File:Wood.png
    &prop=imageinfo
    &iiprop=url
    &format=json
```

**Respuesta:**
```json
{
  "query": {
    "pages": {
      "-1": {
        "imageinfo": [{ "url": "https://static.wikia.com/.../Wood.png" }]
      }
    }
  }
}
```

Útil cuando se necesita la URL directa de una imagen sin parsear wikitext.

### 6. Special:Redirect — imagen sin API call

```
https://terraria.wiki.gg/wiki/Special:Redirect/file/Wood.png
```

Redirige (HTTP 301) a la URL estática del archivo. Permite usar directamente como `model` en Coil sin parsear.

**Importante — URL-encoding del filename (iteración 3):**
- Si el filename tiene espacios (la mayoría: `Terra Blade.png`, `Wood Plank.png`, `'0' Statue.png`...), la URL construida con espacio literal falla sin respuesta HTTP.
- `URLEncoder.encode(filename, "UTF-8")` codifica espacios como `+` → 404 en MediaWiki. **No usar.**
- `android.net.Uri.encode(filename)` codifica espacios como `%20` → 301 OK a la URL final. **Usar este.**
- En KMP, la alternativa KMP-pura es `URLEncoder.encode(...).replace("+", "%20")` (kotlin stdlib).

Verificado con curl real:
- `Special:Redirect/file/Terra Blade.png` (espacio literal) → sin respuesta (falloe).
- `Special:Redirect/file/Terra%20Blade.png` → 301 → URL final.
- `Special:Redirect/file/Terra+Blade.png` → 404.

---

## Tablas Cargo disponibles

**Lista completa confirmada (iteración 17, `action=cargotables`) — solo 12 tablas existen en total:**
```
Drops, Equipinfo, Exclusive, History, Imageinfo, Items, Modifiers,
NPCs, Recipes, Weapon_source, _fileData, _pageData
```
No hay tabla `Events`, `Enemies` ni `Biomes` — si una feature futura las necesita, la data NO viene de Cargo (ver [[../03-Patrones-Kotlin/Static-Domain-Catalog]] para el patrón alternativo usado con Events).

- `Items` — funciona correctamente, incluido el campo `imagefile` (no descartar).

**Campos valiosos disponibles en `Items` (incluidos en iteración 3):**
- `imagefile` — filename directo, sin regex.
- `critical` — probabilidad de crítico (%).
- `velocity` — velocidad de proyectil.
- `autoswing` — booleano (`"1"`).
- `buy` — precio de compra (mismo formato HTML que `sell`).
- `stack` — cantidad apilable máxima.
- `hardmode` — booleano (`"1"` si solo aparece en modo difícil).
- `listcat` — categorías de gameplay separadas por `^` (`broadswords^Melee weapons^craftable items`).

**`NPCs` (confirmada en iteración 17, usada para la feature Bosses):**
- Campos: `name, nameraw, image, type, environment, ai, damage, life, defense, knockback, banner, bannername, money, npcid, immunities`.
- `type HOLDS 'boss'` filtra solo bosses (mismo operador `HOLDS` que en `Items`, ver [[../03-Patrones-Kotlin/Cargo-HOLDS-filter]]).
- `life`, `defense`, `damage`, `knockback` vienen como **HTML con múltiples modos** (`<span class="m-normal">50000</span><span class="ssep">/</span><span class="m-expert">75000</span>...`), no como números planos — hay que limpiar tags Y wikitext `[[...]]` (`[[Category:...]]`, `[[Expert Mode|140]]`), no solo HTML. Regex adicional a la de `sell`/`buy` de Items.
- `nameraw` es el nombre en texto plano (sin el markup `[[Betsy|Betsy]]<span class="eico"...` de `name`) — usar `nameraw`, no `name`, para mostrar/filtrar.
- **Cuidado:** `nameraw` no es único — "Dark Mage" aparece dos veces (tiers T1/T3, tipo `boss` ambos). Ver [[../03-Patrones-Kotlin/Compose-LazyColumn]] riesgo de key duplicada.

**Por explorar (fases futuras):**
- `Enemies` normales (no boss) — dentro de la misma tabla `NPCs`, filtrando `type` distinto de `boss`.
- `Recipes` (joins) — ya en uso parcial vía `RecipesApiImpl`.
- `Drops`, `Modifiers`, `Weapon_source` — sin explorar aún.

Para listar todas las tablas disponibles:
```
GET /api.php?action=cargotables&format=json
```

---

## User-Agent y rate-limit

- **Obligatorio** enviar un `User-Agent` descriptivo. MediaWiki API Guidelines lo requieren.
- Sin autenticación: ~500 requests/periodo por IP (suficiente para MVP).
- Para uso más intensivo: registrar `OAuth` consumer (fuera de alcance del MVP).

User-Agent elegido:
```
TerrariaWikiApp/1.0 (https://github.com/antonioalarconvirseda/terrariawiki; contacto: antonioalarconvirseda@hotmail.com)
```

## Delimitador de listas (¡importante!)

Los campos de tipo "List" en Cargo vienen como string con campos separados por `^`:
```json
"type": "block^crafting material"
```
Para mapear:
```kotlin
"block^crafting material".split("^")
// → ["block", "crafting material"]
```

## Formato del precio de venta (`sell` y `buy`)

`sell` y `buy` vienen como HTML de MediaWiki con un `data-sort-value` (en centavos) y un `title` con la cantidad legible:

```html
<span class="coin" title="20 Gold Coins" data-sort-value="200000">
  <span class="gc">20<i> GC</i></span>
</span>
```

**Para mostrar en la UI: parsear el `title`, NO el `data-sort-value`.** La regex correcta es:
```kotlin
Regex("""title="(\d+)\s+(\w+)\s+Coins"""")
// match.groupValues[1] = "20" (cantidad)
// match.groupValues[2] = "Gold" → abreviar a "GC"
```

Si parseas `data-sort-value="200000"` directamente y le pones "GC", obtienes "200000 GC" (incorrecto). Si parseas el `title`, obtienes "20 GC" (correcto).

Mapeo de moneda: Copper→CC, Silver→SC, Gold→GC, Platinum→PC.
