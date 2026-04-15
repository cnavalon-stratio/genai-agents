# Skill: fetch-dcat-nonckan

Descarga el catálogo DCAT-AP-ES de un portal de datos abiertos que **no usa CKAN**,
aplicando una estrategia por capas (Tier 1 → 2 → 3) hasta obtener el catálogo.

---

## Cuándo usar esta skill

Úsala cuando `detect_opendata_api` devuelva DKAN, OpenDataSoft, o "unknown",
o cuando el usuario proporcione la URL de un portal del que no sabes si tiene CKAN.

**No uses esta skill si el portal es CKAN** — en ese caso usa `/fetch-dcat-ckan`.

---

## Fase 0 — Detección (si no se ha hecho ya)

Si el usuario solo ha dado una URL sin pasar antes por `detect_opendata_api`:

1. Llama a `detect_opendata_api(portal_url=<url>)`.
2. Si devuelve `ckan` → detente y sugiere `/fetch-dcat-ckan`.
3. Si devuelve `dkan` → salta a Fase 3 (Tier 3 DKAN).
4. Si devuelve `opendatasoft` → salta a Fase 3 (Tier 3 ODS).
5. Si devuelve `arcgis_hub` → salta a Fase 3 (Tier 3 ArcGIS Hub).
6. Si devuelve `dcat_direct` → salta directamente a Fase 1 con la `catalog_url` ya detectada.
7. Si devuelve `unknown` → sigue a Fase 1 (Tier 1 directo).

---

## Fase 1 — Tier 1: Catálogo DCAT directo

Llama a `fetch_dcat_catalog_direct(portal_url=<url>)`.

- Si tiene éxito → informa al usuario del fichero descargado y pregunta si quiere importar.
  Sugiere `/import-dcat-dmp` con ese `file_path`.
- Si falla → continúa a Fase 2.

---

## Fase 2 — Tier 2: Descubrimiento via datos.gob.es

Si se trata de una administración pública española y Tier 1 falló:

1. Llama a `discover_dcat_source(org_identifier=<nombre_o_NIF>)`.
   - Si el usuario dio URL pero no nombre, extrae el dominio como término de búsqueda.
2. Muestra los resultados al usuario: organización, URL del catálogo, plataforma.
3. Si hay catálogo DCAT confirmado:
   - Si la plataforma es CKAN → sugiere `/fetch-dcat-ckan` con esa URL.
   - Si la plataforma es DKAN → continúa a Fase 3 (Tier 3 DKAN) con esa URL.
   - Si la plataforma es OpenDataSoft → continúa a Fase 3 (Tier 3 ODS) con esa URL.
   - Si plataforma desconocida → vuelve a Tier 1 con la catalog_url encontrada.
4. Si `discover_dcat_source` no devuelve resultados → continúa a Fase 3.

---

## Fase 3 — Tier 3: APIs específicas

### Si el portal es DKAN:

1. Llama a `list_dkan_datasets(base_url=<url>)` para explorar el catálogo.
2. Muestra resumen al usuario (total datasets, primera página).
3. Pregunta estrategia:
   - **Opción A**: Catálogo completo → `fetch_dcat_catalog_direct` (DKAN con módulo DCAT puede exponer `/catalog.rdf`)
   - **Opción B**: Por datasets individuales → usa `fetch_dcat_dataset` en batches de 20
4. Si el usuario elige opción B, aplica el mismo flujo de batches que en `/fetch-dcat-ckan`:
   - Descarga en batches de 20, tasa 0.5s entre batches si hay errores 429/503
   - Nunca acumules contenido RDF — trabaja solo con `file_path`
   - Informa del progreso cada 20 datasets

### Si el portal es OpenDataSoft:

1. Informa al usuario: "Portal OpenDataSoft detectado — descargando catálogo completo con un único export DCAT."
2. Llama a `fetch_ods_catalog(base_url=<url>)`.
3. Si tiene éxito → informa del fichero y pregunta si quiere importar.
4. Si falla → informa y sugiere revisar manualmente el portal.

### Si el portal es ArcGIS Hub:

1. Informa al usuario: "Portal ArcGIS Hub detectado — descargando catálogo completo desde /data.json."
2. Llama a `fetch_arcgis_catalog(base_url=<url>)`.
3. Si tiene éxito → informa del fichero (formato JSON-LD / DCAT-US) y pregunta si quiere importar.
4. Si falla → informa y sugiere revisar si el portal realmente usa ArcGIS Hub.

---

## Fase 4 — Resumen y propuesta

Al terminar la descarga (por cualquier tier):

- Muestra: portal, número de ficheros descargados, tier utilizado, rutas de los ficheros.
- Si hubo fallos parciales: lista los datasets que fallaron (no aborta el proceso).
- Pregunta: "¿Quieres importar estos metadatos en el DMP? Usa `/import-dcat-dmp`."

---

## Reglas

- **MCP-first**: todas las operaciones van via tools. Nunca hagas HTTP directamente.
- **Nunca acumules RDF**: trabaja solo con `file_path`. No leas ni muestres el contenido RDF.
- **Errores aislados**: un dataset que falla no aborta el proceso. Registra y continúa.
- **Confirma antes de >200 datasets** (opción B): muestra el volumen y pide confirmación.
- **Tier 1 y Tier 2 son siempre preferibles** a descargar dataset a dataset (más eficiente).
- Si ningún tier funciona → informa al usuario con el resumen de lo intentado y sugiere
  contactar al portal para solicitar una URL de catálogo DCAT o acceso a su API.
