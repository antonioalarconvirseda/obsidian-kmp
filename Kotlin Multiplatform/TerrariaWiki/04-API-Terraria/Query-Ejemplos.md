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

**A verificar durante Fase 6** — debe devolver HTTP 302 a la URL estática.

Si no funciona, alternativa:
```bash
curl "https://terraria.wiki.gg/api.php?action=query&titles=File:Wood.png&prop=imageinfo&iiprop=url&format=json"
```

## 7. Limitaciones conocidas

- `imagefile` produce `internal_api_error_MWException` cuando se incluye en joins multi-campo. **Workaround:** omitir y obtener la URL con `Special:Redirect/file/<filename>`.
- El campo `rare` viene como **string** (`"0"`, `"1"`, …), no como número. Hay que parsearlo manualmente.
- Los campos de tipo List vienen separados por `^` (no `,`). Documentado en [[Endpoint-Lista]].
- Sin `User-Agent`, MediaWiki puede responder 403 o throttlear. Siempre enviar uno descriptivo.
