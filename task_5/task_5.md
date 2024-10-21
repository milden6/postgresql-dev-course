# ДЗ-5

### Задание

Протестировать падение производительности при использовании
pgbouncer в разных режимах: statement, transaction, session

### Результат

1. Запустим бенчмарк на стандартном порту без `pgbouncer` для интереса. 100 клиентов в 16 потоков, тест будет продолжаться 60 секунд.

    ```bash
    // подключение напрямую
    pgbench -p 5432 -c 100 -j 16 -T 60 -U postgres -h localhost postgres

    number of transactions actually processed: 52720
    number of failed transactions: 0 (0.000%)
    latency average = 112.301 ms
    initial connection time = 1055.610 ms
    tps = 890.461858 (without initial connection time)

    // подключение через pgbouncer
    pgbench -p 6432 -c 100 -j 16 -T 60 -U postgres -h localhost postgres

    number of transactions actually processed: 83497
    number of failed transactions: 0 (0.000%)
    latency average = 71.917 ms
    initial connection time = 34.125 ms
    tps = 1390.492152 (without initial connection time)
    ```
    Видно, что производительность запросов с `pgbouncer` возросла:
    - Количество выполненных транзакций больше - 83 497 vs 52 720
    - Средняя задержка меньше - 71.917 ms vs 112.301 ms
    - Количество операций в секунду выше - 1390 vs 890

2. Теперь запустим бенчмарк добавив флаг -S чтобы pgbench выполнял только запросы на выборку, иначе будет ошибка в режиме statement - `FATAL:  transaction blocks not allowed in statement pooling mode`. И будем менять настройку `pool_mode`. После изменения настройки делаем ` sudo systemctl restart pgbouncer`

    ```bash
    // session
    number of transactions actually processed: 1040826
    number of failed transactions: 0 (0.000%)
    latency average = 5.764 ms
    initial connection time = 33.831 ms
    tps = 17350.181971 (without initial connection time)

    // transaction
    number of transactions actually processed: 963708
    number of failed transactions: 0 (0.000%)
    latency average = 6.224 ms
    initial connection time = 27.941 ms
    tps = 16066.502130 (without initial connection time)

    // statement
    number of transactions actually processed: 957963
    number of failed transactions: 0 (0.000%)
    latency average = 6.262 ms
    initial connection time = 29.951 ms
    tps = 15970.565145 (without initial connection time)
    ```

    Получился такой результат:

    | Режим        | Кол-во транз. | Средняя задержка | Операций в сек. |
    | ------------ | ------------- | ---------------- | --------------- |
    | session      | 1 040 826     | 5.764 ms         | 17350           |
    | transaction  | 963 708       | 6.224 ms         | 16066           |
    | statement    | 957 963       | 6.262 ms         | 15970           |

### Вывод

Режим session обеспечивает наибольшую производительность и минимальную задержку в рамках данного бенчмарка. Режим transaction показал середину, а statement самые низкие показатели.