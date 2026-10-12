# Команды для практической №1

Все команды выполняются в PowerShell (Windows).

## Подготовка

Остановка Windows-ового PostgreSQL (чтобы не конфликтовал по порту 5432), от имени администратора:
```powershell
Stop-Service postgresql-x64-18
```

Рабочая папка:
```powershell
mkdir "$HOME\lakehouse" -Force
cd "$HOME\lakehouse"
```

## Запуск стека

### 1. MinIO (community-образ, т.к. официальный требует лицензию)
```powershell
docker run -d --name lakehouse-minio `
  -p 9000:9000 -p 9001:9001 `
  -e MINIO_ROOT_USER=minioadmin `
  -e MINIO_ROOT_PASSWORD=minioadmin `
  -v minio-data:/data `
  netvark/minio:latest server /data --console-address ":9001"
```

### 2. PostgreSQL (для метаданных Lakekeeper)
```powershell
docker run -d --name lakehouse-pg `
  -e POSTGRES_PASSWORD=postgres `
  -p 5432:5432 `
  -v pg-data:/var/lib/postgresql/data `
  postgres:17
```

### 3. Миграции Lakekeeper (один раз)
```powershell
docker run --rm `
  -e 'LAKEKEEPER__PG_ENCRYPTION_KEY=This-is-NOT-Secure!' `
  -e 'LAKEKEEPER__PG_DATABASE_URL_READ=postgresql://postgres:postgres@host.docker.internal:5432/postgres' `
  -e 'LAKEKEEPER__PG_DATABASE_URL_WRITE=postgresql://postgres:postgres@host.docker.internal:5432/postgres' `
  quay.io/lakekeeper/catalog:latest-main migrate
```

### 4. Lakekeeper
```powershell
docker run -d --name lakekeeper `
  -e 'LAKEKEEPER__PG_ENCRYPTION_KEY=This-is-NOT-Secure!' `
  -e 'LAKEKEEPER__PG_DATABASE_URL_READ=postgresql://postgres:postgres@host.docker.internal:5432/postgres' `
  -e 'LAKEKEEPER__PG_DATABASE_URL_WRITE=postgresql://postgres:postgres@host.docker.internal:5432/postgres' `
  -p 8181:8181 `
  quay.io/lakekeeper/catalog:latest-main serve
```

Проверка:
```powershell
curl.exe http://localhost:8181/health
```

### 5. Trino
```powershell
docker run -d --name trino `
  -p 8080:8080 `
  -v "${PWD}\catalog:/etc/trino/catalog" `
  trinodb/trino:476
```

Проверка:
```powershell
docker exec trino trino --execute "SHOW CATALOGS"
```

### 6. Metabase (BI)
```powershell
docker run -d --name metabase `
  -p 3000:3000 `
  -v metabase-data:/metabase-data `
  -e "MB_DB_FILE=/metabase-data/metabase.db" `
  metabase/metabase:latest
```

## Конфиги каталогов Trino

Файлы в папке `catalog/`. Пример `catalog/bronze.properties` (для silver и gold — то же самое, меняется только `iceberg.rest-catalog.warehouse`):

```
connector.name=iceberg
iceberg.catalog.type=rest
iceberg.rest-catalog.uri=http://host.docker.internal:8181/catalog
iceberg.rest-catalog.warehouse=bronze
iceberg.rest-catalog.nested-namespace-enabled=true
iceberg.unique-table-location=true
fs.native-s3.enabled=true
s3.endpoint=http://host.docker.internal:9000
s3.path-style-access=true
s3.region=us-east-1
s3.aws-access-key=minioadmin
s3.aws-secret-key=minioadmin
```

`catalog/tpch.properties`:
```
connector.name=tpch
tpch.splits-per-node=4
```

После добавления/изменения файлов:
```powershell
docker restart trino
```

## Регистрация warehouse в Lakekeeper

Файл `wh-bronze.json` (для silver/gold — по аналогии, меняется имя и бакет):
```json
{
  "warehouse-name": "bronze",
  "project-id": "00000000-0000-0000-0000-000000000000",
  "storage-profile": {
    "type": "s3",
    "bucket": "bronze",
    "key-prefix": "",
    "endpoint": "http://host.docker.internal:9000",
    "region": "us-east-1",
    "path-style-access": true,
    "flavor": "s3-compat",
    "sts-enabled": false
  },
  "storage-credential": {
    "type": "s3",
    "credential-type": "access-key",
    "access-key-id": "minioadmin",
    "secret-access-key": "minioadmin"
  }
}
```

Bootstrap и регистрация:
```powershell
'{"accept-terms-of-use": true}' | Set-Content -Path bootstrap.json -Encoding ascii
curl.exe -X POST http://localhost:8181/management/v1/bootstrap `
  -H "Content-Type: application/json" `
  -d "@bootstrap.json"

foreach ($name in "bronze","silver","gold") {
  curl.exe -X POST http://localhost:8181/management/v1/warehouse `
    -H "Content-Type: application/json" `
    -d "@wh-$name.json"
}
```

## Бакеты MinIO

Создать через UI (http://localhost:9001, логин/пароль `minioadmin`/`minioadmin`) — три бакета: `bronze`, `silver`, `gold`.

## TPC-H

Доступные масштабы:
```powershell
docker exec trino trino --execute "SHOW SCHEMAS FROM tpch"
```
Доступны: tiny, sf1, sf100, sf300, sf1000, sf3000, sf10000, sf30000, sf100000. `sf0.1` и `sf0.01` отсутствуют.

Проверка объёма:
```powershell
docker exec trino trino --execute "SELECT count(*) FROM tpch.sf1.lineitem"
```
Ожидаемо: `6001215`.

## Бенчмарк сжатия

Смена кодека — в `catalog/bronze.properties`, строка:
```
iceberg.compression-codec=GZIP
```
или
```
iceberg.compression-codec=ZSTD
```

После смены:
```powershell
docker restart trino
Start-Sleep -Seconds 30
```

Затем в DBeaver пересоздать Bronze-таблицы (DROP + CREATE AS SELECT из tpch.sf1.*) — данные перезапишутся в новом кодеке.

Замер размера:
```sql
SELECT SUM(file_size_in_bytes) / 1024.0 / 1024.0 AS mb
FROM bronze.lakehouse."tpch_lineitem$files";

SELECT SUM(file_size_in_bytes) / 1024.0 / 1024.0 AS mb
FROM bronze.lakehouse."tpch_orders$files";
```

Замер времени — см. `sql/04_benchmark.sql`.

Результаты:
| Кодек | lineitem, МБ | orders, МБ | Время Q3, сек |
|---|---|---|---|
| GZIP | 38.2 | 7.3 | 5.227 |
| ZSTD | 38.6 | 6.7 | 5.066 |

## Остановка / запуск

Остановить стек (данные сохранятся):
```powershell
docker stop trino lakekeeper lakehouse-pg lakehouse-minio metabase
```

Запустить обратно:
```powershell
docker start lakehouse-minio lakehouse-pg
Start-Sleep -Seconds 3
docker start lakekeeper
Start-Sleep -Seconds 3
docker start trino metabase
Start-Sleep -Seconds 30
docker ps
```

Полная очистка (удалит данные!):
```powershell
docker rm -f trino lakekeeper lakehouse-pg lakehouse-minio metabase
docker volume rm minio-data pg-data metabase-data
```