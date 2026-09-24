---
name: unpublish-dmp-products
description: >
  Cambia a estado Unpublished uno o varios Data Products del DataMarketPlace de
  Stratio. Soporta filtros por nombre, keywords, path (carpeta) o una lista
  explícita de IDs, siempre con preview en dry_run y confirmación del usuario
  antes de aplicar los cambios. Resuelve el pathId con list_data_product_paths
  cuando el usuario nombra una carpeta. Tras despublicar ofrece el borrado
  definitivo vía delete_data_product, que solo se ejecuta con confirmación
  explícita del usuario por ser irreversible.
argument-hint: [criterio de selección, e.g. "los dataproducts de biodiversidad" o IDs concretos]
---

# Skill: Despublicar Data Products en el DMP

Cambia el estado de uno o varios Data Products del DataMarketPlace a Unpublished,
con soporte de filtros por nombre, keywords o path, o una lista explícita de IDs.
Opcionalmente, y solo con confirmación explícita del usuario, los elimina después
de forma permanente con `delete_data_product`.

---

## Cuándo usar esta skill

Cuando el usuario pide algo como:
- "despublica los dataproducts de biodiversidad"
- "quita de publicado todos los que importé de MITECO"
- "despublica todo lo que hay en la carpeta Open Data"
- "cambia a despublicado el dataproduct ID 42"
- "archiva / retira los datasets de X"
- "borra / elimina los dataproducts de X" → despublica primero y ofrece el borrado (Fase 5)

---

## Fase 0 — Entender el alcance

`unpublish_data_products` acepta exactamente estos filtros. No inventes parámetros:

| Caso | Parámetro a usar |
|------|-----------------|
| Palabra clave que aparece en el nombre | `name_like` (subcadena, case-insensitive) |
| Etiquetas / keywords concretas | `keywords` (lista; coincide con CUALQUIERA) |
| Acotar a una carpeta del DMP | `path_id` (UUID del path y sus subpaths) |
| El usuario da IDs concretos | `data_product_ids` (lista de enteros) |
| Despublicar "todos" sin filtro | **No permitido** — pide al menos un criterio |

Los filtros son combinables entre sí (p.ej. `keywords` + `path_id`).

⚠️ **No existen** `description_like`, `created_after`, `created_before`, `text_query`
ni `searcher_filters` en esta tool. Si el usuario pide filtrar por descripción, por tema,
por publisher o por fecha, dilo explícitamente y ofrece la alternativa: localizar los
productos con `search_data_products` (que acepta `descriptionLike`, `keywords`, `pathId`)
y despublicar después con `data_product_ids`.

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

**Siempre** empieza con una llamada en modo dry_run para mostrar al usuario qué se va a despublicar:

```
unpublish_data_products(
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

## Fase 2 — Confirmación (siempre obligatoria)

A diferencia de la publicación, **despublicar siempre requiere confirmación explícita**,
independientemente del número de productos:

> "Se van a despublicar **N dataproducts**. Esta acción los retirará del catálogo público. ¿Confirmas?"

Espera la respuesta. No despubliques sin confirmación.

---

## Fase 3 — Despublicar

Ejecuta la operación con los mismos parámetros pero `dry_run=false` (o sin el parámetro):

```
unpublish_data_products(
  name_like="<término>",
  keywords=["<kw1>", ...],
  path_id="<uuid>",
  data_product_ids=[...],
  dry_run=false
)
```

La tool maneja la paginación y despublica en paralelo — una sola llamada es suficiente.

Si el entorno requiere un estado BPM diferente a `"Unpublished"` (p.ej. `"Draft"`),
pásalo en `unpublish_status`:

```
unpublish_data_products(
  ...,
  unpublish_status="Draft"
)
```

---

## Fase 4 — Informe de resultados

Tras la llamada, muestra al usuario:
- Cuántos se despublicaron correctamente
- Cuántos fallaron (con el error)
- Si hay fallos, sugiere reintentar con los IDs fallidos usando `data_product_ids`

Ejemplo de resumen:
```
✅ Despublicados 23/25 dataproducts
❌ 2 errores:
  • [ID 101] Dataset biodiversidad marina — BPM timeout
  • [ID 208] Inventario especies — Permission denied
```

---

## Fase 5 — Ofrecer el borrado definitivo (opcional)

Tras el informe de la Fase 4, **pregunta siempre** si además quiere eliminarlos:

> "Los N dataproducts están despublicados. ¿Quieres **eliminarlos definitivamente** del DMP?
> El borrado elimina también sus assets, data contracts, ficheros subidos y el mapping del
> import RDF. **Es irreversible: no hay papelera ni deshacer.**"

Si el usuario no responde que sí de forma inequívoca, **termina aquí**. Silencio, duda,
"quizá", "ya veré" o un cambio de tema no son un sí.

### Conjunto de candidatos al borrado

El conjunto son **todos los productos del alcance de la Fase 1**, no solo los que cambiaron
de estado en la Fase 3. Incluye por tanto:

- Los que el informe marca `UNPUBLISHED` (despublicados ahora).
- Los que salen con `ERROR` **porque ya estaban despublicados** — la transición BPM falla
  al no haber cambio de estado, pero el producto es igualmente un candidato válido.

Distingue en el informe los `ERROR` que son "ya estaba despublicado" de los fallos reales
(timeout BPM, permisos, producto aún publicado). Menciónalos, pero no los excluyas del
listado: el API de borrado rechaza por sí mismo los que sigan publicados y devuelve el error.

### Confirmación explícita (obligatoria, sin excepciones)

Muestra la lista completa `[ID] nombre` de lo que se va a borrar y pide una confirmación
explícita e inequívoca **aunque solo sea un producto**:

> "Se van a **eliminar de forma permanente** estos N dataproducts:
>   • [ID 101] Dataset biodiversidad marina
>   • [ID 208] Inventario especies
>   ...
> Esta acción **no se puede deshacer**. Escribe `BORRAR` para confirmar."

Espera la respuesta. **Nunca borres sin esa confirmación explícita.** No hay atajos:
- Una confirmación previa de despublicar **no** autoriza el borrado — son dos decisiones distintas.
- Que el usuario dijera "despublícalo y bórralo" al principio **no** sustituye a esta
  confirmación final: se la pides igualmente con la lista concreta delante.

### Ejecución

`delete_data_product` borra **un producto por llamada** y no tiene `dry_run` ni modo bulk:

```
delete_data_product(data_product_id=101)
delete_data_product(data_product_id=208)
...
```

- Usa el **ID numérico** (el campo `ID` del informe / de `search_data_products`), nunca el UUID.
- Lanza las llamadas en paralelo, una por producto.
- Un fallo en uno no aborta el resto: registra y continúa.
- Error típico: el API rechaza el borrado si el producto sigue publicado. En ese caso indica
  que hay que despublicarlo primero y ofrece reintentar.

### Informe de borrado

```
🗑️ Eliminados 24/25 dataproducts
❌ 1 error:
  • [ID 208] Inventario especies — el producto sigue publicado, despublícalo primero
```

---

## Reglas

- Nunca despubliques sin haber mostrado primero el dry_run al usuario.
- **Siempre pide confirmación explícita** antes de despublicar, incluso para un solo producto.
- **Nunca borres sin confirmación explícita del usuario**, ni siquiera un solo producto y ni
  siquiera si pidió "despublica y borra" desde el principio: el borrado es irreversible y
  requiere su propia confirmación con la lista de IDs delante (Fase 5).
- **Despublicar y borrar son dos decisiones separadas**: confirmar la primera nunca autoriza
  la segunda. Si el usuario no contesta o duda, no borres.
- `delete_data_product` es **un producto por llamada**, por ID numérico (no UUID), sin
  `dry_run` ni modo bulk. La lista previa de la Fase 5 hace de preview.
- **Nunca inventes un `path_id`**: resuélvelo con `list_data_product_paths` y confirma
  con el usuario cuando haya varias coincidencias.
- **Si los IDs vienen de un paso anterior** (p.ej. un import report), usa `data_product_ids`
  directamente — no busques por nombre, que puede dar falsos positivos.
- No uses `search_data_products` por separado para luego despublicar manualmente — usa siempre
  `unpublish_data_products`, que filtra y pagina internamente de forma más eficiente.
  La única excepción es el filtrado por descripción, que esta tool no soporta.
- Si por petición del usuario necesitas una búsqueda exploratoria previa, usa paginación
  estricta en `search_data_products` (`size=10`, `page` incremental desde 0) para evitar respuestas gigantes.
- Si el usuario quiere despublicar "todos sin filtro", explica que es necesario al menos un criterio de selección para evitar operaciones accidentales masivas.
