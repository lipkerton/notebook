[[#unittest]]
[[#doctest]]
## unittest
Стандартный пакет в Python. Тесты обычно пишутся в отдельной папке. Чтобы сделать тесты с помощью unittest нужно сначала импортировать класс `TestCase`, от которого будет унаследован класс, где будут содержаться тесты.
```Python
from unittest import TestCase
import cap


class TestCap(TestCase):

	def setUp(self)
		pass

	def tearDown(self):
		pass

	def test_one_word(self):
		text = 'duck'
		result = cap.just_do_it(text)
		self.assertEqual(result, 'Duck')

	def test_mult_word(self):
		text = 'the varible ducks'
		result = cap.just_do_it(text)
		self.assertEqual(result, 'The Varible Ducks')

	def test_words_with_apostrophes(self):
		text = "I'm fresh out of ideas"
		result = cap.just_do_it(text)
		self.assertEqual(result, "I'm Fresh Out Of Ideas")
```
Здесь тестируется функция из файла `cap`. Она выглядит так:
```Python
def just_do_it(text):
	return text.capitalize()
```
Вывод будет таким:
```Bash
F.F
===================================================================
FAIL: test_mult_word (__main__.TestCap.test_mult_word)
-------------------------------------------------------------------
Traceback (most recent call last):
  File "/Users/petrpetuhov/Dev/my_python_prikols/unittest_scratch/test_cap.py", line 21, in test_mult_word
    self.assertEqual(result, 'A Varible Flock Of Ducks')
    ~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
AssertionError: 'A varible flock of ducks' != 'A Varible Flock Of Ducks'
- A varible flock of ducks
?   ^       ^     ^  ^
+ A Varible Flock Of Ducks
?   ^       ^     ^  ^

===================================================================
FAIL: test_words_with_apostrophes (__main__.TestCap.test_words_with_apostrophes)
-------------------------------------------------------------------
Traceback (most recent call last):
  File "/Users/petrpetuhov/Dev/my_python_prikols/unittest_scratch/test_cap.py", line 26, in test_words_with_apostrophes
    self.assertEqual(result, "I'm Fresh Out Of Ideas")
    ~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
AssertionError: "I'm fresh out of ideas" != "I'm Fresh Out Of Ideas"
- I'm fresh out of ideas
?     ^     ^   ^  ^
+ I'm Fresh Out Of Ideas
```
В самом верху буквы `F.F` означают, что программа была протестирована в трех тестах - два провалились (они отмечены буквой `F`) и один был успешно пройден (он отмечен точкой). Далее идут записи о проваленных тестах. Каждая запись указывает название проваленного теста, ошибку и еще указывает те места, которые не совпадают во входных и выходных данных.
А зачем нужны методы `setUp()` и `tearDown()`? Это методы, которые можно прописать для усовершенствования поведения тестов. Они используются, когда для какого-то теста или набора тестов нужно подготовить, например, данные в базе данных - этим занимается метод `setUp()`, а метод `tearDown()` наоборот нужен, чтобы очистить подготовленную основу после проведения тестов. Если эти методы определены, то перед каждым тестом (т.е. перед каждым методом) будет проводиться подготовка через `setUp()` и очистка `tearDown()`.
## doctest
`doctest` так же является встроенной библиотекой, но его метод тестирования в корне отличается от метода тестирования `unittest`.
Здесь мы не делаем новый файл, чтобы писать тесты в нем. Мы можем писать тесты прямо в аннотации к функции:
```Python
import doctest
from string import capwords
def just_do_it(text):
	"""
	>>> just_do_it('duck')
	'Duck'
	>>> just_do_it('a bunch of ducks')
	'A Bunch Of Ducks'
	>>> just_do_it("I'm fresh out of ideas")
	"I'm Fresh Out Of Ideas"
	"""
	return capwords(text)
if __name__ == '__main__':
	doctest.testmod()
```
Файл можно запускать на исполнение как обычно. Если все тесты будут пройдены, то в результате выполнения ничего не случиться. Но если все равно хочется посмотреть полную информацию о тестировании, то в терминале следует написать:
```Bash
python <имя файла>.py -v
```
Появится примерно такой интерфейс:
```Bash
Trying:
    just_do_it('duck')
Expecting:
    'Duck'
ok
Trying:
    just_do_it('a veritable flock of ducks')
Expecting:
    'A veritable flock of ducks'
******************************************************************
File "/Users/petrpetuhov/Dev/my_python_prikols/unittest_scratch/doctest_cap.py", line 9, in __main__.just_do_it
Failed example:
    just_do_it('a veritable flock of ducks')
Expected:
    'A veritable flock of ducks'
Got:
    'A Veritable Flock Of Ducks'
Trying:
    just_do_it("I'm fresh out of ideas")
Expecting:
    "I'm Fresh Out Of Ideas"
ok
1 items had no tests:
    __main__
******************************************************************
1 items had failures:
1 of   3 in __main__.just_do_it
3 tests in 2 items.
2 passed and 1 failed.
***Test Failed*** 1 failures.
```
## nose
Это сторонняя библиотека. От двух предыдущих ее отличает то, что тесты в ней пишутся чисто функциями. Чтобы начать нужно просто создать отдельный файл и произвести установку самой библиотеки в виртуальное окружение.
```Bash
pip install nose
```
С этим однако может возникнуть небольшая проблема. По какой-то причине у меня не запустились nose-тесты на python3.12+. Ошибка была в том, что на каком-то этапе инициализации тестирования программа начала требовать модуль `imp`, который отсутствовал. Я попытался его импортировать, но из этого тоже ничего не вышло. Поэтому я пошел писать код на python3.9. Создал на нем окружение, установил nose и запустил тесты - все сработало.
Тесты пишутся в функциях. Принцип простой - любая функция в файле, которая содержит слово `test` первым в названии будет запущена как тест.
```Python
from nose import eq_


def test_one_word():
	text = 'duck'
	result = cap.just_do_it(text)
	eq_(result, text)
```
Результат будет примерно таким:
```Bash
Trying:
    just_do_it('duck')
Expecting:
    'Duck'
ok
```
Функция `eq_()` будет проверять равняются ли элементы. Такой тип функций (и много других типов) есть в каждом модуле, который предназначен для тестирования.