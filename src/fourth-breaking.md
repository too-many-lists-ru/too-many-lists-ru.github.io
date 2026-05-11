> # Breaking Down

# Разбираем

> `pop_front` should be the same basic logic as `push_front`, but backwards.
> Let's try:

У `pop_front` должна быть та же самая базовая логика, что и у `push_front`, только обращённая.

```rust ,ignore
pub fn pop_front(&mut self) -> Option<T> {
    // need to take the old head, ensuring it's -2
    // надо получить старую голову, убедимся, что количество ссылок -2
    self.head.take().map(|old_head| {                         // -1 old старая
        match old_head.borrow_mut().next.take() {
            Some(new_head) => {                               // -1 new новая
                // not emptying list
                // не пустой список
                new_head.borrow_mut().prev.take();            // -1 old старая
                self.head = Some(new_head);                   // +1 new новая
                // total: -2 old, +0 new
                // всего: -2 старыъ, +0 новых
            }
            None => {
                // emptying list
                // пустой список
                self.tail.take();                             // -1 old
                // total: -2 old, (no new)
                // всего: -2 старых, (нет новых)
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

> ACK.
> *RefCells*.
> Gotta `borrow_mut` again I guess...

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

> *sigh*

*вздох*

> > cannot move out of borrowed content

> не могу извлечь заимствованное содержимое

> Hrm...
> It seems that Box was *really* spoiling us.
> `borrow_mut` only gets us an `&mut Node<T>`, but we can't move out of that!

Грм...
Похоже, Box нас *в действительности* баловал.
`borrow_num` возращает нам только `&mut Nude<T>`, но мы не можем ничего оттуда извлечь!

> We need something that takes a `RefCell<T>` and gives us a `T`.
> Let's check [the docs][refcell] for something like that:

Нам нужно что-то, что возьмёт у нас `RefCell<T>` и вернёт нам `T`.
Давайте проверим доки на предмет наличия чего-то подобного:

> > `fn into_inner(self) -> T`
> >
> > Consumes the RefCell, returning the wrapped value.

> `fn into_inner(self) -> T`
>
> Получает RefCell, возвращает завёрнутое значение.

> That looks promising!

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

> Ah dang.
> `into_inner` wants to move out the RefCell, but we can't, because it's in an `Rc`.
> As we saw in the previous chapter, `Rc<T>` only lets us get shared references into its internals.
> That makes sense, because that's *the whole point* of reference counted pointers: they're shared!

Вот блин.
`into_inner` хочет извлечь <!-- ячейку --> RefCell, но мы не можем её предоставить, потому что она находится в `Rc`.
Как мы видели в предыдущей главе, `Rc<T>` разрешает нам только разделяемую ссылку на свои внутренности.
И это правильно, потому что *главный смысл* указателей с подсчётом ссылок в том, что они разделяемые!

> This was a problem for us when we wanted to implement Drop for our reference counted list, and the solution is the same: `Rc::try_unwrap`, which moves out the contents of an Rc if its refcount is 1.

Это стало для нас проблемой, когда мы хотели реализовать Drop для нашего списка с подсчётом ссылок, и решение не изменилось: <!-- метод --> `Rc::try_unwrap`, который возвращает содержимое Rc, если счётчик ссылок равен 1.

```rust ,ignore
Rc::try_unwrap(old_head).unwrap().into_inner().elem
```

> `Rc::try_unwrap` returns a `Result<T, Rc<T>>`.
> Results are basically a generalized `Option`, where the `None` case has data associated with it.
> In this case, the `Rc` you tried to unwrap.
> Since we don't care about the case where it fails (if we wrote our program correctly, it *has* to succeed), we just call `unwrap` on it.

`Rc::try_unwrap` возвращает `Result<T, Rc<T>>`.
Значения типа Result — расширенный тип `Option` в котором у варианта `None` появляются связанные с ним данные.
В нашем случае — Rc, из которого мы пытаемся достать значение.
Поскольку нас не беспокоят возможные ошибки (если мы правильно написали программу, она *обязана* успешно завершиться), мы просто вызываем `unwrap`.

> Anyway, let's see what compiler error we get next (let's face it, there's going to be one).

В любом случае, давайте посмотрим, когда ошибка компиляции возникнет на этот раз (и давайте будем честными — она обязательно возникнет).

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

> UGH.
> `unwrap` on Result requires that you can debug-print the error case.
> `RefCell<T>` only implements `Debug` if `T` does.
> `Node` doesn't implement Debug.

Да что ты будешь делать?
`unwrap` у Result требует, чтобы тип ошибки мог напечатать себя в отладочном режиме.
А `RefCell<T>` реализует `Debug` только тогда, когда его реализует `T`.
А `Node` не реализует Debug.

> Rather than doing that, let's just work around it by converting the Result to an Option with `ok`:

Вместо того, чтобы сделать это, давайте просто обойдём это, преобразовав Result в Options с помощью вызова `ok`:

```rust ,ignore
Rc::try_unwrap(old_head).ok().unwrap().into_inner().elem
```

> PLEASE.

НУ, ПОЖАЛУЙСТА.

```text
cargo build

```

> YES.

ДА.

> *phew*

*уф*

> We did it.

У нас получилось.

> We implemented `push` and `pop`.

Мы реализовали `push` и `pop`.

> Let's test by stealing the old `stack` basic test (because that's all that we've implemented so far):

Давайте потестируем с помощю нашего базового теста старого стека (поэтому он единственный, который мы на данный момент написали):

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn basics() {
        let mut list = List::new();

        // Check empty list behaves right
        assert_eq!(list.pop_front(), None);

        // Populate list
        list.push_front(1);
        list.push_front(2);
        list.push_front(3);

        // Check normal removal
        assert_eq!(list.pop_front(), Some(3));
        assert_eq!(list.pop_front(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push_front(4);
        list.push_front(5);

        // Check normal removal
        assert_eq!(list.pop_front(), Some(5));
        assert_eq!(list.pop_front(), Some(4));

        // Check exhaustion
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

> *Nailed it*.

*В яблочко*.

> Now that we can properly remove things from the list, we can implement Drop.
> Drop is a little more conceptually interesting this time around.
> Where previously we bothered to implement Drop for our stacks just to avoid unbounded recursion, now we need to implement Drop to get *anything* to happen at all.

Теперь, когда мы можем корректно удалять элементы из списка, мы можем реализовать и типаж Drop.
На этот раз Drop чуть более интересен с концептуальной точки зрения.
В то время, как раньше мы реализовывали Drop для наших стеков просто чтобы избавиться от неограниченной рекурсии, сейчас нам надо реализовать Drop, чтобы хоть-нибудь вообще работало.

> `Rc` can't deal with cycles.
> If there's a cycle, everything will keep everything else alive.
> A doubly-linked list, as it turns out, is just a big chain of tiny cycles!
> So when we drop our list, the two end nodes will have their refcounts decremented down to 1... and then nothing else will happen.
> Well, if our list contains exactly one node we're good to go.
> But ideally a list should work right if it contains multiple elements.
> Maybe that's just me.

`Rc` не умеет работать с циклами.
Если есть цикл, все объекты поддерживают существование друг друга.
Двусвязный список, как выяснилось, всег лишь большая цепочка крошечных циклов!
Так что когда мы уничтожает наш список, два конечных узла уменьшают свои счётчики до 1... а затем ничего не происходит.
Ладно, если наш список содержит точно один узел, можно оставить всё, как есть.
Но в идеалее, список должен корректно работать, если в нём множество элементов. <!-- вот здесь не понял -->
Возможно, это только моё мнение.

> As we saw, removing elements was a bit painful.
> So the easiest thing for us todo is just `pop` until we get None:

Как мы видели, удаление элементов было довольно болезненным.
Поэтому простейший способ для нас — просто вызывать `pop`, пока мы не получим None:

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

> (We actually could have done this with our mutable stacks, but shortcuts are for people who understand things!)

(На самом деле мы могли бы сделать это с помощью наших изменяемых стеков, но короткий путь только для тех, кто знает, куда идти!)

> We could look at implementing the `_back` versions of `push` and `pop`, but they're just copy-paste jobs which we'll defer to later in the chapter.
> For now let's look at more interesting things!

Мы могли бы взглянуть на реализацию `_back` версий методов `push` и `pop`, но это всего лишь копи-паста, к которым мы вернёмся позже в этой главе.
А сейчас давайте посмотрим на более интересные вещи!

[refcell]: https://doc.rust-lang.org/std/cell/struct.RefCell.html
[multirust]: https://github.com/brson/multirust
[downloads]: https://www.rust-lang.org/install.html
