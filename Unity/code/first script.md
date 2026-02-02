стартовый туториал по unity дает такой скрипт управления персонажем:
```C#
using UnityEngine;
using UnityEngine.InputSystem; 

/// <summary>
/// Moves forward/backward and rotates with WASD/Arrow keys.
/// </summary>
public class PlayerController : MonoBehaviour
{
    [Tooltip("Forward/back speed (units/sec).")]
    public float speed = 5.0f;

    [Tooltip("Turn speed (degrees/sec).")]
    public float rotationSpeed = 120.0f;

    private Rigidbody rb; 

    private void Start()
    {
        rb = GetComponent<Rigidbody>();
        if (rb == null) Debug.LogWarning("нужно установить rigidbody");
    }

    private void FixedUpdate() 
    {
        Vector2 moveInput = Vector2.zero;

        // Forward/backward
        if (Keyboard.current.wKey.isPressed || Keyboard.current.upArrowKey.isPressed) moveInput.y = 1f;
        if (Keyboard.current.sKey.isPressed || Keyboard.current.downArrowKey.isPressed) moveInput.y = -1f;

        // Left/right (rotation)
        if (Keyboard.current.aKey.isPressed || Keyboard.current.leftArrowKey.isPressed) moveInput.x = -1f;
        if (Keyboard.current.dKey.isPressed || Keyboard.current.rightArrowKey.isPressed) moveInput.x = 1f;

        // Move in facing direction 
        Vector3 movement = transform.forward * moveInput.y * speed * Time.fixedDeltaTime;
        rb.MovePosition(rb.position + movement);

        // Y-axis rotation (invert when going backwards)
        float turnDirection = moveInput.x;
        if (moveInput.y < 0)
            turnDirection = -turnDirection;

        float turn = turnDirection * rotationSpeed * Time.fixedDeltaTime;
        Quaternion turnRotation = Quaternion.Euler(0f, turn, 0f);
        rb.MoveRotation(rb.rotation * turnRotation);
    }
}
```
и я хочу его построчно разобрать.
в первых двух строках происходят "импорты":
```C#
using UnityEngine;
using UnityEngine.InputSystem;
```
`UnityEngine` - это основная библиотека для работы с элементами Unity.
`UnityEngine.InputSystem` - это что-то вроде фреймворка для работы с вводом в Unity.
далее:
```C#
/// <summary>
/// Moves forward/backward and rotates with WASD/Arrow keys.
/// </summary
```
это интересная штука. это XML комментарий. в отличие от обычного комментария этот добавляет немного интерактивности в код - если навести мышкой на класс, то у него появится описание:
![[comment1.png]]
объявление класса выглядит так:
```C#
public class PlayerController : MonoBehaviour
```
`PlayerController` - это мой кастомный класс, а вот `MonoBehaviour` - это класс из библиотеки `UnityEngine`, от которого я здесь выполняю наследование.
`MonoBehaviour` - базовый класс в контексте Unity. он связывает между собой объект из мира игры и скрипт, поэтому фактически является единственным интерфейсом в Unity, через который можно отправлять инструкции объекту.
>[!important] только классы, которые унаследованы от `MonoBehaviour` могут быть компонентами объектов Unity. только они могут описывать скрипт для поведения объекта.

в блоке кода класса:
```C#
[Tooltip("Forward/back speed (units/sec).")]
public float speed = 5.0f;

[Tooltip("Turn speed (degrees/sec).")]
public float rotationSpeed = 120.0f;
```
`Tooltip` - это подсказка, которая будет появляться в инспекторе, при наведении на поле `Speed`:
![[comment2.png]]
я так понял, что подсказка знает где появится из-за имени переменной.
так как сами поля `Speed` и `RotationSpeed` не объявляются отдельно в коде скрипта, мне кажется, что эти поля автоматически подхватываются в Unity Editor, потому что они являются атрибутами класса-наследника `MonoBehaviour`.
под каждым `Tooltip` объявляется переменная, которая хранит `float` число. используется именно `float` (а не `double`), потому что он занимает меньше места (4 байта против 8 байт), следовательно, оказывает меньшую нагрузку на GPU.
`speed` будет отвечать за скорость объекта на сцене, а `rotationSpeed` - за скорость вращения объекта по сцене.
далее объявляется переменная `rb`:
```C#
private Rigidbody rb;
```
я остановлюсь на ее применении позже, но здесь важно, что свойство переменной `private` не позволяет инспектору захватить ее, следовательно, не позволяет ее увидеть вместе с `Speed` и `RotationSpeed`.
затем начинается первый метод:
```C#
private void Start()
{
	rb = GetComponent<Rigidbody>();
	if (rb == null) Debug.LogWarning("нужно установить rigidbody");
}
```
метод `Start()` - это событийный метод, который описан в классе `MonoBehaviour` и наследуется из него. он вызывается ровно один раз перед началом жизненного цикла.
у каждого скриптового объекта в коде есть жизненный цикл, который выполняется всегда. жизненный цикл выполняют специальные методы. каждый специальный метод берется из класса `MonoBehaviour`. поведение методов можно определить самостоятельно, как я делаю тут.
метод `GetComponent<T>()` просит `GameObject` отдать ему компонент `T`. в данном случае компонент `Rigidbody` должен быть записан в переменную `rb`, которую я объявил ранее. если компонента у объекта не будет, то вместо него в переменную будет записан `null`.
`if (rb == null)` выражение, которое проверяет положился ли компонент `Rigidbody` в переменную или в нее попал `null`. я так думаю, что такой код написан разработчиком специально, чтобы он выкидывал ошибку.
ошибка, которая будет появляться описана тут: `Debug.LogWarning("нужно установить rigidbody")`. так как у моего объекта реально нет никакого `Rigidbody`, то ошибку можно увидеть при тесте:
![[comment3.png]]
на самом деле, ошибка была допущена мной по моей глупости - я прикрепил скрипт к одному из компонентов составного объекта, а не ко всему объекту. мой пылесос состоит из цилиндра и кубика:
![[comment4.png]]
и я случайно накинул скрипт только на `Cylinder`, у которого не было `Rigidbody`.
retarded development
после того как метод `Start()` завершился сразу начинается метод `FixedUpdate()`. его фишка в том, что он будет выполняться каждый равный промежуток времени (каждые $0.02$ секунды). это обеспечивает всем вычислениям физики стабильность и постоянность.
исходя из документации, `FixedUpdate()` предназначен для записи всего того, что связано с физикой. например, его следует использовать, когда мы применяем силу к компоненту `Rigidbody`.
```C#
public void FixedUpdate()
{
	Vector2 moveInput = Vector2.zero;
	...
}
```
переменная `moveInput` имеет тип `Vector2` и значение `Vector2.zero`. на самом деле такая запись идентична такой:
```C#
Vector2 moveInput = new Vector2(0, 0);
```
каждые $0.02$ секунды значение вектора будет устанавливаться в нулевую позицию.
вообще зачем тут вектор?
вектор - это то благодаря чему можно будет определить варианты движения игрока. у него есть две оси - $Ox$ и $Oy$, следовательно, если игрок, например, нажмет клавишу `W` можно будет добавить к первому значению вектора единицу и передвинуть объект вперед.
```C#
    // Forward/backward
    if (Keyboard.current.wKey.isPressed || Keyboard.current.upArrowKey.isPressed)   moveInput.y = 1f;
    if (Keyboard.current.sKey.isPressed || Keyboard.current.downArrowKey.isPressed) moveInput.y = -1f;

    // Left/right (rotation)
    if (Keyboard.current.aKey.isPressed || Keyboard.current.leftArrowKey.isPressed) moveInput.x = -1f;
    if (Keyboard.current.dKey.isPressed || Keyboard.current.rightArrowKey.isPressed)moveInput.x = 1f;
```
в этом коде мы привязываем к клавишам передвижения.
на примере первой условной конструкции:
```C#
if (Keyboard.current.wKey.isPressed || Keyboard.current.upArrowKey.isPressed)
```
обращаюсь к текущему состоянию клавиатуры `Keyboard.current` к клавише `W` (через атрибут `wKey`) и к ее булевому атрибуту `isPressed`, чтобы понять нажата ли клавиша. символ `||` означает логическое ИЛИ, поэтому после него я выполняю то же самое, только вместо клавиши `W` спрашиваю атрибут `isPressed` у объекта верхней стрелочки.
если какое-нибудь или оба условия верны, то выполниться такой код:
```C#
moveInput.y = 1f;
```
к оси $Oy$ добавляется единица в типе `float`.
условия идут одно за другим и на практике это будет выражено в том, что, нажимая несколько клавиш (например, `WD`), программа будет запускать сразу несколько обновлений вектора в промежуток $0.02$ секунды. следовательно, я смогу двигаться диагонально и т.д.
дальше начинаются трудности - формулы:
```C#
Vector3 movement = transform.forward * moveInput.y * speed * Time.fixedDeltaTime;
```
`Vector3` - это трехмерный вектор с тремя осями $Ox, Oy, Oz$.
`transform.forward` - это специальный объект, который указывает "вперед", относительно объекта, которым я управляю (синяя стрелка). значение этого объекта я умножаю на текущее значение атрибута `y` у `moveInput`, а затем умножаю на константу `speed`, то есть на скорость объекта.
`Time.fixedDeltaTime` - это важный параметр, потому что в нем хранится период вызова функции `FixedUpdate()`. если не умножать на него значение, которое у нас получилось в результате прошлого произведения, то при разном значении счетчика кадров будет разная скорость (я обязательно напишу об этом отдельно).
вопрос: я сказал, что тип `Vector3` - это трехмерный вектор, но почему я записываю в него буквально одно значение?
здесь происходит магия векторной математики - `transform.forward` содержит трехмерный вектор (0 0, 1). представим, что `moveInput.y = 1`, тогда получится такая формула:
$$x = \begin{bmatrix}
0 \\ 0 \\ 1
\end{bmatrix} \times 1 \times 5 \times 0.02
$$
следовательно
$$
x = \begin{bmatrix}
0 \\ 0 \\ 0.1
\end{bmatrix}
$$
ну и нельзя не сказать, что
$$
время \times скорость = движение
$$
следовательно
```C#
speed * Time.fixedDeltaTime = фиксированное движение
```
в итоге переменная `movement` будет хранить вектор, который покажет куда и с какой скоростью должен сдвинуться объект.
все это нужно, чтобы правильно использовать метод физического передвижения:
```C#
rb.MovePosition(rb.position + movement);
```
метод `movePosition()` телепортирует объект в нужную точку на поле, но при этом корректно обрабатывает все физические столкновения с `Rigidbody`. в данном случае точкой будет сумма текущей позиции и вектора движения, который я только что вычислял `rb.position + movement`.
```C#
float turnDirection = moveInput.x;
if (moveInput.y < 0) turnDirection = -turnDirection;
```
здесь происходит инверсия управления в случае если объект движется назад - при движении вперед клавиша `D` будет заставлять объект поворачиваться вправо, но при движении назад с клавишей `D` объект будет двигаться влево.
аналогично формуле с расчетом движения нужно так же рассчитать цифру поворота объекта при движениях вправо и влево:
```C#
float turn = turnDirection * rotationSpeed * Time.fixedDeltaTime;
```
нам нужно получить значение поворота в градусах - `turnDirection` определяется через `moveInput.x`, умножается на константу `rotationSpeed` и умножается на время. время здесь для того, чтобы выяснить скорость за один тик функции.
```C#
Quaternion turnRotation = Quaternion.Euler(0f, turn, 0f);
```
класс `Quaternion` позволяет хранить значение вращения и здесь я скажу только о том, что `Quaternion.Equler()` позволяет мне хранить значение поворота. подробнее в отдельном конспекте.
теперь нужно просто применить значение поворота:
```C#
rb.MoveRotation(rb.rotation + turnRotation);
```

