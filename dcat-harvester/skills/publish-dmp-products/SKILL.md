---
name: publish-dmp-products
description: >
  Publica en el DataMarketPlace de Stratio uno o varios Data Products importados
  previamente (vía DCAT-AP o cualquier otro mecanismo). Soporta filtros por nombre,
  keywords, path (carpeta) o una lista explícita de IDs, siempre con preview en
  dry_run y confirmación del usuario antes de aplicar los cambios. Resuelve el
  pathId con list_data_product_paths cuando el usuario nombra una carpeta.
argument-hint: [criterio de selección, e.g. "los dataproducts de biodiversidad" o IDs concretos]
---

# Skill: Publicar Data Products en el DMP

Publica en el DataMarketPlace uno o varios Data Products que hayan sido importados
previamente (vía DCAT-AP o cualquier otro mecanismo), con soporte de filtros por nombre,
keywords o path, o una lista explícita de IDs.

---

## Cuándo usar esta skill

Cuando el usuario pide algo como:
- "publica los dataproducts de biodiversidad"
- "publica todos los que importé de MITECO"
- "publica todo lo que hay en la carpeta Open Data"
- "cambia a publicado el dataproduct ID 42"
- "publica los dataproducts relacionados con calidad del agua"

---

## Fase 0 — Entender el alcance

`publish_data_products` acepta exactamente estos filtros. No inventes parámetros:

| Caso | Parámetro a usar |
|------|-----------------|
| El usuario da un tema/palabra clave que aparece en el nombre | `name_like` (subcadena, case-insensitive) |
| El usuario menciona keywords o etiquetas concretas | `keywords` (lista; coincide con CUALQUIERA) |
| El usuario acota a una carpeta del DMP | `path_id` (UUID del path y sus subpaths) |
| El usuario da IDs explícitos | `data_product_ids` (lista de enteros) |
| El usuario quiere publicar "todos" sin filtro | **No permitido** — pide al menos un criterio |

Los filtros son combinables entre sí (p.ej. `name_like` + `path_id`).

⚠️ **No existen** `description_like`, `created_after` ni `created_before` en esta tool.
Si el usuario pide filtrar por descripción o por fecha de creación, dilo explícitamente
y ofrece la alternativa: localizar los productos con `search_data_products`
(que sí acepta `descriptionLike`) y publicar después con `data_product_ids`.

Si la petición es ambigua, formula **una sola pregunta** para clarificar el criterio.
No procedas si no tienes al menos un filtro claro.

### Resolver el `path_id` cuando el usuario nombra una carpeta

El `path_id` es un UUID — **nunca lo inventes**. Resuélvelo con `list_data_product_paths`:

```
list_data_product_paths(nameLike="<nombre de la carpeta>")
```

Cada nodo se imprime como `• <nombre> ✔ | ID: <uuid> | Path: <metadataPath> | Role: <rol>`;
el ✔ marca las coincidencias y los nodos sin ✔ son sus padres. Usa el valor de `ID`.

- **0 coincidencias** → dilo y lista el árbol completo con `list_data_product_paths()`.
- **Varias ✔** → muéstralas con su `metadataPath` y pide al usuario que elija. No escojas tú.
- **Una sola ✔** → úsala y confirma nombre + `metadataPath`.

Si el usuario no menciona carpeta, no pases `path_id`: la tool usará `DATAMARKET_PATH_ID`
como ámbito por defecto, que **por sí solo no cuenta como filtro** — sigue haciendo falta
`name_like`, `keywords` o `data_product_ids`.

---

## Fase 1 — Preview con dry_run

**Siempre** empieza con una llamada en modo dry_run para mostrar al usuario qué se va a publicar:

```
publish_data_products(
  name_like="<término>",         # si aplica
  keywords=["<kw1>", ...],       # si aplica
  path_id="<uuid>",              # si aplica (resuelto con list_data_product_paths)
  data_product_ids=[...],        # si aplica
  dry_run=true
)
```

Muestra el resultado al usuario:
- Si `total == 0`: informa que no hay productos que coincidan con los filtros y termina.
- Si `total > 0`: muestra la lista y el recuento.

---

## Fase 2 — Confirmación para operaciones masivas

- Si `total <= 10`: procede directamente sin pedir confirmación.
- Si `total > 10`: muestra el resumen y pide confirmación explícita:
  > "Se van a publicar **N dataproducts**. ¿Confirmas?"
  Espera la respuesta. No publiques sin confirmación cuando N > 10.

---

## Fase 3 — Publicar

Ejecuta la publicación con los mismos parámetros pero `dry_run=false` (o sin el parámetro):

```
publish_data_products(
  name_like="<término>",
  keywords=["<kw1>", ...],
  path_id="<uuid>",
  data_product_ids=[...],
  dry_run=false
)
```

La tool maneja la paginación y publica en paralelo — una sola llamada es suficiente.

---

## Fase 4 — Informe de resultados

Tras la llamada, muestra al usuario:
- Cuántos se publicaron correctamente
- Cuántos fallaron (con el error)
- Si hay fallos, sugiere revisar el estado del DMP o reintentar con los IDs fallidos usando `data_product_ids`

Ejemplo de resumen:
```
✅ Publicados 23/25 dataproducts
❌ 2 errores:
  • [ID 101] Dataset biodiversidad marina — BPM timeout
  • [ID 208] Inventario especies — Permission denied
```

---

## Reglas

- **Si los IDs son conocidos de un paso anterior** (p.ej. vienen de un import report),
  usa siempre `data_product_ids` directamente — no hagas una búsqueda por nombre.
  Los IDs son exactos e inequívocos; una búsqueda puede devolver falsos positivos.
- **Nunca inventes un `path_id`**: resuélvelo con `list_data_product_paths` y confirma
  con el usuario cuando haya varias coincidencias.
- Nunca publiques sin haber mostrado primero el dry_run al usuario.
- Si el usuario ya sabe exactamente qué quiere publicar (da IDs), el dry_run es opcional pero recomendado.
- No uses `search_data_products` por separado para luego publicar manualmente — usa siempre
  `publish_data_products`, que filtra y pagina internamente de forma más eficiente.
  La única excepción es el filtrado por descripción, que esta tool no soporta.
- Si el usuario quiere publicar "todos sin filtro", explica que es necesario al menos un criterio de selección para evitar publicaciones accidentales masivas.
