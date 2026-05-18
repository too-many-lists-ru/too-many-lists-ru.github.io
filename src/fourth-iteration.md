# Итерация

Попробуем написать итераторы для нашего неудачливого списка.

## IntoIter

IntoInter, как всегда — самый простой.
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

Но у нас появилось новое интересное усовершенствование.
Раньше у нас был только один «натуральный» порядок перебора элементов, а дек по своей природе двунаправленный.
Что особенного в переборе от начала к концу?
Что если кому-то понадобится итерация в другом направлении?

На самом деле в Rust есть ответ на этот вопрос: `DoubleEndedIterator`.
DoubleEndedIterator *наследует* Iterator (это значит, что все объекты, реализующие DoubleEndedIterator реализуют так же и Iterator) и добавляет один новый метод: `next_back`.
У него такая же сигнатура, как и у `next`, но он перебирает элементы с другого конца.
Семантика DoubleEndedIterator очень удобна для нашего случая: итератор становится деком.
Мы можем брать элементы как с начала, так и с конца, пока два конца не встретятся и в этой точке итератор станет пустым.

Нам нечасто приходится вызывать `next` вручную.
То же самое и с `next_back` — этот метод не так важен пользователям DoubleEndedIterator.
Лучшая часть интерфейса — это метод `rev`, который заворачивает итератор, чтобы создать другой итератор, который возвращает элементы в обратном порядке.
Семантика здесь довольно проста: вызов `next` у обращённого итератора — это на самом деле вызов `next_back`.

В любом случае, поскольку у нас уже есть дек, предоставить новый API довольно просто:

```rust ,ignore
impl<T> DoubleEndedIterator for IntoIter<T> {
    fn next_back(&mut self) -> Option<T> {
        self.0.pop_back()
    }
}
```

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

Прекрасно.

## Iter

Iter не будет таким снисходительным.
Нам снова придётся иметь дело с этими ужасными `Ref`!
Из-за Ref мы не сможем хранить `&Node`, как мы делали раньше.
Вместо этого попробуем хранить `Ref<Node>`.

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

Пока всё хорошо.
Реализация `next` окажется чуть сложнее, но я думаю, это будет та же самая базовая логика, как и в старом IterMut для стека, с некоторыми дополнительными приседаниями из-за RefCell:

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

Ну, понеслась!

`node_ref` живёт недостаточно долго.
В отличие от обычных ссылок, Rust не позволяет нам повторно использовать Ref подобным образом.
Ref, который мы получили из `head.borrow()`, существует до тех пор, пока существует `node_ref`, который уничтожается при вызове `Ref::map`.

Функция, которая нам нужна, существует, и она называется [*map_split*](https://doc.rust-lang.org/std/cell/struct.Ref.html#method.map_split):

```rust ,ignore
pub fn map_split<U, V, F>(orig: Ref<'b, T>, f: F) -> (Ref<'b, U>, Ref<'b, V>) where
    F: FnOnce(&T) -> (&U, &V),
    U: ?Sized,
    V: ?Sized,
```

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

Рррр.
Нам снова нужен `Ref::map` чтобы у нас было правильное время жизни.
Но `Ref::map` возвращает `Ref`, а нам нужен `Option<Ref>`, но для этого нам надо пройти сквозь Ref, чтобы вызвать map для нашего Option...

**долго смотрит вдаль**

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

Ну да.
Правильно.
У нас несколько RefCell.
Чем дальше мы углубляемся в список, тем более вложенными становятся RefCell.
Нам потребуется поддерживать что-то вроде стека Ref для представления всех объектов, на которые мы ссылаемся, потому что если мы перестаём наблюдать за элементом, нам надо снизить на единицу счётчик заимствований у каждого RefCell, который находится перед ним...

Кажется, здесь уже ничего не сделаешь.
Это тупик.
Давайте выбираться из всех этих RefCell.

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

Э-э-э...
Подождите, а что мы теперь возвращаем?
`&T`?
`Ref<T>`?

Ни один из них не работает... у нашего Iter в любом случае нет времени жизни!
И `&T`, и `Ref<T>` требуют объявления какого-то времени жизни перед тем, как мы сможем вызвать `next`.
Но всё, что мы сможем получить из нашего Rc, будет заимствованием Iterator... мозг... кипит... аааааааааааа

Может быть, мы можем... вызвать map... для Rc... чтобы получить `Rc<T>`?
Так вообще можно?
В доках Rc, похоже, ничего подобного нет.
На самом деле, кто-то сделал [крейт](https://crates.io/crates/owning_ref), который это умеет.

Но, подождите, даже если у нас *это* получится, мы получим ещё большую проблему: ужасную угрозу инвалидации итератора.
Раньше мы не сталкивались с инвалидацией итератора, потому что Iter заимствовал список, оставляя его, в целом, неизменяемым.
Однако, если бы наш итератор возвращал Rc, ему вообще бы не пришлось заимствовать список!
Это значит, что люди могут начать вызывать `push` и `pop`, пока у них есть указатель на список!

О, боже, во что всё это выльется?

Ну, в принципе, с `push` всё в порядке.
Мы видим какую-то часть списка и список будет расти за пределами поля нашего зрения.
Ничего страшного.

Однако, с `pop` ситуация другая.
Если элементы извлекаются за пределами поля нашего зрения, это *всё ещё* должно быть нормально.
Мы не можем видеть эти узлы, так что ничего случиться не может.
В то же время если попытаться извлечь узел, на который мы указываем... всё взлетит на воздух!
В частности, вызов `unwrap` у результата, полученного с помощью `try_unwrap`, на самом деле приводит к панике и аварийному завершению программы.

А это ведь здорово.
Мы можем добавлять в список тонны указателей с внутренним владением и одновременно менять их *и это будет просто работать*, если не удалять узла, на который указывает итератор.

И даже тогда у нас не возникнет висячих указателей или чего-то похожего: программа детерминировано запаникует!

Но необходимость иметь дело с инвалидацией итераторов в дополнение к вызову `map` у `Rc` кажется просто... плохой.
`Rc<RefCell>` окончательно нас подвёл.
Интересно, мы столкнулись с инверсией случая с устойчивым стеком.
В то время как устойчивый стек не мог справиться с владением, но мог днями напролёт предоставлять ссылки, наш список не испытывает проблем с владением, не не может справиться со ссылками.

Впрочем, если быть честными, большая часть нашей борьбы вращалась вокруг желания спрятать детали реализации и иметь пристойный API.
Мы *могли бы* сделать всё хорошо, если бы просто передавали везде узлы.

Блин, да мы могли бы сделать несколько конкурентных IterMuts, которые проверяли бы наличие изменяемого доступа к одному и тому же элементу прямо во время работы программы!

Правда, такой дизайн больше подходит для внутренней структуры данных, которая никогда не попадает к пользователям API.
Внутренняя изменчивость отлично подходит для написания безопасных *приложений*.
Но не для *библиотек*.

В любом случае, я отказываюсь от реализации Iter и IterMut.
Можно было бы, но... получается некрасиво.
