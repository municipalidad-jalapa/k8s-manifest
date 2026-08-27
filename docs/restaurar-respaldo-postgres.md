# Restaurar un respaldo de Postgres (SCRUM-163 / HU-68)

Guía con los comandos exactos para restaurar un respaldo, pensada para
alguien que no escribió el CronJob de respaldo.

## Dónde están los respaldos

Cuenta de Azure Storage `ecorutabackups`, contenedor `pg-backups`, un
archivo `.sql.gz` por día (`pg_dump` comprimido con gzip), separados por
entorno: `qa/` y `production/`. Se conservan 14 días (política de ciclo de
vida del storage account, no hay que borrarlos a mano).

## 1. Listar los respaldos disponibles

```bash
az storage blob list \
  --account-name ecorutabackups \
  --account-key "$AZURE_STORAGE_ACCESS_KEY" \
  --container-name pg-backups \
  --prefix production/ \
  -o table
```

(Cambiar `production/` por `qa/` según el entorno.)

## 2. Descargar el respaldo que se quiere restaurar

```bash
az storage blob download \
  --account-name ecorutabackups \
  --account-key "$AZURE_STORAGE_ACCESS_KEY" \
  --container-name pg-backups \
  --name "production/ecoruta-20260827-0300.sql.gz" \
  --file ./ecoruta-restaurar.sql.gz
```

## 3. Copiar el respaldo al pod de Postgres y restaurarlo

**IMPORTANTE:** esto sobrescribe los datos actuales de la base indicada.
Verificar el namespace (`qa` o `production`) antes de correr esto.

```bash
# Conectarse al cluster correcto primero (contexto del GitLab Agent o
# az aks get-credentials, segun como se esté trabajando).

# Copiar el archivo al pod:
kubectl cp ./ecoruta-restaurar.sql.gz qa/postgres-0:/tmp/ecoruta-restaurar.sql.gz

# Restaurar (recrea la base para partir de un estado limpio):
kubectl exec -n qa postgres-0 -- sh -c '
  gunzip -c /tmp/ecoruta-restaurar.sql.gz > /tmp/ecoruta-restaurar.sql &&
  psql -U ecoruta -d postgres -c "DROP DATABASE IF EXISTS ecoruta_restaurada;" &&
  psql -U ecoruta -d postgres -c "CREATE DATABASE ecoruta_restaurada;" &&
  psql -U ecoruta -d ecoruta_restaurada -f /tmp/ecoruta-restaurar.sql
'
```

Se restaura primero a una base separada (`ecoruta_restaurada`), no
directo sobre `ecoruta`, para poder verificar el contenido antes de
decidir promoverla. Para promoverla a la base real:

```bash
kubectl exec -n qa postgres-0 -- sh -c '
  psql -U ecoruta -d postgres -c "ALTER DATABASE ecoruta RENAME TO ecoruta_anterior;" &&
  psql -U ecoruta -d postgres -c "ALTER DATABASE ecoruta_restaurada RENAME TO ecoruta;"
'
```

(`ecoruta_anterior` queda como respaldo de la base previa a la
restauración, por si hace falta revertir. Borrarla manualmente una vez
confirmado que todo está bien.)

## 4. Verificar que la base quedó consultable

```bash
kubectl exec -n qa postgres-0 -- psql -U ecoruta -d ecoruta -c "\dt"
kubectl exec -n qa postgres-0 -- psql -U ecoruta -d ecoruta -c "SELECT count(*) FROM equipo;"
```

## Limitación conocida

Esta historia usa un punto de restauración por día (el último `pg_dump`
disponible), no recuperación a un punto en el tiempo exacto — eso quedó
fuera de alcance (ver nota en SCRUM-163). Si la base se corrompe a media
tarde, el respaldo más reciente disponible es el de las 03:00 UTC de esa
misma madrugada; los cambios posteriores a esa hora se pierden.

## Prueba de esta restauración

Este procedimiento (pg_dump + Blob Storage) fue probado end-to-end en QA
con evidencia real — ver comentario en SCRUM-163 para el detalle completo
del ensayo (datos de prueba insertados, restaurados y verificados).
