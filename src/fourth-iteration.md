> # Iteration

# Итерация

> Let's take a crack at iterating this bad-boy.

Давайте попробуем написать итераторы для этого плохого парня.

> ## IntoIter

## IntoIter

> IntoIter, as always, is going to be the easiest.
> Just wrap the stack and call `pop`:

IntoInter, как всегда, будет самым простым.
Просто завернём стек в новый тип и вызовем `pop`:

```rust ,ignore
pub struct IntoIter<T>(List<T>);

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        self.0.pop_front()
    }
}
```

> But we have an interesting new development.
> Where previously there was only ever one "natural" iteration order for our lists, a Deque is inherently bi-directional.
> What's so special about front-to-back?
> What if someone wants to iterate in the other direction?

Но у нас появилось новое интересное усовершенствование.
В то время как раньше был только один «натуральный» порядок перебора для нашех списков, дек по своей природе двунаправленный.
Что особенного в итерации от начала к коцу?
Что если кому-то понадобится итерация в другом направлении?

> Rust actually has an answer to this: `DoubleEndedIterator`.
> DoubleEndedIterator *inherits* from Iterator (meaning all DoubleEndedIterator are Iterators) and requires one new method: `next_back`.
> It has the exact same signature as `next`, but it's supposed to yield elements from the other end.
> The semantics of DoubleEndedIterator are super convenient for us: the iterator becomes a deque.
> You can consume elements from the front and back until the two ends converge, at which point the iterator is empty.

На самом деле в Rust есть ответ на этот вопрос: `DoubleEndedIterator`.
DoubleEndedIterator *наследует* Iterator (это значит, что все объекты, реализующие DoubleEndedIterator реализуют так же и Iterator) и добавляет один новый метод: `next_back`.
У него такая же сигнатура, как и у `next`, но он перебирает элементы с другого конца.
Семантика DoubleEndedIterator очень удобная для нас: итератор становится деком.
Вы можете брать элементы как с начала, так и с конца, пока два конца не встретятся и в этой точке итератор станет пустым.

> Much like Iterator and `next`, it turns out that `next_back` isn't really something consumers of the DoubleEndedIterator really care about.
> Rather, the best part of this interface is that it exposes the `rev` method, which wraps up the iterator to make a new one that yields the elements in reverse order.
> The semantics of this are fairly straight-forward: calls to `next` on the reversed iterator are just calls to `next_back`.

Как и в случае с Iterator и `next`, оказывается, что `next_back` — это не то, что на самом деле важно пользователям DoubleEndedIterator.
Скорее, лучшая часть интерфейса — предоставление метода `rev`, которые заворачивает итератор для того, чтобы сделать другой итератор, который возвращает элементы в обратном порядке.
Семантика здесь довольно проста: вызов `next` у обращённого итератора — это на самом деле вызов `next_back`.

> Anyway, because we're already a deque providing this API is pretty easy:

В любом случае, поскольку у нас уже есть дек, предоставить новый API довольно просто:

```rust ,ignore
impl<T> DoubleEndedIterator for IntoIter<T> {
    fn next_back(&mut self) -> Option<T> {
        self.0.pop_back()
    }
}
```

> And let's test it out:

И давайте протестируем:

```rust ,ignore
#[test]
fn into_iter() {
    let mut list = List::new();
    list.push_front(1); list.push_front(2); list.push_front(3);

    let mut iter = list.into_iter();
    assert_eq!(iter.next(), Some(3));
    assert_eq!(iter.next_back(), Some(1));
    assert_eq!(iter.next(), Some(2));
    assert_eq!(iter.next_back(), None);
    assert_eq!(iter.next(), None);
}
```


```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 11 tests
test fourth::test::basics ... ok
test fourth::test::peek ... ok
test fourth::test::into_iter ... ok
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test third::test::iter ... ok
test third::test::basics ... ok
test second::test::into_iter ... ok
test second::test::peek ... ok

test result: ok. 11 passed; 0 failed; 0 ignored; 0 measured

```

> Nice.

Прекрасно.

> ## Iter

## Iter

> Iter will be a bit less forgiving.
> We'll have to deal with those awful `Ref` things again!
> Because of Refs, we can't store `&Node`s like we did before.
> Instead, let's try to store `Ref<Node>`s:

Iter не будет таким снисходительным.
Нам снова придётся иметь дело с этими ужасными `Ref`!
Из-за Ref мы не сможем хранить `&Node`, как мы делали раньше.
Вместо этого давайте попробуем хранить `Ref<Node>`.

```rust ,ignore
pub struct Iter<'a, T>(Option<Ref<'a, Node<T>>>);

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter(self.head.as_ref().map(|head| head.borrow()))
    }
}
```

```text
> cargo build

```

> So far so good.
> Implementing `next` is going to be a bit hairy, but I think it's the same basic logic as the old stack IterMut but with extra RefCell madness:

Пока всё хорошо.
Реализация `next` окажется чуть сложнее, но я думаю, это та же самая базовая логика, как и в старом IterMut для стека, с некоторыми дополнительными сложностями RefCell:

```rust ,ignore
impl<'a, T> Iterator for Iter<'a, T> {
    type Item = Ref<'a, T>;
    fn next(&mut self) -> Option<Self::Item> {
        self.0.take().map(|node_ref| {
            self.0 = node_ref.next.as_ref().map(|head| head.borrow());
            Ref::map(node_ref, |node| &node.elem)
        })
    }
}
```

```text
cargo build

error[E0521]: borrowed data escapes outside of closure
   --> src/fourth.rs:155:13
    |
153 |     fn next(&mut self) -> Option<Self::Item> {
    |             --------- `self` is declared here, outside of the closure body
154 |         self.0.take().map(|node_ref| {
155 |             self.0 = node_ref.next.as_ref().map(|head| head.borrow());
    |             ^^^^^^   -------- borrow is only valid in the closure body
    |             |
    |             reference to `node_ref` escapes the closure body here

error[E0505]: cannot move out of `node_ref` because it is borrowed
   --> src/fourth.rs:156:22
    |
153 |     fn next(&mut self) -> Option<Self::Item> {
    |             --------- lifetime `'1` appears in the type of `self`
154 |         self.0.take().map(|node_ref| {
155 |             self.0 = node_ref.next.as_ref().map(|head| head.borrow());
    |             ------   -------- borrow of `node_ref` occurs here
    |             |
    |             assignment requires that `node_ref` is borrowed for `'1`
156 |             Ref::map(node_ref, |node| &node.elem)
    |                      ^^^^^^^^ move out of `node_ref` occurs here
```

> Shoot.

Началось! <!-- вариант Пристретие меня, кто нибудь. >

> `node_ref` doesn't live long enough.
> Unlike normal references, Rust doesn't let us just split Refs up like that.
> The Ref we get out of `head.borrow()` is only allowed to live as long as `node_ref`, but we end up trashing that in our `Ref::map` call.

`node_ref` живёт недостаточно долго.
В отличие от нормальных ссылок, Rust не позвоялет нам просто разделить Ref подобным образом.
Ref, который мы получили из `head.borrow()`, может существовать только до тех пор, пока существует `node_ref`, но в итоге уничтожаем её при вызове `Ref::map`.

> The function we want exists, and it's called *[map_split][]*:

Функциая, которая нам нужна, существует, и она называется [*map_split*][]:

```rust ,ignore
pub fn map_split<U, V, F>(orig: Ref<'b, T>, f: F) -> (Ref<'b, U>, Ref<'b, V>) where
    F: FnOnce(&T) -> (&U, &V),
    U: ?Sized,
    V: ?Sized,
```

> Woof.
> Let's give it a try...

Уф.
Давайте попробуем.

```rust ,ignore
fn next(&mut self) -> Option<Self::Item> {
    self.0.take().map(|node_ref| {
        let (next, elem) = Ref::map_split(node_ref, |node| {
            (&node.next, &node.elem)
        });

        self.0 = next.as_ref().map(|head| head.borrow());

        elem
    })
}
```

```text
cargo build
   Compiling lists v0.1.0 (/Users/ADesires/dev/temp/lists)
error[E0521]: borrowed data escapes outside of closure
   --> src/fourth.rs:159:13
    |
153 |     fn next(&mut self) -> Option<Self::Item> {
    |             --------- `self` is declared here, outside of the closure body
...
159 |             self.0 = next.as_ref().map(|head| head.borrow());
    |             ^^^^^^   ---- borrow is only valid in the closure body
    |             |
    |             reference to `next` escapes the closure body here
```

> Ergh.
> We need to `Ref::Map` again to get our lifetimes right.
> But `Ref::Map` returns a `Ref` and we need an `Option<Ref>`, but we need to go through the Ref to map over our Option...

Рррр.
Нам снова нужен `Ref::map` чтобы у нас было правильное время жизни.
Но `Ref::map` возвращает `Ref`, а нам нужен `Option<ref>`, но для этого нам надо пройти сквозь Ref, чтобы вызвать map для нашего Option...

> **stares into distance for a long time**

**долго смотрит вдаль**

> ??????

??????

```rust ,ignore
fn next(&mut self) -> Option<Self::Item> {
    self.0.take().map(|node_ref| {
        let (next, elem) = Ref::map_split(node_ref, |node| {
            (&node.next, &node.elem)
        });

        self.0 = if next.is_some() {
            Some(Ref::map(next, |next| &**next.as_ref().unwrap()))
        } else {
            None
        };

        elem
    })
}
```

```text
error[E0308]: mismatched types
   --> src/fourth.rs:162:22
    |
162 |                 Some(Ref::map(next, |next| &**next.as_ref().unwrap()))
    |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `fourth::Node`, found struct `std::cell::RefCell`
    |
    = note: expected type `std::cell::Ref<'_, fourth::Node<_>>`
               found type `std::cell::Ref<'_, std::cell::RefCell<fourth::Node<_>>>`
```

> Oh.
> Right.
> There's multiple RefCells.
> The deeper we walk into the list, the more nested we become under each RefCell.
> We would need to maintain, like, a stack of Refs to represent all the outstanding loans we're holding, because if we stop looking at an element we need to decrement the borrow-count on every RefCell that comes before it.................

О.
Правильно.
Есть несколько RefCell.
Чем дальше мы углубляемся в список, тем более вложенными становяся RefCell.
Нам потребуется поддерживать что-то вроде стека Ref для представления всех объектов, на которые мы ссылаемся, потому что если мы перестаём наблидать за элементом, нам надо снизить на единицу счётчкик заимствований у кажого RefCell, который находится перед ним...

> I don't think there's anything we can do here.
> It's a dead end.
> Let's try getting out of the RefCells.

Кажется, мы ничего не можем здесь сделать.
Это тупик.
Давайте выбираться из всех этих RefCell.

> What about our `Rc`s.
> Who said we even needed to store references?
> Why can't we just Clone the whole Rc to get a nice owning handle into the middle of the list?

Что насчёт наших `Rc`?
Кто вообще сказал, что нам нужно хранить ссылки?
Почему мы не можем просто вызвать Clone у Rc, чтобы получить удобный указатель в середине списка?

```rust ,ignore
pub struct Iter<T>(Option<Rc<Node<T>>>);

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter(self.head.as_ref().map(|head| head.clone()))
    }
}

impl<T> Iterator for Iter<T> {
    type Item =
```

> Uh...
> Wait what do we return now?
> `&T`?
> `Ref<T>`?

Э-э-э...
Подождите, а что мы теперь возвращаем?
`&T`?
`Ref<T>`?

> No, none of those work... our Iter doesn't have a lifetime anymore!
> Both `&T` and `Ref<T>` require us to declare some lifetime up front before we get into `next`.
> But anything we manage to get out of our Rc would be borrowing the Iterator... brain... hurt... aaaaaahhhhhh

Нет, ни один из них не работает... у нашего Iter в любом случае нет времени жизни!
И `&T`, и `Ref<T>` требуют объявления какого-то времени жизни перед тем, как мы сможем вызвать `next`.
Но всё, что мы сможем получить из нашего Rc будет заимстованием Iterator... мозг... болит... аааааааааааа!

> Maybe we can... map... the Rc... to get an `Rc<T>`?
> Is that a thing?
> Rc's docs don't seem to have anything like that.
> Actually someone made [a crate][own-ref] that lets you do that.

Может быть, мы можем... вызвать map... для Rc... чтобы получить `Rc<T>`?
Так вообще бывает?
В доках Rc, похоже, ничего подобного нет.
На самом деле, кто сделал [крейт][own-ref], который позволяет это сделать.

> But wait, even if we do *that* then we've got an even bigger problem: the dreaded spectre of iterator invalidation.
> Previously we've been totally immune to iterator invalidation, because the Iter borrowed the list, leaving it totally immutable.
> However if our Iter was yielding Rcs, they wouldn't borrow the list at all!
> That means people can start calling `push` and `pop` on the list while they hold pointers into it!

Но похождите, даже если мы *это* сделаем, у нас появится ещё большая проблема: ужасная угроза инвалидации итератора.
Раньше мы не сталкивались с инвалидацией итератора, потому что Iter заимствовал список, оставляя его в целом неизменяемым.
Однако, если бы наш итератор возвращал Rc, ему вообще бы не пришлось заимствовать список!
Это значит, что люди могут начать вызывать `push` и `pop` у списка, пока они хранят указатели на него.

> Oh lord, what will that do?!

О, боже, что из этого получится?

> Well, pushing is actually fine.
> We've got a view into some sub-range of the list, and the list will just grow beyond our sights.
> No biggie.

Ну, в принципе, с `push` всё в порядке.
Мы видим какую-то часть списка и список будет расти за пределами поля нашего зрения.
Ничего страшного.

> However `pop` is another story.
> If they're popping elements outside of our range, it should *still* be fine.
> We can't see those nodes so nothing will happen.
> However if they try to pop off the node we're pointing at... everything will blow up!
> In particular when they go to `unwrap` the result of the `try_unwrap`, it will actually fail, and the whole program will panic.

Однако, с `pop` ситуация другая.
Если элементы извлекаются за пределами поля нашего зрения, это *всё ещё* должно быть нормально.
Мы не можем видеть эти узлы, так что ничего случиться не может.
В то же время если попытаться извлечь узел, на который мы указываем... всё взорвётся! <!-- судя по всему, можно образнее: всё взлетит на воздух -->
В частности, если вызвать `unwrap` у результата, полученного с помощью `try_unwrap`, это на самом деле приведёт к панике и программа аварийно завершится.

> That's actually pretty cool.
> We can get tons of interior owning pointers into the list and mutate it at the same time *and it will just work* until they try to remove the nodes that we're pointing at.
> And even then we don't get dangling pointers or anything, the program will deterministically panic!

Это действительно здорово.
Мы можем добавлять в список тонны указателей с внутренним владением и одновременно менять их *и это будет просто работать*, если не удалять узла, на которые мы <!-- указывает итератор --> указываем.

И даже тогда у нас не возникнет висячих указателей или чего-то похожего: программа детерминированно запаникует!

> But having to deal with iterator invalidation on top of mapping Rcs just seems... bad.
> `Rc<RefCell>` has really truly finally failed us.
> Interestingly, we've experienced an inversion of the persistent stack case.
> Where the persistent stack struggled to ever reclaim ownership of the data but could get references all day every day, our list had no problem gaining ownership, but really struggled to loan our references.

Но необходимость иметь дело с инвалидацией итераторов в дополнение к вызовам `map` у `Rc` кажется просто... плохой.
`Rc<RefCell>` окончательно подвёл нас.
Интересно, мы столкнулись с инверсией случая с устойчивым стеком.
В то время как устойчивый стек не мог справиться с владением, но мог предоставлять ссылки днями напролёт, нас список не испытвает проблем с владением, не не может справиться с нашими ссылками.

> Although to be fair, most of our struggles revolved around wanting to hide the implementation details and have a decent API.
> We *could* do everything fine if we wanted to just pass around Nodes all over the place.

Хотя, если быть честными, большая часть нашей борьбы вращалась вокруг желания спрятать детали реализации и иметь пристойный API.
Мы *могли бы* сделать всё хорошо, если бы просто передавали везде узлы.

> Heck, we could make multiple concurrent IterMuts that were runtime checked to not be mutable accessing the same element!

Блин, да мы могли бы сделать несколько конкурентных IterMuts, которые бы во время выполнения программы проверяли бы, что нет изменяемого доступа к одному и тому же элементу!

> Really, this design is more appropriate for an internal data structure that never makes it out to consumers of the API.
> Interior mutability is great for writing safe *applications*.
> Not so much safe *libraries*.

На самом деле, такой дизайн больше подходит для внутренней структуры данных, которая никогда не попадает к пользователям API.
Внутренняя изменчивость отлично подходит для написания безопасных *приложений*.
Но не для безопасных *библиотек*.

> Anyway, that's me giving up on Iter and IterMut.
> We could do them, but *ugh*.

В любом случае, я отказываюсь от реализации Iter и IterMut.
Мы могли бы, но... получается некрасиво.

[own-ref]: https://crates.io/crates/owning_ref
[map-split]: https://doc.rust-lang.org/std/cell/struct.Ref.html#method.map_split
