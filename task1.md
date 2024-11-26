# Задание 1. B-tree индексы в PostgreSQL

1. Запустите БД через docker compose в ./src/docker-compose.yml:

2. Выполните запрос для поиска книги с названием 'Oracle Core' и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE title = 'Oracle Core';
   ```
   
   *План выполнения:*
   ```sql

   Seq Scan on t_books  (cost=0.00..3100.00 rows=1 width=33) (actual time=15.319..15.321 rows=1 loops=1)
     Filter: ((title)::text = 'Oracle Core'::text)
     Rows Removed by Filter: 149999
   Planning Time: 0.182 ms
   Execution Time: 14.240 ms
   ```
   
   *Объясните результат:*
   Seq scan -> вынуждены вычитать все строки за линию. Долго плохо 

3. Создайте B-tree индексы:
   ```sql
   CREATE INDEX t_books_title_idx ON t_books(title);
   CREATE INDEX t_books_active_idx ON t_books(is_active);
   ```
   
   *Результат:*
   ```sql
    > CREATE INDEX t_books_title_idx ON t_books(title)
    completed in 322 ms
    > CREATE INDEX t_books_active_idx ON t_books(is_active)
    completed in 52 ms
   ```

4. Проверьте информацию о созданных индексах:
   ```sql
   SELECT schemaname, tablename, indexname, indexdef
   FROM pg_catalog.pg_indexes
   WHERE tablename = 't_books';
   ```
   
   *Результат:*
   индексы создались успешно

5. Обновите статистику таблицы:
   ```sql
   ANALYZE t_books;
   ```
   ```sql
    SELECT relname, n_tup_ins, n_tup_upd, n_tup_del, n_tup_hot_upd 
    FROM pg_stat_user_tables
    WHERE relname = 't_books';
   ```
   *Результат:*

    ```
   relname | n_tup_ins | n_tup_upd | n_tup_del | n_tup_hot_upd
   -----------------------------------------------------------
   t_books | 1500000   |     3     |     0     |      0
   ```
   

6. Выполните запрос для поиска книги 'Oracle Core' и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE title = 'Oracle Core';
   ```
   
   *План выполнения:*
   ```sql
    Index Scan using t_books_title_idx on t_books  (cost=0.43..8.43 rows=1 width=33) (actual time=0.073..0.074 rows=1 loops=1)
      Index Cond: ((title)::text = 'Oracle Core'::text)
    Planning Time: 0.353 ms
    Execution Time: 0.121 ms
   ```
   
   *Объясните результат:*
   используется index scan. он используется потому что у нас нет повторений видимо, так бы должен был быть по хорошему ```bitmap index scan```

7. Выполните запрос для поиска книги по book_id и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE book_id = 18;
   ```
   
   *План выполнения:*
   ```sql
    Index Scan using t_books_id_pk on t_books  (cost=0.43..8.43 rows=1 width=33) (actual time=0.073..0.074 rows=1 loops=1)
     Index Cond: (book_id = 18)
   Planning Time: 0.049 ms
   Execution Time: 0.064 ms
   ```
   
   *Объясните результат:*
   аналогично 6. индекс на  book_id создался когда создавался pk

8. Выполните запрос для поиска активных книг и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE is_active = true;
   ```
   
   *План выполнения:*
   ```sql
      Seq Scan on t_books  (cost=0.00..28.00 rows=74490 width=33) (actual time=0.004..11.851 rows=74770 loops=1)
     Filter: is_active
     Rows Removed by Filter: 75230
   Planning Time: 0.04 ms
   Execution Time: 15.201 ms
   ```
   
   *Объясните результат:*
   предположу что boolean не индексируется 

9. Посчитайте количество строк и уникальных значений:
   ```sql
   SELECT 
       COUNT(*) as total_rows,
       COUNT(DISTINCT title) as unique_titles,
       COUNT(DISTINCT category) as unique_categories,
       COUNT(DISTINCT author) as unique_authors
   FROM t_books;
   ```
   
   *Результат:*
    ```
   total_rows | unique_titles | unique_categories | unique_authors
   ----------------------------------------------------------------
    1500000    |    1500000    |       6           |      1003
   ``` 

10. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_title_idx;
    DROP INDEX t_books_active_idx;
    ```
    
    *Результат:*
    ```sql
    > DROP INDEX t_books_title_idx
    completed in 4 ms
    > DROP INDEX t_books_active_idx
    completed in 2 ms
    ```

11. Основываясь на предыдущих результатах, создайте индексы для оптимизации следующих запросов:
    a. `WHERE title = $1 AND category = $2`
    b. `WHERE title = $1`
    c. `WHERE category = $1 AND author = $2`
    d. `WHERE author = $1 AND book_id = $2`
    
    *Созданные индексы:*
    ```sql

    CREATE INDEX idx_title_category ON t_books(title, category);
    CREATE INDEX idx_title ON t_books(title);
    CREATE INDEX idx_category_author ON t_books(category, author);
    CREATE INDEX idx_author_book_id ON t_books(author, book_id);
    ```
    
    *Объясните ваше решение:*
    создаем индексы, использующие несколько аттрибутов одновременно. оптимизируем для ```where expr, expr ...```

12. Протестируйте созданные индексы.
    
    *Результаты тестов:*
    ```sql
     Index Scan using idx_author_book_id on t_books  (cost=0.43..8.43 rows=1 width=33) (actual time=0.097..0.099 rows=1 loops=1)
      Index Cond: (((author)::text = 'Jonathan Lewis'::text) AND (book_id = 3001))
    Planning Time: 0.150 ms
    Execution Time: 0.134 ms
    ```
    
    *Объясните результаты:*
    опять же аналогично пункту 6. так как повторов нет и индексы созданы делаем ```index scan```, а не ```seq``` или ```bitmap``` index scan

13. Выполните регистронезависимый поиск по началу названия:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';
    ```
    
    *План выполнения:*
    ```sql
    Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=33) (actual time=84.860..84.860 rows=0 loops=1)
      Filter: ((title)::text ~~* 'Relational%'::text)
      Rows Removed by Filter: 150000
    Planning Time: 0.944 ms
    Execution Time: 83.685 ms
    ```
    
    *Объясните результат:*
    проблема ilike не поддерживается для индексирования. ничего лучше seq scan не можем запустить 

14. Создайте функциональный индекс:
    ```sql
    CREATE INDEX t_books_up_title_idx ON t_books(UPPER(title));
    ```
    
    *Результат:*
    ```sql
    > CREATE INDEX t_books_up_title_idx ON t_books(UPPER(title))
    completed in 353 ms
    ```

15. Выполните запрос из шага 13 с использованием UPPER:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE UPPER(title) LIKE 'RELATIONAL%';
    ```
    
    *План выполнения:*
    ```sql
    Seq Scan on t_books  (cost=0.00..3465.00 rows=750 width=33) (actual time=84.370..84.840 rows=0 loops=1)
      Filter: (upper((title)::text) ~~ 'RELATIONAL%'::text)
      Rows Removed by Filter: 150000
    Planning Time: 0.144 ms
    Execution Time: 84.767 ms
    ```
    
    *Объясните результат:*
    предположу что функциональный индекс не поможет если у нас ilike

16. Выполните поиск подстроки:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE '%Core%';
    ```
    
    *План выполнения:*
    ```sql
    Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=33) (actual time=70.564..70.567 rows=1 loops=1)
      Filter: ((title)::text ~~* '%Core%'::text)
      Rows Removed by Filter: 149999
    Planning Time: 0.223 ms
    Execution Time: 71.540 ms
    ```
    
    *Объясните результат:*
    аналогично, для такой операцииilike не поддерживаютя индексы => seq scan

17. Попробуйте удалить все индексы:
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

18. Создайте индекс для оптимизации суффиксного поиска:
    ```sql
    -- Вариант 1: с reverse()
    CREATE INDEX t_books_rev_title_idx ON t_books(reverse(title));
    
    -- Вариант 2: с триграммами
    CREATE EXTENSION IF NOT EXISTS pg_trgm;
    CREATE INDEX t_books_trgm_idx ON t_books USING gin (title gin_trgm_ops);
    ```
    
    *Результаты тестов:*
    ```
    Seq Scan on t_books  (cost=0.00..3475.00 rows=750 width=33) (actual time=45.648..45.746 rows=0 loops=1)
      Filter: (upper((title)::text) ~~ 'RELATIONAL%'::text)
      Rows Removed by Filter: 150000
    Planning Time: 0.145 ms
    Execution Time: 55.780 ms
    
    Bitmap Heap Scan on t_books  (cost=95.15..150.36 rows=15 width=33) (actual time=0.039..0.040 rows=0 loops=1)
    Recheck Cond: ((title)::text ~~* 'Relational%'::text)
    Rows Removed by Index Recheck: 1
    Heap Blocks: exact=1
        -> Bitmap Index Scan on t_books_trgm_idx  (cost=0.00..95.15 rows=15 width=0) (actual time=0.025..0.025 rows=1 loops=1)
                Index Cond: ((title)::text ~~* 'Relational%'::text)
        Planning Time: 0.228 ms
        Execution Time: 0.084 ms
    ```
    
    *Объясните результаты:*
    pg_trgm лучше для поиска по подстрокам, а reverse() индекс
    полезен для суффиксного поиска

19. Выполните поиск по точному совпадению:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title = 'Oracle Core';
    ```
    
    *План выполнения:*
    ```sql
    Index Scan using idx_title on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.023..0.025 rows=1 loops=1)
    Index Cond: ((title)::text = 'Oracle Core'::text)
    Planning Time: 0.414 ms
    Execution Time: 0.256 ms
    ```
    
    *Объясните результат:*
    точно совпадение с полем для которого создавался индекс => index scan аналогично 6

20. Выполните поиск по началу названия:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';
    ```
    
    *План выполнения:*
    ```sql
    Bitmap Heap Scan on t_books  (cost=95.15..150.36 rows=15 width=33) (actual time=0.019..0.019 rows=0 loops=1)
      Recheck Cond: ((title)::text ~~* 'Relational%'::text)
    Rows Removed by Index Recheck: 1
    Heap Blocks: exact=1
    ->  Bitmap Index Scan on t_books_trgm_idx  (cost=0.00..95.15 rows=15 width=0) (actual time=0.013..0.014 rows=1 loops=1)
            Index Cond: ((title)::text ~~* 'Relational%'::text)
    Planning Time: 0.226 ms
    Execution Time: 0.087 ms
    ```
    
    *Объясните результат:*
    триграмы норм будут ообрабатывать строки даже если запрос не будет чувствителен к регистру. поиск по началу строки будет лучше работать с триграмным индексом + ilike

21. Создайте свой пример индекса с обратной сортировкой:
    ```sql
    CREATE INDEX t_books_desc_idx ON t_books(title DESC);
    ```
    
    *Тестовый запрос:*
    ```sql

    EXPLAIN ANALYZE
    SELECT * FROM t_books ORDER BY title DESC LIMIT 10;
    ```
    
    *План выполнения:*
    ```sql
      Limit  (cost=0.42..1.02 rows=10 width=33) (actual time=0.014..0.018 rows=10 loops=1)
    ->  Index Scan using t_books_desc_idx on t_books  (cost=0.42..9062.76 rows=150000 width=33) (actual time=0.011..0.022 rows=10 loops=1)
    Planning Time: 0.321 ms
    Execution Time: 0.017 ms
    ```
    
    *Объясните результат:*
    благодаря индексам запрос с order by не нужно будет проводить сортировку кортежей (а у нас может быть такая ситуация что какие-то данные в памяти, а какие-то на диске сгружены). такая сортировка потенциально будет работать долго. а индекс с обратной сортировкой позволит этого избежать 