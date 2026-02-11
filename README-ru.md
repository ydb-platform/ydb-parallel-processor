## YDB: параллельный пакетный обработчик записей

[См. страницу релизов для загрузки](https://github.com/zinal/ydb-parallel-processor/releases).

Этот инструмент предоставляет базовые возможности для параллельной пакетной обработки записей,
хранящихся в таблицах [YDB](https://ydb.tech).
Его можно использовать как отдельную консольную утилиту или встраивать в пользовательское
приложение (на стеке, совместимом с Java) как библиотеку.

Базовый рабочий процесс, реализованный инструментом, состоит из двух фаз:

1. Инструмент получает ключи записей, используя основной запрос, с применением базовой фильтрации.
   При необходимости может использоваться постраничный вывод, как [описано в документации YDB](https://ydb.tech/docs/ru/dev/paging).
   Для работы постраничного вывода должен быть настроен дополнительный «paging»-запрос.
2. Инструмент использует собранные ключи записей для дальнейшей обработки, с возможностями:
   * Выполнения соединений таблиц и дополнительных вычислений для обогащения данных
     и вывода полных записей в форматах CSV или JSON.
   * Обновления данных — установки значений в существующих записях или вставки новых записей.

Фаза 2 выполняется с использованием настраиваемого пула исполнителей (workers),
что обеспечивает параллельное выполнение и повышенную производительность.
Это значит, что результирующие записи в общем случае не могут быть отсортированы,
так как вывод из разных параллельных потоков будет поступать в неопределённом порядке.

## Требования

- Java SDK 17 или новее (для запуска и сборки)
- Apache Maven (для сборки)

## Запуск инструмента как отдельной программы

```bash
./Run.sh connection.xml jobdef.xml
... или ...
./Run.sh connection.xml jobdef.xml vars.xml
```

* Первый параметр указывает на файл с параметрами подключения.
* Второй параметр указывает на файл с описанием задания (job definition).
* Необязательный третий параметр может быть опущен, либо указывать на файл
  с подставляемыми переменными.

Дополнительную информацию о подстановочных переменных см. в разделе
в конце этого файла (`README-ru.md`), либо в аналогичном разделе в `README.md`.

## Встраивание инструмента в пользовательскую программу

Для встраивания инструмента в собственное приложение может использоваться зависимость Maven:

```xml
        <dependency>
            <groupId>tech.ydb.app</groupId>
            <artifactId>ydb-parallel-processor</artifactId>
            <version>1.3</version>
        </dependency>
```

> [!WARNING]
> В настоящее время не осуществляется публикация артефактов приложения в Maven Central.
> Для работы с указанной выше зависимостью требуется выполнить локальную сборку.

Используя класс `tech.ydb.app.parproc.Tool`, можно реализовать, например, следующее:

```java
var job = new JobDef();
job.setMainQuery("SELECT ...");
job.setDetailsQuery("SELECT ...");
job.getDetailsInput().add("id");
...
var propsConn = new Properties();
propsConn.setProperty("ydb.url", "grpcs://ydb01.localdomain:2135/cluster1/testdb");
...
try (YdbConnector yc = new YdbConnector(propsConn)) {
    try (Tool app = new Tool(yc, job)) {
        app.run();
    }
}
```

## Параметры подключения

Параметры подключения могут быть заданы программно (в виде объекта `java.util.Properties`)
или через XML‑файл свойств.

| **Параметр** | **Описание** |
| --- | --- |
| `ydb.url` | URL подключения к YDB |
| `ydb.cafile` | Путь к файлу сертификата доверенного УЦ (CA) |
| `ydb.auth.mode` | [Режим аутентификации](https://ydb.tech/docs/ru/reference/ydb-sdk/auth#auth-provider) (NONE, ENV, STATIC, METADATA, SAKEY) |
| `ydb.auth.username` | Имя пользователя для аутентификации STATIC |
| `ydb.auth.password` | Пароль для аутентификации STATIC |
| `ydb.auth.sakey` | Имя файла ключа сервисного аккаунта для аутентификации SAKEY |
| `ydb.preferLocalDc` | Предпочитать локальный или ближайший дата‑центр для соединений (true или false, по умолчанию false) |

Пример файла свойств:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
<entry key="ydb.url">grpcs://ydb01.localdomain:2135/cluster1/testdb</entry>
<entry key="ydb.cafile">/home/ydbadmin/Work/cluster1/tls/ca.crt</entry>
<entry key="ydb.auth.mode">STATIC</entry>
<entry key="ydb.auth.username">root</entry>
<entry key="ydb.auth.password">???</entry>
<entry key="ydb.poolSize">1000</entry>
</properties>
```

## Параметры обработки

Параметры обработки могут быть заданы программно (в виде объекта `tech.ydb.app.parproc.JobDef`)
или через XML‑файл конфигурации.

| **Параметр** | **Описание** |
| --- | --- |
| `worker-count` | Количество параллельных рабочих потоков (workers) |
| `queue-size` | Размер очереди между основным запросом и детализирующим запросом |
| `batch-limit` | Максимальное количество ключей в одном батче для детализирующего запроса |
| `output-format` | Формат вывода (CSV, TSV, JSON, CUSTOM1 — см. ниже). По умолчанию CSV, если не указан |
| `output-file` | Имя выходного файла, значение `-` — вывод в STDOUT. По умолчанию `-`, если не указано |
| `isolation` | Уровень изоляции транзакций (SERIALIZABLE_RW, SNAPSHOT_RO, STALE_RO, ONLINE_RO, ONLINE_INCONSISTENT_RO). По умолчанию SERIALIZABLE_RW |
| `timeout` | Таймаут для `query-main`, `query-page` и `query-detail` в миллисекундах, по умолчанию −1 (без ограничения) |
| `query-main` | Основной запрос (выполняется первым) |
| `query-page` | Постраничный запрос (опционально). Требует сортировки по ключам и ограничения количества строк как для него самого, так и для основного запроса. |
| `query-details` | Детализирующий запрос. Принимает ключи из основного и постраничного запросов и реализует дополнительную логику. |
| `input-page` | Список входных колонок для постраничного запроса (подмножество колонок из основного и постраничного запросов) |
| `input-details` | Список входных колонок для детализирующего запроса (подмножество колонок из основного и постраничного запросов), опционально. Если не указан, используются все колонки |

В тегах `query-main`, `query-page` и `query-details` может быть указан опциональный атрибут `timeout`, устанавливающий предельное время выполнения запроса в миллисекундах. В случае превышения указанного времени выполнение запроса прерывается, и выполняется повторное обращение. Механизм таймаутов позволяет защититься от снижения производительности из-за редких, но продолжительных задержек выполнения отдельных запросов.

Поддерживаемые форматы вывода:
- `CSV` — обычный CSV по RFC4180: разделитель — запятая, кавычки — двойные, разделитель строк — CR‑LF
- `TSV` — таб‑разделённый формат, двойные кавычки, CR‑LF
- `JSON` — отдельный JSON‑документ на строку, разделитель строк — CR
- `CUSTOM1` — CSV‑подобный формат с разделителем полей 0x19, разделителем записей LF (0x0A),
  с минимальным (в большинстве случаев отсутствующим) экранированием кавычками.

Пример файла конфигурации:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ydb-parallel-exporter>
    <worker-count>5</worker-count>
    <queue-size>100</queue-size>
    <batch-limit>1000</batch-limit>
    <output-format>CSV</output-format>
    <output-file>example1.csv</output-file>
    <isolation>SNAPSHOT_RO</isolation>
    <query-main timeout="3000"><![CDATA[
SELECT sys_update_tv, id
FROM my_documents VIEW ix_sys_update_tv
WHERE sys_update_tv >= Timestamp('2021-01-01T00:00:00Z')
  AND sys_update_tv  < Timestamp('2021-01-02T00:00:00Z')
ORDER BY sys_update_tv, id   -- Обязательная сортировка по первичному или вторичному ключу
LIMIT 1000;   -- Обязательное ограничение количества возвращаемых строк
]]> </query-main>

    <!-- Колонки для параметров постраничного запроса -->
    <input-page>
        <column-name>sys_update_tv</column-name>
        <column-name>id</column-name>
    </input-page>
    <query-page timeout="3000"><![CDATA[
DECLARE $input AS Struct<sys_update_tv:Timestamp?, id:Text>;
SELECT sys_update_tv, id
FROM my_documents VIEW ix_sys_update_tv
WHERE (sys_update_tv, id) > ($input.sys_update_tv, $input.id) -- Условие страничной навигации (paging)
  AND sys_update_tv < Timestamp('2021-01-02T00:00:00Z') -- Повтор фильтра из основного запроса
ORDER BY sys_update_tv, id   -- Обязательная сортировка по первичному или вторичному ключу
LIMIT 1000;   -- Обязательное ограничение количества возвращаемых строк
]]> </query-page>
    <input-details>
        <column-name>id</column-name>
    </input-details>
    <query-details timeout="10000"><![CDATA[
DECLARE $input AS List<Struct<id:Text>>;
SELECT
    documents.*,
    d1.attr1 AS d1_attr1,
    d2.attr1 AS d2_attr1
FROM AS_TABLE($input) AS input
INNER JOIN my_documents VIEW PRIMARY KEY AS documents
  ON input.id=documents.id
LEFT JOIN my_dict1 AS d1
  ON d1.key=documents.dict1
LEFT JOIN my_dict2 AS d2
  ON d2.key=documents.dict2
WHERE documents.some_state IN ('ONE'u, 'TWO'u, 'THREE'u, 'FOUR'u)
  AND documents.input_tv IS NOT NULL;
]]> </query-details>
</ydb-parallel-exporter>
```

## Подстановочные переменные

Инструмент может опционально применять подстановочные переменные к XML‑файлу с описанием задания.

Подстановки выполняются после разбора XML‑содержимого (то есть файл должен быть корректным XML до подстановок),
но до его преобразования в объект описания задания.

Переменные могут использоваться в значениях атрибутов, текстовых узлов и содержимом CDATA‑секций
в виде `${varname}`, но не в именах тегов и атрибутов.

Пример файла свойств с подстановочными переменными:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
<entry key="file_name">output_with_some_suffix.csv</entry>
<entry key="worker_count">10</entry>
<entry key="start_time">2020-01-01T00:00:00Z</entry>
<entry key="finish_time">2026-01-01T00:00:00Z</entry>
</properties>
```

## Примеры использования

Инструмент YDB parallel processor может применяться для разных задач обработки данных.
Ниже приведены примеры наиболее типичных сценариев.

### 1. Заполнение нового столбца вычисляемыми данными

Этот пример демонстрирует заполнение нового столбца таблицы данными,
вычисляемыми на основе других столбцов с помощью SQL‑запросов.

Предположим, что в таблицу `some_table` был добавлен новый столбец `new_field`
следующей командой:

```SQL
ALTER TABLE some_table ADD COLUMN new_field Text;
```

Изначально столбец `new_field` содержит только `NULL`, и стоит задача
заполнить его вычисляемыми значениями на основе других полей
(и, при необходимости, данных из других таблиц).

**Файл описания задания (`fill-column.xml`):**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ydb-parallel-exporter>
    <worker-count>10</worker-count>
    <queue-size>100</queue-size>
    <batch-limit>1000</batch-limit>
    <output-format>TSV</output-format>
    <output-file>-</output-file>
    <isolation>SERIALIZABLE_RW</isolation>

    <!-- Выбрать записи, которым требуется заполнить новый столбец -->
    <query-main timeout="120000"><![CDATA[
SELECT id FROM some_table
WHERE new_field IS NULL
ORDER BY id;
]]> </query-main>

    <input-details>
        <column-name>id</column-name>
    </input-details>

    <!-- Обновить новый столбец вычисляемыми данными -->
    <query-details timeout="10000"><![CDATA[
DECLARE $input AS List<Struct<id:Text>>;
UPSERT INTO some_table
  SELECT i.id AS id,
      t.old_field || ' 'u || a.some_name AS new_field
  FROM AS_TABLE($input) AS i
  JOIN some_table AS t
    ON t.id = i.id
  LEFT JOIN another_table AS a
    ON a.id = t.ref_a;
]]> </query-details>
</ydb-parallel-exporter>
```

**Запуск:**

```bash
./Run.sh connection.xml fill-column.xml
```

### 2. Архивирование старых записей в другую таблицу

В этом примере показывается перенос старых записей в архивную таблицу
с последующим удалением их из исходной таблицы.

Предположим, что таблица `documents` должна каждый месяц очищаться от документов,
старше 3 месяцев.
Старые документы должны быть перенесены в таблицу `documents_archive`
и удалены из исходной таблицы `documents`.

**Файл описания задания (`archive-records.xml`):**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ydb-parallel-exporter>
    <worker-count>5</worker-count>
    <queue-size>100</queue-size>
    <batch-limit>1000</batch-limit>
    <output-format>TSV</output-format>
    <output-file>-</output-file>
    <isolation>SERIALIZABLE_RW</isolation>

    <!-- Выбрать записи для архивирования (старше 92 дней, ~3 месяцев) -->
    <query-main timeout="120000"><![CDATA[
SELECT id FROM documents
WHERE created_date < CurrentUtcTimestamp() - Interval('P92D');
]]> </query-main>

    <input-details>
        <column-name>id</column-name>
    </input-details>

    <!-- Сначала записать в архивную таблицу, затем удалить из исходной -->
    <query-details timeout="10000"><![CDATA[
DECLARE $input AS List<Struct<id:Text>>;

-- Сначала вставить в архивную таблицу
UPSERT INTO documents_archive
SELECT d.*
FROM AS_TABLE($input) AS i
JOIN documents AS d ON d.id = i.id;

-- Затем удалить из исходной
DELETE FROM documents
ON SELECT * FROM AS_TABLE($input);
]]> </query-details>
</ydb-parallel-exporter>
```

**Запуск:**

```bash
./Run.sh connection.xml archive-records.xml
```

### 3. Выгрузка больших объёмов данных в CSV‑файлы

Этот пример демонстрирует выгрузку больших наборов данных в CSV‑файлы
с использованием параллельной обработки и постраничного вывода
для повышения производительности.

Предположим, что в нормализованной структуре таблиц хранится очень большой объём данных,
а системе BI требуются денормализованные данные для заполнения витрины.
Каждый день запускается задание, которое выгружает данные за последние два дня
в CSV‑файл в виде широкой таблицы, содержащей все необходимые атрибуты
для заполнения витрины данных.
Файл затем загружается в BI‑систему её штатными средствами.

Параметры задания вынесены в отдельный файл с подстановочными переменными,
который генерируется заново перед каждым запуском.

**Файл подстановочных переменных (`export-to-csv_params.xml`):**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
<entry key="file_name">exported_data.csv</entry>
<entry key="worker_count">10</entry>
<entry key="start_time">2020-01-01T00:00:00Z</entry>
<entry key="finish_time">2026-01-01T00:00:00Z</entry>
</properties>
```

**Файл описания задания (`export-to-csv.xml`):**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ydb-parallel-exporter>
    <worker-count>${worker_count}</worker-count>
    <queue-size>100</queue-size>
    <batch-limit>1000</batch-limit>
    <output-format>CSV</output-format>
    <output-file>${file_name}</output-file>
    <output-encoding>UTF-8</output-encoding>
    <isolation>SNAPSHOT_RO</isolation>

    <!-- Основной запрос с постраничной выборкой для больших наборов данных -->
    <query-main timeout="5000"><![CDATA[
SELECT sys_update_tv, id
FROM my_documents VIEW ix_sys_update_tv
WHERE sys_update_tv >= Timestamp('${start_time}')
  AND sys_update_tv < Timestamp('${finish_time}')
ORDER BY sys_update_tv, id
LIMIT 1000;
]]> </query-main>

    <!-- Постраничный запрос для обработки больших наборов данных -->
    <input-page>
        <column-name>sys_update_tv</column-name>
        <column-name>id</column-name>
    </input-page>
    <query-page timeout="5000"><![CDATA[
DECLARE $input AS Struct<sys_update_tv:Timestamp?, id:Text>;
SELECT sys_update_tv, id
FROM my_documents VIEW ix_sys_update_tv
WHERE (sys_update_tv, id) > ($input.sys_update_tv, $input.id)
  AND sys_update_tv < Timestamp('${finish_time}')
ORDER BY sys_update_tv, id
LIMIT 1000;
]]> </query-page>

    <!-- Детализирующий запрос с джойнами и дополнительными фильтрами -->
    <input-details>
        <column-name>id</column-name>
    </input-details>
    <query-details timeout="5000"><![CDATA[
DECLARE $input AS List<Struct<id:Text>>;
SELECT
    documents.*,
    d1.attr1 AS d1_attr1,
    d2.attr1 AS d2_attr1
FROM AS_TABLE($input) AS input
INNER JOIN my_documents VIEW PRIMARY KEY AS documents
  ON input.id=documents.id
LEFT JOIN my_dict1 AS d1
  ON d1.key=documents.dict1
LEFT JOIN my_dict2 AS d2
  ON d2.key=documents.dict2
WHERE documents.some_state IN ('ONE'u, 'TWO'u, 'THREE'u, 'FOUR'u)
  AND documents.input_tv IS NOT NULL;
]]> </query-details>
</ydb-parallel-exporter>
```

**Запуск:**

```bash
./Run.sh connection.xml export-to-csv.xml export-to-csv_params.xml
```

