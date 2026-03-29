я хочу сделать для работы API (через FastAPI), который будет обмениваться сообщениями с клиентами по протоколу SOAP, а потом дописать для этого API клиента на C#, который будет бесконечно слушать сообщения от API.
о SOAP: https://blog.postman.com/soap-api-definition/
# API
писать я буду на FastAPI, поэтому мне нужен его стандартный пак:
```Powershell
python -m pip install "fastapi[standard]"
```
стандратный FastAPI не поддерживает SOAP, поэтому придется поставить еще и расширение.
адрес расширения на PyPi: https://pypi.org/project/fastapi-soap/
## test
после установки я решил сделать тестовый сервер, где буду пробовать код из инструкции на странице в PyPi:
```Python
from fastapi import FastAPI
from pydantic_xml import element
from fastapi_soap import SoapRouter, XMLBody, SoapResponse
from fastapi_soap.models import BodyContent


class Operands(BodyContent, tag="Operands"):
    operands: list[float] = element(tag="Operand")

class Result(BodyContent, tag="Result"):
    value: float

soap = SoapRouter(name='Calculator', prefix='/Calculator')

@soap.operation(
    name="SumOperation",
    request_model=Operands,
    response_model=Result
)
def sum_operation(body: Operands = XMLBody(Operands)):

    result =  sum(body.operands)
    return SoapResponse(
        Result(value=result)
    )


app = FastAPI()
app.include_router(soap)
```
следует обратить внимание на два класса, которые тут используются - `Operands` и `Result`. они определяют содержимое тела XML запросов и ответов. через них я определяю, что мой API ожидает получить и какие формы он будет отправлять в ответ.
в случае с классом `Operands` я определяю, что внутри тела XML будет лежать несколько тегов `Operand`, которые при автоматическом парсинге (через Pydantic) будут складываться в список.
в случае с классом `Result` я определяю, что внутри тела XML будет лежать только один тег `value` с одним значением, которое будет переводиться в тип `float`.
после определения классов я делаю экземпляр `SoapRouter` и кладу его в переменную `soap`. так же в этом экземпляре указан префикс, по которому будут доступны все операции моего API.
чтобы сделать метод, который будет работать с SOAP нужно использовать декоратор `@soap.operation()`. внутри скобок я прописываю имя операции (к этому имени я буду обращаться, чтобы использовать операцию) и выбираю модель запроса с моделью ответа.
аргумент метода указывает на то, что я должен передать ему тело XML (тег `<body></body>`). внутри метода для меня будет доступен словарь `body`, внутри которого лежит содержимое тела XML. это обычный словарь, поэтому я могу из него достать список операндов и сложить его с помощью метода `sum()`.
ответный XML высылается классом `SoapResponse()`, тело которого я определяю через `Result` модель.
## как это тестировать?
я писал код на Windows, поэтому можно использовать два способа - `curl.exe` и `Invoke-WebRequest`.
первый вариант простой - он напоминает стандартный `curl` в UNIX.
сначала можно получить WSDL - это описание сервиса:
```Bash
curl.exe -X GET http://localhost:8000/Calculator/
```
следует обратить внимание на то, что FastAPI автоматически перенаправляет запросы с эндпоинта `http://localhost:8000/Calculator` на эндпоинт `http://localhost:8000/Calculator/` - это означает, что в заголовке `Location` вернется новый адрес, куда мне нужно перейти, чтобы получить свой WSDL. `curl.exe` не ожидает по умолчанию, что его отправят на другую локацию, поэтому в ответ на запрос к `http://localhost:8000/Calculator` не вернет ничего. чтобы получить нормальный ответ нужно либо использовать `http://localhost:8000/Calculator/`, либо указать, что `curl.exe` нужно смотреть туда, куда показывает `Location`, с помощью параметра `-L`.
чтобы провернуть аналогичную операцию с помощью `Invoke-WebRequest`:
```Powershell
Invoke-WebRequest -Method Get -Uri "http://localhost:8000/Calculator/"
```
вернется примерно такой ответ:
```PowerShell
StatusCode        : 200
StatusDescription : OK
Content           : <wsdl:definitions xmlns:soap="http://schemas.xmlsoap.org/wsdl/soap/" xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/" xmlns:xs="http://www.w3.org/2001/XMLSchema"
	xmlns:wsdlsoap="http://schemas.xmlsoap.or...
RawContent        : HTTP/1.1 200 OK
	Content-Length: 1860
	Content-Type: text/xml; charset=utf-8
	Date: Sat, 28 Mar 2026 11:32:28 GMT
	Server: uvicorn

	<wsdl:definitions xmlns:soap="http://schemas.xmlsoap.org/wsdl/soap...
Forms             : {}
Headers           : {[Content-Length, 1860], [Content-Type, text/xml; charset=utf-8], [Date, Sat, 28 Mar 2026 11:32:28 GMT], [Server, uvicorn]}
Images            : {}
InputFields       : {@{innerHTML=; innerText=; outerHTML=<?xml:namespace prefix = "wsdl" /><wsdl:input message="SumOperationRequest"></wsdl:input>; outerText=; tagName=input},
	@{innerHTML=<?xml:namespace prefix = "soap" /><soap:body use="literal"></soap:body>; innerText=; outerHTML=<?xml:namespace prefix = "wsdl" /><wsdl:input
	message="SumOperationRequest"><?xml:namespace prefix = "soap" /><soap:body use="literal"></soap:body></wsdl:input>; outerText=; tagName=input}}
Links             : {}
ParsedHtml        : System.__ComObject
RawContentLength  : 1860
```
теперь я могу получить результат своего метода:
```Bash
curl.exe -X POST --header '{"Content-Type": "text/xml;charset=UTF-8"}' \
	-d "@test_request.xml"
	http://localhost:8000/Calculator/SumOperation
```
если делать с помощью `Invoke-WebRequest`:
```PowerShell
Invoke-WebRequest -Method Post -Headers @{'Content-Type' = 'text/xml;charset=UTF-8'} `
	-InFile "test_request.xml"
	-Uri http://localhost:8000/Calculator/SumOperation
```
придет такой XML:
```PowerShell
StatusCode        : 200
StatusDescription : OK
Content           : <soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/"><soap:Header/><soap:Body><Result>6.0</Result></soap:Body></soap:Envelope>
RawContent        : HTTP/1.1 200 OK
                    Content-Length: 143
                    Content-Type: text/xml; charset=utf-8
                    Date: Sat, 28 Mar 2026 11:47:21 GMT
                    Server: uvicorn

                    <soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope...
Forms             : {}
Headers           : {[Content-Length, 143], [Content-Type, text/xml; charset=utf-8], [Date, Sat, 28 Mar 2026 11:47:21 GMT], [Server, uvicorn]}
Images            : {}
InputFields       : {}
Links             : {}
ParsedHtml        : System.__ComObject
RawContentLength  : 143
```

---
для будущего себя я обращаю внимание, что передать несколько заголовков в `Invoke-WebRequest` сложнее - лучше всего делать это через объявление перменной:
```PowerShell
$headers = @{
	'Content-Type' = 'text/xml;charset=UTF-8'
	'еще один заголовок' = 'что-то'
}
Invoke-WebRequest -Headers $headers ...
```
## аутентификация
### XML_request_response
API сервиса, на который я ориентируюсь, предполагает, что пользователь, перед выполнением основных обращений к ресурсу, пришлет запрос на конкретный эндпоинт с именем, паролем и какими-то другими данными.
запрос должен содержать в себе XML, где описаны нужные поля.
как выглядит такой запрос с точки зрения клиента?
```XML
<body>
	<osmreq xmlns="http://ifx.ru/IFXWebService">
		<mbci>client_type</mbci>
		<mbcv>1.0</mbcv>
		<mbh>OnluHeadline</mbh>
		<mbi>username</mbi>
		<mbla>ru-RU</mbla>
		<mbo>OS Name</mbo>
		<mbp>password</mbp>
		<mbt></mbt>
	</osmreq>
</body>
```
для такого запроса я сразу составлю класс:
```Python
class OpenSessionRequest(BodyContent, tag="osmreq"):
    mbci: str = element(tag="mbci")    # тип клиента.
    mbcv: float = element(tag="mbcv")  # версия клиента.
    mbh: str = element(tag="mbh")      # состав атрибутов новости.
    mbl: str = element(tag="mbl")      # логин пользователя.
    mbla: str = element(tag="mbla")    # язык интерфейса пользователя.
    mbo: str = element(tag="mbo")      # операционная система.
    mbp: str = element(tag="mbp")      # пароль пользователя.
    mbt: str = element(tag="mbt")      # хз.
```
ну и сразу стоит посмотреть на ответ, который должен отправить сервер:
```XML
<body>
	<osmresp xmlns="http://ifx.ru/IFXWebService">
		<mbr>false</mbr>
		<mbsid>dsfahsdjkflsdjf</mbsid>
	</osmresp>
</body>
```
класс для него будет простым:
```Python
class OpenSessionResponse(BodyContent, tag="osmresp"):
    mbr: bool = element(tag="mbr")     # хз.
    mbsid: str = element(tag="mbsid")  # идентификатор открытой сессии.
```
теперь осталось только написать метод, который будет принимать запрос пользователя на подключение и отправлять ему в ответ *cookie* и подтверждающий токен.
я не буду здесь писать кастомные ошибки (схемы и XML для них), потому что мне впадлу.
к сожалению, не все так просто, ведь в оригинальном API есть `namespace`. это означает, что `osmreq xmlns="http://ifx.ru/IFXWebService"` просто не будет прочитан классом `OpenSessionRequest`, т.к. по умолчанию `pydantic-xml` считает, что у тега `osmreq` нет никакого `namespace`, если оно не указано отдельно:
```Python
class OpenSessionResponse(
	BodyContent,
	tag="osmreq",
	nsmap={
		'': 'http://ifx.ru/IFXWebService'
	}
)
```
во1 *что такое `http://ifx.ru/IFXWebService`?*, во2 *что такое `namespace`?*, в3 *почему при указании `namespace` в `nsmap` ключ пустой?*
эта ссылка по сути является просто уникальным набором символов, который никак не влияет на сам XML, т.е. нет какого-то механизма, который существовал бы по этой ссылке и влиял бы на XML.
любой `namespace` объединяет под уникальным идентификатором (в данном случае `http://ifx.ru/IFXWebService`) данный элемент и все его дочерние элементы. насколько я понял, в XML нет прикола с дочерними `namespace` - если у тега объявлен `namespace`, то он перезаписывает любой ранее объявленный `namespace`.
запись `'': 'http://ifx.ru/IFXWebService'` означает, что я сообщаю `pydatic-xml`, что у тега `osmreq` дочерние теги обладают `namespace` по умолчанию `http://ifx.ru/IFXWebService`.
тут есть еще одна проблема - такой класс все равно не прочитает мой XML:
```XML
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/">
    <soapenv:Header>
        <Action soapenv:mustUnderstand="1" xmlns="http://schemas.microsoft.com/ws/2005/05/addressing/none">
            http://ifx.ru/IFX3WebService/IIFXService/OpenSession
        </Action>
    </soapenv:Header>
    <soapenv:Body>
        <osmreq xmlns="http://ifx.ru/IFXWebService">
            <mbci>client_type</mbci>
            <mbcv>1.0</mbcv>
            <mbh>OnlyHeadline</mbh>
            <mbi>lipkerton</mbi>
            <mbla>ru-RU</mbla>
            <mbo>OS Name</mbo>
            <mbp>super-secret</mbp>
            <mbt>what</mbt>
        </osmreq>
    </soapenv:Body>
</soapenv:Envelope>
```
и дело тут не в самом XML, а в логике `pydantic-xml`.
именно с таким XML я обращаюсь к эндпоинту:
```PowerShell
Invoke-WebRequest -Method Post -Headers @{'Content-Type' = 'text/xml;charset=UTF-8'} `
	-InFile ".\XML_requests\request_templ.xml"
	-Uri "http://localhost:8000/IFXService.svc/OpenSession"
```
возвращается мне такой ответ:
```PowerShell
Invoke-WebRequest : <soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/"><soap:Header/><soap:Body><Fault><faultcode>server</faultcode><faultstring>Internal Error: namespace alias 
soap not declared in nsmap</faultstring></Fault></soap:Body></soap:Envelope>
```
и в этом ответе сказано, что в моем `osmreq` не указано `namespace` `soap`. откуда здесь вообще `soap`? а дело в том, что на каком-то этапе `pydantic-xml` подсовывает `soap` пространство имен куда-то в логику моего XML (может быть в момент обработки ошибок?).
решить это можно путем указания пространства имен `soap` в классе:
```Python
class OpenSessionRequest(
    BodyContent,
    tag="osmreq",
    nsmap={
        '': 'http://ifx.ru/IFXWebService',
        'soap': 'http://ifx.ru/IFXWebService'
    }
):
	pass
```
точно так же я исправлю и класс для ответа:
```Python 
class OpenSessionResponse(
    BodyContent,
    tag="osmresp",
    nsmap={
        '': 'http://ifx.ru/IFXWebService',
        'soap': 'http://ifx.ru/IFXWebService'
    }
):
	pass
```
хотя по сути в классе ответа мне не нужно выстраивать логику формирования данных (т.к. он просто высылает данные) - я могу написать в нем что угодно, мне хочется, чтобы ответ моего API был максимально похож на API-референс.
теперь при запросе:
```PowerShell
Invoke-WebRequest -Method Post -Headers @{'Content-Type' = 'text/xml;charset=UTF-8'} `
	-InFile ".\XML_requests\request_templ.xml"
	-Uri "http://localhost:8000/IFXService.svc/OpenSession"
```
будет возвращаться нормальный ответ:
```PowerShell
StatusCode        : 200
StatusDescription : OK
Content           : <soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/"><soap:Header/><soap:Body><osmresp xmlns="http://ifx.ru/IFXWebService"
                    xmlns:soap="http://ifx.ru/IFXWebService"><mbr>false</mbr><mb...
RawContent        : HTTP/1.1 200 OK
                    Content-Length: 250
                    Content-Type: text/xml; charset=utf-8
                    Date: Sun, 29 Mar 2026 09:41:46 GMT
                    Server: uvicorn

                    <soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope...
Forms             : {}
Headers           : {[Content-Length, 250], [Content-Type, text/xml; charset=utf-8], [Date, Sun, 29 Mar 2026 09:41:46 GMT], [Server, uvicorn]}
Images            : {}
InputFields       : {}
Links             : {}
ParsedHtml        : System.__ComObject
RawContentLength  : 25
```
как именно я организовал саму логику эндпоинта?
мне не нужна сложная авторизация - я хочу сделать две вещи:
- проверить пользователя и пароль
- отправить в ответ куки для верификации следующих запросов
такой метод:
```Python
@soap.operation(
    name="OpenSession",
    request_model=OpenSessionRequest,
    response_model=OpenSessionResponse
)
def open_session(body: OpenSessionRequest = XMLBody(OpenSessionRequest)):
    if body.mbi not in my_user:
        return
    if not check_password(body.mbi, body.mbp):
        return
    return SoapResponse(
        OpenSessionResponse(
            mbr="false",
            mbsid=""
        )
    )
```
когда на эндпоинт поступает запрос, я проверяю его с помощью двух условий. во втором условии есть функция `check_password()`:
```Python
pwd_context = CryptContext(
    schemes=["bcrypt", "sha256_crypt"], deprecated="auto"
)

def check_password(username: str, password: str):
    return pwd_context.verify_and_update(password, my_user[username]["hashed_password"])
```
если все в порядке, то метод вернет ответ, который сформирован классом `OpenSessionResponse`.
и это в целом работает.
осталось только прикрепить к этому куки.
### cookie
сервер отправляет `cookie` на клиент в заголовке `Set-Cookie`. это как бы указание клиенту сохранить `cookie` у себя, чтобы позже их использовать.
установить `cookie` в FastAPI довольно просто:
```Python
response = SoapResponse(
	OpenSessionResponse(
		mbr="false",
		mbsid=""
	)
)
response.set_cookie(key="fakesession", value="fake-cookie-session-value")
```
теперь, когда я сделаю запрос со своим XML, то в ответ получу:
```PowerShell
RawContent        : HTTP/1.1 200 OK
	Content-Length: 250
	Content-Type: text/xml; charset=utf-8
	Date: Sun, 29 Mar 2026 12:39:21 GMT
	Set-Cookie: fakesession=fake-cookie-session-value; Path=/; SameSite=lax
	Server: uvico...
```
здесь виден ответ от сервера и заголовок `Set-Cookie`, где и стоит мой заявленный `cookie`.
# listener
чтобы сделать бесконечного слушателя я воспользуюсь шаблоном .NET - `worker`:
```PowerShell
dotnet new worker -o listener
```
такая команда создает набор файлов для фоновой программы C#. что есть в этом наборе?
два основных файла:
- `Program.cs`
- `Worker.cs`
идея такая, что я запускаю вечный фоновый процесс через точку входа - `Program.cs`. в `Program.cs` описана сборка будущего работника - ряд подготовительных этапов к запуску работника и сам запуск:
```C#
using listener;

HostApplicationBuilder builder = Host.CreateApplicationBuilder(args);
builder.Services.AddHostedService<Worker>();

IHost host = builder.Build();
host.Run();
```
первая строка - указатель на пространство имен программы, чтобы все файлы внутри данного пространства имен могли ссылаться друг на друга.
`HostApplicationBuilder` - это контейнер для настроек хоста. хост - это то, где будет крутиться моя программа. `CreateApplicationBuilder()` как раз создает такого хоста с аргументами (если они есть) командной строки.
так как `HostApplicationBuilder` является контейнером, то мне в него нужно добавить сервисы, которые будут вызваны при старте контейнера. именно это делает `AddHostedService<Worker>()`