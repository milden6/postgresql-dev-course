# ДЗ-6

### Задание

Развернуть асинхронную реплику и протестировать производительность по сравнению с single инстансом

cо *: переделать под синхронную реплику

### Результат

1. Тестируем с помощью `pgbench` single инстанс

    ```bash
    // TPC-B режим
    pgbench -p 5433 -c 100 -j 2 -T 60 -U postgres -h localhost postgres

    number of transactions actually processed: 53853
    number of failed transactions: 0 (0.000%)
    latency average = 109.892 ms
    initial connection time = 1066.415 ms
    tps = 909.987835 (without initial connection time)

    // select-only режим
    pgbench -p 5433 -c 100 -j 2 -T 60 -S -U postgres -h localhost postgres

    number of transactions actually processed: 841673
    number of failed transactions: 0 (0.000%)
    latency average = 6.998 ms
    initial connection time = 1133.853 ms
    tps = 14288.948859 (without initial connection time)
    ```

2. Создаем рядом кластер `slave`, делаем его асинхронной репликой и тестируем

    ```bash
    // список кластеров
    15  master  5433 online          postgres /var/lib/postgresql/15/master /var/log/postgresql/postgresql-15-master.log

    15  slave   5434 online,recovery postgres /var/lib/postgresql/15/slave  /var/log/postgresql/postgresql-15-slave.log

    // запустим инициализацию pgbench в мастер кластере
    pgbench -p 5433 -i -U postgres -h localhost postgres

    //проверим в slave кластере, что данные появились
    psql -p 5434 -U postgres

    \d
              List of relations
    Schema |       Name       | Type  |  Owner
    --------+------------------+-------+----------
    public | pgbench_accounts | table | postgres
    public | pgbench_branches | table | postgres
    public | pgbench_history  | table | postgres
    public | pgbench_tellers  | table | postgres
    (4 rows)
    ```

    ```sql
    -- проверяем на мастер кластере, что реплика работает
    postgres=# SELECT usename, application_name, client_addr, state, sync_state FROM pg_stat_replication;
    usename  | application_name | client_addr |   state   | sync_state
    ----------+------------------+-------------+-----------+------------
    postgres | 15/slave         | 127.0.0.1   | streaming | async
    (1 row)
    ```
    Запускаем бенчмарк

    ```bash
    // TPC-B режим
    pgbench -p 5433 -c 100 -j 2 -T 60 -U postgres -h localhost postgres

    number of transactions actually processed: 40041
    number of failed transactions: 0 (0.000%)
    latency average = 147.785 ms
    initial connection time = 1092.220 ms
    tps = 676.660945 (without initial connection time)

    // select-only режим
    pgbench -p 5433 -c 100 -j 2 -T 60 -S -U postgres -h localhost postgres

    number of transactions actually processed: 849054
    number of failed transactions: 0 (0.000%)
    latency average = 6.961 ms
    initial connection time = 925.713 ms
    tps = 14364.768175 (without initial connection time)
    ```

3. Переключаем на синхронную репликацию*
    
    ```sql
    postgres=# SELECT usename, application_name, client_addr, state, sync_state FROM pg_stat_replication;
    usename  | application_name | client_addr |   state   | sync_state
    ----------+------------------+-------------+-----------+------------
    postgres | 15/slave         | 127.0.0.1   | streaming | sync
    (1 row)
    ```

    ```bash
    // TPC-B режим
    pgbench -p 5433 -c 100 -j 2 -T 60 -U postgres -h localhost postgres
    
    number of transactions actually processed: 40807
    number of failed transactions: 0 (0.000%)
    latency average = 145.420 ms
    initial connection time = 885.686 ms
    tps = 687.661946 (without initial connection time)

    // select-only режим
    pgbench -p 5433 -c 100 -j 2 -T 60 -S -U postgres -h localhost postgres
    
    number of transactions actually processed: 856555
    number of failed transactions: 0 (0.000%)
    latency average = 6.889 ms
    initial connection time = 1016.769 ms
    tps = 14515.345845 (without initial connection time)
    ```

### Вывод

Производительность падает при включении репликации, при переходе с single инстанса на асинхронную репликацию. При переходе на синхронную репликацию производительность остается почти на уровне асинхронной.