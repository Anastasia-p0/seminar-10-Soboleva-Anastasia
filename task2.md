# Задание 2: Специальные случаи использования индексов

# Партиционирование и специальные случаи использования индексов

1. Удалите прошлый инстанс PostgreSQL - `docker-compose down` в папке `src` и запустите новый: `docker-compose up -d`.

2. Создайте партиционированную таблицу и заполните её данными:

    ```sql
    -- Создание партиционированной таблицы
    CREATE TABLE t_books_part (
        book_id     INTEGER      NOT NULL,
        title       VARCHAR(100) NOT NULL,
        category    VARCHAR(30),
        author      VARCHAR(100) NOT NULL,
        is_active   BOOLEAN      NOT NULL
    ) PARTITION BY RANGE (book_id);

    -- Создание партиций
    CREATE TABLE t_books_part_1 PARTITION OF t_books_part
        FOR VALUES FROM (MINVALUE) TO (50000);

    CREATE TABLE t_books_part_2 PARTITION OF t_books_part
        FOR VALUES FROM (50000) TO (100000);

    CREATE TABLE t_books_part_3 PARTITION OF t_books_part
        FOR VALUES FROM (100000) TO (MAXVALUE);

    -- Копирование данных из t_books
    INSERT INTO t_books_part 
    SELECT * FROM t_books;
    ```

3. Обновите статистику таблиц:
   ```sql
   ANALYZE t_books;
   ANALYZE t_books_part;
   ```
   
   *Результат:*
   ```sql
   ANALYZE
   Query returned successfully in 1 secs 538 msec.
   ```

5. Выполните запрос для поиска книги с id = 18:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part WHERE book_id = 18;
   ```
   
   *План выполнения:*
   ```sql
   "Seq Scan on t_books_part_1 t_books_part  (cost=0.00..1032.99 rows=1 width=32) (actual time=0.016..4.997 rows=1 loops=1)"
   "  Filter: (book_id = 18)"
   "  Rows Removed by Filter: 49998"
   "Planning Time: 0.118 ms"
   "Execution Time: 5.018 ms"
   ```
   
   *Объясните результат:*
   Скорость выполнения запроса выше, чем без партиционирования, так как поиск выполняется не по всей таблице, а только в одной её части

6. Выполните поиск по названию книги:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part 
   WHERE title = 'Expert PostgreSQL Architecture';
   ```
   
   *План выполнения:*
   ```sql
   "Append  (cost=0.00..3101.01 rows=3 width=33) (actual time=5.791..17.124 rows=1 loops=1)"
   "  ->  Seq Scan on t_books_part_1  (cost=0.00..1032.99 rows=1 width=32) (actual time=5.790..5.791 rows=1 loops=1)"
   "        Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
   "        Rows Removed by Filter: 49998"
   "  ->  Seq Scan on t_books_part_2  (cost=0.00..1034.00 rows=1 width=33) (actual time=6.250..6.250 rows=0 loops=1)"
   "        Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
   "        Rows Removed by Filter: 50000"
   "  ->  Seq Scan on t_books_part_3  (cost=0.00..1034.01 rows=1 width=34) (actual time=5.074..5.074 rows=0 loops=1)"
   "        Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
   "        Rows Removed by Filter: 50001"
   "Planning Time: 0.168 ms"
   "Execution Time: 17.164 ms"
   ```
   
   *Объясните результат:*
   Скорость выполнения такая же, как и без партиций, так как таблица разбивается по book_id, а запрос выбирает по title, поэтому разделение не улучшает поиск

7. Создайте партиционированный индекс:
   ```sql
   CREATE INDEX ON t_books_part(title);
   ```
   
   *Результат:*
   ```sql
   CREATE INDEX
   Query returned successfully in 537 msec.
   ```

8. Повторите запрос из шага 5:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part 
   WHERE title = 'Expert PostgreSQL Architecture';
   ```
   
   *План выполнения:*
   ```sql
   "Append  (cost=0.29..24.94 rows=3 width=33) (actual time=0.036..0.087 rows=1 loops=1)"
   "  ->  Index Scan using t_books_part_1_title_idx on t_books_part_1  (cost=0.29..8.31 rows=1 width=32) (actual time=0.035..0.036         rows=1 loops=1)"
   "        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
   "  ->  Index Scan using t_books_part_2_title_idx on t_books_part_2  (cost=0.29..8.31 rows=1 width=33) (actual time=0.028..0.028         rows=0 loops=1)"
   "        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
   "  ->  Index Scan using t_books_part_3_title_idx on t_books_part_3  (cost=0.29..8.31 rows=1 width=34) (actual time=0.020..0.020         rows=0 loops=1)"
   "        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
   "Planning Time: 0.754 ms"
   "Execution Time: 0.117 ms"
   ```
   
   *Объясните результат:*
   Используется созданный индекс на title, который сильно ускорил скорость выполнения запроса

9. Удалите созданный индекс:
   ```sql
   DROP INDEX t_books_part_title_idx;
   ```
   
   *Результат:*
   ```sql
   DROP INDEX
   Query returned successfully in 125 msec.
   ```

10. Создайте индекс для каждой партиции:
   ```sql
   CREATE INDEX ON t_books_part_1(title);
   CREATE INDEX ON t_books_part_2(title);
   CREATE INDEX ON t_books_part_3(title);
   ```
   
   *Результат:*
   ```sql
   CREATE INDEX
   Query returned successfully in 507 msec.
   ```

11. Повторите запрос из шага 5:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books_part 
    WHERE title = 'Expert PostgreSQL Architecture';
    ```
    
    *План выполнения:*
    ```sql
    "Append  (cost=0.29..24.94 rows=3 width=33) (actual time=0.057..0.086 rows=1 loops=1)"
    "  ->  Index Scan using t_books_part_1_title_idx on t_books_part_1  (cost=0.29..8.31 rows=1 width=32) (actual time=0.056..0.058         rows=1 loops=1)"
    "        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
    "  ->  Index Scan using t_books_part_2_title_idx on t_books_part_2  (cost=0.29..8.31 rows=1 width=33) (actual time=0.013..0.013         rows=0 loops=1)"
    "        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
    "  ->  Index Scan using t_books_part_3_title_idx on t_books_part_3  (cost=0.29..8.31 rows=1 width=34) (actual time=0.012..0.012         rows=0 loops=1)"
    "        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
    "Planning Time: 0.294 ms"
    "Execution Time: 0.137 ms"
    ```
    
    *Объясните результат:*
    Скорость выполнения не изменилась по сравнению с предыдущим случаем. Отдельный индекс для каждой партиции не увеличивает скорость выполнения запроса

12. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_part_1_title_idx;
    DROP INDEX t_books_part_2_title_idx;
    DROP INDEX t_books_part_3_title_idx;
    ```
    
    *Результат:*
    ```sql
    DROP INDEX
    Query returned successfully in 114 msec.
    ```

13. Создайте обычный индекс по book_id:
    ```sql
    CREATE INDEX t_books_part_idx ON t_books_part(book_id);
    ```
    
    *Результат:*
    ```sql
    CREATE INDEX
    Query returned successfully in 222 msec.
    ```

14. Выполните поиск по book_id:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books_part WHERE book_id = 11011;
    ```
    
    *План выполнения:*
    ```sql
    "Index Scan using t_books_part_1_book_id_idx on t_books_part_1 t_books_part  (cost=0.29..8.31 rows=1 width=32) (actual                  time=0.030..0.031 rows=1 loops=1)"
    "  Index Cond: (book_id = 11011)"
    "Planning Time: 0.612 ms"
    "Execution Time: 0.053 ms"
    ```
    
    *Объясните результат:*
    Созданный  индекс увеличил скорость выполнения запроса

15. Создайте индекс по полю is_active:
    ```sql
    CREATE INDEX t_books_active_idx ON t_books(is_active);
    ```
    
    *Результат:*
    ```sql
    CREATE INDEX
    Query returned successfully in 239 msec.
    ```

16. Выполните поиск активных книг с отключенным последовательным сканированием:
    ```sql
    SET enable_seqscan = off;
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE is_active = true;
    SET enable_seqscan = on;
    ```
    
    *План выполнения:*
    ```sql
    "Bitmap Heap Scan on t_books  (cost=852.52..2891.52 rows=75900 width=33) (actual time=2.691..18.158 rows=75405 loops=1)"
    "  Recheck Cond: is_active"
    "  Heap Blocks: exact=1225"
    "  ->  Bitmap Index Scan on t_books_active_idx  (cost=0.00..833.54 rows=75900 width=0) (actual time=2.478..2.479 rows=75405             loops=1)"
    "        Index Cond: (is_active = true)"
    "Planning Time: 0.101 ms"
    "Execution Time: 22.606 ms"
    ```
    
    *Объясните результат:*
    Отключение последовательного сканирования заставило планировщика применить индекс, что на самом деле не увеличило скорость выполнения запроса

17. Создайте составной индекс:
    ```sql
    CREATE INDEX t_books_author_title_index ON t_books(author, title);
    ```
    
    *Результат:*
    ```sql
    CREATE INDEX
    Query returned successfully in 2 secs 744 msec.
    ```

19. Найдите максимальное название для каждого автора:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, MAX(title) 
    FROM t_books 
    GROUP BY author;
    ```
    
    *План выполнения:*
    ```sql
    "HashAggregate  (cost=3530.00..3540.00 rows=1000 width=42) (actual time=102.055..102.229 rows=1003 loops=1)"
    "  Group Key: author"
    "  Batches: 1  Memory Usage: 193kB"
    "  ->  Seq Scan on t_books  (cost=0.00..2780.00 rows=150000 width=21) (actual time=0.010..14.742 rows=150000 loops=1)"
    "Planning Time: 0.699 ms"
    "Execution Time: 102.348 ms"
    ```
    
    *Объясните результат:*
    Индекс не используется, так как не ускоряет выполнение запроса. Не смотря на встроенную сортировку, он не улучшает поиск MAX(title)

20. Выберите первых 10 авторов:
    ```sql
    EXPLAIN ANALYZE
    SELECT DISTINCT author 
    FROM t_books 
    ORDER BY author 
    LIMIT 10;
    ```
    
    *План выполнения:*
    ```sql
    "Limit  (cost=0.42..58.71 rows=10 width=10) (actual time=0.146..0.495 rows=10 loops=1)"
    "  ->  Result  (cost=0.42..5829.42 rows=1000 width=10) (actual time=0.144..0.492 rows=10 loops=1)"
    "        ->  Unique  (cost=0.42..5829.42 rows=1000 width=10) (actual time=0.143..0.489 rows=10 loops=1)"
    "              ->  Index Only Scan using t_books_author_title_index on t_books  (cost=0.42..5454.42 rows=150000 width=10) (actual       time=0.141..0.367 rows=1341 loops=1)"
    "                    Heap Fetches: 0"
    "Planning Time: 0.180 ms"
    "Execution Time: 0.521 ms"
    
    
    *Объясните результат:*
    Cкорость выполнения запроса выросла, так как был задействован составной индекс. В данном слуае он полезен, так как включает сортировку author, а это подразумевает запрос

21. Выполните поиск и сортировку:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE author LIKE 'T%'
    ORDER BY author, title;
    ```
    
    *План выполнения:*
    ```sql
    "Sort  (cost=3155.29..3155.33 rows=15 width=21) (actual time=21.089..21.091 rows=1 loops=1)"
    "  Sort Key: author, title"
    "  Sort Method: quicksort  Memory: 25kB"
    "  ->  Seq Scan on t_books  (cost=0.00..3155.00 rows=15 width=21) (actual time=21.044..21.076 rows=1 loops=1)"
    "        Filter: ((author)::text ~~ 'T%'::text)"
    "        Rows Removed by Filter: 149999"
    "Planning Time: 0.115 ms"
    "Execution Time: 21.116 ms"
    ```
    
    *Объясните результат:*
    Планировщик не использует индекс для запроса с LIKE, ему дешевле сканировать построчно

22. Добавьте новую книгу:
    ```sql
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150001, 'Cookbook', 'Mr. Hide', NULL, true);
    COMMIT;
    ```
    
    *Результат:*
    ```sql
    INSERT 0 1
    Query returned successfully in 170 msec.
    COMMIT
    Query returned successfully in 188 msec.
    ```

23. Создайте индекс по категории:
    ```sql
    CREATE INDEX t_books_cat_idx ON t_books(category);
    ```
    
    *Результат:*
    ```sql
    CREATE INDEX
    Query returned successfully in 229 msec.
    ```

24. Найдите книги без категории:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE category IS NULL;
    ```
    
    *План выполнения:*
    ```sql
    "Index Scan using t_books_cat_idx on t_books  (cost=0.29..8.14 rows=1 width=21) (actual time=0.029..0.031 rows=3 loops=1)"
    "  Index Cond: (category IS NULL)"
    "Planning Time: 0.278 ms"
    "Execution Time: 0.046 ms"
    ```
    
    *Объясните результат:*
    [Ваше объяснение]

25. Создайте частичные индексы:
    ```sql
    DROP INDEX t_books_cat_idx;
    CREATE INDEX t_books_cat_null_idx ON t_books(category) WHERE category IS NULL;
    ```
    
    *Результат:*
    ```sql
    CREATE INDEX
    Query returned successfully in 169 msec.
    ```

26. Повторите запрос из шага 22:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE category IS NULL;
    ```
    
    *План выполнения:*
    ```sql
    "Index Scan using t_books_cat_null_idx on t_books  (cost=0.13..7.97 rows=1 width=21) (actual time=0.019..0.020 rows=3 loops=1)"
    "Planning Time: 0.561 ms"
    "Execution Time: 0.041 ms"
    ```
    
    *Объясните результат:*
    [Ваше объяснение]

27. Создайте частичный уникальный индекс:
    ```sql
    CREATE UNIQUE INDEX t_books_selective_unique_idx 
    ON t_books(title) 
    WHERE category = 'Science';
    
    -- Протестируйте его
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150002, 'Unique Science Book', 'Author 1', 'Science', true);
    
    -- Попробуйте вставить дубликат
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150003, 'Unique Science Book', 'Author 2', 'Science', true);
    
    -- Но можно вставить такое же название для другой категории
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150004, 'Unique Science Book', 'Author 3', 'History', true);
    ```
    
    *Результат:*
    ```sql
    CREATE INDEX
    Query returned successfully in 276 msec.
    ```
    ```sql
    INSERT 0 1
    Query returned successfully in 160 msec.
    ```
    ```sql
    "Bitmap Heap Scan on t_books  (cost=625.16..2278.68 rows=29881 width=21) (actual time=2.145..13.384 rows=29893 loops=1)"
    "  Recheck Cond: ((category)::text = 'Science'::text)"
    "  Heap Blocks: exact=1225"
    "  ->  Bitmap Index Scan on t_books_selective_unique_idx  (cost=0.00..617.69 rows=29881 width=0) (actual time=1.901..1.902             rows=29893 loops=1)"
    "Planning Time: 0.181 ms"
    "Execution Time: 22.105 ms"
    ```
    ```sql
    ERROR:  Key (title)=(Unique Science Book) already exists.duplicate key value violates unique constraint                                 "t_books_selective_unique_idx"
    ERROR:  duplicate key value violates unique constraint "t_books_selective_unique_idx"
    SQL state: 23505
    Detail: Key (title)=(Unique Science Book) already exists.
    ```
    ```sql
    INSERT 0 1
    Query returned successfully in 233 msec.
    ```
    ```sql
    "Bitmap Heap Scan on t_books  (cost=625.16..2278.68 rows=29881 width=21) (actual time=1.780..14.032 rows=29893 loops=1)"
    "  Recheck Cond: ((category)::text = 'Science'::text)"
    "  Heap Blocks: exact=1225"
    "  ->  Bitmap Index Scan on t_books_selective_unique_idx  (cost=0.00..617.69 rows=29881 width=0) (actual time=1.616..1.617             rows=29893 loops=1)"
    "Planning Time: 0.121 ms"
    "Execution Time: 15.633 ms"
    ```
    
    *Объясните результат:*
    [Ваше объяснение]
