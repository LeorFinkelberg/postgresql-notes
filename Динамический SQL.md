Использование динамического SQL существенно повышает гибкость разрабатываемых приложений и позволяет обойти некоторые ограничения языка PL/pgSQL. Например, в блоках PL/pgSQL нельзя использовать команды языка определения данных (DDL), но в код блока PL/pgSQL можно вставить динамический оператор, содержащий такую команду, и выполнить его. 

Следует иметь в виду, что ==динамические операторы== выполняются ==медленнее ==статических и усложняют процессы отладки и сопровождения программ.
### Выполнение динамических операторов `SELECT`

Динамические операторы SQL создаются в виде символьных строк с использованием параметров, указанных в процессе выполнения программы. Эти строки должны содержать допустимые операторы SQL.

Команда `EXECUTE` осуществляет анализ и выполнение динамического оператора, содержащегося в символьной строке.

Синтаксис команды `EXECUTE`
```sql
EXECUTE {query} [INTO [STRICT] {list_of_vars}]
[USING {value_of_params}];
```
Здесь
- `{query}` -- строка или переменная типа `TEXT`, содержащая оператор SQL, который должен быть выполнен,
- `INTO [STRICT] {list_of_vars}` -- одна или несколько переменных, которым присваивается результат выполнения запроса. Если указано `STRICT`, то выдается сообщение об ошибке, в случае если оператор не возвращает ровно одну строку,
- `USING` -- содержит значения входных параметров, на которые можно ссылаться в операторе: `$1`, `$2` и т.д.

Результат, возвращаемый из динамического оператора, может быть:
- скалярным,
- представлять собой запись 
- или состоять из нескольких строк.

Пример
```sql
DO $$
DECLARE 
  v_department_id INTEGER := 110;
  v_ret_value TEXT;
BEGIN
  -- Динамический SQL-запрос
  EXECUTE 'SELECT first_name FROM employees WHERE employee_id = $1'
  INTO v_ret_value
  USING v_department_id;
  RAISE NOTICE 'employees_id = %, first_name = %', v_department_id, v_ret_value;
END $$;
```

Динамический оператор `SELECT` может вернуть всю строку таблицы. В этом случае переменная, которой присваивается результат, должна иметь составной тип, структура которого совпадает со структурой таблицы, из которой извлекается строка.
```sql
DO $$
DECLARE
  sql_query TEXT;
  v_rec employees%ROWTYPE;
  v_department_id employees.department_id%TYPE := 110;
BEGIN
  sql_query := 'SELECT * FROM employees WHERE department_id = $1'
  EXECUTE sql_query
  INTO v_rec
  USING v_department_id;
  --
  RAISE NOTICE 'employee_id = %, fist_name = %', v_department_id, v_rec.first_name;
END $$;
```

Если в таблице `employees` будет несколько строк, то оператор `SELECT` вернет несколько строк и возникнет ошибок. В этом случае переменная, которая присваивается результат, должна иметь тип `REFCURSOR` [[Литература#^af0aa6]]<c. 443>.
```sql
DO $$
DECLARE
  sql_query TEXT;
  v_department_id employees.department_id%TYPE;
  v_rec employees%ROWTYPE;
  v_refcur REFCURSOR; -- объявление курсорной переменной
BEGIN
  sql_query := 'SELECT * FROM employees WHERE first_name = $1';
  -- Открытие курсорной переменной
  OPEN v_refcur FOR EXECUTE sql_query
  USING v_department_id;
  --
  LOOP FETCH v_refcur INTO v_rec;
    EXIT WHEN NOT FOUND;
    RAISE NOTICE '...';
  END LOOP;
  CLOSE v_refcur; -- закрытие курсорной переменной
END $$;
```

В этом примере был использован _динамический курсор_, который содержит результат выполнения динамического запроса. Синтаксис использования _курсорных переменных_ в динамических запросах
```sql
OPEN {cursor_var}
FOR EXECUTE {query} [USING {value_of_params}];
CLOSE {cursor_var};
```

Содержимое курсорной переменной можно записать в массив или в таблицу.
```sql
DO $$
DECLARE
  sql_query TEXT;
  v_fist_name TEXT := 'Jhon';
  v_rec employees%ROWTYPE;
  v_refcur REFCURSOR;
BEGIN
  sql_query := 'SELECT * FROM employees 
                WHERE first_name = $1';
  --
  OPEN v_refcur FOR EXECUTE sql_query
  USING v_first_name;
  --
  LOOP
    FETCH v_refcur INTO v_rec;
    EXIT WHEN NOT FOUND;
    INSERT INTO employees(employee_id, first_name, last_name, job_id, salary)
    VALUES (v_rec.employee_id, v_rec.first_name, v_rec.last_name, v_rec.salary);
  END LOOP;
  CLOSE v_refcur;
END $$;
```
### Использование динамических DML-операторов

Динамический DML-оператор может содержать предложение `RETURNING` для возврата данных о строке, к которой он был применен. Оператор, который содержит это предложение, должен изменять _только одну строку_, в противном случае возникнет ошибка. При отсутствии предложения `RETURNING` динамический DML-оператор будет выполнен, но не вернет результат.
```sql
DO $$
DECLARE
  sql_query TEXT;
  v_emp_id_in INTEGER := 107;
  v_emp_id_out INTEGER; 
  v_add_salary NUMERIC(10, 2) := 1000;
  v_emp_salary NUMERIC(10, 2);
  v_f_name TEXT;
  v_job_id TEXT;
BEGIN
  sql_query := 'UPDATE employees SET salary = salary + $1
                WHERE employee_id = $2
                RETURNING employee_id, first_name, job_id, salary';
  -- 
  EXECUTE sql_query
  INTO v_emp_id_out, v_f_name, v_job_id, v_emp_salary
  USING v_add_salary, v_emp_id_in;
  --
  RAISE NOTICE '...';
END $$;
```

Пример формирования и выполнения динамического оператора `DELETE`, который использует имена таблиц и столбцов в качестве значений переменных
```sql
DO $$
DECLARE
  sql_query TEXT;
  v_tab_name TEXT := 'employees';
  v_col_name TEXT := 'first_name';
  v_col_value TEXT := 'Clara';
  v_ret_name TEXT := 'employee_id';
  v_ret_value TEXT;
BEGIN
  -- Пример экранирования кавычек. Лучше использовать QUOTE_LITERAL()
  sql_query := 'DELETE FROM ' || v_tab_name || 'WHERE ' || v_col_name || ' = ''' || v_col_value || '''' || 'RETURNING ' || v_ret_name;
END $$;
```

При работе с динамическими командами часто приходится использовать экранирование одинарных кавычек. Эту задачу можно упростить, используя специальные функции:
- `QUOTE_IDENT(text)` -- возвращает заданную строку, подходяющую для использования в операторе SQL в качестве идентификатора. Если строка содержит значение, не являющееся идентификатором, то возвращаемое значение обрамляется одинарными кавычками.
- `QUOTE_LITERAL(text)` -- возвращает заданную строку, заключенную в кавычки, подходящую для использования в операторе SQL в качестве строкового литерала.

Пример
```sql
DO $$
DECLARE
  sql_stmt TEXT;
  v_tab_name TEXT := 'employees';
  v_col_name TEXT := 'first_name';
  v_col_value TEXT := 'Bruce';
  v_ret_name TEXT := 'employee_id';
  v_ret_value TEXT;
BEGIN
  sql_stmt := 'DELETE FROM ' || quote_ident(v_tab_name) || 'WHERE ' || quote_ident(v_col_name) || ' = ' || quote_literal(v_col_value) || ' RETURNING ' || v_ret_name;
  --
  EXECUTE sql_stmt
  INTO v_ret_value;
  RAISE NOTICE '...';
END $$;
```
### Использование динамических DDL-операторов

В программах PL/pgSQL, DDL операторы могут выполняться только с использованием динамических операторов. 

Пример 
```sql
DO $$
DECLARE
  sql_stmt TEXT;
BEGIN
  sql_stmt := 'CREATE TABLE dep_count(dep_id INTEGER, dep_ct INTEGER)';
  EXECUTE sql_stmt;
END $$;
```
