Детали можно найти на странице проекта https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN.

Общая схема соединения выглядит так
```bash
_`T1`_ { [INNER] | { LEFT | RIGHT | FULL } [OUTER] } JOIN _`T2`_ ON _`boolean_expression`_
_`T1`_ { [INNER] | { LEFT | RIGHT | FULL } [OUTER] } JOIN _`T2`_ USING ( _`join column list`_ )
_`T1`_ NATURAL { [INNER] | { LEFT | RIGHT | FULL } [OUTER] } JOIN _`T2`_
```

Слова `INNER` и `OUTER` необязательные для всех форм. `INNER` подразумевается по умолчанию, а `LEFT`, `RIGHT` и `FULL` реализованы как _внешние_.
