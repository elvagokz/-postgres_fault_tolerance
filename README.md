Улучшение времени запроса
=========================

## Старый запрос
```
explain analyze verbose
SELECT body 
  FROM posts 
 WHERE body ILIKE '%postgres%awesome%'
    OR body ILIKE '%postgres%amazing%';
                                                           QUERY PLAN                                                            
---------------------------------------------------------------------------------------------------------------------------------
 Seq Scan on public.posts  (cost=0.00..28983.08 rows=27 width=32) (actual time=240.031..5870.792 rows=40 loops=1)
   Output: body
   Filter: ((posts.body ~~* '%postgres%awesome%'::text) OR (posts.body ~~* '%postgres%amazing%'::text))
   Rows Removed by Filter: 220354
 Planning Time: 4.118 ms
 Execution Time: 5870.866 ms
(6 rows)
```
## Новый запрос
```
# Создаём расширения pg_trgm для использования trigram
CREATE EXTENSION IF NOT EXISTS pg_trgm;
# Создаём индекс GIN для столбца body c gin_trgm_ops для работы с trigram
CREATE INDEX ON posts USING gin (body gin_trgm_ops);
explain analyze verbose
SELECT body 
  FROM posts 
 WHERE body ILIKE '%postgres%awesome%'
    OR body ILIKE '%postgres%amazing%';

                                                           QUERY PLAN                                                            
---------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on public.posts  (cost=296.35..467.68 rows=44 width=32) (actual time=9.632..19.273 rows=40 loops=1)
   Output: body
   Recheck Cond: ((posts.body ~~* '%postgres%awesome%'::text) OR (posts.body ~~* '%postgres%amazing%'::text))
   Rows Removed by Index Recheck: 20
   Heap Blocks: exact=59
   ->  BitmapOr  (cost=296.35..296.35 rows=44 width=0) (actual time=8.971..8.971 rows=0 loops=1)
         ->  Bitmap Index Scan on posts_body_idx  (cost=0.00..148.17 rows=22 width=0) (actual time=5.775..5.775 rows=36 loops=1)
               Index Cond: (posts.body ~~* '%postgres%awesome%'::text)
         ->  Bitmap Index Scan on posts_body_idx  (cost=0.00..148.17 rows=22 width=0) (actual time=3.190..3.190 rows=24 loops=1)
               Index Cond: (posts.body ~~* '%postgres%amazing%'::text)
 Planning Time: 1.836 ms
 Execution Time: 19.320 ms
(12 rows)
```
Индексация при нагруженной БД

=========================
1. Прежде чем создавать индекс в продуктивном окружении, лучше сделать:
   + Провести тестирование на тестовой базе.
   + Сделать резервную копию данных.
   + Использовать параметр CONCURRENTLY для создания индекса, если это возможно, чтобы избежать блокировки таблицы.

 Но лучше создание индексов лучше выполнять во вне-пиковое время, чтобы минимизировать воздействие на работу системы.


2. Создание индексов при высокой нагрузке из-за:
   + Блокировки таблицы при индексации.
   + Ещё большая нагрузка на процессор.
   + Риск получить некоректные данные.

     
3. Параметры PostgreSQL, которые можно изменить для ускорения создания индекса при повышенной нагрузке:
   + max_parallel_workers_per_gather чтобы ускорить операции, выполняемые параллельно.
   + maintenance_work_mem для увеличения выделяемой памяти.
   + autovacuum для удаления мёртвых данных
