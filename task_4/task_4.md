# ДЗ-4

### Задание

Получить deadlock при работе в Postgres.

### Результат

1. Создание таблицы `accounts` и добавление записей

    ```sql
    postgres=# CREATE TABLE accounts (
        id integer PRIMARY KEY,
        amount numeric
    );
    CREATE TABLE

    postgres=# INSERT INTO accounts (id, amount) VALUES (1, 50), (2, 60);
    INSERT 0 2
    ```

2. Получение `deadlock`

    ```sql
    -- session 1
    postgres=# BEGIN;
    BEGIN

    -- session 2
    postgres=# BEGIN;
    BEGIN

    -- session 1
    postgres=# UPDATE accounts
    SET amount = amount * 2
    WHERE id = 1;
    UPDATE 1

    -- session 2
    postgres=# UPDATE accounts
    SET amount = amount * 2
    WHERE id = 2;
    UPDATE 1

    -- session 1
    postgres=# UPDATE accounts
    SET amount = amount * 4
    WHERE id = 2;

    -- sesson 2 посмотрим, есть ли блокировки (вывел минимум столбцов, чтобы поместилось)
    postgres=*# SELECT pid, usename, state, wait_event_type, wait_event,
    FROM pg_stat_activity
    WHERE wait_event_type = 'Lock';
      pid   | usename  | state  | wait_event_type |  wait_event
    --------+----------+--------+-----------------+---------------
    122977 | postgres | active | Lock            | transactionid
    (1 row)

    postgres=# UPDATE accounts
    SET amount = amount * 4
    WHERE id = 1;
    ERROR:  deadlock detected
    DETAIL:  Process 122965 waits for ShareLock on transaction 946; blocked by process 122977.
    Process 122977 waits for ShareLock on transaction 947; blocked by process 122965.
    HINT:  See server log for query details.
    CONTEXT:  while updating tuple (0,1) in relation "accounts"

    ```

    > Произошел `deadlock` при выполнении UPDATE во второй сессии из-за конфликта между двумя транзакциями.

    Дополнительно посмотрел `/var/log/postgresql/postgresql-15-main.log`
    
    ```shell
    2024-10-14 16:03:53.791 MSK [122965] postgres@postgres ERROR:  deadlock detected
    2024-10-14 16:03:53.791 MSK [122965] postgres@postgres DETAIL:  Process 122965 waits for ShareLock on transaction 946; blocked by process 122977.
            Process 122977 waits for ShareLock on transaction 947; blocked by process 122965.
            Process 122965: UPDATE accounts
            SET amount = amount * 4
            WHERE id = 1;
            Process 122977: UPDATE accounts
                SET amount = amount * 4
                WHERE id = 2;
    2024-10-14 16:03:53.791 MSK [122965] postgres@postgres HINT:  See server log for query details.
    ```