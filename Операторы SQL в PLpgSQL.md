В PL/pgSQL можно использовать следующие виды операторов SQL:
- Операторы манипулирования данными -- `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `MERGE`.
- Операторы управления транзакциями -- `COMMIT`, `ROLLBACK`, `SAVEPOINT`.

Если нужно выполнить несколько операторов манипулирования данными, то каждый оператор является отдельным запросом к базе данных и отправляется на сервер отдельно от других операторов. Результаты выполнения каждого оператора отправляются обратно клиенту. Выполнение нескольких операторов манипулирования данными приводит к множественным передачам в обоих направлениях, значительно увеличивая сетевой трафик.

Если эти операторы _объединить в блок_, то они отправляются на сервер _как единое целое_. Сервер _выполняет_ эти операторы и отправляет результаты их выполнения обратно клиенту _как единое целое_. Этот процесс более эффективен и _занимает существенно меньшее время_, чем выполнение каждого оператора независимо от других [[Литература#^af0aa6]]<c. 270>.

### Использование оператора `SELECT`

Оператор `SELECT` можно использовать для присвоения значений переменным
```sql
SELECT {list_of_cols_or_expressions}
INTO [STRICT] {list_of_vars}
FROM {list_of_data_sources}
WHERE {cond}
```

Запрос должен возвращать ровно одну строку. Список столбцов и список переменных должны содержать одинаковое количество элементов с совместимыми типами данных.

Если слово `STRICT` будет отсутствовать, то в случае если запрос не вернет строк, переменным будет присвоено значение `NULL`, а если запрос вернет несколько строк, то переменным будет присвоено значение первой строки. Ошибки `NO_DATA_FOUND` и `TOO_MANY_ROWS` в этом случае не возникают. Если же указать `... INTO STRICT`, то, например, в случае, когда запрос не возвращает никаких строк, возникнет ошибка "SQL Error [P0002]: ERROR: query returned no rows".

Извлечение значения одного столбца
```sql
DO $$
DECLARE
  v_emp_id employees.employee_id%TYPE := 120;
  v_emp_salary employees.salary%TYPE;
BEGIN
  SELECT salary INTO v_emp_salary
  FROM employees
  WHERE employee_id = v_emp_id;
  -- 
  RAISE NOTICE 'Results: ';
  RAISE NOTICE 'employee_id = %', v_emp_id;
  RAISE NOTICE 'salary = %', v_emp_salary;
END $$;
```

Если бы нужно было присвоить значения, скажем, двум переменным, то можно было сделать так
```sql
DO $$
DECLARE
  var1 customers.c_name%type;
  var2 customers.credit_limit%type;
BEGIN
  SELECT c_name, credit_limit INTO var1, var2
  FROM customers
  WHERE customer_id = 2;
...
```

Значение, присваиваемое переменной, может быть результатом обработки данных, например значением, возвращаемым агрегатной функцией
```sql
DO $$
DECLARE
  v_dep_id employees.department_id%type := 80;
  v_max_salary employees.salary%type;
BEGIN
  SELECT MAX(salary) INTO v_max_salary
  FROM employees
  WHERE department_id = v_dep_id;
--
  RAISE NOTICE 'Results: ';
  RAISE NOTICE 'department_id = %', v_dep_id;
  RAISE NOTICE 'MAX(salary) = %', v_max_salary;
END $$;
```

Можно извлечь и присвоить переменным значения нескольких столбцов
```sql
DO $$
DECLARE
  v_emp_id employees.employee_id%TYPE := 120;
  v_first_n employees.first_name%TYPE;
  v_last_n employees.last_name%TYPE;
  v_job employees.job_id%TYPE;
  v_emp_salary employees.salary%TYPE;
BEGIN
  SELECT
    employee_id,
    first_name,
    last_name,
    job_id,
    salary
  INTO
    v_emp_id,
    v_first_n,
    v_last_n,
    v_job,
    v_emp_salary
  FROM employees
  WHERE employee_id = v_emp_id;
  ...
END $$;
```

В подобных случаях удобнее использовать переменную, имеющую тип `ROWTYPE`
```sql
DO $$
DECLARE
  v_emp_id employees.employee_id%TYPE := 120;
  v_emp employees%ROWTYPE;
BEGIN
  SELECT * INTO v_emp
  FROM employees
  WHERE employee_id = v_emp_id;
-- 
  RAISE NOTICE 'Results: ';
  RAISE NOTICE '% %', v_emp.last_name, v_emp.salary;
END $$;
```

### Использование оператора `INSERT`

Правила использования операторов `INSERT` в блоках PL/pgSQL соответствуют правилам использования DML операторов PostgreSQL, но в условных выражениях и списке значений можно использовать переменные, которые были определены и инициализированы в блоке.

Вставка в таблицу `products` результатов выполнения оператора `SELECT`
```sql
DO $$
BEGIN
  INSERT INTO products(product_id, product_name, rating_p, price)
  SELECT NEXTVAL('prod_id_seq'), product_name, rating_p, price
  FROM products WHERE rating_p = 3;
END $$;
```

Использование последовательности в этих примерах позволяет обеспечить уникальность значений ключевого столбца `product_id`. 

### Использование оператора `UPDATE`

Синтаксис и правила использования операторов `UPDATE` в блоках PL/pgSQL соответствуют правилам использования DML операторов PostgreSQL, но в условных выражениях и операторах присваивания можно использовать переменные, которые были определены и инициализированы в блоке.

Изменение значения столбца `rating_p` в таблице `products`
```sql
DO $$
DECLARE
  v_prod_id products.product_id%TYPE := 6;
  v_add_rating products.rating_p%TYPE := 2;
  v_prod products%ROWTYPE;
BEGIN
  UPDATE products
  SET rating_p = rating_p + v_add_rating
  WHERE product_id = v_prod_id;
  --
  SELECT * INTO v_prod
  FROM products
  WHERE product_id = v_prod_id;
  --
  RAISE NOTICE 'Results: ';
  RAISE NOTICE 'product_id = %', v_prod.product_id;
  RAISE NOTICE 'rating_p = %', v_prod.rating_p;
END $$;
```

Чтобы привести первую букву каждого слова во предложении удобно использовать функцию `INITCAP`
```sql
SELECT INITCAP('hi THOMAS'); -- Hi Thomas
```

Например в Python мы сделали так
```python
" ".join(elem.capitalize() for elem in "hi THOMAS".split())  # Hi Thomas
```

### Использование оператора `DELETE`

Правила использования этого оператора в блоках PL/pgSQL соответствуют правилам использования DML операторов PostgreSQL.

Удаление из таблицы `products` данных о товарах, имеющих рейтинг 2
```sql
DO $$
DECLARE
  v_prod_rating products.rating_p%TYPE := 2;
BEGIN
  DELETE FROM products
  WHERE rating_p = v_prod_rating;
END $$;
```



