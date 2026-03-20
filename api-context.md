# andrewzc API — Railfan Write Reference

This file covers the endpoints and patterns Railfan uses to make changes to the database. For read operations (finding entities, looking things up), see the shared patterns below.

Base URL: `https://api.andrewzc.net`

---

## Finding entities before writing

Always resolve the entity before writing to it.

```
GET /entities?name=<query>[&list=<list>]
```

Returns name-matched entities with their `list`, `key`, `name`, and `been`. Use this to confirm which entity Andrew means before making changes.

If Andrew mentions a list explicitly, pass `&list=<list>` to narrow results. If not, search across all lists and use context to pick the right one.

```
# Find "Flushing Avenue" across all lists
GET /entities?name=Flushing+Avenue

# Find "Taj Mahal" in the unesco list
GET /entities?name=Taj+Mahal&list=unesco
```

---

## Updating an entity

```
PUT /entities/:list/:key
Content-Type: application/json
{ ...fields to update... }
```

Only the fields you send are updated. Other fields are untouched.

### Mark as been
```json
{ "been": true }
```

### Mark section progress (transit)
```json
{ "section": "done" }
```
Valid values: `done`, `taken`, `visited`, `want`

### Mark as closed / defunct
```json
{ "strike": true }
```

### Set a prop on the entity
```json
{ "props": { "curved": true } }
```
Note: sending `props` replaces the entire props object. Fetch the entity first if you need to merge.

### Set multiple fields at once
```json
{ "been": true, "section": "done" }
```

---

## Creating a new entity

```
POST /entities/:list
Content-Type: application/json
{
  "name": "Leaning Tower of Pisa",
  "link": "https://en.wikipedia.org/wiki/Leaning_Tower_of_Pisa"
}
```

The `name` field is required. Include `link` (Wikipedia URL) whenever you know it — it enables enrichment.

After creating, always enrich:

```
POST /entities/:list/:key/enrich
```

Enrichment cascades: Wikipedia → coordinates → city → reference (if the page has `reference` in tags). This fills in `coords`, `location`, `city`, `country`, `wikiSummary`, and `wikiEmbedding`.

Full create + enrich example:
```
POST /entities/towers
{ "name": "Leaning Tower of Pisa", "link": "https://en.wikipedia.org/wiki/Leaning_Tower_of_Pisa" }
→ returns { "key": "leaning-tower-of-pisa", ... }

POST /entities/towers/leaning-tower-of-pisa/enrich
→ fills in coords, city, country, wikiSummary
```

---

## Updating a page

```
PUT /pages/:key
Content-Type: application/json
{ ...fields to update... }
```

### Change page size
```json
{ "size": "medium" }
```
Valid values: `small`, `medium`, `large`

### Change page icon
```json
{ "icon": "☕️" }
```

---

## Creating a new page

```
POST /pages
Content-Type: application/json
{
  "name": "Cafés",
  "icon": "☕️",
  "type": "place",
  "size": "large"
}
```

Required fields: `name`, `icon`, `type`.
Valid types: `place`, `entity`, `city`, `country`, `state`.
Default size: `large`.

---

## The props system

Some pages in the database are `propertyOf` another list. These represent attributes of entities in the parent list, stored as props.

The four parent lists and their property pages:

**stations** — airport-metros, airport-trams, bumper, curved, deep-level, elevator-stations, infill, intermodal, least-used, mexico-city, metro-places, metro-people, metro-themes, moving-walkways, new-stations, no-exit, offset, parking, part-time, police, short-platform, short-turn, single-platform, sloped, spanish, split-platform, transfer, twin-stations

**metros** — metro-prices, metro-population, metro-widths

**countries** — citizenship, european-union, eurozone, founded, independence, kosovo, landlocked, lefthand, left-to-right, marriage, microstates, nato, no-trains, non-binary, palestine, portuguese, schengen, short-coastline, similar-flags, tariffs, territories, unification, western-sahara

**cities** — china-cities, culture, eurovision, money-heist, olympics, tintin, world-cities

### When Andrew says "X is [property]"

1. Check if `[property]` matches a page key in the list above
2. Identify the parent list (e.g. `curved` → parent is `stations`)
3. Find the entity in the parent list (e.g. search for "Flushing Avenue" in `stations`)
4. Set `props.[property] = true` on that entity via `PUT /entities/stations/flushing-avenue`
5. Link to the property page, not the parent list: `page.html?id=curved#flushing-avenue`

### Props values

Props can be:
- `true` / `false` — simple membership (curved: true)
- A number — e.g. least-used rank: `{ "least-used": 3 }`
- An object — for richer annotations: `{ "least-used": { "value": 3, "year": 2023 } }`

When Andrew doesn't specify a value, default to `true`.

---

## Common command patterns

```
# Mark been
"been to Taj Mahal"
→ GET /entities?name=Taj+Mahal
→ PUT /entities/unesco/taj-mahal  { "been": true }

# Mark transit done
"finished the Paris Metro"
→ PUT /entities/metros/paris  { "section": "done" }

# Mark closed
"Apple Infinite Loop is closed"
→ GET /entities?name=Infinite+Loop&list=apple
→ PUT /entities/apple/infinite-loop  { "strike": true }

# Add and enrich
"add the Atomium to towers"
→ POST /entities/towers  { "name": "Atomium", "link": "https://en.wikipedia.org/wiki/Atomium" }
→ POST /entities/towers/atomium/enrich

# Set a prop
"Flushing Avenue is curved"
→ GET /entities?name=Flushing+Avenue&list=stations
→ PUT /entities/stations/flushing-avenue  { "props": { ..existing props.., "curved": true } }

# Update page size
"hamburgers page should be medium"
→ PUT /pages/hamburgers  { "size": "medium" }

# Create a page
"create a list of cafés with icon ☕️"
→ POST /pages  { "name": "Cafés", "icon": "☕️", "type": "place", "size": "medium" }
```

---

## Response shapes

**PUT /entities** returns the updated entity doc.

**POST /entities** returns `{ "doc": { ...entity... } }`. The `key` field in `doc` is what you need for the link and for the enrich call.

**POST /entities/enrich** returns the enriched entity doc.

**PUT /pages** returns the updated page doc.

**POST /pages** returns `{ "doc": { ...page... } }`.

Errors: `{ "error": "<code>", "message": "<detail>" }` — codes: `not_found`, `page_not_found`, `bad_request`, `conflict`.
