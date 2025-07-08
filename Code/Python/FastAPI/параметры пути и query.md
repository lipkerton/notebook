## параметры пути
### о параметрах в URL
В FastAPI можно описывать функции, которые принимают параметры пути и query-параметры.
Чтобы прописать функцию, которая принимает параметры пути нужно просто добавить к маршруту в декораторе "заглушку", как я это делал в маршрутах Django:
```Python
@app.get("/{name}")
def hello_page(name:str):
	return {"message": f"Hello, {name}"}
```
`/{name}` в данном случае означает, что все записи, которые будут в пути после `/` и до следующего `/` будут переданы в эту функцию, потому что они похожи на шаблон `/{name}`. Например, запросы к эндпоинтам `/Peter`, `/cool`, `/picture` получат ответы именно от этой функции, а запросы к эндпоинтам `/Peter/cool`, `/cool/Peter`, `/picture/beautifull` уже не совпадут с шаблоном `/{name}`.
### конвертеры пути
Конвертер пути - это штука, которая проверяет, что в запросе от пользователя пришло именно то, что ожидает получить на вход конкретная фукнция. 
Я хочу написать функцию, которая будет принимать на вход число, а возвращать некую страницу с товаром по номеру. Будет плохо, если в такую функцию попадет аргумент другого типа, например, строка. Как решит эту ситуацию? С помощью конвертера пути:
```Python
@app.get("/catalog/{index}")
def catalog_item(index:int):
	return {
		"item_id": index
		"title": "Тарелка",
		"description": "Крутая"
	}
```
Следует отметить, что конвертер пути в FastAPI пишется с помощью интерфейса Python - `def catalog_item(index:int)`. И это работает! Если передать в запросе к данному эндпоинту число, то результат получится таким, каким мы его ожидаем:
![[path_param_fastapi.png]]
А если передать в запрос к этому эндпоинту строку, то получится ошибка:
![[error_path_param_fastapi.png]]
### порядок маршрутов
Порядок расположения маршрутов точно так же важен как в любом другом фреймворке. Если я размещу две функции:
```Python
@app.get("/catalog/{slug}")
def catalog_item(slug:str):
	return f"Предмет каталога {slug}!"

@app.get("/catalog/pen")
def catalog_pen():
	return "Предмет каталога pen!"
```
То вторая функция никогда не сработает, потому что маршрут, который указан для нее так же подходит для первой функции. Первая функция будет отлавливать любые запросы такого формата, а запрос `/catalog/pen` как раз подходит для нее.
Даже если типы в функциях будут предопределены, например, так:
```Python
@app.get("/catalog/{index}")
def catalog_item(index:int):
	return f"Предмет каталога с индексом {index}."

@app.get("catalog/{slug}")
def catalog_category(slug:Category):
	return f"Категория каталога {slug}."
```
То маршрут `/category/adventures` попадет в первую функцию, а на страницу вернется ошибка. Чтобы исправить эту ситуацию нужно применить не только типизацию в параметрах, нужно еще добавить конвертер в сам маршрут:
```Python
@app.get("/catalog/{index:int}")
def catalog_item(index:int):
	return f"Предмет каталога с индексом {index}."

@app.get("/catalog/{slug:str}")
def catalog_category(slug:Category):
	return f"Категория каталога {slug.name}."
```
В этом случае параметр пути проверяется при поиске нужного маршрута, а не при вызове функции, поэтому никакой ошибки не произойдет.
### enum и параметры пути
Если мы хотим, чтобы функция принимала параметры пути, но при этом имела ограниченный круг тех параметров, которые она может принять, то можно воспользоваться классом `enum.Enum` [[полезные модули#enum]]. Коротко, в нем определяют пары ключей и значений; эти пары могут быть вызваны буквально из объекта класса. Например:
```Python
from enum import Enum
class Category(Enum):
	adventures = 'adventures'
	action = 'action'
	romantic_comedy = 'romantic_comedy'
print(Category.adventures.name)  # adventures
print(Category.adventures.value)  # adventures
```
Эту механику можно использовать, чтобы ограничить круг возможных аргументов функции:
```Python
@app.get("/catalog/{slug:str}")
def catalog_category(slug:Category):
	return f"Это категория каталога {slug.name}"
```
Далее можно прописать индивидуальное поведение каждому из вариантов, который может получить эта функция:
```Python
@app.get("/catalog/{slug:str}")
def catalog_category(slug:Category):
	if slug is Category.adventures:
		return f"Это категория приключений."
	if slug is Category.action:
		return f"Это категория экшенов."
	if slug is Category.romantic_comedy:
		return f"Это категория романтических комендий."
```
### путь внутри пути
Допустим у нас есть шаблон пути `/file/{file_name}`. Но вдруг мы хотим передать в этот шаблон другой путь, например, такой `home/something/file.txt`. Спецификация OpenAPI не подразумевает такой подход, однако FastAPI может такое реализовать с помощью конвертера пути `path`:
```Python
@app.get("/file/{file_path:path}")
def get_file(file_path):
    from pathlib import Path
    my_path = "file" / Path(file_path)
    with open(my_path, "r") as file:
        result = file.read()
    return result
```
Теперь я могу сделать запрос на эндпоинт, например, `file/cool/file.txt` и функция вернет строку, которая прочитана из файла. Проблема в таком походе заключается только в том, что любой аргумент после `/file/` будет рассматриваться, как часть пути.
## query-параметры
Query-параметры выглядят так:
```
http://127.0.0.1/fruits?name=orange
```
Знак вопроса отделяет тело запроса от параметров. Параметры записываются парами - имя + значение и между собой разделяются знаком амперсанта `&`.
Чтобы сделать функцию, которая принимает query-параметры в FastAPI не нужно делать ничего. Главное, прописать для функции те параметры, которые мы хотим в нее передавать. Прописывается это с помощью стандартного интерфейса функции в  Python:
```Python
@app.get("/fruits")
def get_fruit(name:str):
	return f"Это фрукт {name}!"
```
Теперь, если мы обратимся по адресу `http://127.0.0.1/fruits?name=orange`, то на страницу выведется именно эта фраза.
Чтобы избежать конфликтов и не писать сто функций лучше всего прописывать возможное наличие опциональных query-параметров с помощью параметров по умолчанию:
```Python
def get_fruit(name:str | None = None):
	if name:
		return f"Это фрукт {name}!"
	return f"Это каталог фруктов!"
```
>[!important] FastAPI достаточно умный, чтобы отличать query-параметры от параметров пути автоматически.
### bool в query
Я написал функцию, которая должна принимать query-параметр типа `bool`:
```Python
@app.get("/true_false")
def true_false(bol: bool = False):
    if bol:
        return {"message": "Здесь говорят правду!"}
    return {"message": "Здесь все лгут!"}
```
FastAPI может конвертировать некоторые аргументы, которые были переданы ей в query-параметрах сразу в булевское значение `True`. Например, запросы:
```
http://127.0.0.1:8000/true_false?bol=True
http://127.0.0.1:8000/true_false?bol=true
http://127.0.0.1:8000/true_false?bol=yes
http://127.0.0.1:8000/true_false?bol=1
http://127.0.0.1:8000/true_false?bol=on
```
Отправят в параметр функции `bol` значение `True`.
### параметры пути + query-параметры
Они могут быть указаны вместе в одной функции. Главное нужно соблюдать порядок, стандартный для интерфейса Python: параметры со значениями по умолчанию идут позже позиционных параметров.
```Python
@app.get("/hello_name/{name:str}")
def hello_peter(name: str, second_name: str | None = None):
    if second_name:
        return {"message": f"Привет, {name} {second_name}"}
    return {"message": f"Привет, {name}!"}
```
Никакого специального синтаксиса не нужно - FastAPI определит где какие параметры по именам аргументов
>[!important] Когда мы указываем query-параметр без значения по умолчанию, то он автоматически становится обязательным.
### проверка query-параметра на входе
Иногда хочется проверить входящий query-параметр на соответствие каким-то критериям. 
Такую проверку можно сделать с помощью стандартного Python модуля `typing` и класса `Query` в FastAPI.
Для начала нужно получить `Annotated` из `typing`:
```Python
from typing import Annotated
```
`Annotated` используют, чтобы писать более подробные аннотации к параметрам функции в Python:
```Python
def my_func(number: Annotated[int | None]):
```
Такая формулировка по сути означает то же самое, что и:
```Python
def my_func(number: int | None):
```
В FastAPI она позволяет добавить дополнительную валидацию query-параметра, используя модуль `Query` из пакета `fastapi`:
```Python
from fastapi import Query
from typing import Annotated
...
def my_func(
	number: Annotated[int | None, Query(gt=0, le=1000)] = None
)
```
Параметры:
1) `gt` - greater then
2) `le` - lesser or equal
Точно так же можно проверять и строки:
```Python
from fastapi import Query
from typing import Annotated
...
def my_func(
	string: Annotated[str | None, Query(min_length=1, max_length=50)] = None
)
```
Параметры:
1) `min_length` - проверяет, что в строке больше одного символа.
2) `max_length` - проверяет, что в строке меньше 50 символов.
Так же можно вставлять регулярки для проверки входящих строк:
```Python
string: Annotated[str | None, Query(pattern='<регулярка>')]
```
Для query-параметра так же можно указать, что он принимает список значений.