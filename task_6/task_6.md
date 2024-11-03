# ДЗ-6

### Задание

1. Развернуть ВМ (Linux) с PostgreSQL
2. Залить Тайские перевозки
https://github.com/aeuge/postgres16book/tree/main/database
3. Проверить скорость выполнения сложного запроса (приложен в конце файла скриптов)
4. Навесить индексы на внешние ключ
5. Проверить, помогли ли индексы на внешние ключи ускориться

### Результат

ВМ с PostgreSQL развернута в предыдущих заданиях, тайские первозки на 60 млн. аналогично. Поэтому эти шаги пропускаем.

1. Выполнение запроса без индексов

    ```sql
    WITH all_place AS (
    SELECT count(s.id) as all_place, s.fkbus as fkbus
    FROM book.seat s
    group by s.fkbus
    ),
    order_place AS (
        SELECT count(t.id) as order_place, t.fkride
        FROM book.tickets t
        group by t.fkride
    )
    SELECT r.id, r.startdate as depart_date, bs.city || ', ' || bs.name as busstation,  
        t.order_place, st.all_place
    FROM book.ride r
    JOIN book.schedule as s
        on r.fkschedule = s.id
    JOIN book.busroute br
        on s.fkroute = br.id
    JOIN book.busstation bs
        on br.fkbusstationfrom = bs.id
    JOIN order_place t
        on t.fkride = r.id
    JOIN all_place st
        on r.fkbus = st.fkbus
    GROUP BY r.id, r.startdate, bs.city || ', ' || bs.name, t.order_place,st.all_place
    ORDER BY r.startdate
    limit 10;

      id | depart_date |       busstation       | order_place | all_place
    -----+-------------+------------------------+-------------+-----------
    1096 | 2000-01-01  | Bankgkok, Suvarnabhumi |          36 |        40
    1495 | 2000-01-01  | Surin, Central         |          38 |        40
     871 | 2000-01-01  | Pattaya, South         |          34 |        40
     744 | 2000-01-01  | Pattaya, North         |          36 |        40
     244 | 2000-01-01  | Bankgkok, Eastern      |          35 |        40
    1077 | 2000-01-01  | Surin, Central         |          31 |        40
     647 | 2000-01-01  | Poipet, Train station  |          34 |        40
     180 | 2000-01-01  | Poipet, Train station  |          35 |        40
     202 | 2000-01-01  | Pattaya, South         |          35 |        40
     536 | 2000-01-01  | Surin, Central         |          37 |        40
    (10 rows)

    -- время выполнения: (очень долго...)
    Time: 180307.584 ms (03:00.308)
    ``` 
2. Ускоряем запрос

    В ходе дебага запроса выяснилось, что больше всего времени уходит на `order_place`. Я попробовал добавить индекс `CREATE INDEX idx_tickets_fkride ON book.tickets (fkride);`, это не дало прироста скорости. Пробовал поиграть с `max_parallel_workers_per_gather` и `work_mem` прирост есть, но не кратный. Я решил создать MATERIALIZED VIEW из `order_place`, если в будущем придется обновлять данные, то можно сделать REFRESH MATERIALIZED VIEW.

    ```sql
    CREATE MATERIALIZED VIEW mv_tickets_ride_count AS
    SELECT
        fkride,
        count(id) AS order_place
    FROM
        book.tickets
    GROUP BY
        fkride;

    -- выполняем запрос
    WITH all_place AS (
    SELECT
        count(s.id) AS all_place,
        s.fkbus AS fkbus
    FROM
        book.seat s
    GROUP BY
        s.fkbus
    )

    SELECT
        r.id,
        r.startdate AS depart_date,
        bs.city || ', ' || bs.name AS busstation,
        t.order_place,
        st.all_place
    FROM
        book.ride r
    JOIN book.schedule AS s
        ON
        r.fkschedule = s.id
    JOIN book.busroute br
        ON
        s.fkroute = br.id
    JOIN book.busstation bs
        ON
        br.fkbusstationfrom = bs.id
    JOIN mv_tickets_ride_count t
        ON
        t.fkride = r.id
    JOIN all_place st
        ON
        r.fkbus = st.fkbus
    GROUP BY
        r.id,
        r.startdate,
        bs.city || ', ' || bs.name,
        t.order_place,
        st.all_place
    ORDER BY
        r.startdate
    LIMIT 10;

      id | depart_date |       busstation       | order_place | all_place
    -----+-------------+------------------------+-------------+-----------
    1096 | 2000-01-01  | Bankgkok, Suvarnabhumi |          36 |        40
    1495 | 2000-01-01  | Surin, Central         |          38 |        40
     871 | 2000-01-01  | Pattaya, South         |          34 |        40
     744 | 2000-01-01  | Pattaya, North         |          36 |        40
     244 | 2000-01-01  | Bankgkok, Eastern      |          35 |        40
    1077 | 2000-01-01  | Surin, Central         |          31 |        40
     647 | 2000-01-01  | Poipet, Train station  |          34 |        40
     180 | 2000-01-01  | Poipet, Train station  |          35 |        40
     202 | 2000-01-01  | Pattaya, South         |          35 |        40
     536 | 2000-01-01  | Surin, Central         |          37 |        40
    (10 rows)

    -- время выполнения:
    Time: 3209.385 ms (00:03.209)
    ```
    > Индекс ускориться не помог, а MATERIALIZED VIEW помог 😄