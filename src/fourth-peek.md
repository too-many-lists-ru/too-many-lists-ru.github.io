# Метод `peek`

Хорошо, мы справились с методами `push` и `pop`.
Не буду врать, я немного волнуюсь.
Проверка правильности на этапе компиляции — охрененное чувство.

Надо немного успокоиться и сделать что-нибудь простое: скажем, реализовать `peek_front`.
Этот метод всегда был простым.
Должен и остаться простым, правда?

Правда?

Фактически, я уверена, что могу просто скопировать старое решение!

```rust ,ignore
pub fn peek_front(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        &node.elem
    })
}
```

Но, погодите.
Не в этот раз.

```rust ,ignore
pub fn peek_front(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        // ЗАИМСТВОВАНИЕ!!!!
        &node.borrow().elem
    })
}
```

ХА.

```text
cargo build

error[E0515]: cannot return value referencing temporary value
  --> src/fourth.rs:66:13
   |
66 |             &node.borrow().elem
   |             ^   ----------^^^^^
   |             |   |
   |             |   temporary value created here
   |             |
   |             returns a value referencing data owned by the current function
```

Ладно, просто сожгу свой компьютер.

Здесь та же самая логика, что и в односвязном стеке.
Почему всё по другому?
ПОЧЕМУ?

Ответ на самом деле отражает мораль всей этой главы: RefCell всё усложняет.
До текущего момента RefCell просто доставлял неудобства.
Теперь он превратился в ночной кошмар.

Так что же случилось?
Чтобы в этом разобраться, надо вернуться к определению `borrow`:

```rust ,ignore
fn borrow<'a>(&'a self) -> Ref<'a, T>;
fn borrow_mut<'a>(&'a self) -> RefMut<'a, T>;
```

В разделе о представлении я писала:

> RefCell проверяет возможность заимствования не статически (во время компиляции), а > динамически (во время выполнения).
> Если вы нарушили правила, RefCell вызовет panic! и завершит программу.
> Почему методы возвращают объекты Ref и RefMut?
> Они, по сути, ведут себя как `Rc`, но только по отношению к заимствованию.
> Они также держат объект RefCell заимствованным, пока не выйдут из области видимости.
> **Позже мы вернёмся к этому моменту.**

Это «позже» наступило.

`Ref` и `RefMut` реализуют `Deref` и `DerefMut` соответственно.
В большинстве случаев они ведут себя *точно так же* как `&T` и `&mut T`.
Однако из-за особенностей работы этих типажей, возвращаемая ссылка связана с временем жизни Ref, а не с самой ячейкой RefCell.
Это значит, что Ref должна оставаться доступна до тех пор, пока мы пользуемся полученной ссылкой.

Это нужно, чтобы сохранить корректность.
Когда Ref уничтожается, он сообщает RefCell, что больше не заимствуется.
Таким образом, если бы нам удалось сохранить нашу ссылку дольше, чем существует Ref, мы бы могли создать RefMut на ту же самую область памяти.
Тогда бы у нас одновременны были изменяемая и разделяемая ссылки на одно и то же значение, что полностью ломает систему типов Rust.

Итак, что нам остаётся?
Мы хотим всего лишь вернуть ссылку, но для этого нам надо сохранить Ref.
Но как только мы возвращаем ссылку из `peek`, функция завершается и `Ref` выходит из области видимости.

😖

Насколько я понимаю, сейчас мы оказались в абсолютном тупике.
Нельзя полностью инкапсулировать использование RefCell тем способом, который мы придумали.

Но... что если мы откажемся от идеи полностью скрывать детали реализации?
Что если мы будем возвращать Ref?

```rust ,ignore
pub fn peek_front(&self) -> Option<Ref<T>> {
    self.head.as_ref().map(|node| {
        node.borrow()
    })
}
```

```text
> cargo build

error[E0412]: cannot find type `Ref` in this scope
  --> src/fourth.rs:63:40
   |
63 |     pub fn peek_front(&self) -> Option<Ref<T>> {
   |                                        ^^^ not found in this scope
help: possible candidates are found in other modules, you can import them into scope
   |
1  | use core::cell::Ref;
   |
1  | use std::cell::Ref;
   |
```

Бла, бла, бла.
Надо кое-что импортировать.


```rust ,ignore
use std::cell::{Ref, RefCell};
```

```text
> cargo build

error[E0308]: mismatched types
  --> src/fourth.rs:64:9
   |
64 | /         self.head.as_ref().map(|node| {
65 | |             node.borrow()
66 | |         })
   | |__________^ expected type parameter, found struct `fourth::Node`
   |
   = note: expected type `std::option::Option<std::cell::Ref<'_, T>>`
              found type `std::option::Option<std::cell::Ref<'_, fourth::Node<T>>>`
```

Хмм... всё правильно.
У нас есть `Ref<Node<T>>`, но нам нужен `Ref<T>`.
Мы могли бы отказаться от всякой надежды на инкапсуляцию и просто возвращать то, что есть.
Мы могли бы ещё больше усложнить код и завернуть `Ref<Node<T>>` в новый тип, чтобы предоставлять доступ только к `&T`.

Оба варианта *так себе*.

Вместо этого, углубимся в тему.
Давайте немного *повеселимся*.
Источник нашего веселья — *вот это чудовище*:

```rust ,ignore
fn map<U, F>(orig: Ref<'b, T>, f: F) -> Ref<'b, U>
    where F: FnOnce(&T) -> &U,
          U: ?Sized
```

> Создаёт новую Ref для компонента заимствованных данных.

Да: точно также, как вы можете вызывать map для Option, вы можете вызывать map для Ref.

Я уверена, что кто-то где-то сейчас возбудился из-за *монад* или чего-то подобного, но мне они совершенно не интересны.
К тому же я не уверена, что речь идёт о настоящей монаде, поскольку нет варианта, похожего на None.
Впрочем, я отвлеклась.

Мне нравится, что я нашла крутое решение.
*Люблю такие штуки*.

```rust ,ignore
pub fn peek_front(&self) -> Option<Ref<T>> {
    self.head.as_ref().map(|node| {
        Ref::map(node.borrow(), |node| &node.elem)
    })
}
```

```text
> cargo build
```

Да-да-да-да.

Давайте убедимся, что код работает, немного изменив тест, написанный для нашего стека.
Нам нужно кое-что добавить, чтобы справиться с тем фактом, что Ref не реализуют операцию сравнения.

```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert!(list.peek_front().is_none());
    list.push_front(1); list.push_front(2); list.push_front(3);

    assert_eq!(&*list.peek_front().unwrap(), &3);
}
```


```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 10 tests
test first::test::basics ... ok
test fourth::test::basics ... ok
test second::test::basics ... ok
test fourth::test::peek ... ok
test second::test::iter_mut ... ok
test second::test::into_iter ... ok
test third::test::basics ... ok
test second::test::peek ... ok
test second::test::iter ... ok
test third::test::iter ... ok

test result: ok. 10 passed; 0 failed; 0 ignored; 0 measured

```

Великолепно!