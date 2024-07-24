При создании сложных запросов, содержащих большое число подзапросов, рекомендуется использовать оператор `WITH`. В этом операторе подзапросам присваиваются имена. Эти имена используются в основном запросе как имена таблиц. Использование `WITH` улучшает производительность и облегчает чтение запроса.
```sql
WITH {subquery1_name} AS (subquery1),
     {subquery2_name} AS (subquery2),
     ...
SELECT col1, col2, ...
FROM {table} | {subquery_name} | {view}
WHERE {conds};
```

Найти отдел с максимальной заработной платой, используя оператор `WITH`
```sql
WITH t1 AS ( -- CTE-1
  SELECT
    department_id,
    sum(salary) AS sum_salary_dep
  FROM employees
  GROUP BY 1
),
t2 AS ( -- CTE-2
  SELECT max(sum_salary_dep) AS max_sum_salary_dep
  FROM t1
)
SELECT * FROM t1 
WHERE sum_salary_dep = (SELECT max_sum_salary_dep FROM t2);
```
То есть здесь конструкция `(SELECT max_sum_salary_dep FROM t2)` играет роль `t2.max_sum_salary_dep`.

Вывести данные о клиентах, у которых средняя сумма заказа превышает общую среднюю сумму одного заказа
```sql
WITH t1 AS ( -- средняя сумма корзины каждого пользователям
  SELECT
    customer_id,
    AVG(quantity * unit_price) AS avg_price_customer
  FROM orders JOIN order_items USING (order_id)
  GROUP BY 1
),
t2 AS ( -- средняя сумма корзины по всем пользователям
  SELECT 
    AVG(quantity * unit_price) AS avg_price
  FROM order_items
)
SELECT
  customer_id,
  c_name
FROM customers c
WHERE ( -- связанный скалярный подзапрос 
  SELECT
    avg_price_customer
  FROM t1
  WHERE customer_id = c.customer_id
) > (SELECT avg_price FROM t2) -- Как бы t2.avg_price
```
Здесь нужно пройтись по строкам таблицы `customers`, забрать значение атрибута `c.customer_id`  и передать его в подзапрос. Подзапрос отфильтрует строку из таблицы `t1` и вернет значение атрибута `avg_price_customer` (то есть среднюю сумму корзины для указанного пользователя), которое будет сравниваться со значением атрибута `avg_price` таблицы `t2`.

NB! Из-под подзапроса мы можем ссылаться на псевдоним таблицы `customers`.
