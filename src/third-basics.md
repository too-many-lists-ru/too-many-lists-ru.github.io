> # Basics

# Основы

> We already know a lot of the basics of Rust now, so we can do a lot of the simple stuff again.

Сейчас мы уже знаем множество основ языка Rust, 

> For the constructor, we can again just copy-paste:

В конструкторе мы можем просто использовать копи-пасту:

```rust ,ignore
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None }
    }
}
```

> `push` and `pop` don't really make sense anymore.
> Instead we can provide `prepend` and `tail`, which provide approximately the same thing.

`push` и `pop` в любом случае больше не имеют смысла.
Вместо них мы должны предоставить операции `prepend` (вставить в начало) и `tail` (хвост), которые предоставят приблизительно те же возможности.

> Let's start with prepending.
> It takes a list and an element, and returns a List.
> Like the mutable list case, we want to make a new node, that has the old list as its `next` value.
> The only novel thing is how to *get* that next value, because we're not allowed to mutate anything.

Давайте начнём со вставки.
Она получает список и элемент и возвращает список.
Как и в случае с мутабельным списком, мы хотим создать новый узел, у которого старый список хранится в поле `next`.
Единственное нововведение заключается в том, как *получить* значение для `next`, поскольку мы не можем ничего менять.

> The answer to our prayers is the Clone trait.
> Clone is implemented by almost every type, and provides a generic way to get "another one like this one" that is logically disjoint, given only a shared reference.
> It's like a copy constructor in C++, but it's never implicitly invoked.

Ответом на наши молитвы является типаж Clone.
Clone реализуется практически любым типом и предоставляет универсальный способ получить «что-то похожее на вот это», используя при этом только разделяемую ссылку, при этом клон будет логически отличаться от оригинала.
Метод похож на компирующий конструктор в C++, но никогда не вызывается неявно. <!-- всегда вызывается явно -->

> Rc in particular uses Clone as the way to increment the reference count.
> So rather than moving a Box to be in the sublist, we just clone the head of the old list.
> We don't even need to match on the head, because Option exposes a Clone implementation that does exactly the thing we want.

В частности, Rc испозьуте Clone, как способ увеличить счётчик ссылок на единицу.
Так что вместо того, чтобы перемещать Box в подсписок, мы просто клонируем голову старого списка.
Нам даже не надо выполнять `match` над головой, потому что Option предоставляет реализацию Clone, которая делает точно то, что нам нужно.

> Alright, let's give it a shot:

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

> Wow, Rust is really hard-nosed about actually using fields.
> It can tell no consumer can ever actually observe the use of these fields!
> Still, we seem good so far.

Да уж, Rust поистине непреклонен по поводу фактического использования <!-- объявленных  --> полей.
Можно сказать, ни один потребитель не может на самом деле наблюдать использование этих полей! <!-- вообще не понимаю, что хотел сказать автор Похоже, речь о том, что ни один потребитель не может использовать эти поля незаметно  -->
В любом случае, пока всё выглядит неплохо.

> `tail` is the logical inverse of this operation.
> It takes a list and returns the whole list with the first element removed.
> All that is is cloning the *second* element in the list (if it exists).
> Let's try this:

`tail` — это логическая инверсия этой <!-- предыдущей --> операции.
Она получает список и возвращает список без первого элемента.
Всё, что нужно сделать — это клонировать второй элемент в списке (если он существует).
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

> Hrm, we messed up.
> `map` expects us to return a Y, but here we're returning an `Option<Y>`.
> Thankfully, this is another common Option pattern, and we can just use `and_then` to let us return an Option.

Грм, мы всё испортили.
`map` ожидает, что мы вернём Y, но здесь мы возвращаем `Option<Y>`.
К счастью, есть ещё один известный способ использовать Option, так что мы можем просто вызвать `and_then`, чтобы мы могли вернуть Option.

```rust ,ignore
pub fn tail(&self) -> List<T> {
    List { head: self.head.as_ref().and_then(|node| node.next.clone()) }
}
```

```text
> cargo build

```

> Great.

Великолепно.

> Now that we have `tail`, we should probably provide `head`, which returns a reference to the first element.
> That's just `peek` from the mutable list:

Теперь, когда у нас есть `tail`, мы, возможно, должны написать `head`, чтобы возвращать ссылку на первый элемент.
По сути, это `peek` из мутабельного списка:

```rust ,ignore
pub fn head(&self) -> Option<&T> {
    self.head.as_ref().map(|node| &node.elem)
}
```

```text
> cargo build

```

> Nice.

Здорово.

> That's enough functionality that we can test it:

У нас достаточно функциональность, чтобы мы захотели её протестировать:


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

        // Make sure empty tail works
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

> Perfect!

Превосходно!

> Iter is also identical to how it was for our mutable list:

Iter тоже идентичен реализации из нашего мутабельного списка:

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

> Who ever said dynamic typing was easier?

Кто вообще сказал, что динамическая типизация проще?

> (chumps did)

(это говорили болваны)

> Note that we can't implement IntoIter or IterMut for this type.
> We only have shared access to elements.

Обратите внимание, что мы не можем реализовать IntoIter или IterMut для этого типа.
У нас может быть только разделяемый доступ к элементам.