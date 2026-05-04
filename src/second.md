> # An Ok Singly-Linked Stack

# Нормальный односвязный стек

> In the previous chapter we wrote up a minimum viable singly-linked stack.
> However there's a few design decisions that make it kind of sucky.
> Let's make it less sucky.
> In doing so, we will:

В прошлой главе мы написали минимальный жизнеспособный односвязный стек.
Хотя было несклько проектных решений которые делают его довольно отстойным.
Давайте сделаем его менее отстойным.
Для этого мы:

> * Deinvent the wheel
> * Make our list able to handle any element type
> * Add peeking
> * Make our list iterable

* Переизобретём велосипед
* Сделаем наш список способным работать с элементами любого типа
* Добавим операцию заглядывания (`peek`)
* Реализуем итератор

> And in the process we'll learn about

И в процессе этого мы узнаем о:

> * Advanced Option use
> * Generics
> * Lifetimes
> * Iterators

* Продвинутом использовании Option
* Обобщениях
* Времени жизни
* Итераторах

> Let's add a new file called `second.rs`:

Давайте создадим новый файл `second.rs`:

```rust ,ignore
// in lib.rs
// в lib.rs

pub mod first;
pub mod second;
```

> And copy everything from `first.rs` into it.

И скопируем в него содержимое `first.rs`.
