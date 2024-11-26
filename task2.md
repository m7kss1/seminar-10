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
   ![alt text](image-1.png)

4. Выполните запрос для поиска книги с id = 18:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part WHERE book_id = 18;
   ```
   
   *План выполнения:*
   ```sql
   Seq Scan on t_books_part_1 t_books_part  (cost=0.00..2065.98 rows=2 width=32) (actual time=0.025..15.65 rows=2 loops=1)
      Filter: (book_id = 18)
      Rows Removed by Filter: 99996
    Planning Time: 0.228 ms
    Execution Time: 14.881 ms
   ```
   *Объясните результат:*   
   есть декларации диапазонов при создании партиций => запрос при поиске book_id = 18 попадет в диапазон значений

5. Выполните поиск по названию книги:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part 
   WHERE title = 'Expert PostgreSQL Architecture';
   ```
   
   *План выполнения:*
   
   ```sql
    Gather  (cost=1000.00..5655.50 rows=6 width=33) (actual time=11.921..23.459 rows=2 loops=1)
      Workers Planned: 2
      Workers Launched: 2
      ->  Parallel Append  (cost=0.00..4654.90 rows=3 width=33) (actual time=10.423..13.283 rows=1 loops=3)
            ->  Parallel Seq Scan on t_books_part_2  (cost=0.00..1552.29 rows=1 width=33) (actual time=3.839..3.839 rows=0 loops=3)
                  Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)
                  Rows Removed by Filter: 33333
            ->  Parallel Seq Scan on t_books_part_3  (cost=0.00..1551.31 rows=1 width=34) (actual time=5.839..5.839 rows=0 loops=2)
                  Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)
                  Rows Removed by Filter: 50001
            ->  Parallel Seq Scan on t_books_part_1  (cost=0.00..1551.28 rows=1 width=32) (actual time=10.839..16.839 rows=2 loops=1)
                  Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)
                  Rows Removed by Filter: 99996
   Planning Time: 0.348 ms
   Execution Time: 31.00 ms
   ```
   *Объясните результат:*
   чекаем все 3 партиции. ет индекса на title-> seq scan.  так как данные не зависят мы распараллелим 

6. Создайте партиционированный индекс:
   ```sql
   CREATE INDEX ON t_books_part(title);
   ```
   
   *Результат:*
   ```sql
   CREATE INDEX ON t_books_part(title)
   completed in 1 s 240 ms
   ```

7. Повторите запрос из шага 5:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part 
   WHERE title = 'Expert PostgreSQL Architecture';
   ```
   
   *План выполнения:*
   ```sql
    Append  (cost=4.43..35.09 rows=6 width=33) (actual time=0.116..0.310 rows=2 loops=1)
      ->  Bitmap Heap Scan on t_books_part_1  (cost=4.43..12.16 rows=2 width=32) (actual time=0.114..0.121 rows=2 loops=1)
            Recheck Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)
            Heap Blocks: exact=2
            ->  Bitmap Index Scan on t_books_part_1_title_idx  (cost=0.00..4.43 rows=2 width=0) (actual time=0.103..0.103 rows=2 loops=1)
                  Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)
      ->  Index Scan using t_books_part_2_title_idx on t_books_part_2  (cost=0.42..11.46 rows=2 width=33) (actual time=0.108..0.109 rows=0 loops=1)
            Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)
      ->  Index Scan using t_books_part_3_title_idx on t_books_part_3  (cost=0.42..11.45 rows=2 width=34) (actual time=0.074..0.074 rows=0 loops=1)
            Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)
    Planning Time: 0.913 ms
    Execution Time: 0.455 ms
   ```
   
   *Объясните результат:*
    используем индексы после создания для каждой партиции

8. Удалите созданный индекс:
   ```sql
   DROP INDEX t_books_part_title_idx;
   ```
   
   *Результат:*
   ```sql
    > DROP INDEX t_books_part_title_idx
    completed in 6 ms
    ```

9. Создайте индекс для каждой партиции:
   ```sql
   CREATE INDEX ON t_books_part_1(title);
   CREATE INDEX ON t_books_part_2(title);
   CREATE INDEX ON t_books_part_3(title);
   ```

10. Повторите запрос из шага 5:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books_part 
    WHERE title = 'Expert PostgreSQL Architecture';
    ```
    
   *Результат:*
   ```sql
    Append  (cost=4.43..35.09 rows=6 width=33) (actual time=0.052..0.112 rows=2 loops=1)
      ->  Bitmap Heap Scan on t_books_part_1  (cost=4.43..12.16 rows=2 width=32) (actual time=0.051..0.056 rows=2 loops=1)
            Recheck Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)
            Heap Blocks: exact=2
            ->  Bitmap Index Scan on t_books_part_1_title_idx  (cost=0.00..4.43 rows=2 width=0) (actual time=0.042..0.042 rows=2 loops=1)
                  Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)
      ->      Index Scan using t_books_part_2_title_idx on t_books_part_2  (cost=0.42..11.46 rows=2 width=33) (actual time=0.028..0.028 rows=0 loops=1)
                Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)
      ->  Index Scan using t_books_part_3_title_idx on t_books_part_3  (cost=0.42..11.45 rows=2 width=34) (actual time=0.025..0.025 rows=0 loops=1)
                Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)
    Planning Time: 0.741 ms
    Execution Time: 0.253 ms
   ```

   ищем только в затронутых партициях + используем индексы

11. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_part_1_title_idx;
    DROP INDEX t_books_part_2_title_idx;
    DROP INDEX t_books_part_3_title_idx;
    ```

12. Создайте обычный индекс по book_id:
    ```sql
    CREATE INDEX t_books_part_idx ON t_books_part(book_id);
    ```
    
    *Результат:*
    ```sql
    CREATE INDEX t_books_part_idx ON t_books_part(book_id)
    completed in 142 ms
    ```    

13. Выполните поиск по book_id:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books_part WHERE book_id = 11011;
    ```
    
    *План выполнения:*
    ```sql
    Index Scan using t_books_part_1_book_id_idx on t_books_part_1 t_books_part  (cost=0.29..11.32 rows=2 width=32) (actual time=0.039..0.044 rows=2 loops=1)
      Index Cond: (book_id = 11011)
    Planning Time: 0.555 ms
    Execution Time: 0.041 ms
    ```
    
    *Объясните результат:*
    аналогично. нет повторения + индексы => index scan

14. Создайте индекс по полю is_active:
    ```sql
    CREATE INDEX t_books_active_idx ON t_books(is_active);
    ```
    
    *Результат:*
    ```sql
    CREATE INDEX t_books_active_idx ON t_books(is_active)
    completed in 52 ms
    ```
15. Выполните поиск активных книг с отключенным последовательным сканированием:
    ```sql
    SET enable_seqscan = off;
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE is_active = true;
    SET enable_seqscan = on;
    ```
    
    *План выполнения:*
    ```sql
     Bitmap Heap Scan on t_books  (cost=840.03..2812.08 rows=74805 width=33) (actual time=1.949..8.311 rows=75089 loops=1)
      Recheck Cond: is_active
      Heap Blocks: exact=1224
      ->  Bitmap Index Scan on t_books_active_idx  (cost=0.00..821.33 rows=74805 width=0) (actual time=1.806..1.807 rows=75089 loops=1)
            Index Cond: (is_active = true)
    Planning Time: 0.241 ms
    Execution Time: 11.413 ms
    ```
    
    *Объясните результат:*
    видимо теперь есть повторения + построены индексы

16. Создайте составной индекс:
    ```sql
    CREATE INDEX t_books_author_title_index ON t_books(author, title);
    ```
    
    *Результат:*
    ```sql
    > CREATE INDEX t_books_author_title_index ON t_books(author, title)
    completed in 239 ms
    ```
17. Найдите максимальное название для каждого автора:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, MAX(title) 
    FROM t_books 
    GROUP BY author;
    ```
    
    *План выполнения:*
    ```sql
     HashAggregate  (cost=3474.00..3484.00 rows=1000 width=42) (actual time=50.344..50.434 rows=1003 loops=1)
      Group Key: author
      Batches: 1  Memory Usage: 193kB
      ->  Seq Scan on t_books  (cost=0.00..2724.00 rows=150000 width=21) (actual time=0.004..7.221 rows=150000 loops=1)
    Planning Time: 0.214 ms
    Execution Time: 42.375 ms
    ```
    
    *Объясните результат:*
    нам нужны все данные => придется seq scan

18. Выберите первых 10 авторов:
    ```sql
    EXPLAIN ANALYZE
    SELECT DISTINCT author 
    FROM t_books 
    ORDER BY author 
    LIMIT 10;
    ```
    
    *План выполнения:*
    ```sql
     Limit  (cost=0.42..56.63 rows=10 width=10) (actual time=0.084..0.314 rows=10 loops=1)
      ->  Result  (cost=0.42..5621.42 rows=1000 width=10) (actual time=0.083..0.312 rows=10 loops=1)
            ->  Unique  (cost=0.42..5621.42 rows=1000 width=10) (actual time=0.082..0.310 rows=10 loops=1)
                  ->  Index Only Scan using t_books_author_title_index on t_books  (cost=0.42..5246.42 rows=150000 width=10) (actual time=0.081..0.233 rows=1373 loops=1)
                        Heap Fetches: 0
    Planning Time: 0.102 ms
    Execution Time: 0.413 ms
    ```
    
    *Объясните результат:*
    построен индекс, берем только уникальные => можем запустить index scan

19. Выполните поиск и сортировку:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE author LIKE 'T%'
    ORDER BY author, title;
    ```
    
    *План выполнения:*
    ```sql
    Sort  (cost=3099.29..3099.33 rows=15 width=21) (actual time=22.761..22.762 rows=1 loops=1)
      Sort Key: author, title
      Sort Method: quicksort  Memory: 25kB
      ->  Seq Scan on t_books  (cost=0.00..3099.00 rows=15 width=21) (actual time=22.752..22.754 rows=1 loops=1)
            Filter: ((author)::text ~~ 'T%'::text)
            Rows Removed by Filter: 149999
    Planning Time: 0.215 ms
    Execution Time: 22.880 ms
    ```
    
    *Объясните результат:*
    ilike => не можем использовать индекс => долго вычитываем все

20. Добавьте новую книгу:
    ```sql
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150001, 'Cookbook', 'Mr. Hide', NULL, true);
    COMMIT;
    ```
    

21. Создайте индекс по категории:
    ```sql
    CREATE INDEX t_books_cat_idx ON t_books(category);
    ```

22. Найдите книги без категории:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE category IS NULL;
    ```


23. Создайте частичные индексы:
    ```sql
    DROP INDEX t_books_cat_idx;
    CREATE INDEX t_books_cat_null_idx ON t_books(category) WHERE category IS NULL;
    ```
    

24. Повторите запрос из шага 22:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE category IS NULL;
    ```
    
    *План выполнения:*
    ```sql
     Index Scan using t_books_cat_idx on t_books  (cost=0.29..8.16 rows=1 width=21) (actual time=0.075..0.078 rows=1 loops=1)
      Index Cond: (category IS NULL)
    Planning Time: 0.239 ms
    Execution Time: 0.493 ms
    ```
    
    *Объясните результат:*
    t_books_cat_idx - построен индекс на category и значения уникальны
    
25. Создайте частичный уникальный индекс:
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
    [Вставьте результаты всех операций]
    
    *Объясните результат:*
    [Ваше объяснение]