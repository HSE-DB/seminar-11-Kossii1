## Задание 2

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

4. Создайте полнотекстовый индекс:
    ```sql
    CREATE INDEX t_books_fts_idx ON t_books 
    USING GIN (to_tsvector('english', title));
    ```

5. Найдите книги, содержащие слово 'expert':
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE to_tsvector('english', title) @@ to_tsquery('english', 'expert');
    ```
    
    *План выполнения:*
    ```text
    Bitmap Heap Scan on t_books  (cost=21.03..1336.08 rows=750 width=33) (actual time=0.013..0.013 rows=1 loops=1)
   Recheck Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
   Heap Blocks: exact=1
   ->  Bitmap Index Scan on t_books_fts_idx  (cost=0.00..20.84 rows=750 width=0) (actual time=0.008..0.009 rows=1 loops=1)
         Index Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
    Planning Time: 0.765 ms
    Execution Time: 0.043 ms
   ```
    
    *Объясните результат:*
    
    Полнотекстовый GIN-индекс используется для быстрого поиска слова expert в title. Bitmap Index Scan на индексе отбирает подходящие строки, затем Bitmap Heap Scan перепроверяет их, но так как подходящих книг мало, проверяется всего один блок, поэтому выполнение запросабыстрое.

6. Удалите индекс:
    ```sql
    DROP INDEX t_books_fts_idx;
    ```

7. Создайте таблицу lookup:
    ```sql
    CREATE TABLE t_lookup (
         item_key VARCHAR(10) NOT NULL,
         item_value VARCHAR(100)
    );
    ```

8. Добавьте первичный ключ:
    ```sql
    ALTER TABLE t_lookup 
    ADD CONSTRAINT t_lookup_pk PRIMARY KEY (item_key);
    ```

9. Заполните данными:
    ```sql
    INSERT INTO t_lookup 
    SELECT 
         LPAD(CAST(generate_series(1, 150000) AS TEXT), 10, '0'),
         'Value_' || generate_series(1, 150000);
    ```

10. Создайте кластеризованную таблицу:
     ```sql
     CREATE TABLE t_lookup_clustered (
          item_key VARCHAR(10) PRIMARY KEY,
          item_value VARCHAR(100)
     );
     ```

11. Заполните её теми же данными:
     ```sql
     INSERT INTO t_lookup_clustered 
     SELECT * FROM t_lookup;
     
     CLUSTER t_lookup_clustered USING t_lookup_clustered_pkey;
     ```

12. Обновите статистику:
     ```sql
     ANALYZE t_lookup;
     ANALYZE t_lookup_clustered;
     ```

13. Выполните поиск по ключу в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     ```text
     Index Scan using t_lookup_pk on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.019..0.019 rows=1 loops=1)
        Index Cond: ((item_key)::text = '0000000455'::text)
        Planning Time: 0.221 ms
        Execution Time: 0.035 ms
     ```
     
     *Объясните результат:*
     
    Поиск использует индекс первичного ключа, который напрямую находит строку с нужным item_key без сканирования всей таблицы, поэтому выполнение запроса быстрое и затрагивает минимальное количество блоков.

14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     ```text
        Index Scan using t_lookup_clustered_pkey on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.071..0.072 rows=1 loops=1)
        Index Cond: ((item_key)::text = '0000000455'::text)
        Planning Time: 0.891 ms
        Execution Time: 0.116 ms
     ```
     
     *Объясните результат:*
     
    Поиск использует индекс первичного ключа так же, как и в обычной таблице. Кластеризация улучшает расположение строк на диске, но для одиночного точечного поиска это не сильно влияет, поэтому запрос остаётся быстрым.

15. Создайте индекс по значению для обычной таблицы:
     ```sql
     CREATE INDEX t_lookup_value_idx ON t_lookup(item_value);
     ```

16. Создайте индекс по значению для кластеризованной таблицы:
     ```sql
     CREATE INDEX t_lookup_clustered_value_idx 
     ON t_lookup_clustered(item_value);
     ```

17. Выполните поиск по значению в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ```text
     Index Scan using t_lookup_value_idx on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.033..0.034 rows=0 loops=1)
    Index Cond: ((item_value)::text = 'T_BOOKS'::text)
    Planning Time: 0.420 ms
     Execution Time: 0.051 ms
     ```
     
     *Объясните результат:*
     
    Поиск использует индекс по item_value, который сразу отбирает строки с нужным значением без сканирования всей таблицы, поэтому время выполнения чуть больше, чем при поиске по первичному ключу.

18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ```text
     Index Scan using t_lookup_clustered_value_idx on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.037..0.037 rows=0 loops=1)
    Index Cond: ((item_value)::text = 'T_BOOKS'::text)
    Planning Time: 0.317 ms
    Execution Time: 0.054 ms
     ```
     
     *Объясните результат:*
     
    Поиск использует индекс по item_value так же, как в обычной таблице, но благодаря кластеризации строки с близкими значениями расположены рядом на диске, поэтому проверка блоков происходит чуть быстрее.

19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
     *Сравнение:*
     
    Поиск использует индекс в обеих таблицах, количество проверяемых строк минимально и результат точный. Для одиночного поиска кластеризация практически не влияет, поэтому фактическое время может быть чуть больше или меньше, разницы в производительности нет.