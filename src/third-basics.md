# Основы

Сейчас мы уже знаем основы языка Rust, так что можем делать множество простых вещей.

Конструктор можно просто скопировать из предыдущей главы:

```rust ,ignore
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None }
    }
}
```

Методы `push` и `pop` теперь не имеют смысла.
Вместо них мы напишем `prepend` (вставить в начало) и `tail` (получить хвост), которые делают приблизительно то же самое.

Начнём с `prepend`.
Метод получает элемент и список. и возвращает список.
Как и в случае с изменяемым списком, мы создадим новый узел, у которого старый список помещается в поле `next`.
Единственное нововведение заключается в том, как *получить* значение для `next`, поскольку мы не можем ничего менять.

Ответом на наши молитвы является типаж Clone.
Его реализуют практически все типы.
Он предоставляет универсальный способ получить «похожую штуку» и при этом использует только разделяемую ссылку.
Логически клон отличается от оригинала.
Метод похож на копирующий конструктор в C++, но всегда вызывается явно.

Конкретно Rc использует Clone, чтобы увеличить счётчик ссылок на единицу.
Поэтому вместо того, чтобы перемещать Box в подсписок, мы просто клонируем голову старого списка.
Нам даже не придётся писать оператор `match`, поскольку Option предоставляет реализацию Clone, которая делает именно то, что нам нужно.

Ладно, давайте попробуем:

```rust ,ignore
pub fn prepend(&self, elem: T) -> List<T> {
    List { head: Some(Rc::new(Node {
        elem: elem,
        next: self.head.clone(),
    }))}
}
```

```text
> cargo build

warning: field is never used: `elem`
  --> src/third.rs:10:5
   |
10 |     elem: T,
   |     ^^^^^^^
   |
   = note: #[warn(dead_code)] on by default

warning: field is never used: `next`
  --> src/third.rs:11:5
   |
11 |     next: Link<T>,
   |     ^^^^^^^^^^^^^
```

Да уж, Rust поистине непреклонен по поводу неиспользуемых полей.
Фактически, он предупреждает, что может не компилировать этот код, поскольку никто пользователей библиотеки даже не заметит разницы!
В любом случае, пока всё выглядит неплохо.

`tail` — это логическая инверсия предыдущей операции.
Она получает список и возвращает список без первого элемента.
Всё, что нужно сделать — это клонировать второй элемент (если он существует) и сделать его головой нового списка.
Давайте попробуем:

```rust ,ignore
pub fn tail(&self) -> List<T> {
    List { head: self.head.as_ref().map(|node| node.next.clone()) }
}
```

```text
cargo build

error[E0308]: mismatched types
  --> src/third.rs:27:22
   |
27 |         List { head: self.head.as_ref().map(|node| node.next.clone()) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `std::rc::Rc`, found enum `std::option::Option`
   |
   = note: expected type `std::option::Option<std::rc::Rc<_>>`
              found type `std::option::Option<std::option::Option<std::rc::Rc<_>>>`
```

Грм, здесь мы напортачили.
`map` ожидает, что мы вернём Y, но мы возвращаем `Option<Y>`.
К счастью, есть ещё один известный паттерн для Option, так что мы можем просто вызвать `and_then` вместо `map`.

```rust ,ignore
pub fn tail(&self) -> List<T> {
    List { head: self.head.as_ref().and_then(|node| node.next.clone()) }
}
```

```bash
cargo build
```

Великолепно.

Теперь, когда у нас есть `tail`, мы, возможно, должны написать `head`, чтобы возвращать ссылку на первый элемент.
По сути, это метод `peek`, который мы реализовали для изменяемого списка:

```rust ,ignore
pub fn head(&self) -> Option<&T> {
    self.head.as_ref().map(|node| &node.elem)
}
```

```bash
cargo build
```

Здорово.

Теперь у нас достаточно функциональности, чтобы её протестировать:


```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn basics() {
        let list = List::new();
        assert_eq!(list.head(), None);

        let list = list.prepend(1).prepend(2).prepend(3);
        assert_eq!(list.head(), Some(&3));

        let list = list.tail();
        assert_eq!(list.head(), Some(&2));

        let list = list.tail();
        assert_eq!(list.head(), Some(&1));

        let list = list.tail();
        assert_eq!(list.head(), None);

        // Убеждаемся, что tail работает с пустым списком
        let list = list.tail();
        assert_eq!(list.head(), None);

    }
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 5 tests
test first::test::basics ... ok
test second::test::into_iter ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test third::test::basics ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured

```

Превосходно!

Iter тоже идентичен реализации из нашего изменяемого списка:

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<'_, T> {
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

```rust ,ignore
#[test]
fn iter() {
    let list = List::new().prepend(1).prepend(2).prepend(3);

    let mut iter = list.iter();
    assert_eq!(iter.next(), Some(&3));
    assert_eq!(iter.next(), Some(&2));
    assert_eq!(iter.next(), Some(&1));
}
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 7 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test second::test::into_iter ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured

```

Кто вообще сказал, что динамическая типизация проще?

(наверняка, какие-то болваны)

Обратите внимание, что мы не можем реализовать IntoIter или IterMut для этого типа.
К элементам может быть только разделяемый доступ.
