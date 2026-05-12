# Iter

Теперь попробуем реализовать Iter.
На этот раз мы не можем рассчитывать, что Rust предоставит нам необходимые функции.
Их придётся написать самим.
Основная логика, которую мы хотим сделать — хранить указатель на текущий узел, который мы вёрнём при следующем вызове `next`.
Nакого узла может и не быть (список пуст или мы завершили итерирование), поэтому ссылка будет опциональной.
Возвращая элементы, мы должны передвинуться к следующему узлу, то есть ссылке `next` текущего узла.

Хорошо, давайте попробуем:

```rust ,ignore
pub struct Iter<T> {
    next: Option<&Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

impl<T> Iterator for Iter<T> {
    type Item = &T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

```text
> cargo build

error[E0106]: missing lifetime specifier
  --> src/second.rs:72:18
   |
72 |     next: Option<&Node<T>>,
   |                  ^ expected lifetime parameter

error[E0106]: missing lifetime specifier
  --> src/second.rs:82:17
   |
82 |     type Item = &T;
   |                 ^ expected lifetime parameter
```

О, боже.
Время жизни.
Я слышал про эту штуку.
Я слышал, что это кошмар.

Давайте попробуем что-то новое: видите ошибку `error[E0106]`?
Это код ошибки компилятора.
Мы можем попросить rustc объяснить её с помощью `--explain`:

```text
> rustc --explain E0106
This error indicates that a lifetime is missing from a type. If it is an error
inside a function signature, the problem may be with failing to adhere to the
lifetime elision rules (see below).

Here are some simple examples of where you'll run into this error:

struct Foo { x: &bool }        // error
struct Foo<'a> { x: &'a bool } // correct

enum Bar { A(u8), B(&bool), }        // error
enum Bar<'a> { A(u8), B(&'a bool), } // correct

type MyStr = &str;        // error
type MyStr<'a> = &'a str; //correct
...

```

Так, хм... Это не очень прояснило ситуацию (дока думает, что мы разбираемся в Rust намного лучше, чем есть на самом деле).
Но разве всё не звучит так, что мы должны добавить эти `'a` к нашей структуре?
Давайте попробуем.

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}
```

```text
> cargo build

error[E0106]: missing lifetime specifier
  --> src/second.rs:83:22
   |
83 | impl<T> Iterator for Iter<T> {
   |                      ^^^^^^^ expected lifetime parameter

error[E0106]: missing lifetime specifier
  --> src/second.rs:84:17
   |
84 |     type Item = &T;
   |                 ^ expected lifetime parameter

error: aborting due to 2 previous errors
```

Так, я начинаю видеть закономерность... давайте просто добавим этих малышей везде, где сможем:

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<'a, T> List<T> {
    pub fn iter(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &'a node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;
    fn next(&'a mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &'a node);
            &'a node.elem
        })
    }
}
```

```text
> cargo build

error: expected `:`, found `node`
  --> src/second.rs:77:47
   |
77 |         Iter { next: self.head.map(|node| &'a node) }
   |         ---- while parsing this struct        ^^^^ expected `:`

error: expected `:`, found `node`
  --> src/second.rs:85:50
   |
85 |             self.next = node.next.map(|node| &'a node);
   |                                                  ^^^^ expected `:`

error[E0063]: missing field `next` in initializer of `second::Iter<'_, _>`
  --> src/second.rs:77:9
   |
77 |         Iter { next: self.head.map(|node| &'a node) }
   |         ^^^^ missing `next`
```

Боже.
Мы сломали Rust.

Может быть, нам всё-таки пора разобраться, что за хрень это самое время жизни `'a`?

Время жизни может отпугнуть многих, потому что оно меняет кое-что, что мы знали и любили с начала программистских времён.
Пока нам удавалось избегать упоминания времени жизни, не смотря на то, что всё это время оно было тесно вплетено в наши программы.

В языках программирования со сборкой мусора время жизни не требуется, поскольку сборщик мусора гарантирует, что все объекты волшебным образом проживут столько, сколько нужно.
Большинство данных в Rust *управляется* вручную, так что для них нужно другое решение.
C и C++ дают нам ясный пример, что происходит, если вы даёте людям возможность просто указывать на произвольные данные в стеке: повсеместная неуправляемая небезопасность.
Возникающие ошибки можно условно разделить на два класса:

* Хранить указатель на что-то, чего больше не существует
* Хранить указатель на что-то, что было кем-то изменено

Время жизни решает обе эти проблемы и в 99% случаев делает это совершенно прозрачно.

Так что ж такое время жизни?

Говоря простыми словами, время жизни — это название участка кода (блока/области видимости) где-то в программе.
Вот и всё.
Когда ссылка помечена временем жизни, мы говорим, что она должна быть действительна на *всём* участке.
То, как долго ссылка должна быть и может быть действительна, зависит от множества вещей.
Вся система управления временем жизни представляет собой систему решения ограничений, которая пытается минимизировать участок, где каждая ссылка действительна.
Если ей удаётся найти множество времён жизни, которые удовлетворяют всем ограничениям, ваша программа компилируется!
В противном случае вы получаете ошибку, которая утверждает, что какая-то переменная живёт недостаточно долго.

Нет смысла говорить о времени жизни внутри функции, потому что у компилятора есть вся информация, чтобы вывести ограничения для определения минимального времени жизни.
Однако, на уровне типов и API, у компилятора *нет* всей информации.
Он требует, чтобы вы рассказали ему о взаимоотношениях между различными временами жизни, чтобы он мог понять, что вы делаете.

В приципе, времена жизни *можно было бы* опустить, но тогда проверка заимствований привела бы к чудовищному анализу всей программы и сложным нелокальным ошибкам. (Речь о том, что ошибка в функции A приводила бы к ошибке в вызывающей её функции B и вам предстояла бы большая работа, чтобы найти истинную причину — прим. пер.)
С другой стороны, если вы явно указываете время жизни, Rust может независимо проверить каждую функцию, и все ваши ошибки должны быть довольно локальными (или у ваших типов некорректные сигнатуры).

Впрочем, мы и раньше писали ссылки в сигнатурах функций, и всё было нормально!
Оказыватеся, всё работало потому, что есть несколько распространенных случаев, когда Rust может вывести время жизни за нас.
Речь идёт о *неявном выведении времени жизни* (lifetime elision).

В частности:

```rust ,ignore
// Только одна ссылка на входе, так что выход можно вывести из входа
fn foo(&A) -> &B; // сахар для:
fn foo<'a>(&'a A) -> &'a B;

// Много ссылок на входе, предполагаем, что все они независимы
fn foo(&A, &B, &C); сахар для:
fn foo<'a, 'b, 'c>(&'a A, &'b B, &'c C);

// Методы, выводим все выходные времена жизни из `self`.
fn foo(&self, &B, &C) -> &D; // sugar for: сахар для:
fn foo<'a, 'b, 'c>(&'a self, &'b B, &'c C) -> &'a D;
```

Итак, что *значит* `fn foo<'a>(&'a A) -> &'a B`?
Всего лишь то, что входная переменная должна жить по крайней мере столько же, сколько и выходная.
Следовательно, если мы храним выходные переменные в течение длительного времени, участок, в котором должны быть действительны и входные переменные, становится больше.
После того, как вы прекращаете использовать выходную переменную, компилятор понимает, что входная переменная тоже стала недействительной.

Благодаря этой системе правил Rust гарантирует, что переменную нельзя использовать после освобождения, и нельзя изменить, пока на неё есть внешние ссылки.
По сути, Rust всего лишь проверяет, что наложенные ограничения не конфликтуют друг с другом!

Хорошо.
Так.
Iter.

Откатимся к коду без времени жизни:

```rust ,ignore
pub struct Iter<T> {
    next: Option<&Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

impl<T> Iterator for Iter<T> {
    type Item = &T;
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

Мы должны добавить времена жизни только в сигнатуры функций и типов:

```rust ,ignore
// Iter это обобщённый тип с каким-то временем, остальное ему не важно
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

// Никакого врмени жизни: у List нет никаких связанных времён жизни
impl<T> List<T> {
    // Здесь мы объявляем новое время жизни для заимствования, чтобы
    // создать итератор. Теперь &self должен оставаться действительным до тех
    // пор, пока существует Iter.
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

// Здесь *нужно* время жизни, потому что оно есть у Iter
impl<'a, T> Iterator for Iter<'a, T> {
    // Здесь тоже нужно, поскольку это декларация типа
    type Item = &'a T;

    // Здесь ничего менять не нужно, всё будет работать, как и раньше
    // Self всё ещё невероятно чудесен и хайпов
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

Ладно, надеюсь, на этот раз у нас получилось.

```text
cargo build

error[E0308]: mismatched types
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.map(|node| &node) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

error[E0308]: mismatched types
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.map(|node| &node);
   |                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&'a second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`
```

(╯°□°)╯︵ ┻━┻

Так.
Хорошо.
Мы исправили ошибки с временем жизни, но у нас появился новый тип ошибок.

Нам нужны `&Node`, но у нас есть только `&Box<Node>`.
Ладно, это достаточно просто, мы всего лишь должны разыменовать бокс, чтобы получить нашу ссылку:

```rust ,ignore
impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &*node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &*node);
            &node.elem
        })
    }
}
```

```text
cargo build
   Compiling lists v0.1.0 (/Users/ADesires/dev/temp/lists)
error[E0515]: cannot return reference to local data `*node`
  --> src/second.rs:77:43
   |
77 |         Iter { next: self.head.map(|node| &*node) }
   |                                           ^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.map(|node| &*node) }
   |                      ^^^^^^^^^ cannot move out of borrowed content

error[E0515]: cannot return reference to local data `*node`
  --> src/second.rs:85:46
   |
85 |             self.next = node.next.map(|node| &*node);
   |                                              ^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.map(|node| &*node);
   |                         ^^^^^^^^^ cannot move out of borrowed content
```

(ﾉಥ益ಥ）ﾉ﻿ ┻━┻

Мы забыли вызвать `as_ref`, так что мы передаём бокс в `map`, что означает, что он будет удалён, что означает, что наши ссылки станут висячими (перестанут ссылаться на реальный объект).

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.as_ref().map(|node| &*node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_ref().map(|node| &*node);
            &node.elem
        })
    }
}
```

```text
cargo build
   Compiling lists v0.1.0 (/Users/ADesires/dev/temp/lists)
error[E0308]: mismatched types
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.as_ref().map(|node| &*node) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

error[E0308]: mismatched types
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.as_ref().map(|node| &*node);
   |                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&'a second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

```

😭

`as_ref` добавил ещё один уровень косвенности и его надо убрать:


```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.as_deref() }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_deref();
            &node.elem
        })
    }
}
```

```bash
cargo build
```

🎉 🎉 🎉

Функции as_deref и as_deref_mut стабильны, начиная с Rust 1.40.
До этого вам надо было писать `map(|node| &**node)` и `map(|node| &mut**node)`.
Вам может показаться, что эта штука с `&**` на самом деле корявая, и в этом вы не ошибаетесь.
Но, как хорошее вино, Rust со временем становится только лучше и нам больше не нужно так писать.
Обычно Rust справляется с такими преобразованиями неявно, посредством процесса, называемого *автоматическим разыменованием* (deref coercion), во время которого он может вставлять \* по всему коду, чтобы код прошёл проверку типов.
Такой подход работает благодаря проверке заимствований, которая гарантирует, что мы никогда не сломаем указатели!

Но в нашем случае замыкание в сочетании с тем фактом, что нам нужен `Option<&T>` вместо `&T`, оказалось слишком сложным.
В результате пришлось явно указывать время жизни.
К счастью, по моему опыту, такое случается довольно редко.

Для полноты картины: мы *могли бы* дать *другую* подсказку с помощью оператора `::<>` который на сленге называют *турборыбой* (turbofish):

```rust ,ignore
    self.next = node.next.as_ref().map::<&Node<T>, _>(|node| &node);
```

Смотрите, `map` — это обобщённая функция:

```rust ,ignore
pub fn map<U, F>(self, f: F) -> Option<U>
```

Турборыба `::<>` позволяет нам сообщить компилятору, какими, как нам кажется, должны быть обобщённые типы.
В данном случае `::<&Node<T>, _>` говорит «функция должна вернуть `&Node<T>`, а второй тип меня не волнует».

Это, в свою очередь сообщает компилятору, что к `&node` следует применить автоматическое разыменование, так что нам не придётся писать все эти \* вручную!

Но в данном случае я не думаю, что так следует писать.
Это был всего лишь плохо завуалированный предлог, чтобы рассказать про автоматическое разыменование и оператор `::<>`, весьма полезный время от времени. 😅

> Let's write a test to be sure we didn't no-op it or anything:

Давайте напишем тест, чтобы убедиться, что мы всё сделали правильно и не оставили без внимания ничего важного:

```rust ,ignore
#[test]
fn iter() {
    let mut list = List::new();
    list.push(1); list.push(2); list.push(3);

    let mut iter = list.iter();
    assert_eq!(iter.next(), Some(&3));
    assert_eq!(iter.next(), Some(&2));
    assert_eq!(iter.next(), Some(&1));
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 5 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::peek ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured

```

Блин, да.

> Finally, it should be noted that we *can* actually apply lifetime elision here:

Наконец, следует заметить, что мы действительно *можем* полагаться на неявное выведение времени жизни:

```rust ,ignore
impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.as_deref() }
    }
}
```

эквивалентно:

```rust ,ignore
impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.as_deref() }
    }
}
```

Ура, меньше времён жизни!

> Or, if you're not comfortable "hiding" that a struct contains a lifetime, you can use the Rust 2018 "explicitly elided lifetime" syntax,  `'_`:

Или, если вам некомфортно «скрывать», что у структуры есть время жизни, мы можете использовать синтаксис «явно пропущенного времени жизни» `'_`, который появился в Rust 2018.

```rust ,ignore
impl<T> List<T> {
    pub fn iter(&self) -> Iter<'_, T> {
        Iter { next: self.head.as_deref() }
    }
}
```
