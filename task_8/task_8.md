# ДЗ-8

### Задание

1. Сгенерировать таблицу с 1 млн jsonb документов
2. Создать индекс
3. Обновить 1 из полей в json
4. Убедиться в блоатинге TOAST
5. Придумать методы избавится от него и проверить на практике
6. Не забываем про блоатинг индексов* 

### Результат

1. Создаем и заполняем таблицу

    ```sql
    CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    data JSONB
    );

    INSERT INTO documents (data)
    SELECT jsonb_build_object(
        'name', md5(random()::text),
        'age', (random() * 100)::int,
        'city', md5(random()::text)
    )
    FROM generate_series(1, 1000000);

    ```

2. Для ускорения выборок создаем GIN-индекс для jsonb:

    ```sql
    CREATE INDEX idx_data ON documents USING GIN (data);
    ```

3. Обновляем поле age

    ```sql
    UPDATE documents
    SET data = jsonb_set(data, '{age}', to_jsonb((random() * 100)::int))
    WHERE id % 10 = 0;

    ```

4. Убеждаемся в блоатинге TOAST

    ```sql
    SELECT
    tbl.oid::regclass AS table_name,
    tbl.relname AS table,
    toast.relname AS toast_table,
    pg_size_pretty(pg_table_size(tbl.oid)) AS table_size,
    pg_size_pretty(pg_total_relation_size(tbl.oid) - pg_table_size(tbl.oid)) AS toast_size,
    pg_size_pretty(pg_total_relation_size(tbl.oid)) AS total_size
    FROM pg_class tbl
    JOIN pg_class toast ON tbl.reltoastrelid = toast.oid
    WHERE tbl.relname = 'documents';

     table_name |   table   |  toast_table   | table_size | toast_size | total_size
    ------------+-----------+----------------+------------+------------+------------
     documents  | documents | pg_toast_16831 | 156 MB     | 193 MB     | 349 MB
     (1 row)

    -- Как видно размер TOAST составляет 193 MB. Пойдем дальше и убедимся в том, что именно из-за поля data используется TOAST
    
    SELECT avg(pg_column_size(data)) AS avg_column_size
    FROM documents; 
    ----------------------
    112.9901800000000000
    (1 row)

    -- Как видно порог в 2 КБ превышен и мы попадаем в TOAST
    ```
5. Методы, как избавится от блоатинга

    Судя по документации и статьям, можно попробовать следующие подходы:
    
    1. Не создавать индексы GIN индексы, если это возможно т.к. они приводят к блоатингу.
    2. Использовать pg_repack или pg_squeeze

    Теперь проверим на практике эти подходы

    1. Повторим создание таблицы documents и update, но без индекса на поле data и сделаем замеры.
        ```sql
        SELECT
        tbl.oid::regclass AS table_name,
        tbl.relname AS table,
        toast.relname AS toast_table,
        pg_size_pretty(pg_table_size(tbl.oid)) AS table_size,
        pg_size_pretty(pg_total_relation_size(tbl.oid) - pg_table_size(tbl.oid)) AS toast_size,
        pg_size_pretty(pg_total_relation_size(tbl.oid)) AS total_size
        FROM pg_class tbl
        JOIN pg_class toast ON tbl.reltoastrelid = toast.oid
        WHERE tbl.relname = 'documents';
        table_name |   table   |  toast_table   | table_size | toast_size | total_size
        -----------+-----------+----------------+------------+------------+------------
        documents  | documents | pg_toast_16842 | 156 MB     | 21 MB      | 178 MB
        (1 row)

        -- Не создавая индекс мы уменьшили размер toast с 193 MB до 21 MB
        ```
    
    2. Проверим pg_repack в действии
        ```sql
        -- установим
        sudo apt update
        sudo apt install postgresql-15-repack

        -- подключаем в psql
        CREATE EXTENSION IF NOT EXISTS pg_repack;

        -- замеряем до pg_repack;
         table_name |   table   |  toast_table   | table_size | toast_size | total_size
        ------------+-----------+----------------+------------+------------+------------
         documents  | documents | pg_toast_16852 | 156 MB     | 192 MB     | 349 MB
         (1 row)

        ```

        ```bash
        // применяем
        pg_repack postgres --table documents
        ```

        ```sql
         table_name |   table   |  toast_table   | table_size | toast_size | total_size
        ------------+-----------+----------------+------------+------------+------------
         documents  | documents | pg_toast_16852 | 142 MB     | 187 MB     | 330 MB

         -- немного ужалась таблица
        ```