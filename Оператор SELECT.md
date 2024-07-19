Вычислить мощность какой-нибудь группы без явной группировки можно так
```sql
SELECT count(*) FROM (
  SELECT * 
  FROM orders
  WHERE status = 'Pending'
) t; -- обязательно нужно задать псевдоним для подзапроса
```

Вывести данные о сотрудниках, которые работают в отделе 30, расположив их в порядке убывания
```sql
SELECT
  employee_id,
  first_name,
  last_name
FROM employees
WHERE department_id = 30
ORDER BY rating_e DESC; -- атрибут сортировки не обязательно должен быть в списке SELECT
```
Вывести имена и адрес клиентов, столбец `address` которых содержит символ `_`
```sql
SELECT c_name, address
FROM Customers
WHERE address LIKE '%/_' ESCAPE '/'
```
Здесь для поиска символа `_` при построении шаблона используется опция `ESCAPE '/'`. Символ, который в шаблоне будет располагаться после `/`, будет рассматриваться как символ поиска. Вместо символа `/` можно использовать и другие символы, например `!`. 

Вывести количество строк, в которых атрибут `salary`, принимает значение из диапазона от 6000 до 8000
```sql
SELECT count(*) FROM (
    SELECT employee_id, first_name, department_id
    FROM employees
    -- или не принадлежит диапазону
    -- WHERE salary NOT BETWEEN 6000 AND 8000
    WHERE salary BETWEEN 6000 AND 8000
) t; -- обязательно нужен псевдоним для подзапроса
```

Список значений для `IN` может формироваться в результате выполнения оператора `SELECT` (подзапроса).

NB: Следует иметь в виду то, что если список значений в `IN` будет содержать `NULL`, то результат выполнения оператора не будет содержать строк, у которых проверяемый столбец имеет значение `NULL`, так как результат сравнения с `NULL` имеет значение НЕ ОПРЕДЕЛЕНО (UNKNOWN).

Особенность использования оператора `NOT IN (...)`. Путь требуется вывести данные о сотрудниках, которые не работают в отделах с определенными номерами
```sql
SELECT employee_id, first_name, last_name, department_id
FROM employees
WHERE department_id NOT IN (30, 50, 60, 100, NULL);
```
Результат выполнения этого запроса ==не будет содержать строк!== Это произойдет потому, что оператор
```sql
X NOT IN (A1, A2, ..., AN)
```
эквивалентен выражению
```sql
X <> A1 AND X <> A2 AND ... AND X <> AN
```
Если один из `A{i}` будет иметь значение `NULL`, то результат этого выражения будет иметь значение НЕ ОПРЕДЕЛЕНО (UNKNOWN).

Получить данные о сотрудниках, у которых неизвестен номер руководителя
```sql
SELECT employee_id, first_name, last_name, department_id
FROM employees
WHERE manager_id IS NULL;
```

В предложении `SELECT`, кроме списка столбцов таблиц, участвующих в запросе, могут присутствовать _вычисляемые столбцы_, которые представляют собой выражения, состоящие из имен столбцов, констант, функций и арифметических операций.

Вывести `last_name` сотрудников, у которых `last_name` содержит две и более буквы `e`
```sql
SELECT last_name FROM employees WHERE last_name ~* '.*e.*e';
```

Вывести для сотрудника `employee_id = 145` период времени между датой приема на работу и сегодняшним днем
```sql
SELECT AGE(hire_date) FROM employees WHERE employee_id = 145;
```

Результат, который возвращает функция `AGE()`, имеет тип `INTERVAL`. Используя функцию `EXTRACT()`, можно выбрать и использовать определенную часть этого результата.

Вывести данные о договорах, которые были оформлены в воскресенье (численное значение дня недели)
```sql
-- dow = day of week
SELECT * FROM orders WHERE EXTRACT(dow FROM order_date) = 0;
```

Вывести `employee_id` сотрудников, работающих в 30-ом отделе, и суммарную зарплату каждого сотрудника за весь период их работы. Данные расположить в порядке убывания суммарной зарплаты
```sql
SELECT
  employee_id,
  salary * (
    EXTRACT(year FROM AGE(hire_date)) * 12 + EXTRACT(month FROM AGE(hire_date))
  ) AS sum_salary
FROM employees
WHERE department_id = 30
ORDER BY sum_salary DESC;
```

Вывести данные о сотрудниках, которые были приняты на работу в 1999 году, в воскресенье (SUNDAY)
```sql
SELECT
  employee_id,
  first_name,
  last_name,
  hire_date,
  TO_CHAR(hire_date, 'DAY')
FROM employees
WHERE EXTRACT(year FROM hire_date) = 1999 AND RTRIM(TO_CHAR(hire_date, 'DAY')) = 'SUNDAY';
```

В этом примере следует обратить внимание на необходимость использования функции `RTRIM()` для удаления правых пробелов. Это необходимо сделать, так как функция `TO_CHAR(hire_date, 'DAY')` возвращает строку, содержащую количество символов, равное длине самого длинного названия дня недели -- `WEDNESDAY`, дополняя более короткие названия пробелами справа.

Вывести данные о полной зарплате сотрудников, которые работают в отделе 30. Значение полной зарплаты равно `salary * (1 + commission_pct)`
```sql
SELECT
  employee_id,
  first_name,
  last_name,
  salary,
  commission_pct AS com_pct,
  ROUND(COALESCE(salary * (1 + commission_pct), salary)) AS total_salary
FROM employees
WHERE department_id = 30
ORDER BY total_salary DESC;
```

Без использования функции `COALESCE()` полная зарплата сотрудников, у которых `commission_pct` имеет значение `NULL`, также имела бы значение `NULL`.

Функцию `COALESCE()` для бинарного случая можно интерпретировать как "Если один элемент из пары равен `NULL`, то возвращаем тот, что не `NULL`.

Вывести данные о полной зарплате сотрудников, которые работают в отделе 30 и полная зарплата которых больше 3000. Данные расположить в порядке убывания полной зарплаты
```sql
SELECT
  employee_id,
  first_name,
  last_name,
  salary,
  ROUND(COALESCE(salary * (1 + commission_pct), salary)) AS total_salary
FROM employees
WHERE department_id = 30 AND COALESCE(salary * (1 + commission_pct), salary) > 3000
ORDER BY total_salary DESC;
```

NB! В этом запросе следует обратить внимание на то, что пвседонимы столбцов (total_salary) можно использовать в предложении `ORDER BY`, но нельзя использовать в предложении `WHERE` [[Литература#^af0aa6]]<c. 123>

Для сотрудников, зарплата которых больше 1200, вывести столбец, который должен содержать полное имя сотрудника, зарплату и несколько звездочек (`*`), по одной звездочке на каждые $1000 зарплаты
```sql
SELECT
  CONCAT(first_name, ' ', last_name) AS full_name,
  salary,
  LPAD('', DIV(salary, 1000)::int, '*')
FROM employee
WHERE salary > 1200;
```

Вывести названия городов (city), в которых 4-ая буква `t`, а последняя `e`
```sql
SELECT city FROM locations WHERE city ~* '^...t.*e';
```

Вывести даты текущего месяца, которые выпадают на воскресенье
```sql
SELECT idx + '2024-07-01'::date
FROM GENERATE_SERIES(0, 30) t(idx)
WHERE EXTRACT(dow, idx + '2024-07-01'::date) = 0;
```

Вывести данные об отделах, названия которых состоят более чем из одного слова. Результат выполнения запроса должен содержать: `deprment_id`, `department_name`, второе слово в названии отдела
```sql
SELECT
  deptment_id,
  deprtment_name
  ltrim(substring(department_name, strpos(department_name, ' '))) AS second_word
FROM departments
WHERE department_name ~* '.*\s.*';
```

Найти сотрудника с наибольшей зарплатой
```sql
SELECT
  employee_id,
  salary
FROM employees
WHERE salary = (SELECT MAX(salary) FROM employee)  -- подзапрос
```

NB! Списки столбцов в предложениях `SELECT` и `GROUP BY` должны совпадать [[Литература#^af0aa6]]<c. 131>