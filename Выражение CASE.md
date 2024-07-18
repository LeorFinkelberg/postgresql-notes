Для реализации логики ветвления используется выражение `CASE`. Можно использовать два варианта выражения `CASE`:
- Выражение `CASE` с параметром,
- Выражение `CASE` с условием.
### Выражение CASE с параметром

Выражение `CASE` с параметром (под параметром понимается атрибут таблицы) имеет следующий синтаксис
```bash
CASE {parameter}
    WHEN {value1} THEN {res1}
   [WHEN {value2} THEN {res2}
     ...
    WHEN {valueN} THEN {resN}]
   [ELSE {res_ELSE}]
END
```

Если предложение `ELSE` отсутствует, то выражение `CASE` вернет результат `NULL`. Возвращаемый результат может быть _значением_ или _выражением_. Выражения `parameter` и `value` должны иметь один и тот же тип данных. Все возвращаемые значения должны иметь одинаковый тип данных.
```sql
SELECT
  department_id,
  employee_id,
  first_name,
  last_name,
  job_id,
  salary,
  CASE department_id
    WHEN 10 THEN 1000
    WHEN 30 THEN 1200
    WHEN 40 THEN 1500
    ELSE 500
  END AS bonus
FROM employees
WHERE department_id IN (10, 20, 30, 40)
ORDER BY department_id;
```

Вывести данные о сотрудниках и размере их премии, которая задана как часть заработной платы, размер которой зависит от отдела, где работает сотрудник
```sql
SELECT
  department_id,
  employee_id,
  first_name,
  last_name,
  job_id,
  salary,
  CASE deprment_id
    WHEN 10 THEN 0.1 * salary
    WHEN 30 THEN 0.2 * salary
    WHEN 40 THEN 0.3 * salary
  END AS bonus
FROM employees
WHERE department_id IN (10, 20, 30, 40)
ORDER BY department_id;
```

### Выражение CASE с условием

Синтаксис
```bash
CASE
  WHEN {cond1} THEN {res1}
 [WHEN {cond2} THEN {res2}
  ...
  WHEN {condN} THEN {resN}]
 [ELSE {res_ELSE}]
END;
```

Вывести данные о сотрудниках и размере их премии, которая зависит от зарплаты сотрудника
```sql
SELECT
  department_id,
  employee_id,
  first_name,
  last_name,
  job_id,
  salary,
  CASE 
    WHEN salary > 10000 THEN 5000
    WHEN salary > 7000 THEN 3000
    WHEN salary > 5000 THEN 2000
    ELSE 1000
  END AS bonus
FROM employees
WHERE department_id in (10, 20, 30, 40)
ORDER BY department_id;
```