> # Pop

# Извлечение

> Like `push`, `pop` wants to mutate the list.
> Unlike `push`, we actually want to return something.
> But `pop` also has to deal with a tricky corner case: what if the list is empty?
> To represent this case, we use the trusty `Option` type:

Как и `push`, функции `pop` требуется изменить список.
В отличие от `push`, нам на самом деле надо вернуть какое-то значение.
Но метод `pop` также должен учитывать каверзный граничный случай: что если список пуст?
Чтобы учесть этот случай, воспользуемся проверенным типом `Option`:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    // TODO
    // дописать
}
```

> `Option<T>` is an enum that represents a value that may exist.
> It can either be `Some(T)` or `None`.
> We could make our own enum for this like we did for Link, but we want our users to be able to understand what the heck our return type is, and Option is so ubiquitous that *everyone* knows it.
> In fact, it's so fundamental that it's implicitly imported into scope in every file, as well as its variants `Some` and `None` (so we don't have to say `Option::None`).

`Option<T>` — это перечисление, которое представляет значение, которое может существовать.
Оно может быть либо `Some(T)`, либо `None`.
Мы могли бы завести для этого собственный тип, как мы сделали с Link, но мы хотим, чтобы наши пользователи понимали, что это из себя представляет хренов возвращаемый тип, а Option настолько вездесущий, что про него все знают. <!-- в оригинале эвфемизм heck = hell -->
Фактически, он настолько фундаментальный, что неявно импортируется в область видимости каждого файла вместе с вариантами `Some` и `None` (так что мы не должны писать `Option::None`).

> The pointy bits on `Option<T>` indicate that Option is actually *generic* over T.
> That means that you can make an Option for *any* type!

Угловые скобки в `Option<T>` указывают на то, что на самом деле Option — *обобщённый* тип над T.
Это значит, что вы можете сделать опциональным значение *любого* типа!

> So uh, we have this `Link` thing, how do we figure out if it's Empty or has More?
> Pattern matching with `match`!

Что же, у нас есть `Link`, как нам узнать, содержит ли она `Empty` или `More`?
Конечно с помощью сопоставления с образцом и оператора `match`!

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match self.head {
        Link::Empty => {
            // TODO
            // дописать
        }
        Link::More(node) => {
            // TODO
            // дописать
        }
    };
}
```

```text
> cargo build

error[E0308]: mismatched types
  --> src/first.rs:27:30
   |
27 |     pub fn pop(&mut self) -> Option<i32> {
   |            ---               ^^^^^^^^^^^ expected enum `std::option::Option`, found ()
   |            |
   |            this function's body doesn't return
   |
   = note: expected type `std::option::Option<i32>`
              found type `()`
```

> Whoops, `pop` has to return a value, and we're not doing that yet.
> We *could* return `None`, but in this case it's probably a better idea to return `unimplemented!()`, to indicate that we aren't done implementing the function.
> `unimplemented!()` is a macro (`!` indicates a macro) that panics the program when we get to it (\~crashes it in a controlled manner).

Ой, `pop` должна возвращать значение, но мы пока этого не делаем.
Мы *могли бы* вернуть `None`, но возможно в данном случае лучше вернуть `unimplemented!()`, чтобы показать, что мы ещё не завершили реализацию функции.
`unimplemented!()` — это макрос (`!` указывает на макрос), который вызывает панику, когда программа пытается получить это значение (приводит к контролируемому сбою).

<!-- Вроде как намёк на игру Rust, которая хранит в дампы памяти в папке \~crashes. -->

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match self.head {
        Link::Empty => {
            // TODO
            // дописать
        }
        Link::More(node) => {
            // TODO
            // дописать
        }
    };
    unimplemented!()
}
```

> Unconditional panics are an example of a [diverging function][diverging].
> Diverging functions never return to the caller, so they may be used in places where a value of any type is expected.
> Here, `unimplemented!()` is being used in place of a value of type `Option<T>`.

Безусловная паника — это пример [расходящейся функции][diverging].
<!-- Термин из сходящихся и расходящихся рядов. Значение сходящегося ряда можно посчитать, в то время, как функция, вычисляющее значение расходящегося ряда, выполняется бесконечно.

Функция, выполняющая бесконечные вычисления, называется расходящейся. -->
Расходящиеся функции никогда не возвращают управления вызывающей функции, поэтому их можно использовать в любом месте, где ожидается значение любого типа.
Здесь `unimplemented!()` используется вместо значения типа `Option<T>`.

> Note also that we don't need to write `return` in our program.
> The last expression (basically line) in a function is implicitly its return value.
> This lets us express really simple things a bit more concisely.
> You can always explicitly return early with `return` like any other C-like language.

Также обратие внимание, что мы не обязаны писать `return` в нашей программе.
Последнее выражение (обычно, последняя строка) функции считается её возвращаемым значением.
Это позволяет нам выражать действительно простые вещи намного лаконичнее.
Вы всегда можете сделать досрочный возврат с помощью оператора `return`, как и в любом другом C-подобном языке.

```text
> cargo build

error[E0507]: cannot move out of borrowed content
  --> src/first.rs:28:15
   |
28 |         match self.head {
   |               ^^^^^^^^^
   |               |
   |               cannot move out of borrowed content
   |               help: consider borrowing here: `&self.head`
...
32 |             Link::More(node) => {
   |                        ---- data moved here
   |
note: move occurs because `node` has type `std::boxed::Box<first::Node>`, which does not implement the `Copy` trait
  --> src/first.rs:32:24
   |
32 |             Link::More(node) => {
   |                        ^^^^
```

> Come on Rust, get off our back!
> As always, Rust is hella mad at us.
> Thankfully, this time it's also giving us the full scoop!
> By default, a pattern match will try to move its contents into the new branch, but we can't do this because we don't own self by-value here.

Да ладно, Rust, прекрати!
Он, как всегда, чертовски сердит.
К счастью, сейчас он предоставляет нам полную информацию!
По умолчанию, при совпадении с образцом происходит перемещение значения в совпавшую ветку, но мы не можем этого сделать потому что здесь мы не владеем значением.

```text
help: consider borrowing here: `&self.head`
```

> Rust says we should add a reference to our `match` to fix that. 🤷‍♀️
> Let's try it:

Rust утверждает, что мы должны добавить ссылку в оператор `match`, чтобы исправить ошибку. 🤷‍♀️
Попробуем:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match &self.head {
        Link::Empty => {
            // TODO
            // дописать
        }
        Link::More(node) => {
            // TODO
            // дописать
        }
    };
    unimplemented!()
}
```

```text
> cargo build

warning: unused variable: `node`
  --> src/first.rs:32:24
   |
32 |             Link::More(node) => {
   |                        ^^^^ help: consider prefixing with an underscore: `_node`
   |
   = note: #[warn(unused_variables)] on by default

warning: field is never used: `elem`
  --> src/first.rs:13:5
   |
13 |     elem: i32,
   |     ^^^^^^^^^
   |
   = note: #[warn(dead_code)] on by default

warning: field is never used: `next`
  --> src/first.rs:14:5
   |
14 |     next: Link,
   |     ^^^^^^^^^^
```

> Hooray, compiling again!
> Now let's figure out that logic.
> We want to make an Option, so let's make a variable for that.
> In the Empty case we need to return None.
> In the More case we need to return `Some(i32)`, and change the head of the list.
> So, let's try to do basically that?

Ура, снова компилируется!
Давайте теперь разберёмся, как это работает.
Мы хотим создать Option, так что давайте создадим для этого переменную.
В случае пустого списка мы должны вернуть `None`.
В случае непустого списка, мы должны вернуть `Some(i32)` и заменить голову списка.
Что ж, давайте попробуем это сделать?

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    let result;
    match &self.head {
        Link::Empty => {
            result = None;
        }
        Link::More(node) => {
            result = Some(node.elem);
            self.head = node.next;
        }
    };
    result
}
```

```text
> cargo build
   Compiling lists v0.1.0 (/Users/ADesires/dev/temp/lists)
error[E0507]: cannot move out of borrowed content
  --> src/first.rs:35:29
   |
35 |                 self.head = node.next;
   |                             ^^^^^^^^^ cannot move out of borrowed content

```

> *head*

> *desk*

*голова*

*стол*

<!-- Обыгрывается голова списка, бьюсь головой о стол. -->

> We're trying to move out of `node` when all we have is a shared reference to it.

Мы пытаемся забрать владение `node` в то время как у нас есть только разделяемая ссылка на него.

> We should probably step back and think about what we're trying to do.
> We want to:

Возможно, нам следует вернуться на шаг назад и поудмать, что мы пытаемся сделать.
Мы хотим:

> * Check if the list is empty.
> * If it's empty, just return None
> * If it's *not* empty
>     * remove the head of the list
>     * remove its `elem`
>     * replace the list's head with its `next`
>     * return `Some(elem)`

* Проверить, не пустой ли список.
* Если пустой, просто вернуть `None`.
* Если *не* пустой:
    * удалить голову списка
    * удалить `elem`
    * заменить голову списка следующим узлом
    * вернуть `Some(elem)`

> The key insight is we want to *remove* things, which means we want to get the head of the list *by value*.
> We certainly can't do that through the shared reference we get through `&self.head`.
> We also "only" have a mutable reference to `self`, so the only way we can move stuff is to *replace it*.
> Looks like we're doing the Empty dance again!

Ключевое озарение заключается в том, что мы хотим что-то *удалить*, а это значит, что доступ к голове списка нам нужен *по значению*.
Мы определённо не можем этого сделать через разделяемую ссылку, которую получаем с помощью `&self.head`.
Также, у нас есть *только* изменяемая ссылка на `self`, так что единственный способ переместить что-либо — это *замена*.
Выглядти так, словно нам снова предстоит отплясывать вокруг Empty!

> Let's try that:

Попробуем:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    let result;
    match mem::replace(&mut self.head, Link::Empty) {
        Link::Empty => {
            result = None;
        }
        Link::More(node) => {
            result = Some(node.elem);
            self.head = node.next;
        }
    };
    result
}
```

```text
cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

> O M G

Боже мой!

> It compiled without *any* warnings!!!!!

Всё скомпилировалось *без единого* предупреждения!!!!!

> Actually I'm going to apply my own personal lint here: we made this `result` value to return, but actually we didn't need to do that at all!
> Just as a function evaluates to its last expression, every block also evaluates to its last expression.
> Normally we supress this behaviour with semi-colons, which instead makes the block evaluate to the empty tuple, `()`.
> This is actually the value that functions which don't declare a return value -- like `push` -- return.

На самом деле я собираюсь применить здесь собственную подсказку: мы сделали `result` возвращаемым значением, но на самом деле оно нам вообще не нужно!
Точно также, как и функция имеет значение свого последнего выражения, каждый блок также имеет значение своего последнего выражения.
Обычно мы ставим в конце операторов точку с запятой, а это значит, что блок возвращает пустой кортеж, `()`.
Фактически, это то самое значение, которое возвращают функции без возвращаемого значения, такие, как `push`.

> So instead, we can write `pop` as:

Так что мы можем переписать `pop` так:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match mem::replace(&mut self.head, Link::Empty) {
        Link::Empty => None,
        Link::More(node) => {
            self.head = node.next;
            Some(node.elem)
        }
    }
}
```

> Which is a bit more concise and idiomatic.
> Note that the Link::Empty branch completely lost its braces, because we only have one expression to evaluate.
> Just a nice shorthand for simple cases.

Что чуть лаконичнее и идиоматичнее.
Обратите внимание, что ветка `Link::Empty` полностью избавилась от фигурных скобок, потому что состоит теперь только из одного выражения.
Просто удобная короткая запись для простых случаев.

```text
cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

> Nice, still works!

Прекрасно, всё ещё работает!

[ownership]: first-ownership.html
[diverging]: https://doc.rust-lang.org/nightly/book/ch19-04-advanced-types.html#the-never-type-that-never-returns
