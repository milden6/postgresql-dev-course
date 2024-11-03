# ДЗ-7

### Задание

1. Создать таблицу с продажами.
2. Реализовать функцию выбор трети года (1-4 месяц - первая треть, 5-8 - вторая и тд)
а. через case
b. * (бонуса в виде зачета дз не будет) используя математическую операцию (лучше 2+ варианта)
с. предусмотреть NULL на входе
3. Вызвать эту функцию в SELECT из таблицы с продажами, уведиться, что всё отработало

### Результат

1. Создаем и заполняем таблицу с продажами

    ```sql
    CREATE TABLE sales (
    id SERIAL PRIMARY KEY,
    sale_date DATE
    );

    -- Заполняем таблицу 1 млн записей
    INSERT INTO sales (sale_date)
    SELECT
        DATE '2024-01-01' + (random() * 365)::INT AS sale_date
    FROM
        generate_series(1, 1000000);
    ```
2. Создаем функцию определения трети года через CASE

    ```sql
    CREATE OR REPLACE FUNCTION get_third_of_year_case(month INTEGER)
    RETURNS INTEGER AS $$
    BEGIN
        RETURN CASE
            WHEN month IS NULL THEN NULL
            WHEN month BETWEEN 1 AND 4 THEN 1
            WHEN month BETWEEN 5 AND 8 THEN 2
            WHEN month BETWEEN 9 AND 12 THEN 3
            ELSE NULL
        END;
    END;
    $$ LANGUAGE plpgsql;

    -- проверяем
    SELECT id, sale_date, EXTRACT(MONTH FROM sale_date) AS month,
    get_third_of_year_case(EXTRACT(MONTH FROM sale_date)::INTEGER) AS third_of_year
    FROM sales
    LIMIT 10;

     id | sale_date  | month | third_of_year
    ----+------------+-------+---------------
      1 | 2024-01-09 |     1 |             1
      2 | 2024-08-13 |     8 |             2
      3 | 2024-07-22 |     7 |             2
      4 | 2024-09-27 |     9 |             3
      5 | 2024-04-01 |     4 |             1
      6 | 2024-12-25 |    12 |             3
      7 | 2024-05-28 |     5 |             2
      8 | 2024-02-13 |     2 |             1
      9 | 2024-04-06 |     4 |             1
     10 | 2024-02-24 |     2 |             1
    (10 rows)
    ```
3. Функция определения трети года через мат. операцию (Вариант 1: целочисленное деление и добавление единицы для корректировки смещения)

    ```sql
    CREATE OR REPLACE FUNCTION get_third_of_year_math_1(month INTEGER)
    RETURNS INTEGER AS $$
    BEGIN
        RETURN CASE
            WHEN month IS NULL THEN NULL
            ELSE ((month - 1) / 4) + 1
        END;
    END;
    $$ LANGUAGE plpgsql;

    -- проверяем
    SELECT id, sale_date, EXTRACT(MONTH FROM sale_date) AS month,
    get_third_of_year_math_1(EXTRACT(MONTH FROM sale_date)::INTEGER) AS third_of_year
    FROM sales
    LIMIT 10;

     id | sale_date  | month | third_of_year
    ----+------------+-------+---------------
      1 | 2024-01-09 |     1 |             1
      2 | 2024-08-13 |     8 |             2
      3 | 2024-07-22 |     7 |             2
      4 | 2024-09-27 |     9 |             3
      5 | 2024-04-01 |     4 |             1
      6 | 2024-12-25 |    12 |             3
      7 | 2024-05-28 |     5 |             2
      8 | 2024-02-13 |     2 |             1
      9 | 2024-04-06 |     4 |             1
     10 | 2024-02-24 |     2 |             1
    (10 rows)
    ```

4. Функция определения трети года через мат. операцию (Вариант 2: деление месяца на 4 и применение CEIL для округления вверх)

    ```sql
    CREATE OR REPLACE FUNCTION get_third_of_year_math_2(month INTEGER)
    RETURNS INTEGER AS $$
    BEGIN
        RETURN CASE
            WHEN month IS NULL THEN NULL
            ELSE CEIL(month / 4.0)
        END;
    END;
    $$ LANGUAGE plpgsql;

    -- проверяем
    SELECT id, sale_date, EXTRACT(MONTH FROM sale_date) AS month,
    get_third_of_year_math_2(EXTRACT(MONTH FROM sale_date)::INTEGER) AS third_of_year
    FROM sales
    LIMIT 10;

     id | sale_date  | month | third_of_year
    ----+------------+-------+---------------
      1 | 2024-01-09 |     1 |             1
      2 | 2024-08-13 |     8 |             2
      3 | 2024-07-22 |     7 |             2
      4 | 2024-09-27 |     9 |             3
      5 | 2024-04-01 |     4 |             1
      6 | 2024-12-25 |    12 |             3
      7 | 2024-05-28 |     5 |             2
      8 | 2024-02-13 |     2 |             1
      9 | 2024-04-06 |     4 |             1
     10 | 2024-02-24 |     2 |             1
    (10 rows)
    ```