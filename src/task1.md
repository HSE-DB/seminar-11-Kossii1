# Задание 1: BRIN индексы и bitmap-сканирование

1. Удалите старую базу данных, если есть:
   ```shell
   docker compose down
   ```

2. Поднимите базу данных из src/docker-compose.yml:
   ```shell
   docker compose down && docker compose up -d
   ```

3. Обновите статистику:
   ```sql
   ANALYZE t_books;
   ```

4. Создайте BRIN индекс по колонке category:
   ```sql
   CREATE INDEX t_books_brin_cat_idx ON t_books USING brin(category);
   ```

5. Найдите книги с NULL значением category:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE category IS NULL;
   ```
   
   *План выполнения:*
   ```text
   Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.010..0.010 rows=0 loops=1)
   Recheck Cond: (category IS NULL)
   ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.007..0.008 rows=0 loops=1)
         Index Cond: (category IS NULL)
   Planning Time: 0.517 ms
   Execution Time: 0.045 ms
   ```
   
   *Объясните результат:*

   Для поиска используется BRIN-индекс, который по метаданным страниц показывает, где могут быть NULL. Индекс возвращает кандидатов, затем Bitmap Heap Scan перепроверяет условие, но фактически строк с category NULL нет, поэтому результат - 0 при минимальных затратах.

6. Создайте BRIN индекс по автору:
   ```sql
   CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author);
   ```

7. Выполните поиск по категории и автору:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books 
   WHERE category = 'INDEX' AND author = 'SYSTEM';
   ```
   
   *План выполнения:*
    ```text
   Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=9.693..9.694 rows=0 loops=1)
   Recheck Cond: ((category)::text = 'INDEX'::text)
   Rows Removed by Index Recheck: 150000
   Filter: ((author)::text = 'SYSTEM'::text)
   Heap Blocks: lossy=1224
   ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.579..0.579 rows=12240 loops=1)
         Index Cond: ((category)::text = 'INDEX'::text)
   Planning Time: 0.528 ms
   Execution Time: 9.770 ms
   ```
   
   *Объясните результат (обратите внимание на bitmap scan):*

   Используется только BRIN-индекс по category, потому что BRIN по author не даёт селективности для комбинированного условия. Bitmap Index Scan по category возвращает набор страниц-кандидатов, а не точные строки, поэтому Bitmap Heap Scan читает lossy-блоки и перепроверяет условие по строкам. В результате отбрасываются все строки при Index Recheck, затем фильтр по author дополнительно проверяется, но подходящих строк нет, поэтому итог - подходящих строк нет и выполнение получается относительно дорогим.

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   ```text
   Sort  (cost=3099.11..3099.12 rows=5 width=7) (actual time=25.773..25.775 rows=6 loops=1)
   Sort Key: category
   Sort Method: quicksort  Memory: 25kB
   ->  HashAggregate  (cost=3099.00..3099.05 rows=5 width=7) (actual time=25.727..25.728 rows=6 loops=1)
         Group Key: category
         Batches: 1  Memory Usage: 24kB
         ->  Seq Scan on t_books  (cost=0.00..2724.00 rows=150000 width=7) (actual time=0.007..7.760 rows=150000 loops=1)
   Planning Time: 0.426 ms
   Execution Time: 25.909 ms
   ```
   
   *Объясните результат:*
   
   Выполняется последовательное сканирование всей таблицы, так как нужно прочитать все строки для получения уникальных значений, затем с помощью HashAggregate формирует множество категорий, после чего выполняет сортировку для ORDER BY, при этом BRIN индекс не используется, поскольку он не ускоряет DISTINCT и ORDER BY по всей таблице.

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   ```text
    Aggregate  (cost=3099.03..3099.05 rows=1 width=8) (actual time=6.946..6.947 rows=1 loops=1)
   ->  Seq Scan on t_books  (cost=0.00..3099.00 rows=14 width=0) (actual time=6.941..6.942 rows=0 loops=1)
         Filter: ((author)::text ~~ 'S%'::text)
         Rows Removed by Filter: 150000
   Planning Time: 1.926 ms
   Execution Time: 6.969 ms
   ```
   
   *Объясните результат:*
   
   Выполняется последовательное сканирование всей таблицы, так как BRIN индекс по author неэффективен для префиксного поиска и селективность маленькая, поэтому оптимизатор читает все строки, отфильтровывает их по LIKE и затем агрегирует результат.

10. Создайте индекс для регистронезависимого поиска:
    ```sql
    CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title));
    ```

11. Подсчитайте книги, начинающиеся на 'O':
    ```sql
    EXPLAIN ANALYZE
    SELECT COUNT(*) 
    FROM t_books 
    WHERE LOWER(title) LIKE 'o%';
    ```
   
   *План выполнения:*
   ```text
    Aggregate  (cost=3475.88..3475.89 rows=1 width=8) (actual time=25.012..25.013 rows=1 loops=1)
   ->  Seq Scan on t_books  (cost=0.00..3474.00 rows=750 width=0) (actual time=25.005..25.007 rows=1 loops=1)
         Filter: (lower((title)::text) ~~ 'o%'::text)
         Rows Removed by Filter: 149999
   Planning Time: 0.565 ms
   Execution Time: 25.031 ms
   ```
   
   *Объясните результат:*
   
   Созданный индекс не используется, потому что запрос использует LIKE с префиксом, но оптимизатор решает, что последовательное сканирование всей таблицы быстрее для такой селективности. Все строки проверяются функцией LOWER, фильтруются по условию и затем агрегируются.

12. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_brin_cat_idx;
    DROP INDEX t_books_brin_author_idx;
    DROP INDEX t_books_lower_title_idx;
    ```

13. Создайте составной BRIN индекс:
    ```sql
    CREATE INDEX t_books_brin_cat_auth_idx ON t_books 
    USING brin(category, author);
    ```

14. Повторите запрос из шага 7:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE category = 'INDEX' AND author = 'SYSTEM';
    ```
   
   *План выполнения:*
   ```text
    Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=0.551..0.551 rows=0 loops=1)
   Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Rows Removed by Index Recheck: 8815
   Heap Blocks: lossy=72
   ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.032..0.032 rows=720 loops=1)
         Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Planning Time: 0.292 ms
   Execution Time: 0.631 ms
   ```
   
   *Объясните результат:*
   
   Составной BRIN индекс по category и author позволяет сразу отбирать страницы, где одновременно могут быть нужные значения, что сокращает количество строк для перепроверки в heap. Bitmap Heap Scan проверяет каждую строку на условие и отбрасывает несоответствующие, поэтому количество проверок стало значительно меньше, а выполнение запроса быстрее.