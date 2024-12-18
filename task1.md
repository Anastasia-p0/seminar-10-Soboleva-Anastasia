# Задание 1. B-tree индексы в PostgreSQL

1. Запустите БД через docker compose в ./src/docker-compose.yml:

2. Выполните запрос для поиска книги с названием 'Oracle Core' и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE title = 'Oracle Core';
   ```
   
   *План выполнения:*
```sql
"Seq Scan on t_books  (cost=0.00..3155.00 rows=1 width=33) (actual time=15.170..15.204 rows=1 loops=1)"
"  Filter: ((title)::text = 'Oracle Core'::text)"
"  Rows Removed by Filter: 149999"
"Planning Time: 0.080 ms"
"Execution Time: 15.225 ms"
   ```
   *Объясните результат:*
   не использованы индексы, время выполнения составило 15.225 ms.

3. Создайте B-tree индексы:
   ```sql
   CREATE INDEX t_books_title_idx ON t_books(title);
   CREATE INDEX t_books_active_idx ON t_books(is_active);
   ```
   
   *Результат:*
   ```sql
   CREATE INDEX
   Query returned successfully in 716 msec.
   ```

5. Проверьте информацию о созданных индексах:
   ```sql
   SELECT schemaname, tablename, indexname, indexdef
   FROM pg_catalog.pg_indexes
   WHERE tablename = 't_books';
   ```
   
   *Результат:*
   ```sql
   "soboleva"	"t_books"	"t_books_title_idx"	"CREATE INDEX t_books_title_idx ON soboleva.t_books USING btree (title)"
   "soboleva"	"t_books"	"t_books_active_idx"	"CREATE INDEX t_books_active_idx ON soboleva.t_books USING btree (is_active)"
   ```
   
   *Объясните результат:*
   Индексы созданы

6. Обновите статистику таблицы:
   ```sql
   ANALYZE t_books;
   ```
   
   *Результат:*
   ```sql
   ANALYZE
   Query returned successfully in 435 msec.;
   ```

7. Выполните запрос для поиска книги 'Oracle Core' и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE title = 'Oracle Core';
   ```
   
   *План выполнения:*
   ```sql
   "Index Scan using t_books_title_idx on t_books  (cost=0.42..8.44 rows=1
   width=33) (actual time=0.045..0.046 rows=1 loops=1)"
   "  Index Cond: ((title)::text = 'Oracle Core'::text)"
   "Planning Time: 1.291 ms"
   "Execution Time: 0.067 ms"
   ```
   
   *Объясните результат:*
   Время выполнения сильно сократилось, так как был создан индекс на title, что ускорило поиск.

9. Выполните запрос для поиска книги по book_id и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE book_id = 18;
   ```
   
   *План выполнения:*
   ```sql
   "Seq Scan on t_books  (cost=0.00..3155.00 rows=1 width=33) (actual time=0.014..15.086 rows=1 loops=1)"
   "  Filter: (book_id = 18)"
   "  Rows Removed by Filter: 149999"
   "Planning Time: 0.401 ms"
   "Execution Time: 15.116 ms"
   ```
   
   *Объясните результат:*
   Скорость выполнения такая же, как и без индексов, так как на book_id индекс не был создан, а другие индексы в данном случае бесполезны

11. Выполните запрос для поиска активных книг и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE is_active = true;
   ```
   
   *План выполнения:*
   ```sql
   "Seq Scan on t_books  (cost=0.00..2780.00 rows=74965 width=33) (actual time=0.010..24.058 rows=75405 loops=1)"
   "  Filter: is_active"
   "  Rows Removed by Filter: 74595"
   "Planning Time: 0.129 ms"
   "Execution Time: 27.808 ms"
   ```
   
   *Объясните результат:*
   Скорость поиска не увеличилась с созданием индекса на is_active, так как разброс принимаемых уникальных значений очень маленький

11. Посчитайте количество строк и уникальных значений:
   ```sql
   SELECT 
       COUNT(*) as total_rows,
       COUNT(DISTINCT title) as unique_titles,
       COUNT(DISTINCT category) as unique_categories,
       COUNT(DISTINCT author) as unique_authors
   FROM t_books;
   ```
   
   *Результат:*
    
    150000	150000	6	1003

11. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_title_idx;
    DROP INDEX t_books_active_idx;
    ```
    
    *Результат:*
    ```sql
    DROP INDEX
    Query returned successfully in 268 msec.
    ```

13. Основываясь на предыдущих результатах, создайте индексы для оптимизации следующих запросов:
    a. `WHERE title = $1 AND category = $2`
    b. `WHERE title = $1`
    c. `WHERE category = $1 AND author = $2`
    d. `WHERE author = $1 AND book_id = $2`
    
    *Созданные индексы:*
    ```sql
    a. CREATE INDEX t_books_title_idx ON t_books(title);
    CREATE INDEX t_books_category_idx ON t_books(category);
    ```
    ```sql
    b. CREATE INDEX t_books_title_idx ON t_books(title);
    ```
    ```sql
    c. CREATE INDEX t_books_category_idx ON t_books(category);
    CREATE INDEX t_books_author_idx ON t_books(author);
    ```
    ```sql
    d. CREATE INDEX t_books_author_idx ON t_books(author);
    CREATE INDEX t_books_book_id_idx ON t_books(book_id);
    ```
    
    *Объясните ваше решение:*
    Индексы созданы в соответсвии с запросами

15. Протестируйте созданные индексы.
    
    *Результаты тестов:*
    ```sql
    a.
    "Index Scan using t_books_title_idx on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.020..0.021 rows=0 loops=1)"
    "  Index Cond: ((title)::text = 'Oracle Core'::text)"
    "  Filter: ((category)::text = 'History'::text)"
    "  Rows Removed by Filter: 1"
    "Planning Time: 0.453 ms"
    "Execution Time: 0.040 ms"
    ```
    ```sql
    b.
    "Index Scan using t_books_title_idx on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.045..0.047 rows=1 loops=1)"
    "  Index Cond: ((title)::text = 'Oracle Core'::text)"
    "Planning Time: 0.134 ms"
    "Execution Time: 0.073 ms"
    ```
    ```sql
    c.
    "Bitmap Heap Scan on t_books  (cost=5.41..428.73 rows=29 width=33) (actual time=0.069..0.296 rows=26 loops=1)"
    "  Recheck Cond: ((author)::text = 'Author_748'::text)"
    "  Filter: ((category)::text = 'History'::text)"
    "  Rows Removed by Filter: 106"
    "  Heap Blocks: exact=125"
    "  ->  Bitmap Index Scan on t_books_author_idx  (cost=0.00..5.40 rows=148 width=0) (actual time=0.035..0.036 rows=132 loops=1)"
    "        Index Cond: ((author)::text = 'Author_748'::text)"
    "Planning Time: 0.160 ms"
    "Execution Time: 0.335 ms"
    ```
    ```sql
    d.
    "Index Scan using t_books_book_id_idx on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.150..0.150 rows=0 loops=1)"
    "  Index Cond: (book_id = 1000)"
    "  Filter: ((author)::text = 'Author_748'::text)"
    "  Rows Removed by Filter: 1"
    "Planning Time: 0.177 ms"
    "Execution Time: 0.191 ms"
    ```
    
    *Объясните результаты:*
    Использование индексов ускорило запросы, однако индекс на category можно было не вводить, так как он нигде не используется из-за того, что очень маленькая дифференциация принимаемых значений

17. Выполните регистронезависимый поиск по началу названия:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';
    ```
    
    *План выполнения:*
    ```sql
    "Seq Scan on t_books  (cost=0.00..3155.00 rows=15 width=33) (actual time=141.617..141.618 rows=0 loops=1)"
    "  Filter: ((title)::text ~~* 'Relational%'::text)"
    "  Rows Removed by Filter: 150000"
    "Planning Time: 0.572 ms"
    "Execution Time: 141.641 ms"
    ```
    
    *Объясните результат:*
    Индекс BTree по умолчанию не учитывает нечувствительность к регистру. Поэтому он не работает при регистронезависимом поиске

19. Создайте функциональный индекс:
    ```sql
    CREATE INDEX t_books_up_title_idx ON t_books(UPPER(title));
    ```
    
    *Результат:*
    ```sql
    CREATE INDEX
    Query returned successfully in 715 msec.
    ```

20. Выполните запрос из шага 13 с использованием UPPER:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE UPPER(title) LIKE 'RELATIONAL%';
    ```
    
    *План выполнения:*
    ```sql
    "Seq Scan on t_books  (cost=0.00..3530.00 rows=750 width=33) (actual time=71.175..71.176 rows=0 loops=1)"
    "  Filter: (upper((title)::text) ~~ 'RELATIONAL%'::text)"
    "  Rows Removed by Filter: 150000"
    "Planning Time: 0.155 ms"
    "Execution Time: 71.212 ms"
    ```
    
    *Объясните результат:*
    Индекс не использовался, так как он не ускоряет процесс, потому что как минимум проходиться по всем строкам, чтобы привести всё в к верхнему регистру

22. Выполните поиск подстроки:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE '%Core%';
    ```
    
    *План выполнения:*
    ```sql
    "Seq Scan on t_books  (cost=0.00..3155.00 rows=15 width=33) (actual time=126.356..126.403 rows=1 loops=1)"
    "  Filter: ((title)::text ~~* '%Core%'::text)"
    "  Rows Removed by Filter: 149999"
    "Planning Time: 0.163 ms"
    "Execution Time: 126.425 ms"
    ```
    
    *Объясните результат:*
    Так как выполнен поиск совпадений не с начала строки, BTree-индекс бесполезен и не используется

24. Попробуйте удалить все индексы:
    ```sql
    DO $$ 
    DECLARE
        r RECORD;
    BEGIN
        FOR r IN (SELECT indexname FROM pg_indexes 
                  WHERE tablename = 't_books' 
                  AND indexname != 'books_pkey')
        LOOP
            EXECUTE 'DROP INDEX ' || r.indexname;
        END LOOP;
    END $$;
    ```
    
    *Результат:*
    ```sql
    DO
    Query returned successfully in 1 min 11 secs.
    ```
    
    *Объясните результат:*
    Все существующие индексы были удалены

26. Создайте индекс для оптимизации суффиксного поиска:
    ```sql
    -- Вариант 1: с reverse()
    CREATE INDEX t_books_rev_title_idx ON t_books(reverse(title));
    
    -- Вариант 2: с триграммами
    CREATE EXTENSION IF NOT EXISTS pg_trgm;
    CREATE INDEX t_books_trgm_idx ON t_books USING gin (title gin_trgm_ops);
    ```
    
    *Результаты тестов:*
    ```sql
    Первый вариант:
    "Seq Scan on t_books  (cost=0.00..3155.00 rows=15 width=33) (actual time=153.020..153.102 rows=1 loops=1)"
    "  Filter: ((title)::text ~~* '%Core%'::text)"
    "  Rows Removed by Filter: 149999"
    "Planning Time: 0.642 ms"
    "Execution Time: 153.127 ms"
    ```
    ```sql
    Второй вариант:
    ERROR:  operator class "gin_trgm_ops" does not exist for access method "gin" 
    ```
    
    *Объясните результаты:*
    Первый вариант не ускоряет поиск, так как проходит по всем строкам, чтобы их перевернуть, поэтому индекс бесполезен

    Второй вариант должен ускорить поиск,так как используется триграммный индекс, который сразу находит строки, в которых есть все необходимые триграммы, без необходимости просматривать все строки

28. Выполните поиск по точному совпадению:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title = 'Oracle Core';
    ```
    
    *План выполнения:*
    ```sql
    "Seq Scan on t_books  (cost=0.00..3155.00 rows=1 width=33) (actual time=20.755..20.791 rows=1 loops=1)"
    "  Filter: ((title)::text = 'Oracle Core'::text)"
    "  Rows Removed by Filter: 149999"
    "Planning Time: 0.118 ms"
    "Execution Time: 20.853 ms"
    ```
    
    *Объясните результат:*
    На предыдущем этапе не получилось создать триграммный индекс, но если бы он был создан, то ускорил бы поиск, так как искал бы строки с полным совпадением триграмм, а не пробегался по всем

30. Выполните поиск по началу названия:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';
    ```
    
    *План выполнения:*
    ```sql
    "Seq Scan on t_books  (cost=0.00..3155.00 rows=15 width=33) (actual time=184.488..184.489 rows=0 loops=1)"
    "  Filter: ((title)::text ~~* 'Relational%'::text)"
    "  Rows Removed by Filter: 150000"
    "Planning Time: 0.214 ms"
    "Execution Time: 184.518 ms"
    ```
    
    *Объясните результат:*
    На предыдущем этапе не получилось создать триграммный индекс, но если бы он был создан, то ускорил бы поиск, так как искал бы строки,в которых есть все необходимые триграммы,  а не пробегался по всем

32. Создайте свой пример индекса с обратной сортировкой:
    ```sql
    CREATE INDEX t_books_desc_idx ON t_books(title DESC);
    ```
    
    *Тестовый запрос:*
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books
    ORDER BY title DESC;
   ```
    
    *План выполнения:*
    ```sql
    "Index Scan using t_books_desc_idx on t_books  (cost=0.42..9288.72 rows=150000 width=33) (actual time=0.048..59.639 rows=150000 loops=1)"
    "Planning Time: 0.389 ms"
    "Execution Time: 67.867 ms"
    ```
    
    *Объясните результат:*
    Индекс ускорил запрос, так уже изначально отсортировал данные в нужном порядке
