## база
Я пишу JavaScript код в WSL, поэтому мне понадобился сервер для обращения к моим HTML документам.
В общем случае JavaScript код - это код, который пишется, чтобы выполнить какую-то сложную логику на странице. "Сложная" означает, что эту логику не могут выполнить CSS стили или простой HTML код.
JavaScript код не требует скачки какого-то движка - интерпретатор этого языка встроен в клиент (браузер). 
Поэтому JS код можно писать прямо в HTML документе, а потом запускать просто обращаясь к странице.
Я начну с того, что запущу сервер:
```Bash
python -m http.server 8000
```
В конце указывают порт - он может быть любым.
Дальше я напишу самый простой HTML документ:
```HTML
<html>
  <head>
    <title>Страница с примером кода JavaScript</title>
    <script>
      alert("Hello world!");
    </script>
  </head>
  <body>
    <h3>Это текст основной страницы</h3>
  </body>
</html>
```
JS код здесь начинается внутри тега `<script>`. Функция `alert()` выведет push-уведомление на экран при запуске страницы.
Здесь есть еще одна особенность - кодировка. Если запустить код таким образом, то на странице вместо надписи "Это текст основной страницы", появится набор рандомных букв. Чтобы это исправить нужно объявить кодировку для HTML документа:
```HTML
<html>
  <head>
    <meta charset="utf-8">
    <title>Страница с примером кода на JavaScript</title>
    <script>
      alert("Hello world!");
	</script>
  </head>
  <body>
    Это текст основной страницы
  </body>
</html>
```
## где размещать код JavaScript?
В заголовке или в теле. Код JavaScript всегда размещается внутри тега `<script>`. 
Далее я пишу код, где JavaScript написан внутри заголовка, но сама функция вызывается из тела.
```HTML
<html>
  <head>
    <script>
      function openWindow() {
	    alert("Hello world!");
      }
    </script>
  </head>
  <body>
    <button type="button" onclick="openWindow()">Open window</button>
  </body>
</html>
```
А теперь код, который размещен в теле и вызывается из тела:
```HTML
<html>
  <head>
  </head>
  <body>
    <script>
      function openWindow() {
        alert("Hello world!");
      }
    </script>
    <button type="Button" onclick="openWindow()">Open window</button>
  </body>
</html>
```