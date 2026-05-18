# Разборка

У `pop_front` должна быть та же самая базовая логика, что и у `push_front`, но обращённая.
Попробуем:

```rust ,ignore
pub fn pop_front(&mut self) -> Option<T> {
    // надо получить старую голову, убедимся, что количество ссылок -2
    self.head.take().map(|old_head| {                         // -1 old_head
        match old_head.borrow_mut().next.take() {
            Some(new_head) => {                               // -1 new_head
                // не пустой список
                new_head.borrow_mut().prev.take();            // -1 old_head
                self.head = Some(new_head);                   // +1 new_head
                // всего: -2 старых, +0 новых
            }
            None => {
                // пустой список
                self.tail.take();                             // -1 old_head
                // всего: -2 старых, (новых нет)
            }
        }
        old_head.elem
    })
}
```

```text
> cargo build

error[E0609]: no field `elem` on type `std::rc::Rc<std::cell::RefCell<fourth::Node<T>>>`
  --> src/fourth.rs:64:22
   |
64 |             old_head.elem
   |                      ^^^^ unknown field
```

ТАК.
*RefCells*.
Чувствую, нам снова нужен `borrow_mut`...

```rust ,ignore
pub fn pop_front(&mut self) -> Option<T> {
    self.head.take().map(|old_head| {
        match old_head.borrow_mut().next.take() {
            Some(new_head) => {
                new_head.borrow_mut().prev.take();
                self.head = Some(new_head);
            }
            None => {
                self.tail.take();
            }
        }
        old_head.borrow_mut().elem
    })
}
```

```text
cargo build

error[E0507]: cannot move out of borrowed content
  --> src/fourth.rs:64:13
   |
64 |             old_head.borrow_mut().elem
   |             ^^^^^^^^^^^^^^^^^^^^^^^^^^ cannot move out of borrowed content
```

*вздох*

> cannot move out of borrowed content

Не могу извлечь заимствованное содержимое.

Грм...
Похоже, *на самом деле* Box нас баловал.
`borrow_num` возвращает нам `&mut Nude<T>`, но мы не можем извлечь оттуда ничего!

Нам нужно что-то, что возьмёт у нас `RefCell<T>` и вернёт нам `T`.
Давайте проверим [доки](https://doc.rust-lang.org/std/cell/struct.RefCell.html) на предмет наличия чего-то похожего:

> `fn into_inner(self) -> T`
>
> Получает RefCell, возвращает завёрнутое значение.

Выглядит многообещающе!

```rust ,ignore
old_head.into_inner().elem
```

```text
> cargo build

error[E0507]: cannot move out of an `Rc`
  --> src/fourth.rs:64:13
   |
64 |             old_head.into_inner().elem
   |             ^^^^^^^^ cannot move out of an `Rc`
```

Вот блин.
`into_inner` хочет извлечь RefCell, но мы не можем её предоставить, потому что она находится в `Rc`.
В предыдущей главе мы видели, что `Rc<T>` разрешает доступ к своему значению только по разделяемой ссылке.
И это правильно, потому что *главный смысл* указателей с подсчётом ссылок в том, что они разделяемые!

Это стало для нас проблемой, когда мы писали drop для списка с подсчётом ссылок.
Воспользуемся тем же решением, а именно, методом `Rc::try_unwrap`, который возвращает содержимое Rc, если счётчик ссылок равен 1.

```rust ,ignore
Rc::try_unwrap(old_head).unwrap().into_inner().elem
```

`Rc::try_unwrap` возвращает `Result<T, Rc<T>>`.
Значения типа Result — расширенный тип `Option` в котором у варианта `None` появляются связанные с ним данные.
В нашем случае — Rc, из которого мы пытаемся достать значение.
Поскольку нас не беспокоят возможные ошибки (если мы правильно написали программу, она *обязана* завершиться успешно), мы просто вызываем `unwrap`.

В любом случае, давайте посмотрим, когда ошибка компиляции возникнет на этот раз (будем честными — она обязательно возникнет).

```text
> cargo build

error[E0599]: no method named `unwrap` found for type `std::result::Result<std::cell::RefCell<fourth::Node<T>>, std::rc::Rc<std::cell::RefCell<fourth::Node<T>>>>` in the current scope
  --> src/fourth.rs:64:38
   |
64 |             Rc::try_unwrap(old_head).unwrap().into_inner().elem
   |                                      ^^^^^^
   |
   = note: the method `unwrap` exists but the following trait bounds were not satisfied:
           `std::rc::Rc<std::cell::RefCell<fourth::Node<T>>> : std::fmt::Debug`
```

Да что ты будешь делать?
`unwrap` у Result требует, чтобы тип ошибки мог печатать себя при отладке.
А `RefCell<T>` реализует `Debug` только тогда, когда его реализует `T`.
А `Node` не реализует Debug.

Вместо того, чтобы добавлять реализацию, выполним обходной маневр, преобразовав Result в Options с помощью вызова `ok`:

```rust ,ignore
Rc::try_unwrap(old_head).ok().unwrap().into_inner().elem
```

НУ, ПОЖАЛУЙСТА.

```text
cargo build

```

ДА.

*уф*

Получилось.

Мы реализовали `push` и `pop`.

Давайте протестируем реализацию с помощью нашего старого базового теста для стека (потому что он единственный, который мы пока написали):

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn basics() {
        let mut list = List::new();

        // Проверяем, что пустой список ведёт себя правильно
        assert_eq!(list.pop_front(), None);

        // Заполняем список
        list.push_front(1);
        list.push_front(2);
        list.push_front(3);

        // Проверяем обычное удаление
        assert_eq!(list.pop_front(), Some(3));
        assert_eq!(list.pop_front(), Some(2));

        // Вставляем новые значения, просто чтобы проверить, что ничего не сломается
        list.push_front(4);
        list.push_front(5);

        // Проверяем обычное удаление
        assert_eq!(list.pop_front(), Some(5));
        assert_eq!(list.pop_front(), Some(4));

        // Проверяем граничный случай
        assert_eq!(list.pop_front(), Some(1));
        assert_eq!(list.pop_front(), None);
    }
}
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 9 tests
test first::test::basics ... ok
test fourth::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::basics ... ok
test fifth::test::iter_mut ... ok
test third::test::basics ... ok
test second::test::iter ... ok
test third::test::iter ... ok
test second::test::into_iter ... ok

test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured

```

*В яблочко*.

Теперь, научившись корректно удалять элементы из списка, мы можем реализовать и типаж Drop.
На этот раз Drop чуть более интересен с концептуальной точки зрения.
Раньше мы реализовывали Drop для стека, чтобы избавиться от неограниченной рекурсии.
Сейчас нам надо реализовать Drop, чтобы хоть-нибудь вообще заработало.

`Rc` не умеет работать с циклическими ссылками.
Если есть цикл, объекты поддерживают существование друг друга.
А двусвязный список, как выяснилось — одна большая цепочка из крошечных циклов!
Так что когда мы уничтожаем наш список, два конечных узла уменьшают свои счётчики до 1... а затем ничего не происходит.
Ладно, когда наш список содержит ровно один узел, счётчик ссылок сбрасывается но нуля.
Но в идеале список должен корректно работать даже когда нём больше одного элемента.
Возможно, это только моё мнение.

Как мы видели, удаление элементов оказалось довольно болезненным.
Поэтому простейший способ для нас — это просто вызывать `pop`, пока мы не получим None:

```rust ,ignore
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        while self.pop_front().is_some() {}
    }
}
```

```text
cargo build

```

(На самом деле мы могли бы сделать это с помощью наших изменяемых стеков, но короткий путь только для тех, кто знает, куда идти!)

Можно было заняться реализацией версий `_back` для методов `push` и `pop`, но это всего лишь копи-паста.
К ней мы вернёмся позже в этой главе.
А сейчас давайте посмотрим на более интересные вещи!
