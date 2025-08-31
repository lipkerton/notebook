Я был удивлен, когда узнал об этой штуке, потому что раньше не видел ее ни на одном курсе Stepik/ЯП/прч.
Да, это механизм сопоставления шаблону, который встроен в базовый синтаксис Python.
Например, у нас есть такие строки:
```Python
test_1 = "stop~http://privet.ru"
test_2 = "start~http://poka.ru"
```
И мы хотим для каждого из действий (`"start"/"stop"`) предусмотреть свой вариант поведения программы.
Такую историю можно было бы реализовать с помощью `if-elif-else`:
```Python
def main(test_line):
	test_line = test_line.split("~")
	if test_line[0] == "stop":
		print("stop")
	elif test_line[0] == "start":
		print("start")
	else:
		print(None)
```
Однако такой вариант 1) беспонтовый и 2) дедовский. Круче можно реализовать это с помощью `match-case`:
```Python
def main(test_line):
	test_line = test_line.split("~")
	match test_line:
		case "stop", *_:
			print("stop")
		case "start", *_:
			print("start")
```
По сути эта штука - усовершенствованный `if-elif-else`, однако у нее применение более специфичное: `if-elif-else` проверяет значение на правду или ложность, а `match-case` именно сопоставляет значение с изначальным шаблоном.
Структура:
```Python
match <структура, которую будем сопоставлять>:
	case <шаблон>:
		<действия в случае совпадения с шаблоном>
```
Особенно классная штука в том, что шаблон может быть устроен сложнее:
```Python
test_1 = ['Peter', 1, (25, 1, 2000)]
test_2 = ['Gleb', 2, (29, 1, 2000)]

def main(test_line):
	match test_line:
		case ['Peter', index, (day, month, year)]:
			print(day, month, year)
		case ['Gleb', index, (day, month, year)]:
			print(day, month, year)
```
И еще круче:
```Python
def main(test_line):
	match test_line:
		case ['Peter', index, (day, month, year) as birthday]:
			print(birthday)
		case ['Gleb', index, (day, month, year) as birthday]:
			print(birthday)
```
Такое тоже сработает - теперь можно вызывать переменную с именем `birthday`. Нужно посмотреть работает ли штука с `as` вне `match-case`.
