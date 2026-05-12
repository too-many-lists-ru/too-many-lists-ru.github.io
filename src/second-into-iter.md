# IntoIter

Перебор коллекций в Rust выполняется с помощью типажа *Iterator*.
Он ненамного сложнее, чем `Drop`:

```rust ,ignore
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

Новая штука здесь — это `type Item`.
Она объявляет, что каждая реализация типажа Iterator имеет *связанный с ней тип*, называемый Iter.
В это тот самый тип, который возвращает функция `next`.

Причина по которой интерфейс Iterator возвращает значения `Option<Self::Item>` в том, что он объединяет концепции `has_next` (есть ли следующий элемент?) и `get_next` (верни следующий элемент).
Когда у вас есть следующее значение, вы возвращаете `Some(value)`, а когда нет — `None`.
Такой подход делает API более эргономичным и безопасным как для реализации, так и для использования, поскольку позволяет убрать проверки и логику между `has_next` и `get_next`.
Прекрасно!

К сожалению, (пока) в Rust нет ничего похожего на оператор `yield`, так что нам предстоит самостоятельно реализовывать логику перебора.
Есть три различных вида итераторов, которые должна постараться реализовать каждая коллекция:

* IntoIter (итератор по значениям) — `T`
* IterMut (итератор по изменяемым ссылкам) — `&mut T`
* Iter (итератор по разделяемым ссылкам) — `&T`

На самом деле у нас уже есть всё, что нужно, чтобы реализовать IntoIter используя интерфейс List: нам всего лишь надо раз за разом вызывать `pop`.
Поэтому мы просто реализуем IntoInter, как новый тип поверх List:


```rust ,ignore
// Кортежи — альтернативная форма структур,
// полезная для тривиальных обёрток вокруг других типов.
pub struct IntoIter<T>(List<T>);

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        // получаем доступ к полям кортежа по номеру
        self.0.pop()
    }
}
```

Ну, а теперь напишем тест:

```rust ,ignore
#[test]
fn into_iter() {
    let mut list = List::new();
    list.push(1); list.push(2); list.push(3);

    let mut iter = list.into_iter();
    assert_eq!(iter.next(), Some(3));
    assert_eq!(iter.next(), Some(2));
    assert_eq!(iter.next(), Some(1));
    assert_eq!(iter.next(), None);
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 4 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::peek ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured

```

Прекрасно!