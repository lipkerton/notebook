Это краулер и одновременно скрапер. 
**Краулер (или веб-паук)**  означает, что программа может автоматически получать данные из сети. А **скрапер** означает, что программа может еще и читать эти данные.
## как сделать проект?
scrapy - это фреймворк, поэтому его запуск чем-то напоминает запуск django-проекта. Сначала нужно загрузить scrapy:
```Bash
pip install scrapy
```
дальше нужно стартануть проект:
```Bash
scrapy startproject <имя_проекта>
```
в папке, из которой была отправлена команда `startproject` появится папка проекта, в которой уже подготовлена вся необходимая структура. 
```Bash
tutorial
├── scrapy.cfg
└── tutorial
    ├── __init__.py
    ├── items.py  # для специальных объектов scrapy.
    ├── middlewares.py  # дополнительный софт.
    ├── pipelines.py  # пока так и не понял для чего.
    ├── settings.py  # настройки проекта.
    └── spiders  # папка, где будут лежать пауки.
        └── __init__.py
```
Далее нужно просто зайти в папку с пауками и создать своего первого паука.
Паук создается на основании класса, который должен быть унаследован от класса `scrapy.Spider`.
```Python
class QuotesSpider(scrapy.Spider):
	name = 'quotes'

	def start_requests(self):
		pass
```
Переменная `name` нужна, чтобы мы могли запускать паука командой из консоли. 
Функция `start_requests()` - это стандартная функция, с нее начинается действие паука при его вызове. Она должна принимать список ссылок и возвращать итерируемый объект, который состоит из этих ссылок.
```Python
class QuotesSpider(scrapy.Spider):
	name = 'quotes'

	def start_requests(self):
		urls = [
			'https://quotes.toscrape.com/page/1/',
			'https://quotes.toscrape.com/page/2/'
		]
		for url in urls:
			yield scrapy.Request(url=url, callback=self.parse)
```
Далее нужно написать функцию `parse()`, которая упомянута в колбэке - именно туда будут отправляться все ответы на запросы, которые происходят в функции `start_requests()`.
```Python
class QuotesSpider(scrapy.Spider):
	name = 'quotes'

	def start_requests(self):
		urls = [
			'https://quotes.toscrape.com/page/1/',
			'https://quotes.toscrape.com/page/2/'
		]
		for url in urls:
			yield scrapy.Request(url=url, callback=self.parse)

	def parse(self, response):
		print(response)
```
Ура, теперь у нас есть объект response, из которого можно доставать все элементы страницы.
```Python
class QuotesSpider(scrapy.Spider):
	name = 'quotes'

	def start_requests(self):
		urls = [
			'https://quotes.toscrape.com/page/1/',
			'https://quotes.toscrape.com/page/2/'
		]
		for url in urls:
			yield scrapy.Request(url=url, callback=self.parse)

	def parse(self, response):
		quote = response.xpath(
			'//div[@class="quote"]/span[1]/text()'
		).getall()
		print(quote)
```
Функция `xpath()` позволяет написать выражение на языке `xpath`, которое приведет нас к нужному элементу на странице. В данном случае я написал:
1) `//div[@class="quote"]` - найди все теги `div` в документе, у которых атрибут `class` равняется `quote`.
2) `/span[1]` - слеш означает, что от того, что мы нашли до мы опустимся на уровень ниже, в нашем случае мы "нырнем" в найденные теги `div` и внутри найдем тег `span`. Цифра `[1]` означает, что мы будем искать только первый тег `span` после `div`, а не все теги `span`, которые могут оказаться внутри `div`.
3) `/text()` - означает, что я применяю функцию `text()` к найденным тегам, чтобы она извлекла из них текстовое содержимое.
Затем функция `getall()` собирает все найденное в список.
Структура HTML примерно такая:
```HTML
<div class="quote" itemscope itemtype="http://schema.org/CreativeWork">
        <span class="text" itemprop="text">
	        “It is our choices, Harry, that show what we truly are, far more than our abilities.”
	    </span>
        <span>by 
	        <small class="author" itemprop="author">J.K. Rowling</small>
	        <a href="/author/J-K-Rowling">(about)</a>
        </span>
        <div class="tags">
            Tags:
            <meta class="keywords" itemprop="keywords" content="abilities,choices"> 
            <a class="tag" href="/tag/abilities/page/1/">abilities</a>
            <a class="tag" href="/tag/choices/page/1/">choices</a>
        </div>
</div>
```