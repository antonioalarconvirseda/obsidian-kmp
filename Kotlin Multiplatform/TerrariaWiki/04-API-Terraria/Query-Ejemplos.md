# API de Terraria — Query Examples

Ejemplos reales verificados con `curl` el 2026-07-25.

---

## 1. Listar 3 items (nombre + tipo + rareza)

```bash
curl "https://terraria.wiki.gg/api.php?action=cargoquery&tables=Items&fields=name,type,rare&limit=3&format=json"
```

**Respuesta real:**
```json
{
  "cargoquery": [
    { "title": { "name": "'0' Statue" } },
    { "title": { "name": "'1' Statue" } },
    { "title": { "name": "'2' Statue" } }
  ]
}
```

Nota: cuando NO se incluyen `rare` o `type`, esos campos no aparecen en `title`. Hay que pedirlos explícitamente.

## 2. Filtrar un item por nombre

```bash
curl "https://terraria.wiki.gg/api.php?action=cargoquery&tables=Items&fields=name,type,rare&where=name='Wood'&format=json"
```

**Respuesta real:**
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

✅ **Confirmado funcional.** El delimitador de listas es `^`.

## 3. Listar campos disponibles en una tabla

```bash
curl "https://terraria.wiki.gg/api.php?action=cargofields&table=Items&format=json"
```

**Campos devueltos (los 46 confirmados):**
```
autoswing, axe, bait, bodyslot, bonus, buffs, buy, consumable, critical,
damage, damagetype, debuffs, defense, fishing, hammer, hardmode, hheal,
image, imageequipped, imagefile, imageplaced, internalname, itemid,
knockback, listcat, mheal, mana, name, pick, placeable, placedheight,
placedwidth, rare, research, sell, stack, tag, tooltip, toolspeed, type,
unobtainable, usetime, velocity
```

## 4. Búsqueda full-text

```bash
curl "https://terraria.wiki.gg/api.php?action=query&list=search&srsearch=Slime&format=json&srlimit=2"
```

**Respuesta (resumida):**
```json
{
  "query": {
    "searchinfo": { "totalhits": 837 },
    "search": [
      { "title": "Slimes", "pageid": 9609, "snippet": "..." },
      { "title": "Banners (enemy)", "pageid": 14563, "snippet": "..." }
    ]
  }
}
```

## 5. Detalle de página por nombre

```bash
curl "https://terraria.wiki.gg/api.php?action=parse&page=Wood&format=json&prop=text|wikitext"
```

Devuelve `parse.text["*"]` (HTML) y `parse.wikitext["*"]` (fuente). Útil para descripciones largas, infoboxes, recetas inline.

## 6. URL de imagen (Special:Redirect)

```bash
curl -I "https://terraria.wiki.gg/wiki/Special:Redirect/file/Wood.png"
```

Devuelve **HTTP 301** a la URL estática (verificado). Importante: el filename debe ir **URL-encoded con `%20`** para espacios, NO con `+` (que da 404).

```bash
# Espacios como %20 → OK
curl -I "https://terraria.wiki.gg/wiki/Special:Redirect/file/Terra%20Blade.png"
# → HTTP/2 301 location: https://terraria.wiki.gg/wiki/Special:Redirect/file/Terra_Blade.png

# Espacios como + → 404
curl -I "https://terraria.wiki.gg/wiki/Special:Redirect/file/Terra+Blade.png"
# → HTTP/2 404
```

En Kotlin usar `android.net.Uri.encode(filename)` (Android-only) o `URLEncoder.encode(filename, "UTF-8").replace("+", "%20")` (KMP-puro).

## 7. Item con todos los campos (iteración 3)

```bash
curl "https://terraria.wiki.gg/api.php?action=cargoquery&tables=Items&fields=name,type,rare,sell,damage,defense,knockback,usetime,tooltip,internalname,itemid,listcat,imagefile,stack,hardmode,buy,autoswing,critical,velocity&where=name='Terra%20Blade'&format=json"
```

**Respuesta (campos clave):**
```json
{
  "cargoquery": [{
    "title": {
      "name": "Terra Blade",
      "type": "weapon^crafting material",
      "rare": "8",
      "sell": "<span class=\"coin\" title=\"20 Gold Coins\" data-sort-value=\"200000\">...</span>",
      "damage": "85",
      "defense": "",
      "knockback": "6.5",
      "usetime": "18",
      "imagefile": "Terra Blade.png",
      "listcat": "broadswords^projectile melee^Melee weapons^craftable items",
      "stack": "9999",
      "hardmode": "1",
      "autoswing": "1",
      "critical": "4",
      "velocity": "12"
    }
  }]
}
```

## 9. Listar bosses (tabla `NPCs`, iteración 17)

```bash
curl "https://terraria.wiki.gg/api.php?action=cargoquery&tables=NPCs&fields=nameraw,type,image,life,defense,damage,knockback&where=type%20HOLDS%20%22Boss%22&limit=5&format=json"
```

**Respuesta real (recortada, un boss):**
```json
{
  "cargoquery": [{
    "title": {
      "nameraw": "Betsy",
      "type": "boss",
      "image": "<span class=\"npcimg\">[[File:Animated Betsy.gif|link=]]</span>",
      "life": "<span class=\"npcstat\"><span class=\"m-normal m-journey\">50000</span><span class=\"ssep\">/</span><span class=\"m-expert mode-exclusive expert\"><span class=\"s\">75000</span></span>...</span>",
      "defense": "<span class=\"npcstat\"><span class=\"m-all\">38</span></span>",
      "damage": "...(bloque HTML largo, múltiples modos + notas de contacto/proyectil)...",
      "knockback": "<span class=\"npcstat\"><span class=\"m-all\">100%</span></span>"
    }
  }]
}
```

A diferencia de `Items`, aquí **no existe un campo plano equivalente a `imagefile`** para los stats — hay que limpiar HTML *y* wikitext `[[...]]` del campo completo. Ver [[Cargo-envelopes]] y [[../03-Patrones-Kotlin/Cargo-HOLDS-filter]].

## 10. Comprobar tablas Cargo existentes (iteración 17)

```bash
curl "https://terraria.wiki.gg/api.php?action=cargotables&format=json"
```

```json
{"cargotables":["Drops","Equipinfo","Exclusive","History","Imageinfo","Items","Modifiers","NPCs","Recipes","Weapon_source","_fileData","_pageData"]}
```

Usado para confirmar (antes de implementar Events) que **no existe tabla `Events`** — decisión documentada en [[../03-Patrones-Kotlin/Static-Domain-Catalog]].

## 11. Limitaciones conocidas (actualizado iteración 3)

- `imagefile` **SÍ funciona** (corregido en iteración 3). El `internal_api_error_MWException` que documentábamos antes no se reproduce con la lista actual de campos. Usar `imagefile` directamente.
- El campo `rare` viene como **string** (`"0"`, `"1"`, …), no como número. Hay que parsearlo manualmente.
- Los campos de tipo List vienen separados por `^` (no `,`). Documentado en [[Endpoint-Lista]].
- Sin `User-Agent`, MediaWiki puede responder 403 o throttlear. Siempre enviar uno descriptivo.
- `sell` y `buy`: parsear el `title="(\d+) (\w+) Coins"` para mostrar el valor legible. NO usar `data-sort-value` (es valor en centavos).
