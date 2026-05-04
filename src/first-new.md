> # New

# Функция `new`

> To associate actual code with a type, we use `impl` blocks:

Чтобы связать написанный код с типом, используем блоки `impl`:

```rust ,ignore
impl List {
    // TODO, make code happen
    // TODO, воплотить код в жизнь
}
```

> Now we just need to figure out how to actually write code.
> In Rust we declare a function like so:

Сейчас нам осталось только разобраться, как правильно писать код.
В Rust мы объявляем функцию подобным образом:

```rust ,ignore
fn foo(arg1: Type1, arg2: Type2) -> ReturnType {
    // body
    // тело
}
```

> The first thing we want is a way to *construct* a list.
> Since we hide the implementation details, we need to provide that as a function.
> The usual way to do that in Rust is to provide a static method, which is just a normal function inside an `impl`:

Первая вещь, котая нам нужна: способ *конструирования* списка.
Поскольку мы спрятали детали реализации, нам нужно предоставить эту возможность, как функцию.
Обычный способ сделать это в Rust — написать статический метод, который по сути является обычной функцией внутри блока `impl`:

```rust ,ignore
impl List {
    pub fn new() -> Self {
        List { head: Link::Empty }
    }
}
```

> A few notes on this:

Несколько замечаний по поводу этого кода:

> * Self is an alias for "that type I wrote at the top next to `impl`".
>   Great for not repeating yourself!
> * We create an instance of a struct in much the same way we declare it, except instead of providing the types of its fields, we initialize them with values.
> * We refer to variants of an enum using `::`, which is the namespacing operator.
> * The last expression of a function is implicitly returned.
>   This makes simple functions a little neater.
>   You can still use `return` to return early like other C-like languages.

* `Self` — это псевдоним для «тот тип, который я написал в заголовке `impl`».
  Отлично подходит, чтобы не повторяться!
* Мы создаём экземпляр структуры практически также, как объявляем её, за исключением того, чтобы вместо указания типов её полей, мы инициализируем их значениями.
* Мы ссылаемся на варианты перечисления, используя `::`, то есть операр указания пространства имён.
* Последнее выражение функции считается её результатом.
  Такой подход делает простые функции немного лаконичнее.
  Но вы всё ещё можете использовать `return` для досрочного возврата из функции, как и в других языках, похожих на C.