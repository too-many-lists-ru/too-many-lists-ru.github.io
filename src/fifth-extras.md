# Всё остальное

Теперь, написав `push` и `pop`, мы можем взять остальные функции из старой реализации стека, каким бы странным это ни казалось.
Только в операциях, изменяющих длину списка, надо менять и указатель на хвост.

Конечно, поскольку у нас везде небезопасные указатели, придётся подкорректировать код!
И, раз уж мы будем его корректировать, почему бы не воспользоваться случаем и не убедиться, что мы ничего не упустили!

Что ж, начнём копи-пастить код из старой реализации:

```rust ,ignore
// ...

pub struct IntoIter<T>(List<T>);

pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}
```

IntoIter выглядит нормально, но `Iter` и `IterMut` нарушают наше правило никогда не использовать безопасные указатели в наших типах.
Перестрахуемся и перепишем их на сырые указатели:

```rust ,ignore
pub struct IntoIter<T>(List<T>);

pub struct Iter<'a, T> {
    next: *mut Node<T>,
}

pub struct IterMut<'a, T> {
    next: *mut Node<T>,
}

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }

    pub fn iter(&self) -> Iter<'_, T> {
        Iter { next: self.head }
    }

    pub fn iter_mut(&mut self) -> IterMut<'_, T> {
        IterMut { next: self.head }
    }
}
```

Выглядит хорошо!

```text
error[E0392]: parameter `'a` is never used
  --> src\fifth.rs:17:17
   |
17 | pub struct Iter<'a, T> {
   |                 ^^ unused parameter
   |
   = help: consider removing `'a`, referring to it in a field, 
     or using a marker such as `PhantomData`

error[E0392]: parameter `'a` is never used
  --> src\fifth.rs:21:20
   |
21 | pub struct IterMut<'a, T> {
   |                    ^^ unused parameter
   |
   = help: consider removing `'a`, referring to it in a field, 
     or using a marker such as `PhantomData`
```

Выглядит нехорошо!
О каком [PhantomData](https://doc.rust-lang.org/std/marker/struct.PhantomData.html) они пишут?

> Тип нулевого размера.
> Используется для маркировки объектов, которые «ведут себя так», словно владеют `T`.
> 
> Добавление поля `PhantomData<T>` к вашему типу говорит компилятору, что он действует так, будто хранит значение типа `T`, хотя на самом деле это не так.
> Эта информация используется при вычислении определённых свойств безопасности.
> 
> Для более глубокого объяснения, как использовать `PhantomData<T>`, читайте, пожалуйста, [Nomicon](https://doc.rust-lang.org/nightly/nomicon/).

Эй, потише, мы говорим о книге, которую написала *я*.
Не про ту другую книгу, которую, написал какой-то неимоверный *ботан*!
Держу пари, если он и напишет какую-нибудь структуру данных, это будет стек на основе массива, а *не* связный список.

> Неиспользуемые параметры времени жизни
> 
> Чаще всего PhantomData применяют для описания структур с неиспользуемым параметром времени жизни, обычно как часть небезопасного кода.

А, так мы объявили в нашем типе время жизни, но на самом деле его не используем.
Мы *могли бы* пойти по пути PhantomData, но я оставлю его для двусвязного списка из следующей главы, где такое решение *на самом деле* имеет смысл.

Сейчас мы в интересной ситуации, когда на самом деле нам не нужен тип PhantomData.
*Мне кажется*.
Я просто надеюсь, что это правда, а если miri будет ругаться, я уступлю и попробую прикрутить PhantomData.

Что мы на самом деле собираемся сделать: вернём ссылки в итераторы, поскольку они удобны для перебора элементов.
Кажется, что с этим не должно быть проблем, поскольку мы соблюдаем вложенность: создаём итератор, использует безопасные ссылки, удаляем итератор.

Только после завершения итерации можно вызывать у списка `push` и `pop`, которым нужен указатель на хвост и боксы.
А пока на время итерации мы *разыменуем* груду сырых указателей, поэтому здесь возможно перемешивание, но мы будем думать об этих ссылках, как о повторно заимствованных небезопасных указателях.

*Я* даже не уверена на 100%, но я хочу попробовать и проверить!

```rust ,ignore
pub struct IntoIter<T>(List<T>);

pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }

    pub fn iter(&self) -> Iter<'_, T> {
        unsafe {
            Iter { next: self.head.as_ref() }
        }
    }

    pub fn iter_mut(&mut self) -> IterMut<'_, T> {
        unsafe {
            IterMut { next: self.head.as_mut() }
        }
    }
}
```

Если мы будем хранить ссылки, нам надо преобразовать сырые указатели в Options для ссылок.
Мы *могли бы* проверять указатели на null, но это один из тех редких случаев, когда я *допускаю* использование неприятных методов [ptr::as_ref](https://doc.rust-lang.org/std/primitive.pointer.html#method.as_ref-1) и [ptr::as_mut](https://doc.rust-lang.org/std/primitive.pointer.html#method.as_mut).

*Обычно* я рекомендую избегать этих методов, как чумы, потому что они делают неожиданные и неприятные вещи и по сути повторно вводят ссылки, тогда как моё «простое правило» — не делать так!

Эти методы сопровождаются множеством предупреждений, но самое интересное вот это:

> Необходимо соблюдать правила Rust о псевдонимах, поскольку возвращаемое время жизни `'a` выбирается произвольно и не отражает реальное время жизни данных.
В частности, в течение этого времени жизни, доступ к этой памяти (чтение или запись) не должен осуществляться ни через один другой указатель

Смотрите, это то, о чём мы говорили на протяжении 25 страниц!
Я уже говорила, что использование ссылок здесь *определённо* допустимо, так что проблема псевдонимов решена!
Но есть и другая проблема — сигнатура:

```rust ,ignore
pub unsafe fn as_mut<'a>(self) -> Option<&'a mut T>
```

Вы видите, что время жизни вообще не привязано к входному значению, потому что `self` передаётся по значению?
Да, это то, что мы называем «неограниченным временем жизни» и это очень неприятная штука.
Это время претворяется настолько длинным, насколько нам нужно, даже «статическим»!

Способ *справиться* с ним — поместить его туда, где оно *будет* ограничено, что обычно означает «вернуть его из функции как можно быстрее, так что теперь его будет ограничивать сигнатура функции».

Боже, я нервничаю, но мы будем двигаться дальше!
Позаимствуем несколько реализаций итератора из стека:

```rust ,ignore
impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        self.0.pop()
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        unsafe {
            self.next.map(|node| {
                self.next = node.next.as_ref();
                &node.elem
            })
        }
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        unsafe {
            self.next.take().map(|node| {
                self.next = node.next.as_mut();
                &mut node.elem
            })
        }
    }
}
```

> Moment of truth time...

Время момента истины...

```text
cargo test

running 15 tests
test fifth::test::basics ... ok
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::iter ... ok
test third::test::basics ... ok

test result: ok. 15 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out;
```

```text
MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri test

running 15 tests
test fifth::test::basics ... ok
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 15 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

ДА!!!
Вот тебе, **ГОЛОС ЗА КАДРОМ**!
Иногда и я не ошибаюсь!

> **ГОЛОС ЗА КАДРОМ:** но разве вся суть не в том, что ошибки нужны, чтобы научить читателя?

ДА, НО ИНОГДА СУТЬ В ТОМ, ЧТО Я ПРАВА И ВСЕ ДОЛЖНЫ ПРИСЛУШИВАТЬСЯ КО МНЕ, КОГДА Я ГОВОРЮ О НЕБЕЗОПАСНОМ КОДЕ, ПОТОМУ ЧТО Я ПРОВЕЛА МНОГО ВРЕМЕНИ В РАЗМЫШЛЕНИЯХ О НАДЁЖНОСТИ РЕАЛИЗАЦИИ ИТЕРАТОРОВ!
ЛАДЫ?!
ЛАДЫ.

Наконец, вот `peek` и `peek_mut`.

```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    unsafe {
        self.head.as_ref()
    }
}

pub fn peek_mut(&mut self) -> Option<&mut T> {
    unsafe {
        self.head.as_mut()
    }
}
```

Я даже не буду их тестировать, потому что я больше не ошибаюсь.

> **ГОЛОС ЗА КАДРОМ**: `cargo build`


```text
error[E0308]: mismatched types
  --> src\fifth.rs:66:13
   |
25 | impl<T> List<T> {
   |      - this type parameter
...
64 |     pub fn peek(&self) -> Option<&T> {
   |                           ---------- expected `Option<&T>` 
   |                                      because of return type
65 |         unsafe {
66 |             self.head.as_ref()
   |             ^^^^^^^^^^^^^^^^^^ expected type parameter `T`, 
   |                                found struct `fifth::Node`
   |
   = note: expected enum `Option<&T>`
              found enum `Option<&fifth::Node<T>>`

```

ПРЕКРАСНО.

```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    unsafe {
        self.head.as_ref().map(|node| &node.elem)
    }
}

pub fn peek_mut(&mut self) -> Option<&mut T> {
    unsafe {
        self.head.as_mut().map(|node| &mut node.elem)
    }
}
```

> I guess I am going to *continue* to make mistakes, so we're going to be extra careful and add a new test I'm going to call "miri food": something that just messes around and mixes up our APIs a bunch to help miri catch our mistakes.

Полагаю, я всё-таки *буду* ошибаться время от времени, так что мы добавим новый тест, который я назову «приманка для miri»: что-то, что вызывает наши методы в произвольном порядке и помогает miri искать наши ошибки.

```rust ,ignore
#[test]
fn miri_food() {
    let mut list = List::new();

    list.push(1);
    list.push(2);
    list.push(3);

    assert!(list.pop() == Some(1));
    list.push(4);
    assert!(list.pop() == Some(2));
    list.push(5);

    assert!(list.peek() == Some(&3));
    list.push(6);
    list.peek_mut().map(|x| *x *= 10);
    assert!(list.peek() == Some(&30));
    assert!(list.pop() == Some(30));

    for elem in list.iter_mut() {
        *elem *= 100;
    }

    let mut iter = list.iter();
    assert_eq!(iter.next(), Some(&400));
    assert_eq!(iter.next(), Some(&500));
    assert_eq!(iter.next(), Some(&600));
    assert_eq!(iter.next(), None);
    assert_eq!(iter.next(), None);

    assert!(list.pop() == Some(400));
    list.peek_mut().map(|x| *x *= 10);
    assert!(list.peek() == Some(&5000));
    list.push(7);

    // Ну а здесь пусть запуститься деструктор
}
```


```text
cargo test

running 16 tests
test fifth::test::basics ... ok
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test fifth::test::miri_food ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test fourth::test::peek ... ok
test second::test::into_iter ... ok
test second::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::iter ... ok
test second::test::iter ... ok
test third::test::basics ... ok

test result: ok. 16 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out



MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri test

running 16 tests
test fifth::test::basics ... ok
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test fifth::test::miri_food ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test fourth::test::peek ... ok
test second::test::into_iter ... ok
test second::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::iter ... ok
test second::test::iter ... ok
test third::test::basics ... ok

test result: ok. 16 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

Идеально.
