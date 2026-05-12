# Метод `pop`

Как и `push`, метод `pop` меняет список.
Но, в отличие от `push`, метод `pop` возвращает какое-то значение.
При этом он должен учитывать каверзный граничный случай: что, если список пуст?
Чтобы представить этот случай, воспользуемся проверенным типом `Option`:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    // дописать
}
```

`Option<T>` — это перечисление, которое представляет *опциональное* значение, которое, возможно, существует.
Вариант `Some(T)` соответствует существующему значению, а `None` — отсутствию значения.
Как и в случае с Link, можно было бы придумать собственный тип, но мы хотим, чтобы пользователи понимали, что именно представляет из себя возвращаемое значение, а Option настолько вездесущ, что все про него все знают.
Фактически, он настолько фундаментален, что неявно импортируется в область видимости каждого модуля вместе с вариантами `Some` и `None` (поэтому мы и не пишем `Option::None`).

Угловые скобки в `Option<T>` означают, что Option, на самом деле — *обобщённый* тип над T.
То есть, опциональным можно сделать значение *любого* типа!

Хорошо, у нас есть ссылка `Link`, как нам узнать, содержит ли она `Empty` или `More`?
Конечно же с помощью сопоставления с образцом и оператора `match`!

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match self.head {
        Link::Empty => {
            // дописать
        }
        Link::More(node) => {
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

Ой, `pop` должна вернуть значение, но этого мы пока не написали.
Мы *могли бы* вернуть `None`, но, возможно в данном случае лучше вернуть `unimplemented!()`, чтобы показать, что мы ещё не завершили реализацию функции.
`unimplemented!()` — это макрос (`!` указывает на макрос), который вызывает панику, когда к нему обращаются.
Происходит так называемый контролируемый сбой.

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match self.head {
        Link::Empty => {
            // дописать
        }
        Link::More(node) => {
            // дописать
        }
    };
    unimplemented!()
}
```

Безусловная паника — это пример [расходящейся функции][diverging].
Расходящиеся функции не возвращают управления вызывающей функции, поэтому их можно использовать в любом месте, где ожидается значение любого типа.
В нашем случае `unimplemented!()` используется вместо значения типа `Option<T>`.

Обратите внимание, что в методе `pop` мы не используем оператор `return`.
Значением функции становится её последнее выражение (обычно, последняя строка).
Благодаря этому, действительно простые функции можно сделать намного короче.
Однако, вы всегда можете завершить функцию досрочно с помощью оператора `return`, как и в любом другом C-подобном языке.

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

Да ладно, Rust, перестань!
Компилятор, как всегда, на что-то ругается.
К счастью, сейчас он предоставил нам всю информацию!
По умолчанию, при совпадении с образцом происходит перемещение значения в совпавшую ветку, но, поскольку мы не владеем значением, так делать нельзя.

```text
help: consider borrowing here: `&self.head`
```

Rust утверждает, что мы должны добавить ссылку в оператор `match`, чтобы исправить ошибку. 🤷‍♀️
Попробуем:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match &self.head {
        Link::Empty => {
            // дописать
        }
        Link::More(node) => {
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

Ура, снова компилируется!
Теперь давайте обсудим логику работы нашего метода.
Мы хотим создать Option, так что пусть у нас будет переменная для опционального значения.
В случае пустого списка мы присвоим ей `None`.
В случае непустого списка, мы присвоим ей `Some(i32)` и заменим голову списка.
Что ж, попробуем?

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

*голова*

*стол*

Мы пытаемся забрать владение `node` в то время как узел доступен нам только по разделяемой ссылке.

Возможно, нам следует вернуться на шаг назад и подумать, что мы пытаемся сделать.
Мы хотим:

* Проверить, не пустой ли список.
* Если пустой, просто вернуть `None`.
* Если *не* пустой:
    * удалить голову списка
    * удалить `elem`
    * заменить голову списка следующим узлом
    * вернуть `Some(elem)`

Ключевая мысль заключается в том, что мы хотим что-то *удалить*, а это значит, что доступ к голове списка нам нужен *по значению*.
Мы определённо не можем ничего менять через разделяемую ссылку, которую получаем с помощью `&self.head`.
Кроме того, имея на руках *только* изменяемую ссылку на `self`, мы можем переместить что-либо единственным способом — через *замену*.
Выглядит так, словно нам опять предстоит танец вокруг Empty!

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

Боже мой!

Всё скомпилировалось *без единого* предупреждения!!!!!

Я собираюсь выступить в роли линтера для нашего кода.
Мы сделали `result` возвращаемым значением, но на самом деле эта переменная нам вообще не нужна!
Также, как и функция получает значение своего последнего выражения, каждый блок получает значение своего последнего выражения.
Обычно мы ставим в конце операторов точку с запятой, а это значит, что блок возвращает пустой кортеж, `()`.
На самом деле, пустой кортеж — то самое значение, которое возвращают функции без возвращаемого значения, такие, как `push`.

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

Так чуть лаконичнее и идиоматичнее.
Обратите внимание, что ветка `Link::Empty` полностью избавилась от фигурных скобок, потому что состоит теперь только из одного выражения.
Удобная короткая запись для простых случаев.

```text
cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

Прекрасно, всё ещё работает!

[ownership]: first-ownership.html
[diverging]: https://doc.rust-lang.org/nightly/book/ch19-04-advanced-types.html#the-never-type-that-never-returns
