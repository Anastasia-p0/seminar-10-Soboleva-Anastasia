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
   [Ваше объяснение]

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
   Скорость выполнения такая же, как и без индексов, так как на book_id индекс не был создан

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
    ```sql
    150000	150000	6	1003
    ```

11. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_title_idx;
    DROP INDEX t_books_active_idx;
    ```
    
    *Результат:*
    [Вставьте результат выполнения]

12. Основываясь на предыдущих результатах, создайте индексы для оптимизации следующих запросов:
    a. `WHERE title = $1 AND category = $2`
    b. `WHERE title = $1`
    c. `WHERE category = $1 AND author = $2`
    d. `WHERE author = $1 AND book_id = $2`
    
    *Созданные индексы:*
    [Вставьте команды создания индексов]
    
    *Объясните ваше решение:*
    [Ваше объяснение]

13. Протестируйте созданные индексы.
    
    *Результаты тестов:*
    [Вставьте планы выполнения для каждого случая]
    
    *Объясните результаты:*
    [Ваше объяснение]

14. Выполните регистронезависимый поиск по началу названия:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';
    ```
    
    *План выполнения:*
    [Вставьте план выполнения]
    
    *Объясните результат:*
    [Ваше объяснение]

15. Создайте функциональный индекс:
    ```sql
    CREATE INDEX t_books_up_title_idx ON t_books(UPPER(title));
    ```
    
    *Результат:*
    [Вставьте результат выполнения]

16. Выполните запрос из шага 13 с использованием UPPER:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE UPPER(title) LIKE 'RELATIONAL%';
    ```
    
    *План выполнения:*
    [Вставьте план выполнения]
    
    *Объясните результат:*
    [Ваше объяснение]

17. Выполните поиск подстроки:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE '%Core%';
    ```
    
    *План выполнения:*
    [Вставьте план выполнения]
    
    *Объясните результат:*
    [Ваше объяснение]

18. Попробуйте удалить все индексы:
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
    [Вставьте результат выполнения]
    
    *Объясните результат:*
    [Ваше объяснение]

19. Создайте индекс для оптимизации суффиксного поиска:
    ```sql
    -- Вариант 1: с reverse()
    CREATE INDEX t_books_rev_title_idx ON t_books(reverse(title));
    
    -- Вариант 2: с триграммами
    CREATE EXTENSION IF NOT EXISTS pg_trgm;
    CREATE INDEX t_books_trgm_idx ON t_books USING gin (title gin_trgm_ops);
    ```
    
    *Результаты тестов:*
    [Вставьте планы выполнения для обоих вариантов]
    
    *Объясните результаты:*
    [Ваше объяснение]

20. Выполните поиск по точному совпадению:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title = 'Oracle Core';
    ```
    
    *План выполнения:*
    [Вставьте план выполнения]
    
    *Объясните результат:*
    [Ваше объяснение]

21. Выполните поиск по началу названия:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';
    ```
    
    *План выполнения:*
    [Вставьте план выполнения]
    
    *Объясните результат:*
    [Ваше объяснение]

22. Создайте свой пример индекса с обратной сортировкой:
    ```sql
    CREATE INDEX t_books_desc_idx ON t_books(title DESC);
    ```
    
    *Тестовый запрос:*
    ```sql
    SELECT * FROM t_books
    ORDER BY title DESC;
   ```
    
    *План выполнения:*
    [Вставьте план выполнения]
    
    *Объясните результат:*
    [Ваше объяснение]
