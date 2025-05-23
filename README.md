# Первая задача 

```sql
select name from t1 where id = 50000;
```

Запускаем запрос с анализом:
```sql
explain analyze select name from t1 where id = 50000;
```
![Pasted image 20250313085642](https://github.com/user-attachments/assets/98e85a0d-709d-466d-b21a-0e164d899184)

Результаты выполнения показывают, что используется последовательное сканирование (Seq Scan), несмотря на то, что нам нужна всего одна строка из 10 миллионов - высокая селективность (нужно извлечь одну строку из миллионов) — это явный сигнал к тому, что следует использовать индекс.

Теперь наша задача — создать индекс на колонку `id` таблицы `t1`.  
Нужно выбрать подходящий тип индекса. PostgreSQL поддерживает несколько типов: `Hash`, `B-tree`, `GiST`, `SP-GiST`, `GIN`, `RUM`, `BRIN` и др. Выбор зависит от типа запроса и данных.

В нашем случае используется **поиск по точному значению** (`id = 50000`), поэтому, помимо `B-tree`, подойдёт и **Hash-индекс**. Он тоже предназначен для поиска по равенству и может быть чуть эффективнее в некоторых ситуациях. Однако стоит учитывать:
- До PostgreSQL 10 Hash-индексы работают без WAL
- Они **не подходят** для диапазонных запросов (`BETWEEN`, `<`, `>`).
```sql
select a.amname, p.name, pg_indexam_has_property(a.oid,p.name)
from pg_am a,
unnest(array['can_order','can_unique','can_multi_col','can_exclude']) p(name)
where a.amname = 'hash' order by a.amname;
```
![Pasted image 20250416163213](https://github.com/user-attachments/assets/4fd17e11-b978-46fa-a3dd-7f632cdad97c)

- В большинстве случаев `B-tree` остаётся универсальнее, но если точно знаешь, что запросы всегда будут вида `WHERE id = ...`, можно использовать `Hash` (Hash-индекс может быть много быстрее [Hash indexes are faster than Btree indexes?](https://amitkapila16.blogspot.com/2017/03/hash-indexes-are-faster-than-btree.html))

В данной ситуации используется **Hash-индекс**, так как запрос выполняется по точному совпадению значения (`id = 50000`), а значит, такой тип индекса подходит.
Создание индекса:
```sql
create index on t1 using hash(id);
```
![Pasted image 20250416164348](https://github.com/user-attachments/assets/697ae8d1-0084-4c54-980d-6044d369e0a3)

После создания индекса повторно запускаем `EXPLAIN ANALYZE`:

![Pasted image 20250416164440](https://github.com/user-attachments/assets/5c05923b-54bb-4f7e-9e00-4f7d0977ac9d)

Время выполнения запроса сократилось примерно до 5 мс, что на порядки быстрее, чем без индекса

# Вторая задача

```sql
select max(t2.day) from t2 left join t1 on t2.t_id = t1.id and t1.name like 'a%';
```
В этом запросе происходит `LEFT JOIN` таблицы `t2` с `t1`, при этом в результате используется только `max(t2.day)`. Ни одна колонка из `t1` не участвует в `SELECT`. `LEFT JOIN` здесь лишний — его можно смело удалить без потери смысла, что также сократит издержки на выполнение запроса.

Переписанный запрос
```sql
select max(day) from t2;
```
Анализуем новый запрос
```sql
explain analyze select max(day) from t2;
```
![Pasted image 20250416165216](https://github.com/user-attachments/assets/e4c0ccb8-a709-4b43-8bc6-b63545ab36df)

Чтобы получить максимальное значение из столбца `day`, PostgreSQL выполняет полный скан всей таблицы. При этом выбирается только одна строка, что говорит о высокой селективности. Такой подход неэффективен при большом объёме данных.
Чтобы ускорить выполнение запроса, можно создать индекс, который поможет быстро находить максимальное значение.
Поскольку нас интересует именно **максимальное** значение, создаём индекс по полю `day` в порядке **убывания**. Это позволит СУБД сразу обратиться к первой записи в индексе — и она уже будет нужной.

```sql
create index on t2 (day desc);
```

![Pasted image 20250416165303](https://github.com/user-attachments/assets/0d242709-a180-422e-85e6-5e2e84b68c3f)

После создания индекса повторно запускаем `EXPLAIN ANALYZE`:

![Pasted image 20250416165407](https://github.com/user-attachments/assets/9d0162c1-fbff-4d1d-bfb5-940e774ae7c3)

Время выполнения запроса сократилось примерно до 5 мс, что на порядки быстрее, чем без индекса
# Третья задача 

```sql
select day from t2 where t_id not in ( select t1.id from t1);
```

Пробуем проанализировать запрос, но он оказывается слишком тяжёлым — выполнять тесты на таких объёмных таблицах неэффективно.  
Поэтому создаём отдельную таблицу с 10% данных для более удобного и быстрого тестирования.
```sql
create table t1_test as                                                            
select * from t1
limit 10000;

create table t2_test as                                                            
select * from t2
limit 5000;
```
![Pasted image 20250416170600](https://github.com/user-attachments/assets/155e8985-330d-42cf-965c-571fc64bdbf7)

Проводим анализ запроса на новой, укороченной таблице
```sql
explain analyze select day from t2_test where t_id not in ( select t1_test.id from t1_test );
```
![Pasted image 20250416170733](https://github.com/user-attachments/assets/08bdfe7f-75ed-47b2-a8d2-ae081e3b5f13)

С помощью плана выполнения мы узнали следующее:
- Сначала выполняется последовательный скан таблицы `t1` с хешированием значения `id`.
- Значения `t_id` из `t2` проверяются на совпадение с этим хешем, и если совпадения нет, тогда запись исключается.

На первый взгляд может быть неочевидно, где именно можно оптимизировать запрос. Но мы знаем, что антисоединение (`ANTI JOIN`) можно выразить через конструкцию `NOT EXISTS`

> *"Антисоединение возвращает строки из первого набора, только если для них не нашлось соответствия во втором. То есть если во втором наборе находится подходящая строка, то строка из первого набора в результат не попадёт — и дальше её можно не проверять. Это позволяет эффективно вычислять предикат `NOT EXISTS`."*  
> — _Документация PostgreSQL (глава 17, стр. 441)_

Переписываем запрос с использованием `NOT EXISTS`
```sql
select day from t2 where not exists ( select 1 from t1 where t_id = id);
```

На первый взгляд может показаться, что производительность стала хуже — ведь теперь подзапрос зависит от внешнего (коррелированный подзапроc).  
Но с другой стороны, это даёт возможности для дальнейшей оптимизации, так как `NOT EXISTS` в PostgreSQL может быть преобразован в более эффективные формы, особенно при наличии индексов
```sql
explain analyze select day from t2_test where not exists ( select 1 from t1_test where t1_test.id = t2_test.t_id);
```
![Pasted image 20250416171509](https://github.com/user-attachments/assets/e1545dbe-8373-43a7-942a-66fcdfbf1842)

Для соединения хеширования  
- Первым этапом в памяти строится хеш-таблица: строки `t1` читаются последовательно, и для каждой из них вычисляется хеш-функция от значений полей, входящих в условие соединения (в нашем примере это числовые идентификаторы `id`). Размер хеш-таблицы в памяти ограничен значением `work_mem × hash_mem_multiplier`.  Наилучшая эффективность достигается, если вся хеш-таблица помещается в этот объём памяти целиком (источник — курс QPT). Но в нашем примере, скорее всего, места в памяти не будет хватать.
- Сопоставление: на втором этапе мы последовательно читаем второй набор строк. Если с таким же хеш-кодом мы нашли пару — происходит сопоставление.

Наша задача теперь — настроить параметры `work_mem` и `hash_mem_multiplier`,  
чтобы выполнить запрос за время менее 10 секунд
```sql
alter system set work_mem to '128MB';
alter system set hash_mem_multiplier to '12';
select pg_reload_conf();
```
![Pasted image 20250416172456](https://github.com/user-attachments/assets/469b4997-2049-4fb7-ad86-1a4c64e2cf01)

Cнова анализуем запрос
```sql
explain analyze select day from t2 where not exists ( select 1 from t1 where t1.id = t2.t_id);
```
![Pasted image 20250416172655](https://github.com/user-attachments/assets/7c42c693-c481-4534-8ac6-95b6b59db52f)

Время выполнения запроса сократилось примерно 7 секунд.
# Четвертая задача

```sql
explain analyze select day from t2 where t_id in ( select t1.id from t1 where t2.t_id = t1.id) and day > to_char(date_trunc('day',now()- '1 months'::interval),'yyyymmdd');
```
Анализуем данный запрос
![Pasted image 20250416172942](https://github.com/user-attachments/assets/97486712-dc61-4f59-b958-97a0e76d1076)

Здорово, что мы **ничего не меняли**, а запрос уже выполняется примерно за **10 секунд** — можно было бы сказать, что задача решена. Но это не наша цель - нам важно понять, почему запрос изначально работал медленно, а не просто добиться результата.
Убираем созданные индексы
```sql
drop index t1_id_idx;
drop index t2_day_idx;
```
![Pasted image 20250416175133](https://github.com/user-attachments/assets/fa49ffc8-aac9-4aae-ad6b-0fcdf7a1746a)

 Анализируем запрос в укороченных таблицах
```sql
explain analyze select day from t2_test where t_id in ( select t1_test.id from t1_test where t2_test.t_id = t1_test.id) and day > to_char(date_trunc('day',now()- '1 months'::interval),'yyyymmdd');
```
![Pasted image 20250416180300](https://github.com/user-attachments/assets/564bf5c6-9c31-4790-892d-41fab6031ef3)

Из плана выполнения видно:
- Выполняется последовательный скан всей таблицы `t2_test`.  
- Применяются две фильтрации:  
 - по дате (найти записи за последний месяц),  
 - и проверка, есть ли связанная запись в `t1_test`.
 
Первая фильтрация по дате даёт высокую селективность → можно заменить последовательный скан на индексный.
Создаём индекс по дате:
```sql
create index on t2 (day desc);
```
![Pasted image 20250416181920](https://github.com/user-attachments/assets/16117438-0a70-4ad9-8e59-953cea313adf)

Вторая часть (поиск `id` в `t1`) тоже может быть ускорена, если использовать индекс:
```sql
create index on t1 using hash(id);
```
![Pasted image 20250416181644](https://github.com/user-attachments/assets/8075eedb-3403-4255-b88b-a6d33e2778bc)

Теперь повторно анализируем запрос, но уже не в укороченных таблицах, а на реальных данных
```sql
explain analyze select day from t2 where t_id in ( select t1.id from t1 where t2.t_id = t1.id) and day > to_char(date_trunc('day',now()- '1 months'::interval),'yyyymmdd');
```
![Pasted image 20250416182043](https://github.com/user-attachments/assets/a234489c-5307-4e9f-9914-e362f37b6aa2)

Время выполнения запроса сократилось примерно 6 мс, добился результата

