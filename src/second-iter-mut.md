> # IterMut

# Итератор по мутабельным ссылкам

> I'm gonna be honest, IterMut is WILD.
> Which in itself seems like a wild thing to say; surely it's identical to Iter!

Я хочу быть честным: IterMut это БЕЗУМИЕ.
Что само по себе звучит безумно: ведь он должен быть похож на Iter!

> Semantically, yes, but the nature of shared and mutable references means that Iter is "trivial" while IterMut is Legit Wizard Magic.

Семантически, да, но природа разделяемых и мутабельных ссылок такова, что Iter можно считать «тривиальным», в то время как IterMut — это Подлинная Магия Волшебника.

> The key insight comes from our implementation of Iterator for Iter:

Ключевая идея берётся из нашей реализации Iterator для Iter:

```rust ,ignore
impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> { /* stuff что-то */ }
}
```

> Which can be desugared to:

Что можно преобразовать в:

```rust ,ignore
impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next<'b>(&'b mut self) -> Option<&'a T> { /* stuff что-то */ }
}
```

> The signature of `next` establishes *no* constraint between the lifetime of the input and the output!
> Why do we care?
> It means we can call `next` over and over unconditionally!

Сигнатура `next` *не* устанавливает никаких ограничений между временем жизни входной и выходной переменных!
Почему нас это волнуте?
Это означает, что мы можем вызывать `next` снова и снова без всяких условий!

```rust ,ignore
let mut list = List::new();
list.push(1); list.push(2); list.push(3);

let mut iter = list.iter();
let x = iter.next().unwrap();
let y = iter.next().unwrap();
let z = iter.next().unwrap();
```

> Cool!

Круто!

> This is *definitely fine* for shared references because the whole point is that you can have tons of them at once.
> However mutable references *can't* coexist.
> The whole point is that they're exclusive.

Это *совершенно нормально* для разделяемых ссылок, потому что их смысл в том, что их одновременно может быть очень много.
В то же время мутабельные ссылки *не могут* сосуществовать.
Их смысл в том, что они эксклюзивные.

> The end result is that it's notably harder to write IterMut using safe code (and we haven't gotten into what that even means yet...).
> Surprisingly, IterMut can actually be implemented for many structures completely safely!

В результате, написать безопасный код для IterMut гораздо сложнее (и мы, кстати, пока не выяснили, что это значит...).
Удивительно, но IterMut можно совершенно безопасно реализовать для многих структур данных!

> We'll start by just taking the Iter code and changing everything to be mutable:

Начнём с того, что просто возьмём код Iter и перепишем всё так, чтобы оно стало мутабельным:

```rust ,ignore
pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}

impl<T> List<T> {
    pub fn iter_mut(&self) -> IterMut<'_, T> {
        IterMut { next: self.head.as_deref_mut() }
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_deref_mut();
            &mut node.elem
        })
    }
}
```

```text
> cargo build
error[E0596]: cannot borrow `self.head` as mutable, as it is behind a `&` reference
  --> src/second.rs:95:25
   |
94 |     pub fn iter_mut(&self) -> IterMut<'_, T> {
   |                     ----- help: consider changing this to be a mutable reference: `&mut self`
95 |         IterMut { next: self.head.as_deref_mut() }
   |                         ^^^^^^^^^ `self` is a `&` reference, so the data it refers to cannot be borrowed as mutable

error[E0507]: cannot move out of borrowed content
   --> src/second.rs:103:9
    |
103 |         self.next.map(|node| {
    |         ^^^^^^^^^ cannot move out of borrowed content
```

> Ok looks like we've got two different errors here.
> The first one looks really clear though, it even tells us how to fix it!
> You can't upgrade a shared reference to a mutable one, so `iter_mut` needs to take `&mut self`.
> Just a silly copy-paste error.

Хорошо, выглядит так, словно у нас тут две разные ошибки.
Первая очень понятная, в сообщении даже написано, как её исправить!
Нельзя преобразовать разделяемую ссылку в мутабельную, так что `iter_mut` должен принимать `&mut self`.
Обычная глупая ошибка копи-пасты.

```rust ,ignore
pub fn iter_mut(&mut self) -> IterMut<'_, T> {
    IterMut { next: self.head.as_deref_mut() }
}
```

> What about the other one?

А что насчёт второй?

> Oops!
> I actually accidentally made an error when writing the `iter` impl in the previous section, and we were just getting lucky that it worked!

Ой!
На самом дела я случайно ошибся при написании `iter` в прошлом разделе, и нам просто повезло, что всё заработало!

> We have just had our first run in with the magic of Copy.
> When we introduced [ownership][ownership] we said that when you move stuff, you can't use it anymore.
> For some types, this makes perfect sense.
> Our good friend Box manages an allocation on the heap for us, and we certainly don't want two pieces of code to think that they need to free its memory.

Мы только что впервые столкнулись с магией копирования.
Когда мы ввели понятие [владения][ownership], мы сказали, когда вы передаёте владение, вы больше не можете использовать переменную.
Для некоторых типов это имеет смысл.
Наш добрый друг Box управляет размещением памяти в куче и мы конечно же не хотим, чтобы два куска кода думали, что они оба должны освободить эту память.

> However for other types this is *garbage*.
> Integers have no ownership semantics; they're just meaningless numbers!
> This is why integers are marked as Copy.
> Copy types are known to be perfectly copyable by a bitwise copy.
> As such, they have a super power: when moved, the old value *is* still usable.
> As a consequence, you can even move a Copy type out of a reference without replacement!

В то же время для других типов всё это *мусор*.
Целые числа не обладают семантикой владения, это всего лишь бессмысленные числа!
Это причина, по которой целые числа реализуют типаж Copy.
Известно, что типы, помеченные как Copy, идеально копируется побитовым копированием.
Поэтому они обладают сверхспособностью: при перемещении стаорое значение *всё ещё* можно использовать.
Как следствие, вы даже можете забрать копируемый тип из ссылки без замены!

> All numeric primitives in Rust (i32, u64, bool, f32, char, etc...) are Copy.
> You can also declare any user-defined type to be Copy as well, as long as all its components are Copy.

Копируемыми являются все численные примитивы Rust (i32, u64, bool, f32, char и т. д.).
Вы также можете обхявить любой свой тип копируемым, если все его компоненты — также копируемы.

> Critically to why this code was working, shared references are also Copy!
> Because `&` is copy, `Option<&>` is *also* Copy.
> So when we did `self.next.map` it was fine because the Option was just copied.
> Now we can't do that, because `&mut` isn't Copy (if you copied an &mut, you'd have two &mut's to the same location in memory, which is forbidden).
> Instead, we should properly `take` the Option to get it.

Ключевым моментом, объясняющим, почему этот код работал, является то, что разделяемые ссылки также копируемые!
А, поскольку `&` можно копировать, `Option<&>` *также* можно копировать.
Так что когда мы вызывали `self.next.map`, это работало потому что что Option просто копировался.
Но теперь мы не можем так сделать, потому что `&mut` не копируется (если вы скопируете &mut, у вам будет две мутабельные сслыки на одно и то же место в памяти, что запрещено).
Вместо копирования, мы должны забрать владение, вызвав метод `take`.


```rust ,ignore
fn next(&mut self) -> Option<Self::Item> {
    self.next.take().map(|node| {
        self.next = node.next.as_deref_mut();
        &mut node.elem
    })
}
```

```text
> cargo build

```

> Uh... wow.
> Holy shit!
> IterMut Just Works!

Так... подождите.
Охренеть!
IterMut просто работает!

> Let's test this:

Давайте протестируем:

```rust ,ignore
#[test]
fn iter_mut() {
    let mut list = List::new();
    list.push(1); list.push(2); list.push(3);

    let mut iter = list.iter_mut();
    assert_eq!(iter.next(), Some(&mut 3));
    assert_eq!(iter.next(), Some(&mut 2));
    assert_eq!(iter.next(), Some(&mut 1));
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 6 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::peek ... ok

test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured

```

> Yep.
> It works.

Да.
Он работает.

> Holy shit.

Охренеть.

> What.

Хотя.

> Ok I mean it actually *is* supposed to work, but there's usually something stupid that gets in the way!
> Let's be clear here:

Ладно, я имею в виду, что IterMut должен работать, но обычно этому мешает какая-нибудь глупость!
Проясним:

> We have just implemented a piece of code that takes a singly-linked list, and returns a mutable reference to every single element in the list at most once.
> And it's statically verified to do that.
> And it's totally safe.
> And we didn't have to do anything wild.

Мы только что реализовали код, который принимает односвязный список и возвращает мутабельные ссылки на каждый отдельный элемент списка максимум один раз.
И он статически проверен.
А также полностью безопасен.
И нам не пришлось писать ничего безумного.

> That's kind of a big deal, if you ask me.
> There are a couple reasons why this works:

Как по мне, это довольно серьёзное дело.
Есть пара причин, поэтому это работает.

> * We `take` the `Option<&mut>` so we have exclusive access to the mutable reference.
>   No need to worry about someone looking at it again.
> * Rust understands that it's ok to shard a mutable reference into the subfields of the pointed-to struct, because there's no way to "go back up", and they're definitely disjoint.

* Мы вызываем `take` у `Option<&mut>`, так что получаем эксклюзивный доступ к мутабельной ссылке.
  Не надо беспокоиться, что кто-то ещё сможет получить к ней доступ.
* Rust понимает, что это нормально — разбить мутабельную ссылку на отдельные поля указуемой структуры, потому что нет способа «собрать её обратно» и поля определённо не пересекаются.

> It turns out that you can apply this basic logic to get a safe IterMut for an array or a tree as well!
> You can even make the iterator DoubleEnded, so that you can consume the iterator from the front *and* the back at once!
> Woah!

Ту же самую базовую логику можно применить для реализации безопасного IterMut для массива или дерева!
Вы можете даже сделать итератор DoubleEnded, так что могли бы перебирать элементы одновременно и от начала к концу, и от конца к началу!
Вот как!

[ownership]: first-ownership.md
