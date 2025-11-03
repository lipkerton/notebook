# DDD
DDD - Domain-Driven Design. Способ архитектуры приложений, который подразумевает совместную работу разработчиков приложений и экспертов в области (domain).
Как правило на основе DDD работают очень большие системы.
DDD архитектура состоит из нескольких основных папок. Их полное описание и предназначение можно прочитать здесь: https://www.telerik.com/blogs/getting-started-domain-driven-design-aspnet-core
А я начну создавать проект DDD.
# SaleCommission
Гайд выше подсказывает, что нужно начать с создания решения в директории:
```Bash
dotnet new solution -n eSales
```
Параметры:
- `-n` имя для будущего решения.
Далее создаем библиотеку классов:
```Bash
dotnet new classlib -n Commission.Domain
```
Библиотека классов - это проект, который содержит используемые структуры данных C#, такие проекты не создают с целью их запуска. Грубо говоря библиотека классов - это набор вспомогательных модулей для основного проекта в группе проектов.
Однако в DDD у библиотеки классов с именем `Domain` существует особое предназначение - классы в ней описывают бизнес-логику всего проекта.
После создания библиотеки ее нужно добавить в мое решение:
```Bash
dotnet sln add Commission.Domain/Commission.Domain.proj
```
Теперь я создам внутри моего `Domain` папку `Entities` и внутри нее напишу первый файл `SaleCommission.cs`:
```C#
namespace SaleCommission
{
	public class SaleCommission
	{
		public Guid Id { get; private set; }
	}
}
```
Запись `public Guid Id` объявляет новое свойство класса `Id` с типом данных `Guid`. Приставка `{ get; private set }` означает, что это свойство может быть свободно взято и использовано вне класса, однако оно не может быть изменено вне класса, т.к. для действия `set` установлен режим `private`.
Я дописал еще переменных из гайда:
```C#
namespace SaleCommission
{
    public class SaleCommission
    {
        public Guid Id { get; private set; }
        public decimal Amount { get; private set; }
        public DateTime Date { get; private set; }
        public Guid SaleId { get; private set; }
        public string CurrencyCode { get; private set; }

    }
}
```

И последняя выдала мне предупреждение:
> Non-nullable property 'CurrencyCode' must contain a non-null value when exiting constructor. Consider adding the 'required' modifier or declaring the property as nullable.

Которое скорее всего означает, что я строковая переменная `CurrencyCode` в текущем формате может быть со значением `null` или с каким-нибудь строковым значением. Подсказка просит определить в явном формате обязательно ли эта переменная должна иметь какое-то значение по выходу из конструктора или нет.

---
Чтобы объявить переменную обязательной мне нужно поставить слово `required` в запись:
```C#
public required string CurrencyCode { get; private set; }
```
Однако в этом случае я так же столкнусь с ошибкой - C# потребует, чтобы моя переменная имела `public/private` статус не ниже, чем статус моего класса:
>Required member 'SaleCommission.CurrencyCode' cannot be less visible or have a setter less visible than the containing type 'SaleCommission'.

Она означает, что у меня возникает логическое противоречие - компилятор при компиляции попытается заполнить обязательное поле `CurrencyCode`, но не сможет, потому что `setter` для поля является приватным.
`required` как бы говорит, что свойство должно быть установлено при инициализации объекта класса, а `private` говорит, что оно может быть установлено только изнутри класса.

---
В сущности мне все равно будет ли эта переменная равна `null` или будет заполнена строкой, поэтому я могу заверить компилятор, что мне похуй.
Это делается с помощью знака `?` рядом с типом переменной:
```C#
public string? CurrencyCode { get; private set; }
```

---
Вообще почему возникает это предупреждение здесь и не возникает, например, когда я пишу:
```C#
public bool IsProcessed { get; private set; }
```
Потому что в  C# есть типы данных, которые без инициализации задают значение `null` своей переменной и `string` является именно таким типом. А вот `bool` нет, т.к. если я не задам `bool`, то внутри этой переменной просто будет лежать `False`.

---
Теперь я продолжу код:
```C#
using System.ComponentModel.DataAnnotations;

namespace SaleCommission
{
    public class SaleCommission
    {
        public Guid Id { get; private set; }
        public decimal Amount { get; private set; }
        public DateTime Date { get; private set; }
        public Guid SaleId { get; private set; }
        public string? CurrencyCode { get; private set; }
        public string? CommissionType { get; private set; }
        public bool IsProcessed { get; private set; }

        public Currency Currency => new Currency(CurrencyCode);
    } 
}
```
Последняя строка означает, что мы создаем свойство для класса `SaleCommission`, но не простое, как предыдущие, а особенное. Строка говорит, что она будет создавать новые экземпляры класса `Currency` со значением `CurrencyCode` каждый раз, когда я обращаюсь к свойству `Currency` класса `SaleCommission`.

---
Теперь важно напрячься. На самом деле все это время я делал модель будущей сущности в БД. По аналогии с фреймворками для взаимодействия с БД в Python, в C# так же есть фреймворк для взаимодействия с БД - **Entity Framework**, который позволяет создавать модели и на основе моделей делать таблицы и грузить в них данные.
Когда я объявлял свойства класса `SaleCommission`, я задавал поля для сущности в БД. Теперь мне нужно добавить в код строку, которая позволит Entity Framework читать данные из БД и превращать их в объекты C#:
```C#
using System.ComponentModel.DataAnnotations;

namespace SaleCommission
{
    public class SaleCommission
    {
        public Guid Id { get; private set; }
        public decimal Amount { get; private set; }
        public DateTime Date { get; private set; }
        public Guid SaleId { get; private set; }
        public string? CurrencyCode { get; private set; }
        public string? CommissionType { get; private set; }
        public bool IsProcessed { get; private set; }

        public Currency Currency => new Currency(CurrencyCode);

		private SaleCommission() {}
    } 
}
```
Я долго не мог понять зачем нужна эта строка. Для начала стоит разобраться что она делает - это приватный конструктор класса для моего же класса `SaleCommission`. Он нужен для EF, чтобы читать данные из БД.
EF прочитает данные таким образом:
1) вызовет приватный пустой конструктор класса
2) возьмет данные из БД
3) значения полей из БД он подставит в свойства созданного экземпляра класса.
Пока что я не углубляюсь в причину такого поведения, просто говорю, что для EF важно иметь пустой конструктор, который поможет ему читать БД и отдавать данные в код.

---
Далее я делаю второй конструктор, но уже публичный:
```C#
public SaleCommission(
	Guid sale_id,
	decimal amount,
	DateTime date,
	Currency currency,
	string commission_type
)
{
	if (amount <= 0) throw new ArgumentException(
		"Amount must be greater then zero."
	);
	if (string.IsNullOrEmpty(commission_type)) throw new ArgumentException(
		"CommissionType cannot be null or empty."
	);

	Id = Guid.NewGuid();
	SaleId = sale_id;
	Amount = amount;
	Date = date;
	CurrencyCode = currency.Code;
	CommissionType = commission_type;
	IsProcessed = false;
}
```
Этот конструктор принимает все параметры, которые указаны в свойствах класса и валидирует часть из них:
```C#
if (amount <= 0) throw new ArgumentException(
	"Amount must be greater then zero."
);
```
Здесь видно условную конструкцию с условием `amount <= 0`, за неисполнение которой выбрасывается исключение о неверном значении аргумента.
Этот конструктор нужен, чтобы записывать новые данные в БД.
Почему EF использует пустой приватный конструктор для извлечения данных и публичный конструктор для загрузки данных? Потому что публичный конструктор буду использовать я, следовательно, данные должны быть валидированы + параметр `currency` является экземпляром класса, который я не могу прямо так в нативном виде разместить в БД, поэтому в этом публичном конструкторе из экземпляра извлекается его свойство:
```C#
CurrencyCode = currency.Code;
```
## конструкторы
В C#, как и в Python, существует понятие "конструктор", однако работает оно немного иначе.
Во-первых, конструктор своей записью напоминает стандартный метод класса:
```C#
public class MyClass
{
	public decimal Number { get; set; }

	public MyClass() 
	{
		Number = 0;
	}
}
```
Однако важный момент состоит в том, что имя конструктора *точно повторяет имя класса*. 
Метод же выглядит так:
```C#
public class MyClass
{
	public decimal Number { get; set; }

	public MyClass()
	{
		Number = 0;
	}

	public decimal Calc()
	{
		return Number * 0.2;
	}
}
```
У метода так же есть возвращаемый тип.
В данном случае конструктор будет использован, если я введу такой код:
```C#
var number = new MyClass();
```
Но что если я хочу добавить в свой класс более сложную логику? Например, могу ли я сделать так, чтобы у меня была возможность и создать экземпляр со значением `Number = 0`, и создать экземпляр с произвольным значением?
```C#
public class MyClass
{
	public decimal Number { get; set; }

	public MyClass()
	{
		Number = 0;
	}
	public MyClass(decimal number)
	{
		Number = number
	}
}
```
Да, могу. Это называется **перегрузка (overloading)** - несколько конструкторов, чтобы контролировать создание экземпляров класса.
## принципы DDD
У создания классов в DDD есть примерно такой же набор принципов, как и у создания классов в Python в парадигме ООП.
В данном фрагменте кода были реализовано четыре из них:
- **Инкапсуляция (Encapsulation)** - инкапсуляция означает набор правил для создания экземпляров данного класса, которые ограничивают определенным образом процесс инициализации его объектов. Например, выше я написал два конструктора и один сразу сделал приватным и это пример работы инкапсуляции - я запретил создавать пустые сущности на основании класса, чтобы не загрузить в БД пустые объекты.
- Инварианты ()