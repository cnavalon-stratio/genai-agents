# Skill: Despublicar Data Products en el DMP

Cambia el estado de uno o varios Data Products del DataMarketPlace a Unpublished,
con soporte de filtros por nombre y/o keywords, o una lista explícita de IDs.

---

## Cuándo usar esta skill

Cuando el usuario pide algo como:
- "despublica los dataproducts de biodiversidad"
- "quita de publicado todos los que importé de MITECO"
- "cambia a despublicado el dataproduct ID 42"
- "despublica los dataproducts relacionados con calidad del agua"
- "archiva / retira los datasets de X"

---

## Fase 0 — Entender el alcance

Antes de ejecutar nada, identifica de qué tipo es la petición:

| Caso | Parámetro a usar |
|------|-----------------|
| El usuario da un tema/palabra clave en el nombre | `name_like` |
| El usuario menciona keywords o etiquetas concretas | `keywords` |
| El usuario da IDs explícitos | `data_product_ids` |
| El usuario quiere despublicar "todos" sin filtro | **No permitido** — pide al menos un criterio |

Si la petición es ambigua, formula **una sola pregunta** para clarificar el criterio.
No procedas si no tienes al menos un filtro claro.

---

## Fase 1 — Preview con dry_run

**Siempre** empieza con una llamada en modo dry_run para mostrar al usuario qué se va a despublicar:

```
unpublish_data_products(
  name_like="<término>",       # si aplica
  keywords=["<kw1>", ...],     # si aplica
  data_product_ids=[...],      # si aplica
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

## Reglas

- Nunca despubliques sin haber mostrado primero el dry_run al usuario.
- **Siempre pide confirmación explícita** antes de despublicar, incluso para un solo producto.
- No uses `search_data_products` por separado para luego despublicar manualmente — usa siempre `unpublish_data_products` que lo hace internamente de forma más eficiente.
- Si el usuario quiere despublicar "todos sin filtro", explica que es necesario al menos un criterio de selección para evitar operaciones accidentales masivas.
