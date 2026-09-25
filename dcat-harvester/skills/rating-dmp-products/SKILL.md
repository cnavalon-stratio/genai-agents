---
name: rating-dmp-products
description: >
  Consulta y establece el rating de calidad (valor de 0 a 100) de uno o varios Data
  Products del DataMarketPlace de Stratio con las tools get_data_product_rating y
  set_data_product_rating. Al establecerlo, el valor es siempre obligatorio: nunca se
  asume ni se inventa. Ambas tools identifican el producto por UUID, así que la skill
  lo resuelve con search_data_products cuando el usuario aporta un nombre, un ID
  numérico o una carpeta, y hace una llamada por cada dataproduct. El rating que se
  establece se guarda como manual y sustituye al automático y a su evidencia.
argument-hint: [valor 0-100 + criterio de selección, e.g. "85 a los dataproducts de biodiversidad"; o solo el criterio para consultar]
---

# Skill: Consultar y establecer el rating de Data Products en el DMP

Lee el rating de calidad de uno o varios Data Products con `get_data_product_rating`
y lo fija con `set_data_product_rating`, resolviendo previamente el UUID de cada producto.

---

## Cuándo usar esta skill

**Consulta** — cuando el usuario pide algo como:
- "¿qué rating tiene el dataproduct ID 42?"
- "dime la calidad de los dataproducts de biodiversidad"
- "¿están valorados los que importé de MITECO?"
- "¿el rating de este producto es manual o automático?"

**Establecer** — cuando el usuario pide algo como:
- "pon rating 85 a los dataproducts de biodiversidad"
- "valora con un 70 el dataproduct ID 42"
- "sube la calidad de los que importé de MITECO a 90"
- "asigna un rating de 50 a todo lo que hay en la carpeta Open Data"

---

## Fase 0 — Determinar la operación

| Lo que pide el usuario | Modo | Recorrido |
|---|---|---|
| Preguntar, ver o comparar ratings | **Consulta** | Fase 1 → Fase 2 → informe |
| Asignar, cambiar, subir o bajar un rating | **Establecer** | Fase 1 → Fase 2 → Fase 3 → Fase 4 → Fase 5 |

Si la petición es ambigua ("revisa el rating de X"), haz **una sola pregunta**: ¿solo consultarlo
o también cambiarlo? No asumas que quiere escribir.

---

## Fase 1 — Identificar los dataproducts y resolver su UUID

Ambas tools identifican el producto **por UUID**, no por el ID numérico.
Resuélvelo siempre antes de llamar; **nunca inventes un UUID**.

| Lo que aporta el usuario | Cómo obtener el UUID |
|---|---|
| El UUID directamente | Úsalo tal cual |
| Un nombre o tema ("los de biodiversidad") | `search_data_products(nameLike="<término>")` |
| Etiquetas / keywords | `search_data_products(keywords=["<kw1>", ...])` |
| Una carpeta del DMP | `list_data_product_paths(nameLike="<carpeta>")` → `search_data_products(pathId="<uuid>")` |
| Un ID numérico (p.ej. de un import report) | `search_data_products` por nombre/carpeta y **empareja el campo `ID`** — ver abajo |

Cada resultado de `search_data_products` se imprime como:

```
[1] <nombre>
    ID: 1234 | UUID: 7709f44a-....
```

Toma el valor de **`UUID`**, no el de `ID`.

⚠️ **`search_data_products` no admite filtrar por ID numérico.** Si el usuario solo tiene IDs
(típico tras un import o tras `/publish-dmp-products`), búscalos por nombre, keywords o `pathId`
y empareja tú el campo `ID` de cada resultado con los IDs pedidos. Si algún ID no aparece en la
búsqueda, dilo explícitamente y no lo proceses — no adivines a qué producto corresponde.

Resolución de coincidencias:
- **0 resultados** → informa al usuario y termina; no amplíes el criterio por tu cuenta.
- **Varios resultados y el usuario quería uno** → muestra la lista `[ID] nombre` y pide que elija.
- **Varios resultados y el usuario quería todos** → continúa con la lista completa.

Usa paginación estricta en la búsqueda (`size=10`, `page` incremental desde 0) para evitar
respuestas gigantes.

---

## Fase 2 — Consultar el rating actual

`get_data_product_rating` acepta **un solo producto por llamada**. Haz **una llamada por cada
dataproduct** y lánzalas en paralelo:

```
get_data_product_rating(data_product_uuid="<uuid-1>")
get_data_product_rating(data_product_uuid="<uuid-2>")
...
```

La respuesta trae `Data Product ID`, `Data Product UUID`, `Value`, `Level`,
`Type` (**Automatic** o **Manual**), `Set by`, `Modified At` y, en los ratings automáticos,
la `Evidence` con la que se calculó.

⚠️ **Un producto sin rating y un producto inexistente devuelven lo mismo.** La API responde 404
en ambos casos y la tool lo traduce a *"has no rating (or it does not exist in this tenant)"*.
No afirmes que el producto no existe: di que **no tiene rating asignado**, y si el UUID salió de
una búsqueda previa, el producto existe con seguridad.

En **modo consulta**, este es el resultado final — preséntalo (Fase 6) y termina.

En **modo establecer**, esta fase es el paso previo recomendado: consulta el rating actual antes
de sobrescribirlo para poder mostrar en el preview qué valor se va a perder, y muy especialmente
si el rating actual es **automático** (al escribir se pierde también su evidencia).

---

## Fase 3 — Obtener el valor de rating (OBLIGATORIO, solo modo establecer)

`value` es un **parámetro obligatorio de la tool**: no hay valor por defecto y **nunca debes
inventarlo, asumirlo ni deducirlo** del contenido del dataproduct ni de su rating anterior.

Si el usuario no ha dado un número explícito, pregunta y espera respuesta:

> "¿Qué valor de rating quieres asignar? Es un número de **0 a 100**:
>   · `low` [0-40)   · `moderate` [40-60)   · `good` [60-80)   · `excellent` [80-100]"

Validaciones antes de seguir:
- Debe ser numérico y estar en el rango **0-100** (ambos incluidos). Fuera de rango, la API
  rechaza la llamada: corrígelo con el usuario, no lo recortes tú.
- Si el usuario solo da una etiqueta cualitativa ("excelente", "malo"), **no elijas tú el número**:
  muéstrale la banda correspondiente y pídele el valor concreto.
- El **nivel** (`low` / `moderate` / `good` / `excellent`) lo deriva la API a partir del valor.
  No se envía ni se puede fijar a mano.

---

## Fase 4 — Preview y confirmación (solo modo establecer)

Muestra qué se va a puntuar, con qué valor y **qué rating tenía antes** (Fase 2):

```
Se va a asignar rating 85 (good) a 3 dataproducts:
  • [ID 1234] Inventario de especies      — actual: 40 (moderate, Manual)
  • [ID 1235] Calidad del aire 2024       — actual: 72 (good, Automatic) ⚠ se pierde la evidencia
  • [ID 1236] Espacios protegidos         — actual: sin rating
```

- **Un solo producto** con UUID o ID inequívoco: procede directamente.
- **Dos o más productos**: pide confirmación explícita antes de aplicar.

Advierte siempre de que el rating se guarda como **manual** y **sustituye al rating automático**
del producto y a su evidencia. El valor anterior no se recupera: para volver atrás hay que fijar
otro valor a mano.

---

## Fase 5 — Aplicar el rating (solo modo establecer)

`set_data_product_rating` acepta **un solo producto por llamada** y no tiene modo bulk ni `dry_run`.
Haz **una llamada por cada dataproduct**:

```
set_data_product_rating(data_product_uuid="<uuid-1>", value=85)
set_data_product_rating(data_product_uuid="<uuid-2>", value=85)
...
```

- Lanza las llamadas en paralelo, una por producto; no esperes entre ellas.
- El mismo `value` se repite en cada llamada salvo que el usuario haya pedido valores distintos
  por producto.
- Un fallo en uno **no aborta** el resto: registra el error y continúa.

Usa el `Level` devuelto por la API en el informe — no lo calcules tú.

---

## Fase 6 — Informe de resultados

**Modo consulta:**

```
⭐ Rating de 3 dataproducts

  [ID 1234] Inventario de especies    → 40 (moderate) · Manual    · user@stratio.com · 2026-09-01
  [ID 1235] Calidad del aire 2024     → 72 (good)     · Automatic · —                · 2026-08-14
  [ID 1236] Espacios protegidos       → sin rating asignado
```

**Modo establecer:**

```
⭐ Rating aplicado a 3/3 dataproducts (valor 85 → good)

  ✅ [ID 1234] Inventario de especies      40 → 85 (good)
  ✅ [ID 1235] Calidad del aire 2024       72 → 85 (good)
  ❌ [ID 1236] Espacios protegidos         → Permission denied
```

Si hay fallos, muestra el error tal cual lo devuelve la API y sugiere reintentar solo
con los UUID fallidos.

Error típico al escribir: **403 / permission denied** — la operación requiere permiso WRITE
sobre el recurso `DATA_PRODUCT_RATING`. Indícaselo al usuario en lugar de reintentar en bucle.

---

## Reglas

- **No escribas cuando solo te piden leer**: ante la duda, consulta con `get_data_product_rating`
  y pregunta antes de establecer nada.
- **El valor de rating es obligatorio** al establecerlo: si el usuario no lo da, pregúntalo. Nunca
  lo inventes, nunca uses un valor por defecto y nunca lo deduzcas de la calidad aparente del
  dataproduct ni de su rating anterior.
- **Valida el rango 0-100 antes de llamar.** Fuera de rango, corrige con el usuario.
- **Nunca inventes un UUID**: resuélvelo siempre con `search_data_products` (o `list_data_product_paths`
  + `search_data_products` cuando el criterio sea una carpeta).
- **UUID, no ID numérico**: ambas tools fallan si les pasas el `ID`. Del resultado de búsqueda,
  coge siempre el campo `UUID`.
- **Una llamada por dataproduct** en ambas tools: no existe modo bulk. No agrupes ni envíes listas.
- **"Sin rating" no significa "no existe"**: la API devuelve 404 en ambos casos. No concluyas que
  el producto no existe a partir de esa respuesta.
- El rating manual **sustituye al automático** y a su evidencia — avisa siempre antes de aplicarlo,
  y con más motivo si la consulta previa devolvió `Type: Automatic`.
- Un error en un producto no aborta el proceso; informa al final con el desglose.
- Si el usuario quiere puntuar "todos" sin ningún criterio de selección, pide al menos un filtro
  para evitar cambios masivos accidentales.
