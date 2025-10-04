`pytest.mark` используется для произвольного именования тестов pytest, чтобы их можно было вызвать из командной строки с помощью этого имени отдельно от других тестов.
Например:
```Python
import pytest


def func():
	return 1 + 1


@pytest.mark.reports
def test_func():
	assert func() == 2
```
Теперь можно запустить тесты так:
```
pytest -v -m reports
```
Однако при подобном запуске pytest начнет делать предупреждения:
```Python
PytestUnknownMarkWarning: Unknown pytest.mark.reports - is this a typo?  You can register custom marks to avoid this warning - for details, see https://docs.pytest.org/en/stable/how-to/mark.html
    @pytest.mark.reports
```
Чтобы их избежать нужно зарегистрировать кастомные маркеры в `pytest.ini`:
```
[pytest]
markers = 
	reports: my custom marker!
```
`pytest.ini` можно разместить в директории рядом файлами тестов, а можно перекинуть в другую папку, но в этом случае потребуется указать путь до файла в параметре `-c`:
```
pytest -v -m reports -c ./settings/pytest.ini
```