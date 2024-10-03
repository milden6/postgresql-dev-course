# ДЗ-1

### Задание
1. Развернуть ВМ (Linux) с PostgreSQL

2. Залить Тайские перевозки
https://github.com/aeuge/postgres16book/tree/main/database

3. Посчитать количество поездок - select count(*) from book.tickets;

### Результат

1. Установка БД согласно [официальной документации](https://www.postgresql.org/download/linux/debian/)
    
    ```bash
    sudo apt install postgresql
    ```
    Проверка, что сервис запущен работу:
    ```bash
    systemctl status postgresql.service    
    ```

2. Скачивание и загрузка Тайских перевозок объемом `60 млн.` строк 

    ```bash
    wget https://storage.googleapis.com/thaibus/thai_medium.tar.gz 
    
    tar -xf thai_medium.tar.gz
    
    psql < thai.sql
    ```

3. Подсчет количества поездок
    
    ```sql
    select count(*) from book.tickets;
    ```

    Результат

    ```sql
     count
    ----------
     53997475
    (1 row)
    ```