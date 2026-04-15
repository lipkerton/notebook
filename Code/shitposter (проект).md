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
так как `HostApplicationBuilder` является контейнером, то мне в него нужно добавить сервисы, которые будут вызваны при старте контейнера. именно это делает `AddHostedService<Worker>()`.
когда я говорю, что добавляю сервисы, то на самом деле я имею в виду классы, которые записываются в этот контейнер. смысл контейнера в механике *DI (Dependency Injection)* и, если говорить кратко, то она заключается в автоматической инициализации объектов классов и параметров.
## DDD
и в этот момент ревьюер написал мне, что я дурачок и сделал все не так. так как C# довольно масштабный язык с кучей нюансов, на нем необходим писать чисто. особенно важно в таком контексте правильно выстроить структуру приложения.
основной ориентир - *domain driven development (DDD)*. эта структура применяется, когда у приложения есть бизнес-логика + логика разработки и они находятся раздельно. чтобы это работало приложение делится на четрые части:
- `Domain` - модели и константы
- `Infrastructure` - репозитории (логика хранения данных)
- `Application` - бизнес логика
- `API` - интерфейс всего приложения
так же есть несколько мелких вещей. 
`Domain` должен содержать описание моделей приложения, но при этом не должен быть раздутым за счет любой дополнительной логики. например, я не могу размещать там интерфейсы для классов `Infrastructure`, так как скорее всего реализация таких интерфейсов требует дополнительных пакетов NuGet, которых в `Domain` быть не должно.
`Infastructure` реализует только логику работы с хранилищами.
`Application` - это вроде оркестратора, который должен запускать бизнес-логику и интегрировать в нее хранилища и модели из `Domain`.
`API` - это точка входа в приложение + настройки приложения.
## связи
я сделал `Infrastructure`, `Application`, `Domain` участки приложения с помощью шаблона `classlib`:
```PowerShell
dotnet new classlib -o listener.Infrastructure
dotnet new classlib -o listener.Application
dotnet new classlib -o listener.Domain
```
а вот точка входа будет реализована за счет шаблона `worker`:
```PowerShell
dotnet new worker -o listener.API
```
теперь эти проекты нужно связать между собой референсами, иначе они не будут знать друг о друге ничего. у референсов так же есть своя система - они долны идти лесенкой. `API` ссылается на `Application`, который ссылается на `Ifrastructure`, который ссылается на `Domain`.
чтобы их связать нужно использовать:
```PowerShell
dotnet add reference .\listener.Infrastructure\ --project .\listener.Domain\
```
после команды появившаяся ссылка будет отображаться в файле проекта `listener.Domain`:
```XML
<ItemGroup>
  <ProjectReference Include="..\listener.Infrastructure\listener.Infrastructure.csproj" />
</ItemGroup>
```
такое нужно проделать со всеми проектами:
```PowerShell
dotnet add reference .\listener.Application\ --project .\listener.Infrastrucrture\
dotnet add reference .\listener.API\ --project .\listener.Application\
```
забавный факт - потом, когда я буду устанавливать пакеты в нужные проекты, эти пакеты будут наследоваться каждым проектом выше в цепочке. так, `Domain` вообще не знает о существовании всех остальных проектов в приложении, а `Application` будет наследовать пакеты для работы с репозиториями из `Infrastructure`.
## domain
чтобы не делать криво, в `Domain` нужно прописать структуру объекта новости. когда я получу новость от API мне нужно будет распарсить ее в этот объект.
сам объект будет классом, но я не могу просто создать файл и запихать туда класс. каждый такой класс должен хранится отдельно от другого в папке `Entities`:
```C#
namespace listener.Domain.Entities;

public class NewsItem
{
    public required string Id { get; set; }
    public required DateTime PubDate { get; set; }
    public required string Header { get; set; }
    public string Content { get; set; } = string.Empty;
}
```
свойство `required` предполагает, что в экземпляре класса `NewsItem` обязано быть значение для этого поля. тут стоит сразу обсудить разницу между полями:
```C#
public required string Id { get; set; }
// ИЛИ
public string? Id { get; set; }
// ИЛИ
public string Id { get; set; } = string.Empty;
```
первое означает, что внутри поля обязано быть значение и оно не может быть пустой строкой. т.е. такая запись:
```C#
public required string Id { get; set; } = string.Empty;
```
не прокатит при инициализации экземпляра класса без значения.
запись `string?` означает, что внутрь свойства при инициализации экземпляра может быть записана либо строка, либо `null`, а запись `= string.Empty` означает, что при инициализации экземпляра без значения для поля в него будет записана пустая строка.
и это все, что пока будет написано внутри `Domain`.
# infrastructure
как я уже сказал, тут должна быть реализация репозиториев. каждый репозиторий должен являться классом, но для каждого из них должен быть написан интерфейс для обращения.
стоит вспомнить, что основа всего проекта - это один *DI* контейнер, к которому я подключаю все остальное в формате сервисов. репозитории так же являются сервисами. поэтому их нужно прописать в директории `Services`. но начну я с интерфейсов - они находятся внутри `Services` в директории `Interfaces`. 
## logrepository
для теста пусть репозиторием для новостей будут консольные логи:
```C#
using listener.Domain.Entities;

namespace listener.Infrastructure.Repositories.Interfaces;

public interface ILogRepository
{
    Task SaveNews(NewsItem[] newsItems, CancellationToken cancelToken);
}
```
что такое интерфейсы? интерфейс это что-то вроде описания того, что должен реализовать класс, который ссылается на этот интерфейс. например, в данном случае мой будущй класс `LogRepository` должен реализовать метод `SaveNews` с указанными параметрами.
прикол интерфейсов не только в том, что это просто описание для будущего класса, но еще и в том, что классы могут наследовать несколько интерфейсов (при том, что класс может наследовать только один другой класс). кстати, интерфейс не накладывает ограничения на то как именно будет реализован метод - синхронный или асинхронный.
тут есть еще один интересный момент: ранее я написал для метода `SaveNews` параметр `IEnumerable<NewsItem>`. по сути `NewsItem[]` и `IEnumerable<NewsItem>` схожие в данном контексте конструкции, но чем они на самом деле различаются?
`NewsItem[]` - это массив. массив в C# хранит непрерывно свои элементы в памяти. к элементам массива можно получить доступ по индексу. `IEnumerable<NewsItem>` - это голый интерфейс любого перечисления; в нем нет реализации доступа по индексу, нет возможности изменить элемент по индексу и т.д.; при этом `IEnumerable` может стать другой коллекцией, т.к. у него есть методы трансформации.
в данном случае я отдаю предпочтение массиву, потому что всегда нужно держать в голове, что проект может масштабироваться в будущем, поэтому внутри сервиса мне может понадобится работа с элементами внутри массива.
когда интерфейс сделан, можно пойти написать реализацию. реализацию я буду писать  внутри `Services`:
```C#
using listener.Infrastructure.Repositories.Interfaces;

namespace listener.Infrastructure.Repositories;

public class LogRepository : ILogRepository
{
    private readonly ILogger<LogRepository> _logger;

    public LogRepository(
        ILogger<LogRepository> logger
    )
    {
        _logger = logger;
    }
}
```
пока что я только сделал класс, а внутрь него поместил атрибут класса `_logger` и конструктор класса `LogRepository(ILogger<LogRepository> logger){...}`. конструктор класса будет вызываться каждый раз, когда будет создаваться экземпляр класса. внутрь экземпляра каждый раз будет передан объект `ILogger<LogRepository>`.
теперь внутри класса нужно написать метод `SaveNews()`:
```C#
public Task SaveNews(
	NewsItem[] newsItems,
	CancellationToken cancelToken
)
{
	foreach (NewsItem newsItem in newsItems)
	{
		_logger.LogInformation(">>> Получена новость:\nID: {Id}\nHeader: {Header}\n", newsItem.Id, newsItem.Header);

		if (!string.IsNullOrEmpty(newsItem.Content))
		{
			_logger.LogDebug("Сontent: {Content}\n", newsItem.Content);
		}
	}
	return Task.CompletedTask;
}
```
метод работает асинхронно. внутри метода цикл `foreach` перебирает полученные новости и печатает каждую новость в лог.
хотя это не самая правильная реализация репозитория (т.к. здесь нет, строго говоря, репозитория - метод просто печатает новости в лог), тут есть пара интересных моментов. во-первых, я реализую метод в соответствии с интерфейсом, потому что это вольность, которую позволяет сделать интерфейс. во-вторых, я делаю внутри цикл `foreach`, который будет перебирать элементы в списке новостей один за другим. внутри для каждой новости логгер будет печатать информацию в лог.
объект `Task` - это задача, которую я могу запустить в асинхронном формате и синхронном формате. эту задачу я повешу как сервис и зарегистрирую его в контейнере. именно поэтому в конце функции я верну `CompletedTask`, чтобы задача была обозначена как завершенная в списке задач.

>[!important] если внутри метода нет `await`, то он считается синхронным, поэтому применение к нему слова `async` (`public async Task`) будет лишним - он все равно будет выполнен синхронно и компилятор об этом предупредит.

зачем, если метод синхронный, мне вообще нужна реализация через `Task`? формально, не нужна, но я уже прописал в своем интерфейсе требование, чтобы мой  метод возвращал `Task`, так же как и интерфейсы всех других репозиториев будут возвращать `Task`. ну и потом, кто знает, может быть в метод все же будут добавлены асинхронные команды и тогда мне не нужно будет вносить много правок в интерфейс.
## jsonrepository
дополнительно к реализации репозитория через логгирование, я могу так же сделать реализацию репозитория через запись в JSON файлы. сначала интерфейс:
```C#
using listener.Domain.Entities;

namespace listener.Infrastructure.Repository.Interfaces;

public interface IJsonRepository
{
    Task SaveNews(NewsItem[] newsItems, CancellationToken cancelToken);
}
```
и сама реализация:
```C#
using System.Text.Json;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;
using listener.Domain.Entities;
using listener.Infrastructure.Repositories.Interfaces;

namespace listener.Infrastructure.Repositories;

public class JsonRepository : IJsonRepository
{
    private readonly ILogger<JsonRepository> _logger;
    private readonly RepositorySettings _settings;

    public JsonRepository(
        ILogger<JsonRepository> logger,
        IOptions<RepositorySettings> settings
    )
    {
        _logger = logger;
        _settings = settings.Value;
    }
}
```
как и в прошлый раз, я написал тут только начало, но тут уже куча интересных механик. самое новое - `IOptions`. `IOptions` - это надстройка над `IConfiguration`, а `IConfiguration` позвояет брать из `appsettings.json` параметры, которые мне нужны для работы `JsonRepository`.
для начала следует посмотреть на сами параметры:
```JSON
"RepositorySettings": {
  "JsonRepository": {
	"JsonResultFolder": "JsonResult",
	"CleanupInterval": 30
  }
}
```
так как я планирую добавить еще один репозиторий - для него будет сделана еще одна порция настроек. поэтому `JsonRepository` является вложенным словарем с двумя настройками - папкой для хранения результатов и значением промежутка между очистками этой директории результатов.
эти параметры я планирую использовать в реализации интерфейса `IJsonRepository`. чтобы их вытащить можно воспользоваться `IConfiguration` или `IOptions`.
`IConfiguration` в целом прост и понятен для использования:
```C#
string jsonResultFolder = _configuration.GetValue<string>(
	"APISettings:RepositorySettings:JsonRepository:JsonResultFolder",
	"JsonResult"
);
```
здесь я через экземпляр `IConfiguration` обращаюсь к `appsettings` с параметрами, как к словарю.
однако с этим есть проблема - так как я по сути вбиваю такую большую строку первым аргументом для метода `GetValue`, то я легко могу ошибиться где-то по пути.
чтобы этого не допустить можно использовать `IOptions`. `IOptions` предлагает сделать схему параметров из `appsettings`, чтобы сериализовать `appsettings.json` в объекты C#:
```C#
namespace listener.Domain.Configuration;

public class RepositorySettings
{
    public JsonRepository jsonRepository { get; set; } = new();
}

public class JsonRepositorySettings
{
    public string jsonResultFolder { get; set; } = string.Empty;
    public DateTime cleanupInterval { get; set; }
}
```
когда я сделал такие схемы, я могу их использовать в своем классе в таком формате:
```C#
private readonly RepositorySettings _settings;
...
IOptions<RepositorySettings> settings;
```
при этом внутри `settings` мои параметры будут лежать в ключе `Value`:
```C#
_settings = settings.Value;
...
_settings.jsonRepository.jsonResultFolder;
```
кстати, `IOptions` видит параметры в `appsettings.json`, т.к. он по сути пользуется механизмами `IConfiguration`, который является регистрируемым в *DI* контейнере сервисом. но к регистрации сервисов я вернусь позже.
главное, что теперь внутри моего `JsonRepository` я могу написать корректно метод `SaveNews`:
```C#
public async Task SaveNews(NewsItem[] newsItems, CancellationToken cancelToken) {
	try
	{
		string jsonResultFolder = _settings.jsonRepository.jsonResultFolder;
		if (!Directory.Exists(jsonResultFolder)) Directory.CreateDirectory(jsonResultFolder);
		string fileName = $"{DateTime.UtcNow:yyyyMMddHHmmssfffffff}.json";
		string filePath = Path.Combine(jsonResultFolder, fileName);

		JsonSerializerOptions jsonOptions = new JsonSerializerOptions { WriteIndented = true };
		string json = JsonSerializer.Serialize(newsItems, jsonOptions);

		await File.WriteAllTextAsync(filePath, json, cancelToken);
		_logger.LogInformation("Сохранено {count} новостей в файл: {path}", newsItems.Count(), filePath);
	}
	catch (Exception ex)
	{
		_logger.LogError("Ошибка при сохранении новостей в файл! {ex}", ex);
	}
}
```
здесь я сначала получаю путь к результирующей папке из `appsettings.json` и тут же выполняю проверку "а существует ли такая папка вообще?" - если директории нет, то я ее создаю с помощью стандартной библиотеки `Directory`.
далее я комбинирую путь до директории с именем файла, куда вшит форматер даты и времени, чтобы имя файла состояло только из уникальной комбинации цифр.
внутри C# есть удобный сериализатор JSON файлов, который дает мне возможность просто взять мой массив с объектами `NewsItems` и распаковать его в готовый JSON объект.
## configureservices
когда интерфейсы и реализации интерфейсов сделаны, осталось только зарегать новые сервисы в главном файле библиотеки - `ConfigureServices.cs`:
```C#
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using listener.Domain.Configuration;
using listener.Infrastructure.Repositories;
using listener.Infrastructure.Repositories.Interfaces;

namespace listener.Infrastructure;

public static class ConfigureServices
{
    public static IServiceCollection AddInfrastructureServices(
        this IServiceCollection services,
        IConfiguration configuration
    )
    {
        services.Configure<RepositorySettings>(
            configuration.GetSection("RepositorySettings")
        );
        services.AddScoped<IJsonRepository, JsonRepository>();
        services.AddScoped<ILogRepository, LogRepository>();
        return services;
    }
}
```
здесь я беру статический класс `IServiceCollection`, который должен регистрировать все сервисы, которые я описал в `Infrastructure`. грубо говоря, в моем проекте есть коллекция сервисов, которые лежат в контейнере и будут задействованы на старте приложения. в этом файле я как бы регистрирую ту часть сервисов, которые относятся к `Infrastructure`.
в классе `ConfigureServices` реализован один единственный метод - `AddInfrastructureServices`, который должен вернуть список сервисов. сервисы эти записываются в контейнер типа `IServiceCollection`.
что за `this IServiceCollection services`? это *метод расширения*. его механика такова: берем оригинальный объект (класс `IServiceCollection`) и добавляем к нему новый метод. именно добавление нового метода и называется *расширением*.
а как это будет выглядеть в итоговом `Program.cs`?
```C#
services.Service.AddInfrastructureServices(builder.Configuration);
```
как именно `Program.cs` понимает, что метод был расширен? все просто - на стадии компиляции C# сканирует код и находит статические классы с *методами расширения*.
после того как сервис добавлен часть `Infrastructure` закончена.
# application
теперь нужно описать `application`. это место, где происходит вся бизнес-логика приложения и, следовательно, вся грязь.
здесь мне нужно написать как минимум два сервися и два интерфейса - новостной сервис-оркестратор и сервис, который содержит все методы для обращения к API. мне хочется их разделить, чтобы написать внутри второго сервиса только саму логику методов; при этом определить когда и как их запускать я хочу в первом сервисе.
## interfaxgateway
интерфейс этого сервиса будет таким:
```C#
using listener.Domain.Entities;
namespace listener.Application.Services.Interfaces;

public interface IInterfaxGateway
{
    Task<bool> OpenSession (CancellationToken cancelToken);
    Task<NewsItem[]?> GetRealtimeNewsByProduct (CancellationToken cancelToken);
    Task<NewsItem> GetEntireNewsByID (NewsItem newsItem, CancellationToken cancelToken);
}
```
три метода с обращениями на разные эндпоинты. первый будет писать запрос на аутентификацию на сервисе. второй отправит запрос для получения новостей за период. третий для каждой новости, которая была получена предыдущим запросом, отправляет запрос для получения содержания новости.
тут сложность в том, что каждый метод должен что-то вернуть. `OpenSession` простой - возвращает успех или неуспех операции аутентификации. `GetRealtimeNewsByProduct` сложнее, потому что он должен вернуть либо список новостей, либо ничего в случае неудачи - именно поэтому в возвращаемом типе метода есть знак вопроса `Task<NewsItem[]?>`. третий метод *всегда* должен возвращать объект `NewsItem` и при этом  принимает он на вход всегда список из объектов `NewsItem`.
теперь реализация:
```C#
public class InterfaxGateway : IInterfaxGateway
{
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly ILogger<InterfaxGateway> _logger;
    private readonly APISettings _settings;

    public InterfaxGateway(
        IHttpClientFactory httpClientFactory,
        ILogger<InterfaxGateway> logger,
        IOptions<APISettings> settings
    )
    {
        _httpClientFactory = httpClientFactory;
        _logger = logger;
        _settings = settings.Value;
    }
}
```
когда я делаю вступление к классу, то все выглядит примерно так же как и в классах-репозиториях - мне нужно получить логгер и настройки. здесь так же добавляется  `HttpClientFactory` - это генератор экземпляров класса `HttpClient`, который будет по запросу создавать мне клиентов для обращения к API.
если коротко, то фабрика помогает избежать издержек создания экземпляров `HttpClient` вручную. нужно это потому, что создание `HttpClient` вручную всегда требует сокет (сокет это связка IP и порта), т.е. занимает свободный порт компьютера; при создании кучи клиентов вручную есть риск, что сокеты вскоре закончатся, потому что даже при завершении TCP соединения сам сокет для него еще занят некоторое время (это называется `TIME_WAIT`).
теперь я напишу первый метод - он должен выполнять аутентификацию:
```C#
public async Task<bool> OpenSession(CancellationToken cancelToken)
{
	HttpClient client = _httpClientFactory.CreateClient("SOAPClient");
	string APIUrl = _settings.OpenSession.Endpoint;
	string xmlPath = Path.Combine(
		_settings.XMLRequestsFolder,
		_settings.OpenSession.XML
	);
	string xmlContent = await File.ReadAllTextAsync(xmlPath, cancelToken);
	
	StringContent content = new StringContent(xmlContent, Encoding.UTF8, "text/xml");

	_logger.LogInformation("Запрос на {url}", APIUrl);
	HttpResponseMessage response = await client.PostAsync(APIUrl, content, cancelToken);

	string responseContent = await response.Content.ReadAsStringAsync(cancelToken);
	_logger.LogInformation("{Body}", responseContent);
	return response.IsSuccessStatusCode;
}
```
сразу же первая строка тут немного смущает - что такое `SOAPClient`? об этой штуке можно думать, как о профиле клиента, на основе которого фабрика будет создавать клиента для будущего запроса.
сам профиль создается при регистрации сервиса:
```C#
services.AddSingleton<CookieContainer>();
services.AddHttpClient("SOAPClient")
	.ConfigurePrimaryHttpMessageHandler(sp => {
		CookieContainer cookieContainer = sp.GetRequiredService<CookieContainer>();
		return new HttpClientHandler
		{
			CookieContainer = cookieContainer,
			UseCookies = true
		};
	}
);
```
так как мне нужно будет работать с куки, я регистрирую сначала синглтон куки контейнера. затем я пишу свой профиль HTTP клиента, чтобы так же добавить его в контейнер.
`ConfigurePrimaryHttpMessageHandler()` - это метод, который будет создавать обработчик сообщений HTTP. он будет в автоматическом режиме следить за всем, что необходимо для обмена HTTP запросами - куки, сертификаты, прокси и т.д. почему это `Primary` обработчик? потому что в C# реализована цепочка обработчиков HTTP обмена, а `Primary` - это основной обработчик, который непосредственно отправляет запрос и получает ответ. метод `ConfigurePrimaryHttpMessageHandler()` позволяет мне сконфигурировать такой обработчик и здесь я его конфигурирую так, чтобы все мои HTTP запросы использовали куки.
