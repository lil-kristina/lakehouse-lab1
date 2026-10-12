# Практическая №1 — Вариант 1 - Q3 «Shipping Priority»

**ФИО:** Седова Кристина Александровна
**Вариант:** Q3 «Shipping Priority»
**Масштаб:** sf0.1

## Что сделано
- Развёрнут Lakehouse: MinIO → Lakekeeper → Trino
- Подготовлен каталог `tpch` в Trino для генерации тестовых данных
- ...

## Стек
| Компонент | Роль | Почему |
|---|---|---|
| MinIO | S3-хранилище | Open-source, S3-совместимое API |
| PostgreSQL | Метаданные Lakekeeper | Хранит каталог таблиц |
| Lakekeeper | Iceberg REST Catalog | Open-source REST-каталог |
| Trino | SQL-движок | Быстрый, поддерживает Iceberg |
| Superset | BI | Официальный коннектор к Trino |

## Структура репозитория
- `отчет.md` — отчёт по форме
- `commands.md` — все команды терминала
- `sql/` — SQL-скрипты
- `catalog/` — конфиги Trino
