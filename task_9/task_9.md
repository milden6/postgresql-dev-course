# ДЗ-9

### Задание

1. Проанализировать данные о зарплатах сотрудников с использованием оконных функций (https://www.db-fiddle.com/f/eQ8zNtAFY88i8nB4GRB65V/0)
    а) На сколько было увеличение с предыдущей зарплатой
    б) если это первая зарплата - вместо NULL вывести 0

### Результат

1. Выполняем код из скрипта https://www.db-fiddle.com/f/eQ8zNtAFY88i8nB4GRB65V/0

    ``` sql
    -- весь скрипт вставлять не буду, но вставлю результат выполнения
    CREATE TYPE
    CREATE TABLE
    INSERT 0 4
    CREATE TABLE
    INSERT 0 3
    CREATE TABLE
    INSERT 0 5
    CREATE TABLE
    INSERT 0 4
    CREATE TABLE
    INSERT 0 4
    CREATE TABLE
    INSERT 0 2
    ```

2. Выводим данные о зарплатах сотрудников с использованием оконных функций
    
    ```sql
    SELECT fk_employee, from_date, amount,
    COALESCE(amount - LAG(amount) OVER (PARTITION BY fk_employee ORDER BY from_date), 0) AS salary_increase
    FROM salary
    ORDER BY fk_employee, from_date

    +-------------+------------+--------+-----------------+
    | fk_employee | from_date  | amount | salary_increase |
    +-------------+------------+--------+-----------------+
    |           1 | 2024-01-01 | 100000 |               0 |
    |           1 | 2024-02-01 | 200000 |          100000 |
    |           1 | 2024-03-01 | 300000 |          100000 |
    |           2 | 2023-01-01 | 200000 |               0 |
    |           3 | 2024-03-01 | 200000 |               0 |
    +-------------+------------+--------+-----------------+
    (5 rows)
    ```

    - LAG: позволяет получить значение зарплаты из предыдущей строки в пределах каждой группы сотрудников, организованных по полю fk_employee.
    - COALESCE: позволяет заменить NULL значением 0 для первой записи зарплаты каждого сотрудника, где нет предыдущей записи.