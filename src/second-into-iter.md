> # IntoIter

# Копирующий итератор (итератор по значениям)

> Collections are iterated in Rust using the *Iterator* trait.
> It's a bit more complicated than `Drop`:

Перебор коллекций в Rust выполняется с помощью типажа *Iterator*.
Он ненамного сложнее, чем `Drop`:

```rust ,ignore
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

> The new kid on the block here is `type Item`.
> This is declaring that every implementation of Iterator has an *associated type* called Item.
> In this case, this is the type that it can spit out when you call `next`.

Новая штука здесь — это `type Item`.
<!-- New Kids On The Block -- Новенькие на районе -- американская группа. -->
Она объявляет, что каждая реализация Итератора имеет *связанный с ней тип*, называемый Iter.
В данном случае это тот самый тип, который возвращает функция `next`.

> The reason Iterator yields `Option<Self::Item>` is because the interface coalesces the `has_next` and `get_next` concepts.
> When you have the next value, you yield `Some(value)`, and when you don't you yield `None`.
> This makes the API generally more ergonomic and safe to use and implement, while avoiding redundant checks and logic between `has_next` and `get_next`.
> Nice!

Причина по которой Iterator генерирует значения `Option<Self::Item>` в том, что этот интерфейс объединяет концепции «есть ли ещё элементы?» и «верни следующий элемент».
Если у вас есть следующее значение, вы возвращаете `Some(value)`, а, если нет — `None`.
Такой подход делает API в целом более эргономичным и безопасным для использования и реализации, поскольку позволяет убрать проверки и логику между этапами.
Прекрасно!

> Sadly, Rust has nothing like a `yield` statement (yet), so we're going to have to implement the logic ourselves.
> Also, there's actually 3 different kinds of iterator each collection should endeavour to implement:

К сожалению, в Rust нет ничего похожего на оператор `yield` (пока), так что нам предстоит самостоятельно реализовать эту логику.
Также, есть три различных вида итераторов, которые каждая коллекция должна постараться реализовать:

> * IntoIter - `T`
> * IterMut - `&mut T`
> * Iter - `&T`

* IntoIter (итератор по значениям) — `T`
* IterMut (итератор по мутабельным сслыкам) — `&mut T`
* Iter (итератор по сслыкам) — `&T`

> We actually already have all the tools to implement IntoIter using List's interface: just call `pop` over and over.
> As such, we'll just implement IntoIter as a newtype wrapper around List:

На самом деле у нас уже есть всё, что нужно, чтобы реализовать IntoIter используя интерфейс Списка: просто раз за разом вызываем `pop`.
Поэтому мы просто реализуем IntoInter, как новый тип поверх List:


```rust ,ignore
// Tuple structs are an alternative form of struct,
// useful for trivial wrappers around other types.
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
        // access fields of a tuple struct numerically
        // получаем доступ к полям кортежа по номеру
        self.0.pop()
    }
}
```

> And let's write a test:

А теперь напишем тест:

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

> Nice!

Прекрасно!