> # Peeking

# Метод Peek

> Alright, we made it through `push` and `pop`.
> I'm not gonna lie, it got a bit emotional there.
> Compile-time correctness is a hell of a drug.

Хорошо, мы справились с методами `push` и `pop`.
Не буду врать, я немного разволновалась.
Проверка правильности на этапе компиляции — охнеренное чувство.

> Let's cool off by doing something simple: let's just implement `peek_front`.
> That was always really easy before.
> Gotta still be easy, right?

Надо немнго успокоиться и сделать что-нибудь простое: скажем, реализовать `peek_front`.
Это то, что всегда было простым раньше.
Должно и остаться простым, правда?

> Right?

Правда?

> In fact, I think I can just copy-paste it!

Фактически, я уверена, что могу просто скопи-пастить старое решение!

```rust ,ignore
pub fn peek_front(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        &node.elem
    })
}
```

> Wait.
> Not this time.

Подождите.
Не в этот раз.

```rust ,ignore
pub fn peek_front(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        // BORROW!!!!
        &node.borrow().elem
    })
}
```

> HAH.

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

> Ok I'm just burning my computer.

Ладно, просто сожгу свой компьютер.

> This is exactly the same logic as our singly-linked stack.
> Why are things different.
> WHY.

Здесь та же самая логика, что и в нашем односвязном стеке.
Почему всё по другому?
ПОЧЕМУ.

> The answer is really the whole moral of this chapter: RefCells make everything sadness.
> Up until now, RefCells have just been a nuisance.
> Now they're going to become a nightmare.

Ответ на самом деле представляет мораль всей этой главы: RefCell делает всё грустным.
До текущего момента RefCell просто доставлял неудобства.
Теперь он стал ночным кошмаром.

> So what's going on?
> To understand that, we need to go back to the definition of `borrow`:

Так что случилось?
Чтобы в этом разобраться, нам надо вернуться к определению <!-- метода --> `borrow`:

```rust ,ignore
fn borrow<'a>(&'a self) -> Ref<'a, T>
fn borrow_mut<'a>(&'a self) -> RefMut<'a, T>
```

> In the layout section we said:

В разделе о представлении мы говорили:

<!-- скопи-пастить перевод из раздела Lyaout -->

> > Rather than enforcing this statically, RefCell enforces them at runtime.
> > If you break the rules, RefCell will just panic and crash the program.
> > Why does it return these Ref and RefMut things?
> > Well, they basically behave like `Rc`s but for borrowing.
> > Also they keep the RefCell borrowed until they go out of scope.
> > **We'll get to that later.**

> It's later.

Это «позже» настало.

> `Ref` and `RefMut` implement `Deref` and `DerefMut` respectively.
> So for most intents and purposes they behave *exactly* like `&T` and `&mut T`.
> However, because of how those traits work, the reference that's returned is connected to the lifetime of the Ref, and not the actual RefCell.
> This means that the Ref has to be sitting around as long as we keep the reference around.

`Ref` и `RefMut` реализуют `Deref` и `DerefMut` соответственно.
Так что в большинстве случаев они ведут себя *точно так же* как `&T` и `&mut T`.
Однако из-за особенностей работы этих типажей, возвращаемая ссылка связана с временем жизни Ref, а не с самой ячейкой RefCell.
Это значит, что Ref должна оставаться доступна до тех пор, пока мы её используем.

> This is in fact necessary for correctness.
> When a Ref gets dropped, it tells the RefCell that it's not borrowed anymore.
> So if we *did* manage to hold onto our reference longer than the Ref existed, we could get a RefMut while a reference was kicking around and totally break Rust's type system in half.

Это нужно, чтобы сохранить корректность.
Когда Ref уничтожается, он сообщает RefCell, что больше не заимствуется.
Таким образом если бы нам удалось сохранить нашу ссылку дольше, чем существовала Ref, мы бы могли получить RefMut, в то время, как ссылка где-то слоняется и полностью сломали бы надвое систему типов Rust.

> So where does that leave us?
> We only want to return a reference, but we need to keep this Ref thing around.
> But as soon as we return the reference from `peek`, the function is over and the `Ref` goes out of scope.

Итак, что нам остаётся?
Мы всего лишь хотим вернуть ссылку, но для этого мы должны сохранить Ref.
Но как только мы возвращаем ссылку из `peek`, функция завершается из `Ref` выходит из области видимости.

😖

> As far as I know, we're actually totally dead in the water here.
> You can't totally encapsulate the use of RefCells like that.

Насколько я понимаю, сейчас мы полностью оказались в тупике.
Вы не можете полностью инкапсулировать использование RefCell подобным образом.

> But... what if we just give up on totally hiding our implementation details?
> What if we returns Refs?

Но... что если мы откажемся от идеи полностью скрывать детали реализации?
Что если мы будуем возвращать Ref?

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

> Blurp.
> Gotta import some stuff.

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

> Hmm... that's right.
> We have a `Ref<Node<T>>`, but we want a `Ref<T>`.
> We could abandon all hope of encapsulation and just return that.
> We could also make things even more complicated and wrap `Ref<Node<T>>` in a new type to only expose access to an `&T`.

Хмм... всё правильно.
У нас есть `Ref<Node<T>>`, но нам нужен `Ref<T>`.
Мы могли бы отказаться от всякой надежды на инкапсуляцию и просто возвращать его.
Мы могли бы ещё больше усложнить код и завернуть `Ref<Node<T>>` в новый тип, чтобы предоставлять доступ только к `&T`.

> Both of those options are *kinda* lame.

Оба варианта *как бы* увечные.

> Instead, we're going to go deeper down.
> Let's have some *fun*.
> Our source of fun is *this beast*:

Вместо этого, углубимся в тему.
Давайте немного *повеселимся*.
Источник нашего веселья — *вот это чудовище*:

```rust ,ignore
map<U, F>(orig: Ref<'b, T>, f: F) -> Ref<'b, U>
    where F: FnOnce(&T) -> &U,
          U: ?Sized
```

> > Make a new Ref for a component of the borrowed data.

> Создаёт новую Ref для компонента заимствованных данных.

> Yes: just like you can map over an Option, you can map over a Ref.

Да: точно также, как вы можете вызывать map для Option, вы можете вызывать map для Ref.

> I'm sure someone somewhere is really excited because *monads* or whatever but I don't care about any of that.
> Also I don't think it's a proper monad since there's no None-like case, but I digress.

Я уверена, что кто-то где-то действительно возбудился из-за *монад* или чего-то подобного, но меня они совершенно не волнуют.
К тому же я не думаю, что это настоящая монада, поскольку нет варианта, похожего на None, но я отвлеклась.

> It's cool and that's all that matters to me.
> *I need this*.

Это круто, и это всё, что имеет для меня значение.
*Мне это нужно*.

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

> Awww yissss

Да-да-да-да.

> Let's make sure this is working by munging up the test from our stack.
> We need to do some munging to deal with the fact that Refs don't implement comparisons.

Давайте убедимся, что это работает, немного изменив тест, написанный для нашего стека.
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

> Great!

Великолепно!