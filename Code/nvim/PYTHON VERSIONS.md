Это был самый жопоразрывающий EXPERIENCE: вим не может увидеть мою версию питона на компьютере. Нужно поставить ему настройку в файл `init.vim` :

```Bash
let g:python3_host_prog = 'path/to/python3'
let g:python2_host_prog = 'path/to/python2'
```

Проблема одна - я использую не `init.vim` , а `init.lua` и при этом я не шарю как написать на языке lua то, что было написано на языке vim. То есть вот эти команды сверху нужно переписать на lua. Как это сделать? В душе не ебу. Но спустя два часа поиска появился спаситель ([https://stackoverflow.com/a/70207074](https://stackoverflow.com/a/70207074)).

```Lua
vim.g.python3_host_prog = 'path/to/python3'
vim.g.python2_host_prog = 'path/to/python2'
```

Чтобы neovim зарегистрировал версию языка в нее необходимо установить библиотеку `pynvim` . Это необходимо, т.к., чтобы зарегистрировать версию neovim будет пытаться сделать импорт `nvim`.

```Bash
python3.13 -m pip install pynvim
```