> # Layout

# Представление данных

> The key to our design is the `RefCell` type.
> The heart of RefCell is a pair of methods:

Ключевым элементом нашей архитектуры является тип `RefCell`.
Сердцем `RefCell` является пара методов:

```rust ,ignore
fn borrow(&self) -> Ref<'_, T>;
fn borrow_mut(&self) -> RefMut<'_, T>;
```

> The rules for `borrow` and `borrow_mut` are exactly those of `&` and `&mut`: you can call borrow` as many times as you want, but `borrow_mut` requires exclusivity.

Правила для `borrow` и `borrow_mut` точно такие же, как и для `&` и `&mut`: вы можете вызвать `borrow` столько раз, сколько захотите, но `borrow_mut` требует эксклюзивности.

> Rather than enforcing this statically, RefCell enforces them at runtime.
> If you break the rules, RefCell will just panic and crash the program.
> Why does it return these Ref and RefMut things?
> Well, they basically behave like `Rc`s but for borrowing.
> They also keep the RefCell borrowed until they go out of scope.
> We'll get to that later.

Вместо статической проверки RefCell проверяет их во время выполнения.
Если вы нарушаете правила, RefCell вызовет panic! и завершит программу.
Для чего она <!-- ячейка, cell --> возвращает эти объекты Ref и RefMut?
Ну, они по сути ведёт себя как `Rc`, но только по отношению к заимствованию.
Они также оставляют заимстованным сам объект RefCell, пока не выйдут из области видимости.
Позже мы обсудим этот момент.

> Now with Rc and RefCell we can become... an incredibly verbose pervasively mutable garbage collected language that can't collect cycles!
> Y-yaaaaay...

Теперь, имея на руках Rc и RefCell, мы можем сделать из Rust... невероятно многословным, полностью изменяемым языком со сборкой мусора, который не умеет собирать циклические ссылки!
О-фи-геть!

> Alright, we want to be *doubly-linked*.
> This means each node has a pointer to the previous and next node.
> Also, the list itself has a pointer to the first and last node.
> This gives us fast insertion and removal on *both* ends of the list.

Ладно, нам <!-- всего лишь --> нужна *двусвязность*.
Это значит, в каждом узле хранится указатель на следующий и предыдущий узел.
Также, у самого списка есть указатели на первый и последний узел.
Это даёт нам быструю вставку и удаление на *обоих* концах списка.

> So we probably want something like:

Так что, возможно, мы хотим что-то такое:

```rust ,ignore
use std::rc::Rc;
use std::cell::RefCell;

pub struct List<T> {
    head: Link<T>,
    tail: Link<T>,
}

type Link<T> = Option<Rc<RefCell<Node<T>>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
    prev: Link<T>,
}
```

```text
> cargo build

warning: field is never used: `head`
 --> src/fourth.rs:5:5
  |
5 |     head: Link<T>,
  |     ^^^^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: field is never used: `tail`
 --> src/fourth.rs:6:5
  |
6 |     tail: Link<T>,
  |     ^^^^^^^^^^^^^

warning: field is never used: `elem`
  --> src/fourth.rs:12:5
   |
12 |     elem: T,
   |     ^^^^^^^

warning: field is never used: `next`
  --> src/fourth.rs:13:5
   |
13 |     next: Link<T>,
   |     ^^^^^^^^^^^^^

warning: field is never used: `prev`
  --> src/fourth.rs:14:5
   |
14 |     prev: Link<T>,
   |     ^^^^^^^^^^^^^
```

> Hey, it built!
> Lots of dead code warnings, but it built!
> Let's try to use it.

Ого, оно собралось!
Много предупреждений о неиспользуемых переменных, но оно собралось!
Давайте попробуем его использовать.