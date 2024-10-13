# ДЗ-3

### Задание

Проверить на практике работу с autovacuum в Postgres.

### Результат

1. Создана таблица `cinemas` и заполнена сгенерированными данными (1 млн. строк)

    ```sql
    postgres=# CREATE TABLE cinemas (
        id serial PRIMARY KEY,
        name text NOT NULL,
        location text NOT NULL
    );

    postgres=# INSERT INTO cinemas (name, location)
    SELECT 
        'Cinema ' || generate_series(1, 1000000) AS name,
        (ARRAY['New York', 'Los Angeles', 'Chicago', 'Houston', 'Phoenix', 
            'Philadelphia', 'San Antonio', 'San Diego', 'Dallas', 'San Jose'])
        [floor(random() * 10 + 1)::int] AS location;

    ```

2. Размер таблицы после вставки

    ```sql
    postgres=# SELECT pg_size_pretty(pg_total_relation_size('"public"."cinemas"'));
    pg_size_pretty
    ----------------
    79 MB
    (1 row)
    ```

3. Обновление каждой строчки таблицы 5 раз (добавление символов)

    ```sql
    -- напишем для этого анонимную процедуру

    postgres=# DO $$
    DECLARE
        i INTEGER;
    BEGIN
        FOR i IN 1..5 LOOP
            UPDATE cinemas
            SET name = name || chr(trunc(65 + random() * 26)::int) || chr(trunc(65 + random() * 26)::int)
            WHERE true;
            RAISE NOTICE 'Iteration % done', i;
        END LOOP;
    END $$;
    ```

4. Количество мертвых строчек в таблице и когда последний раз приходил автовакуум

    ```sql
    postgres=# SELECT relname, last_autovacuum, n_dead_tup FROM pg_stat_user_tables WHERE relname = 'cinemas';
    relname |        last_autovacuum        | n_dead_tup
    ---------+-------------------------------+------------
    cinemas | 2024-10-13 07:48:09.937245+03 |    5000000
    (1 row)
    ```

    Спустя некоторое время происходит автовакуум

    ```sql
    postgres=# SELECT relname, last_autovacuum, n_dead_tup FROM pg_stat_user_tables WHERE relname = 'cinemas';
    relname |        last_autovacuum        | n_dead_tup
    ---------+-------------------------------+------------
    cinemas | 2024-10-13 08:02:47.500585+03 |          0
    (1 row)
    ```
5. Еще раз делаем обновление каждой строчки таблицы 5 раз (добавление символов) и смотрим размер таблицы

    ``` sql
    -- обновляем
    postgres=# DO $$
    DECLARE
        i INTEGER;
    BEGIN
        FOR i IN 1..5 LOOP
            UPDATE cinemas
            SET name = name || chr(trunc(65 + random() * 26)::int) || chr(trunc(65 + random() * 26)::int)
            WHERE true;
            RAISE NOTICE 'Iteration % done', i;
        END LOOP;
    END $$;

    -- проверяем размер
    postgres=# SELECT pg_size_pretty(pg_total_relation_size('"public"."cinemas"'));
    pg_size_pretty
    ----------------
    569 MB
    (1 row)

    ```
6. Выключение `autovacuum` для таблицы cinemas
    
    ```sql
    postgres=# ALTER TABLE cinemas SET (autovacuum_enabled = off);
    ```

7. Обновление данных в таблице cinemas 10 раз, после отключения `autovacuum` и вывод размера таблицы

    ```sql
    -- обновляем
    postgres=# DO $$
    DECLARE
        i INTEGER;
    BEGIN
        FOR i IN 1..10 LOOP
            UPDATE cinemas
            SET name = name || chr(trunc(65 + random() * 26)::int) || chr(trunc(65 + random() * 26)::int)
            WHERE true;
            RAISE NOTICE 'Iteration % done', i;
        END LOOP;
    END $$;

    -- проверяем размер
    postgres=# SELECT pg_size_pretty(pg_total_relation_size('"public"."cinemas"'));
    pg_size_pretty
    ----------------
    1176 MB
    (1 row)

    ```

    > После отключения `autovacuum` и выполнения UPDATE "мертвые" строки не были очищены, поэтому размер таблицы сильно вырос.

8. Включение `autovacuum`

    ```sql
    postgres=# ALTER TABLE cinemas SET (autovacuum_enabled = on);
    ```

9. Задание со звездочкой
    
    > Написать анонимную процедуру, в которой в цикле 10 раз обновятся все строчки в искомой таблице и вывести номер шага цикла

    ``` sql
    -- выше эта процедура написана и использована в рамках выполенения задания, но на всякий случай приведена отдельно
    DO $$
    DECLARE
        i INTEGER;
    BEGIN
        FOR i IN 1..10 LOOP
            UPDATE cinemas
            SET name = name || chr(trunc(65 + random() * 26)::int) || chr(trunc(65 + random() * 26)::int)
            WHERE true;
            RAISE NOTICE 'Iteration % done', i;
        END LOOP;
    END $$;
    ```