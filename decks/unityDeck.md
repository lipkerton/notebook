TARGET DECK: unityDeck

Какой класс выступает связью между объектом на сцене и кодом, который для него написан? #flashcard 
`MonoBehaviour`. От него наследуется собственный класс, где пишется логика объекта.

Какой класс я должен использовать, чтобы считывать клавиши с клавиатуры? #flashcard 
`InputAction`:
```C#
private InputAction moveAction;
private void Awake() {
	moveAction = new InputAction(
		name: "Move",
		type: InputActionType.Value
	)
}
```

Что такое *композит*? #flashcard 
Композит - это сущность, которая участвует в системе считывания клавиш с клавиатуры, где связывает несколько клавиш в группу, чтобы результатом нажатия каждой клавиши, или нескольких, или всех сразу стал единый вектор.

Как я могу с помощью *композита* объединить несколько клавиш управления, чтобы они возвращали единый двумерный вектор? #flashcard 
Использовать метод `InputAction` - `.AddCompositeBinding()`:
```C#
InputAction moveAction = new InputAction(
	name: "move",
	type: InputActionType.Value
)
moveAction.addCompositeBinding("2DVector")
	.With("Up", "<Keyboard>\w")
	.With("Down", "<Keyboard>\s")
	.With("Right", "<Keyboard>\d")
	.With("Left", "<Keyboard>\a");
```