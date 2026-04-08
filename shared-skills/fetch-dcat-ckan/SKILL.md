---
name: fetch-dcat-ckan
description: >
  Descarga todos los metadatos DCAT-AP-ES de un portal CKAN de una administración
  pública española. Detecta automáticamente si el portal usa CKAN y obtiene los
  datasets en formato RDF (Turtle, JSON-LD o RDF/XML) listos para importar en el DMP.
argument-hint: [URL del portal, e.g. https://catalogo.datosabiertos.miteco.gob.es]
---

# Skill: Fetch DCAT-AP desde portal CKAN

## Fase 1: Detección del portal

1. Llama a `detect_opendata_api(portal_url=<URL_del_argumento>)`.
2. Evalúa el resultado:
   - Si `platform == "ckan"` → continúa con `api_base_url` devuelta.
   - Si `platform == "dkan"` o `"opendatasoft"` → informa al usuario que este servidor
     solo soporta CKAN por ahora. Sugiere añadir soporte específico.
   - Si `platform == "unknown"` → informa que no se detectó API estándar.
     Pregunta si desea intentar scraping manual o proporcionar la URL base de CKAN directamente.
3. Confirma con el usuario la URL base que se usará antes de continuar.

## Fase 2: Descubrimiento de datasets

1. Llama a `list_ckan_datasets(base_url=<api_base_url>, page=0, page_size=100)`.
2. Muestra al usuario un resumen:
   - Total de datasets disponibles
   - Rango de fechas de modificación (el más antiguo y el más reciente)
   - Organizaciones presentes (si hay variedad)
3. Pregunta al usuario:
   ```
   ¿Qué datasets deseas descargar?
     [1] Todos ({total} datasets)
     [2] Solo los modificados desde una fecha (para sincronización)
     [3] Filtrar por texto (título o descripción)
     [4] Seleccionar un subset manualmente
   ```
4. Según la elección:
   - Opción 1: confirma el volumen antes de proceder (puede ser lento para >500 datasets).
   - Opción 2: pide la fecha y usa el parámetro `since_modified`.
   - Opción 3: pide el texto y usa el parámetro `query`.
   - Opción 4: muestra lista y espera selección por número.

## Fase 3: Descarga de metadatos DCAT-AP-ES

**Elige la estrategia según lo que haya pedido el usuario:**

- Si el usuario quiere **todos los datasets** (sin límite ni filtro) → usa `fetch_dcat_catalog` primero:
  1. Llama a `fetch_dcat_catalog(base_url=<api_base_url>, format="ttl")`.
     - Si tiene éxito → un único fichero con todos los datasets. Ve directamente a Fase 4.
     - Si falla (portal sin plugin ckanext-dcat) → continúa con la descarga individual.

- Si el usuario especifica **un subconjunto** (N primeros, filtro por fecha, por texto, selección manual) → ve directamente a la descarga individual, no uses `fetch_dcat_catalog`.

**Descarga individual por dataset** (subconjunto o fallback del catálogo completo):

Procesa los datasets en **batches de 20** para no saturar el portal:

Para cada dataset:
1. Llama a `fetch_dcat_dataset(base_url=<api_base_url>, dataset_id=<name>, format="auto")`.
2. Si tiene éxito: acumula únicamente `file_path` y `dataset_id`. **No acumules ni menciones el contenido RDF.**
3. Si falla: registra el error con el motivo y continúa (no abortes el proceso completo).

Muestra progreso tras cada batch: `[20/847] descargados...`

## Fase 4: Resumen y siguiente paso

Al terminar, presenta:
```
✅ Descarga completada
   Total en el portal:     847
   Descargados con éxito:  841
   Fallidos:               6

Datasets fallidos:
  - calidad-aire-2010: No se encontraron endpoints RDF
  - ...

Ficheros guardados en /tmp/dcat-*.ttl

¿Deseas importar los 841 datasets descargados en el DMP?
Usa /import-dcat-dmp para proceder.
```

## Reglas invariables

- **Nunca acumules ni muestres el contenido RDF** — solo trabaja con `file_path`. El RDF está en disco.
- Si un dataset falla, registra y continúa. No abortes el proceso completo.
- Informa del progreso cada 20 datasets para que el usuario sepa que el proceso avanza.
- Respeta siempre el rate limiting del portal: añade una pausa de 0.5s entre batches si
  el portal devuelve errores 429 o 503.
