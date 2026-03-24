я хочу создать сервис, который будет постоянно включен. демон, другими словами.
чтобы реализовать такую штуку в .NET уже есть шаблон - `worker`:
```C#
dotnet new worker -o MyWorker
```
эта команда создаст директорию сервиса и сразу напишет два основных файла - `Program.cs` и `Worker.cs`.
внутри `Program.cs` такой текст:
```C#
using InterfaxTalker;

var builder = Host.CreateApplicationBuilder(args);
builder.Services.AddHostedService<Worker>();

var host = builder.Build();
host.Run();
```
для начала я хочу изменить `var` на типы из документации (https://learn.microsoft.com/en-us/dotnet/core/extensions/workers):
```C#
HostApplicationBuilder builder = Host.CreateApplicationBuilder(args);
...
IHost host = builder.Build();
```
так будет понятно, что я использую класс:
- `HostApplicationBuilder` - позволяет строить сервисы, смысл которых - работать фоново в системе (что идеально подходит для моей задачи). именно через него я буду настраивать свой процесс.
- `IHost` - это обертка, которая заворачивает конфигурацию в готовый хост. проверяются зависимости и фиксируется конфигурация.
что вообще происходит сейчас в таком коде?
для начала инструкция:
```C#
using InterfaxTalker;
```
сообщает, что я подключаю пространство имен `InterfaxTalker` - директорию текущего проекта. делаю я это, чтобы рабочий файл `Program.cs` видел другой рабочий файл `Worker.cs`.
далее:
```C#
HostApplicationBuilder builder = Host.CreateApplicationBuilder(args);
```
`Host` - это класс, который предоставляет методы для создания хоста. в данном случае .NET сразу использует метод `CreateApplicationBuilder(args)`; этот метод дожен принять аргументы командной строки, чтобы, если нужно, с их помощью отредактировать конфигурацию приложения.
`builder` - переменная, которая будет экземпляром класса `HostApplicationBuilder`. этот класс содержит в себе конфигурацию текущего хоста (хост в данном случае является приложением, которое будет постоянно работать).
