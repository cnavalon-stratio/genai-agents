---
name: sync-dcat-changes
description: >
  Sincroniza los cambios en un portal CKAN con el DataMarketPlace: detecta datasets
  nuevos o modificados desde la última sincronización y los importa en el DMP.
  Implementa sincronización delta — solo procesa lo que ha cambiado.
argument-hint: [URL del portal CKAN, e.g. https://catalogo.datosabiertos.miteco.gob.es]
---

# Skill: Sincronización delta DCAT-AP → DMP

## Fase 1: Determinar punto de referencia

1. Pregunta al usuario la fecha desde la que sincronizar:
   ```
   ¿Desde qué fecha sincronizamos?
     [1] Desde la última sincronización registrada: {ultima_fecha}
     [2] Introducir fecha manualmente (ISO 8601, e.g. 2024-06-01T00:00:00Z)
     [3] Sincronización completa (todos los datasets)
   ```
2. Guarda la fecha de inicio de esta sincronización para futuras referencias
   (anótala en la respuesta al usuario para que pueda usarla la próxima vez).

## Fase 2: Descubrimiento de cambios

1. Llama a `list_ckan_datasets(base_url=<portal>, since_modified=<fecha_referencia>)`.
2. Si `total == 0`: informa que no hay cambios desde esa fecha. Proceso completado.
3. Si `total > 0`: muestra resumen:
   ```
   📋 Cambios detectados desde {fecha}:
      Datasets nuevos o modificados: {total}
      {preview de los primeros 5}

   ¿Proceder con la sincronización?
   ```

## Fase 3: Descarga y import de cambios

Sigue el mismo flujo que `/import-dcat-dmp` (mismas estrategias de batching y polling):

1. `fetch_dcat_dataset(base_url, dataset_id, format="auto")` → guarda el RDF en `/tmp/dcat-{id}.ttl`.
   Acumula solo `file_path`, nunca el contenido RDF.
2. Lanza todos los imports con `import_dcat_file(rdf_file_path=..., wait=False)` → recoge importIds.
3. Un único `get_import_reports(import_ids=[...], wait=True)` para obtener todos los resultados.
   - El DMP gestiona internamente si el dataset es nuevo o es una actualización (basado en URI).

## Fase 4: Reporte de sincronización

```
🔄 Sincronización completada

  Fecha de referencia:    {fecha_referencia}
  Datasets procesados:    {total}
  Actualizados/creados:   {successful}
  Errores:                {errors}

  Próxima sincronización: usa since_modified="{fecha_inicio_esta_sync}"
```

## Reglas invariables

- La sincronización delta es preferible a la sincronización completa para portales grandes.
- El DMP gestiona internamente si el dataset es nuevo o es una actualización.
- Guarda siempre la fecha de inicio de la sync para que el usuario pueda usarla como
  referencia en la próxima ejecución.
- Si hay más de 500 datasets modificados, advierte antes de proceder — puede indicar
  una sincronización completa accidental.
