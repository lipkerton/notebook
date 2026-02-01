Это язык программирования, который является частью PostgreSQL. В нем реализованы все базовые приемы других стандартных языков программирования - функции, переменные, циклы, условные конструкции и проч.
# hello world
Скрипт начинается с ключевого слова `DO` и символов `$$`:
```SQL
do $$
begin
	raise notice 'Hello, world!';
end $$;
```
Завершается скрипт ключевым словом `END` и двумя символами `$$`. Сразу после завершения скрипта начинается его выполнение. В данном случае в консоль будут выведены слова:
![[pl_pg_sql_1.png]]
Символ `$$` в контексте PL/pgSQL означает ограничитель, внутри которого можно начинать писать код. Этот же символ можно использовать и в стандартных запросах на SQL:
```SQL
select $$мистер 'Никто'$$;
```
В этом случае он так же сработает как ограничитель строки. Часто его используют, чтобы лишний раз не экранировать кавычки.
Кстати, кавычка в PostgreSQL может быть экранирована с помощью второй кавычки:
![[pl_pg_sql_2.png]]
# begin declare
Код в PL/pgSQL делится на блоки. `begin` и `declare` - два таких блока.
Назначение блока `begin` - содержать в себе какую-то скриптовую логику. Назначение блока `declare` - содержать в себе объявления переменных.
Блоки могут существовать автономно друг от друга, а могут быть связаны. Например, блок `declare` объявляет какие-то переменные, а блок `begin` работает с ними. Как в таком скрипте:
```SQL
do $$
declare
	message text := 'Hello, world!';
	a int := 1;
	b int := 2;
begin
	raise notice '%', message;
	raise notice '% + % = %', a, b, a + b;
end $$;
```
В этом коде объявляется три переменные в блоке `declare` - `message`, `a`, `b`. В блоке `begin` выполняется форматирование строк в стиле PL/pgSQL - в строку на место процента будут в порядке очереди подставлены все значения, которые указаны после строки через запятую.
После выполнения кода произойдет:
![[pl_pg_sql_3.png]]
# if elif else
Условная конструкция размещается в блоке `begin`:
```SQL
do $$
declare
	number := 100;
begin
	if number % 2 = 0 then
		raise notice 'Это четное число!';
	else
		raise notice 'Это нечетное число!';
	end if;
end $$;
```
Сюда же, как и во всех других языках программирования, можно добавить `elif` с дополнительным условием проверки.
# loop
## loop
`loop` - это аналог цикла `while` для PL/pgSQL:
```SQL
do $$
	declare
		a int := 10;
	begin
		loop
			a := a - 1;
			raise notice '%', a
			exit when a < 0;
		end loop;
end $$;
```
Условие выхода из цикла прописывается в конструкции `exit when`.
Результатом такой записи будет:
![[pl_pg_sql_4.png]]
## for loop
Разновидность цикла `loop`, которая является аналогом цикла `for`:
```SQL
for i in 0..n loop
	result := result + i
end loop;
```

# функции и процедуры
В PL/pgSQL функции и процедуры отличаются тем, что первые обязаны возвращать какое-нибудь значение, а вторые обязаны выполнять действие, но не обязаны возвращать значение.
## функция
Создать функцию:
```SQL
create or replace function <имя функции>(<аргументы функции>)
returns <тип возвращаемого значения>
as
$$
<тело функции>
$$
language plpgsql;
```
Внутри функций можно объявлять свои переменные с помощью блока `declare`:
```SQL
$$
declare
	a integer;
	b integer;
begin
<тело функции>
end;
$$
```
Так же этим переменным можно сразу присваивать значения:
```SQL
$$
declare
	a integer := 1;
	b integer := 2;
begin
<тело функции>
end;
$$
```
## процедура
Создать процедуру:
```SQL
create or replace procedure <имя процедуры>(<аргументы процедуры>)
as
$$
<тело процедуры>
$$
language plpgsql;
```
В процедуре все то же самое, что и в функции, но она ничего не возвращает.
## примеры
Функция, которая суммирует два числа:
```SQL
create or replace function sum_two_numbers(a integer, b integer)
returns integer
as
$$
declare
	result integer;
begin
	result := a + b;
	return result;
end;
$$
language plpgsql;
```
Функция, которая складывает числа в промежутке между $0$ и числом $n$:
```SQL
create or replace function sum_numbers(n integer)
returns integer
as
$$
declare
	result integer := 0;
	i integer
begin
	for i in 0..n loop
		result := result + i
	end loop;
	return result;
end;
$$
language plpgsql;
```
На вход функции так же можно передавать ((( объекты ))).
Например, я хотел написать функцию, которая будет парсить переданную на вход запись (имя, фамилия, отчество) и отдавать склеенную строку из них. Чтобы это реализовать можно было бы сделать функцию, которая принимает отдельно три аргумента; но это как-то слишком не практично, поэтому я подумал, что можно было бы сделать функцию, которая принимает прямо всю строку данных из таблицы и парсит ее сама:
```SQL
-- таблица
create table client(
	client_id bigint generated always as identity primary key,
	first_name text not null,
	last_name text,
	middle_name text
)
-- функция
create or replace function print_fio(client_record client)
returns text
as 
$$
declare
	result text := client_record.first_name;
begin
	if client_record.last_name is not null and client_record.middle_name is not null then
		result := result || ' ' || client_record.middle_name || ' ' || client_record.last_name;
	elsif client_record.last_name is null and client_record.middle_name is not null then
		result := result || ' ' || client_record.middle_name;
	elsif client_record.last_name is not null and client_record.middle_name is null then
		result := result || ' ' || client_record.last_name;
	end if;
	return result;
end;
$$
language plpgsql;
```
Теперь после создания функции я могу вызывать ее так:
```SQL
select print_fio(client) from client;
```
Как это работает?
## передача объектов в фукнции и процедуры
Запись:
```SQL
create or replace function print_fio(client_record client)
...
```
Означает, что функция ожидает на вход не всю таблицу, а строку данных в этой таблице. То есть она ожидает получить объект-строку из таблицы `client`.
Когда я пишу:
```SQL
select print_fio(client) from client;
```
Это означает, что я передаю каждую строку таблицы на вход в функцию.
## исключения
В функциях и процедурах можно отлавливать исключения таким синтаксисом:
```SQL
begin
	result := a / b;
exception
	when division_by_zero then
		raise exception 'Ошибка: деление на ноль!';
		return null;
end;
```
Нужно понимать, что это все является блоком исключения, то есть внутри процедуры это будет выглядеть так:
```SQL
create or replace function divide_by(a numeric, b numeric)
returns numeric
as
$$
declare
	result numeric;
begin
	begin
		result := a / b
	exception
		when division_by_zero then
			raise exception 'Ошибка: деление на ноль!';
			return null;
	end;
return result;
end;
$$
```