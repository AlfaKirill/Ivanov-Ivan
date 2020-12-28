Informcontact.  
Решение тестового задания для кандидатов на должность SQL Developer.  
Кобелев Сергей kobelev.serg.al@gmail.com



## 1. Одномерный массив из 100 элементов заполнен случайными, целочисленными значениями. Напишите на любом языке программирования процедуру по сортировке элементов массива.

Ничего умнее пузырька на питоне показать не могу.
Конечно можно нагуглить какой-нибудь O(nlogn) алгоритм, или что-нибудь в этом роде, но зачем, если я его не напишу просто так?

sort_bubble является требуемой процедурой.

```python
import random

def sort_bubble(arr):
    arr_length = len(arr)
    was_swap_performed = True
    while was_swap_performed:
        was_swap_performed = False
        for j in range (0, arr_length - 1):
            if arr[j] > arr[j+1]:
                temp_var = arr[j]
                arr[j] = arr[j+1]
                arr[j+1] = temp_var
                was_swap_performed = True
    return arr

values_arr = [random.randint(-1000, 1000) for i in range(100)]
print(values_arr)
print(sort_bubble(values_arr))
```

# **Задания 2, 2.1, 2.2, 3 описаны для Oracle, а я ищу работу на PostgreSQL. Я интерпретирую задания для постгреса и выполняю их на нём(с ораклом не знаком)**


## 2. Создайте таблицу:

### Изначальное задание

```sql
-- Create table
create table TST_SimpleTable
(  id           number not null,
  sProductName varchar2(50),
  dControlDate date,
  nQuantityIn  number,
  nQuantityOut number);


-- Add comments to the columns
comment on column TST_SimpleTable.id
  is 'Идентификатор';
comment on column TST_SimpleTable.sProductName
  is 'Наименование';
comment on column TST_SimpleTable.dControlDate
  is 'Дата';
comment on column TST_SimpleTable.nQuantityIn
  is 'Количество прибыло';
comment on column TST_SimpleTable.nQuantityOut
  is 'Количество убыло';
```

## Реализация на Postgres
**Camel-case неюзабелен в Postgres(если конечно не ставить везде кавычки), поэтому он заменён на camel-case**


```sql
CREATE SCHEMA informcontact;

CREATE TABLE informcontact.tst_simple_table
(
    id             integer NOT NULL,
    s_product_name varchar(50),
    d_control_date date,
    n_quantity_in  integer,
    n_quantity_out integer
);

COMMENT ON COLUMN informcontact.tst_simple_table.id
IS 'Идентификатор';
COMMENT ON COLUMN informcontact.tst_simple_table.s_product_name
IS 'Наименование';
COMMENT ON COLUMN informcontact.tst_simple_table.d_control_date
IS 'Дата';
COMMENT ON COLUMN informcontact.tst_simple_table.n_quantity_in
IS 'Количество прибыло';
COMMENT ON COLUMN informcontact.tst_simple_table.n_quantity_out
IS 'Количество убыло';
```



## 2.1. Напишите код для заполнения таблицы TST_SimpleTable значениями из 1000 строк согласно правилу:
id – уникальное целочисленное значение;

sProductName — строковое наименование товара. Придумайте 3-5 наименований товаров на Ваш выбор и используйте их рандомно для заполнения;

dControlDate – случайная, неуникальная дата, из диапазона 01.01.2098 .. 31.12.2099;

nQuantityIn — целочисленное, положительное зачение в диапазоне 10-20. Количество товара <sProductName> поступившее на склад на дату <dControlDate>.

nQuantityOut — целочисленное, положительное зачение в диапазоне 5-15. Количество товара <sProductName> убывшее со склада на дату <dControlDate>.

## Реализация на Postgres
Выбрал 4 товара с наименованиями

1. Брус Сосна 40x40,
2. Станок кругопильный BELMASH TS-250R,
3. Шлифмашина Dewalt DWE6423,
4. Пломбир ванильный Milk Republic


```sql
-- Для генерации уникального целочисленного значения используем временный сиквенс

CREATE SEQUENCE informcontact.tst_simple_table_fill_seq
    AS integer
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE;


INSERT INTO informcontact.tst_simple_table
(
        id,
        s_product_name,
        d_control_date,
        n_quantity_in,
        n_quantity_out
)
SELECT  nextval('informcontact.tst_simple_table_fill_seq'::regclass),
        ('{Брус Сосна 40x40, Станок кругопильный BELMASH TS-250R, Пломбир ванильный Milk Republic, Шлифмашина Dewalt DWE6423}'::varchar[])[1 + floor(random()*4)],
        '2098-01-01'::date + interval '1 day' * (floor(random() * ('2099-12-31'::date - '2098-01-01'::date))),
        10 + floor(random() * 10),
        5 + floor(random() * 10)
FROM    generate_series(1, 1000);


DROP SEQUENCE informcontact.tst_simple_table_fill_seq;
```





## 2.2. Напишите запрос, который выведет остаток товаров на складе, на каждый день , из заданного двумя датами (дата начала и дата окончания) диапазона. При этом следует учесть остатки товаров сформировавшиеся на складе на дату предшествующую дате начала.

Для 'запроса' используем prepared statement. Первый аргумент($1) - дата начала из условия, второй($1) - дата окончания из условия.
Дата начала и дата окончания могут быть любыми по отношению к датам в таблице (у нас в таблице диапазон 01.01.2098 .. 31.12.2099)

Упрощённо
1. считаем балансы по каждому товару на дату начала
2. определяем список всех дней в интервале и всех отслеживаемых товаров
3. вычисляем изменения баланса в каждый день интервала для каждого отслеживаемого товара
4. для каждой даты интервала и каждого товара прибавляем к изначальному балансу(пункт а)) сумму изменений баланса(пункт в) за предыдущие дни из интервала

```sql
PREPARE tst_balance_check(date, date) AS
    WITH product_qty_at_start AS
    (
        SELECT  temporary_storage.s_product_name product_name ,
                SUM(temporary_storage.n_quantity_in - temporary_storage.n_quantity_out) quantity_balance
        FROM    informcontact.tst_simple_table temporary_storage
        WHERE   temporary_storage.d_control_date < $1
        GROUP BY
                temporary_storage.s_product_name
    ),
    date_name_list AS
    (
        SELECT  $1::date + date_increment.date_increment balance_check_date,
                products_list.products_list AS product_name
        FROM    generate_series(0, ($2 - $1)) date_increment
                CROSS JOIN UNNEST('{Брус Сосна 40x40, Станок кругопильный BELMASH TS-250R, Пломбир ванильный Milk Republic, Шлифмашина Dewalt DWE6423}'::varchar[]) products_list
    ),
    every_date_balance_change AS
    (
        SELECT  date_name_list.balance_check_date,
                date_name_list.product_name,
                SUM(COALESCE(temporary_storage.n_quantity_in, 0) - COALESCE(temporary_storage.n_quantity_out, 0)) n_quantity_balance
        FROM    date_name_list
                LEFT JOIN  informcontact.tst_simple_table temporary_storage
                    ON  date_name_list.balance_check_date = temporary_storage.d_control_date
                        AND date_name_list.product_name = temporary_storage.s_product_name
        GROUP BY
                date_name_list.balance_check_date,
                date_name_list.product_name
    )
    SELECT  first_balance.balance_check_date,
            first_balance.product_name,
            COALESCE(product_qty_at_start.quantity_balance, 0) + SUM(COALESCE(second_balance.n_quantity_balance, 0)) quantity
    FROM    every_date_balance_change first_balance
            INNER JOIN  every_date_balance_change second_balance
                ON  first_balance.product_name = second_balance.product_name
                    AND first_balance.balance_check_date >= second_balance.balance_check_date
            -- какие-то товары могли не попасть на склад до даты начала, поэтому left join + у этих товаров остаток 0 на дату перед датой начала
            LEFT JOIN  product_qty_at_start
                ON  first_balance.product_name = product_qty_at_start.product_name
    GROUP BY
            first_balance.balance_check_date,
            first_balance.product_name,
            product_qty_at_start.quantity_balance
    ORDER BY
            first_balance.balance_check_date,
            first_balance.product_name;
```

Пример использования

```
EXECUTE tst_balance_check('2098-08-01'::date, '2098-08-30'::date);
```

Часть возвращаемого результата

```
balance_check_date, product_name                       , quantity

2098-08-01        , Брус Сосна 40x40                   , 272
2098-08-01        , Пломбир ванильный Milk Republic    , 314
2098-08-01        , Станок кругопильный BELMASH TS-250R, 451
2098-08-01        , Шлифмашина Dewalt DWE6423          , 313
2098-08-02        , Брус Сосна 40x40                   , 271
2098-08-02        , Пломбир ванильный Milk Republic    , 314
2098-08-02        , Станок кругопильный BELMASH TS-250R, 451
2098-08-02        , Шлифмашина Dewalt DWE6423          , 313
2098-08-03        , Брус Сосна 40x40                   , 280
2098-08-03        , Пломбир ванильный Milk Republic    , 314
2098-08-03        , Станок кругопильный BELMASH TS-250R, 462
2098-08-03        , Шлифмашина Dewalt DWE6423          , 313

```

Я немного его оптимизировал, и пытался придумать, как сделать ещё оптимальнее, но не смог.
Если первая цтеха очень часто будет улетать в Sequence Scan из-за низкой селективности запроса, и с ней можно смириться, то LEFT JOIN informcontact.tst_simple_table в цтехе every_date_balance_change вылетает в напряжный hash right/left join, и прикольно было бы придумать запрос, где вместо LEFT там INNER.


**Если вы не ответите на это задание, и по нему не будет беседы, то отправьте, пожалуйста, правильное по-вашему мнению решение задачи на мой email kobelev.serg.al@gmail.com**


## 3. Необходимо написать функцию, возвращающую значение поля sCode из произвольной таблицы. В качестве входных параметров передаются имя таблицы и идентификатор записи.

Вот ее вариант:

```
function GetCodeForTable(spTableName in varchar2, idpRecord in number) return varchar2
is
  svResult varchar2(100);
begin
  execute immediate 'select sCode from ' || spTableName || ' where id = ' || to_char(idpRecord) into svResult;
  return svResult;
end;
```

**Вопрос — что бы Вы поменяли в данной функции и почему?**



### Вариант указанной функции на PostgreSQL

```
CREATE FUNCTION informcontact.get_code_for_table
(
    sp_table_name varchar,
    id_p_record   integer
)
    RETURNS varchar
    LANGUAGE plpgsql
AS
$$
DECLARE
    sv_result varchar(100);
BEGIN
    EXECUTE 'select sCode from ' || sp_table_name || ' where id = ' || id_p_record::text
        INTO sv_result;
END;
$$;
```

### Изменение №1. Добавить схему
Не знаю, как на Oracle, но на Postgres лучше всегда использовать наименования схем вместе с наименованием таблиц. Иначе мы идём в search_path, а оттуда наверняка попадём в схему по имени юзера $USERNAME или в public, чего лучше не делать. Ведь если на базе с паблика не ревокнуты привилегии, то кто-нибудь может создать функцию с таким же названием в схеме public, но делающую что-то очень плохое(от вашего лица).

Поэтому для начала добавим схему

```
CREATE FUNCTION informcontact.get_code_for_table
(
    sp_table_schema_name varchar,
    sp_table_name        varchar,
    id_p_record          integer
)
    RETURNS varchar
    LANGUAGE plpgsql
AS
$$
DECLARE
    sv_result varchar(100);
BEGIN
    EXECUTE 'select sCode from ' || sp_table_schema_name || '.' || sp_table_name || ' where id = ' || id_p_record::text
        INTO sv_result;
END;
$$;
```

### Изменение №2. У всех переменных проставить префикс нижнее подчёркивание
Для того, чтобы чётко отличать переменные от столбцов, я повсеместно использую нижнее подчёркивание у аргументов и переменных.

```
CREATE FUNCTION informcontact.get_code_for_table
(
    _sp_table_schema_name varchar,
    _sp_table_name        varchar,
    _id_p_record          integer
)
    RETURNS varchar
    LANGUAGE plpgsql
AS
$$
DECLARE
    _sv_result varchar(100);
BEGIN
    EXECUTE 'select sCode from ' || _sp_table_schema_name || '.' || _sp_table_name || ' where id = ' || _id_p_record::text
        INTO _sv_result;
END;
$$;
```


### Изменение №3. Нужно избежать каких-либо конкатенаций строк в Dynamic SQL.
Обычная конкатенация строк в Dynamic SQL открывает дверь для инъекций(вместо _sp_table_name можно подставить целый запрос).

И нужно что-то придумывать для обхода.

Как это сделать в PostgreSQL я не знал, но я, например, использовал библу на питоне для Plain-SQL запросов и там параметры передавал через специализированное подобие format, которое отсеивало инъекции, переводя текст в SQL Identifier/SQL Literal.

Почитал, оказалось, что в современном PG используют тоже самое.

Для передачи наименований объектов используется format(тип айдентифаер %I), для передачи 'константных' значений используется либо тоже формат(тип литерал %L), либо саб-директива USING директивы EXECUTE.
Получим


```
CREATE FUNCTION informcontact.get_code_for_table
(
    _sp_table_schema_name varchar,
    _sp_table_name        varchar,
    _id_p_record          integer
)
    RETURNS varchar
    LANGUAGE plpgsql
AS
$$
DECLARE
    __sv_result varchar(100);
BEGIN
    EXECUTE format('select sCode from %I.%I  where id = $1', _sp_table_schema_name, _sp_table_name)
        USING _id_p_record
        INTO _sv_result;
    RETURN  _sv_result;
END;
$$;
```
