---
name: detect-opendata-api
description: >
  Detecta qué tipo de API de datos abiertos tiene un portal web de una administración
  pública española (CKAN, DKAN, OpenDataSoft, o requiere scraping). Útil para evaluar
  si un nuevo portal puede integrarse de forma automática antes de intentar la extracción.
argument-hint: [URL del portal, e.g. https://www.miteco.gob.es]
---

# Skill: Detección de API de datos abiertos

## Paso 1: Sondeo del portal

Llama a `detect_opendata_api(portal_url=<URL_del_argumento>)`.

## Paso 2: Interpretación y recomendación

Según el resultado, presenta:

**Si es CKAN:**
```
✅ Portal CKAN detectado
   URL base API: {api_base_url}
   Versión CKAN: {ckan_version}

   Este portal puede integrarse automáticamente.
   → Usa /fetch-dcat-ckan {api_base_url} para descargar los metadatos.
```

**Si es DKAN:**
```
✅ Portal DKAN detectado
   URL base: {api_base_url}

   Este portal puede integrarse automáticamente.
   → Usa /fetch-dcat-nonckan {api_base_url} para descargar los metadatos.
```

**Si es OpenDataSoft:**
```
✅ Portal OpenDataSoft detectado
   URL base: {api_base_url}

   Este portal puede integrarse automáticamente.
   → Usa /fetch-dcat-nonckan {api_base_url} para descargar los metadatos.
```

**Si es ArcGIS Hub:**
```
✅ Portal ArcGIS Hub detectado
   URL del catálogo: {catalog_url}

   Este portal puede integrarse automáticamente.
   → Usa /fetch-dcat-nonckan {catalog_url} para descargar los metadatos.
```

**Si es unknown:**
```
❌ No se detectó API estándar

   Opciones:
     1. El portal puede tener CKAN bajo otra ruta — proporciona la URL exacta del catálogo.
     2. El portal puede no tener API — habría que hacer scraping de sus páginas web.
     3. El portal puede tener una API propietaria — revisa su sección de "Acceso a datos".

   ¿Quieres que intente con una URL alternativa o diferente?
```
