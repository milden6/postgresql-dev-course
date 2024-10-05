# ДЗ-2

### Задание

Проверить на практике работу с уровнями изоляции транзакций и увидеть разницу в их работе с данными из разных пользовательских сессий.

### Результат

1. Создана и заполнена таблица `users`

    ```sql
    postgres=# SELECT * FROM users;

    id |  full_name   | phone_number
    ----+--------------+--------------
    1 | Ryan Gosling | +14245776627
    2 | Jim Gordon   | +13242312354
    3 | Nicolas Cage | +123144432
    (3 rows)
    ```
2. Текущий уровень изоляции

    ```sql
    postgres=# SHOW TRANSACTION ISOLATION LEVEL;

    transaction_isolation
    -----------------------
    read committed
    (1 row)
    ```
3. Работа в двух сессиях (уровень изоляции `read committed`)

    ```sql
    -- session 1
    postgres=# BEGIN;
    BEGIN

    -- session 2
    postgres=# BEGIN;
    BEGIN

    -- session 1
    postgres=*# INSERT INTO users (full_name, phone_number) VALUES ('Margot Robbie', '+127826782363');
    INSERT 0 1
    postgres=*

    -- session 2
    postgres=*# SELECT * FROM users;
    id |  full_name   | phone_number
    ----+--------------+--------------
    1 | Ryan Gosling | +14245776627
    2 | Jim Gordon   | +13242312354
    3 | Nicolas Cage | +123144432
    (3 rows)
    ```

    > Видите ли вы новую запись и если да, то почему?

    Нет, новую запись я не вижу, потому что используется уровень изоляции `read committed` при котором запрос SELECT (без FOR UPDATE/SHARE) видит только те данные, которые были зафиксированны до начала выполнения запроса.

    ```sql
    -- session 1
    postgres=*# COMMIT;
    COMMIT
    postgres=#

    -- session 2
    postgres=*# SELECT * FROM users;
    id |   full_name   | phone_number
    ----+---------------+---------------
    1 | Ryan Gosling  | +14245776627
    2 | Jim Gordon    | +13242312354
    3 | Nicolas Cage  | +123144432
    5 | Margot Robbie | +127826782363
    (4 rows)
    ```

    > Видите ли вы новую запись и если да, то почему?

    Теперь новая запись появилась, потому что произошел COMMIT в первой сессии и данные зафиксировались.

4. Работа в двух сессиях (уровень изоляции `repeatable read`)

    ```sql
    -- session 1
    postgres=# BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
    BEGIN

    -- session 2
    postgres=# BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
    BEGIN

    -- session 1
    postgres=*# INSERT INTO users (full_name, phone_number) VALUES ('Bruce Willis', '+864893937373');
    INSERT 0 1
    postgres=*#

    -- session 2
    postgres=*# SELECT * FROM users;
    id |   full_name   | phone_number
    ----+---------------+---------------
    1 | Ryan Gosling  | +14245776627
    2 | Jim Gordon    | +13242312354
    3 | Nicolas Cage  | +123144432
    5 | Margot Robbie | +127826782363
    (4 rows)

    ```

    > Видите ли вы новую запись и если да, то почему?

    Нет, потому что уровень изоляции `repeatable read` видит только данные, зафиксированные до начала транзакции.

    ```sql
    -- session 1
    postgres=*# COMMIT;
    COMMIT

    -- session 2
    postgres=*# SELECT * FROM users;
    id |   full_name   | phone_number
    ----+---------------+---------------
    1 | Ryan Gosling  | +14245776627
    2 | Jim Gordon    | +13242312354
    3 | Nicolas Cage  | +123144432
    5 | Margot Robbie | +127826782363
    (4 rows)

    ```

    > Видите ли вы новую запись и если да, то почему?

    Новой записи все равно нет, потому что во второй сессии транзакция не завершена, а при уровене изоляции `repeatable read` нельзя увидеть изменений зафиксированных параллельными транзакциями во время выполнения транзакции.