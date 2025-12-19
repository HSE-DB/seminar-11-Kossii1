## Задание 3

1. Создайте таблицу с большим количеством данных:
    ```sql
    CREATE TABLE test_cluster AS 
    SELECT 
        generate_series(1,1000000) as id,
        CASE WHEN random() < 0.5 THEN 'A' ELSE 'B' END as category,
        md5(random()::text) as data;
    ```

2. Создайте индекс:
    ```sql
    CREATE INDEX test_cluster_cat_idx ON test_cluster(category);
    ```

3. Измерьте производительность до кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    ```text
    Bitmap Heap Scan on test_cluster  (cost=59.17..7696.73 rows=5000 width=68) (actual time=19.740..132.034 rows=500305 loops=1)
   Recheck Cond: (category = 'A'::text)
   Heap Blocks: exact=8334
   ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..57.92 rows=5000 width=0) (actual time=18.663..18.664 rows=500305 loops=1)
         Index Cond: (category = 'A'::text)
    Planning Time: 0.411 ms
    Execution Time: 144.509 ms
    ```
    
    *Объясните результат:*

    Запрос использует индекс, который отбирает страницы с подходящими строками. Поскольку строки с категорией 'A' разбросаны по всей таблице, Bitmap Heap Scan перепроверяет много блоков heap, что увеличивает время выполнения.

4. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```
    
    *Результат:*
     ```text
    workshop=# CLUSTER test_cluster USING test_cluster_cat_idx;
    CLUSTER
    ```

5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    ```text
    Bitmap Heap Scan on test_cluster  (cost=5549.86..20105.52 rows=497733 width=39) (actual time=9.913..54.455 rows=500305 loops=1)
   Recheck Cond: (category = 'A'::text)
   Heap Blocks: exact=4170
   ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..5425.42 rows=497733 width=0) (actual time=9.482..9.483 rows=500305 loops=1)
         Index Cond: (category = 'A'::text)
    Planning Time: 0.477 ms
    Execution Time: 67.049 ms
    ```
    
    *Объясните результат:*

    После кластеризации строки с категорией 'A' расположены подряд на диске, поэтому Bitmap Heap Scan проверяет меньше блоков heap, что сокращает время выполнения почти вдвое.

6. Сравните производительность до и после кластеризации:
    
    *Сравнение:*
    
    До кластеризации строки с категорией 'A' были разбросаны по всей таблице, поэтому проверялось много блоков heap и время выполнения было 144 мс. После кластеризации эти строки стали расположены подряд, количество проверяемых блоков уменьшилось почти вдвое и время выполнения сократилось до 67 мс, что делает поиск заметно быстрее.