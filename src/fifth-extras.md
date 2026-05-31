> # Extra Junk

# Ещё немного хлама 

> Now that `push` and `pop` are written, everything else is acutally exactly the same as the stack case, weirdly.
> Only operations that change the length of the list need to touch the tail pointer.

Теперь, когда написаны `push` и `pop`, всё остальное на самом деле выглядит точно также, как и со стеком, как ни странно.
Только операции, которые меняют длину списка, нуждаются в изменении указателя на хвост.

> But of course, now that everything's unsafe pointers we need to rewrite the code to use those!
> And if we're going to be touching all the code, we might as well take the chance to make sure we aren't missing something.

Но конечно, сейчас, когда всё у нас везде небезопасные указатели, нам надо перезаписать код, чтобы использовать их!
И если мы хотим переписать весь код, почему бы не воспользоваться случаем и не убедиться, что мы ничего не упустили!

> But anyway, let's start copy-pasting code from the stack implementation:

Но в любом случае, давайте начнём копи-пастить код из реализации стека:

```rust ,ignore
// ...

pub struct IntoIter<T>(List<T>);

pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}
```

> IntoIter looks fine, but `Iter` and `IterMut` are breaking our simple rule of never using safe pointers in our types anymore.
> Let's be safe and change those to use raw pointers:

IntoIter выглядит нормально, но `Iter` и `IterMut` нарушают наше простое правило никогда больше не использовать безопасные указатели в наших типах.
Давайте перестрахуемся и перепишем их на сырые указатели:

```rust ,ignore
pub struct IntoIter<T>(List<T>);

pub struct Iter<'a, T> {
    next: *mut Node<T>,
}

pub struct IterMut<'a, T> {
    next: *mut Node<T>,
}

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }

    pub fn iter(&self) -> Iter<'_, T> {
        Iter { next: self.head }
    }

    pub fn iter_mut(&mut self) -> IterMut<'_, T> {
        IterMut { next: self.head }
    }
}
```

> Looks good!

Выглядит хорошо!

```text
error[E0392]: parameter `'a` is never used
  --> src\fifth.rs:17:17
   |
17 | pub struct Iter<'a, T> {
   |                 ^^ unused parameter
   |
   = help: consider removing `'a`, referring to it in a field, 
     or using a marker such as `PhantomData`

error[E0392]: parameter `'a` is never used
  --> src\fifth.rs:21:20
   |
21 | pub struct IterMut<'a, T> {
   |                    ^^ unused parameter
   |
   = help: consider removing `'a`, referring to it in a field, 
     or using a marker such as `PhantomData`
```

> Doesn't look good!
> What's this [PhantomData](https://doc.rust-lang.org/std/marker/struct.PhantomData.html) they're on about?

Выглядит нехорошо!
Что это за [PhantomData](https://doc.rust-lang.org/std/marker/struct.PhantomData.html), о котором они пишут?

> > Zero-sized type used to mark things that “act like” they own a `T`.
> >
> > Adding a `PhantomData<T>` field to your type tells the compiler that your type acts as though it stores a value of type `T`, even though it doesn’t really.
> > This information is used when computing certain safety properties.
> >
> > For a more in-depth explanation of how to use `PhantomData<T>`, please see [the Nomicon](https://doc.rust-lang.org/nightly/nomicon/).

> Тип нулевого размера используется для маркировки объектов, которые «ведут себя так», словно они владеют `T`.
> 
> Добавление поля `PhantomData<T>` к вашему типу говорит компилятору, что ваш типа действует так, как будто хранит значение типа `T`, хотя на самом деле это не так.
> Эта информация используется при вычислении определённых свойств безопасности.
> 
> Для более глубокого объяснения, как использовать `PhantomData<T>`, читайте, пожалуйста, [Nomicon](https://doc.rust-lang.org/nightly/nomicon/).

> Hey don't get hasty there, we're reading the book that *I* wrote.
> Not that other book that some huge *nerd* probably wrote!
> I bet if they write a data structure in there it's something lame like an Array Stack and *not* a Linked List.

Эй, без спешки, мы говорим про книгу, которая написала *я*.
Не про ту другую книгу, которую, написал какой-то неимоверный *ботан*!
Держу пари, если они и напишут какую-нибудь структуру данных это будет что-то вроде стека на основе массива, а *не* связный список.

> > Unused lifetime parameters
> >
> > Perhaps the most common use case for PhantomData is a struct that has an unused lifetime parameter, typically as part of some unsafe code.

> Неиспользуемые параметры времени жизни
> 
> Возможно наиболее часто PhantomData используют для описания структур с неиспользуемым параметром времени жизни, обычно как часть какого-то небезопасного кода.

> Ah so we're naming a lifetime in our type but not actually using it.
> We *could* go down the PhantomData path, but I want to save that for the doubly-linked list in the next chapter that will *really* need it.

А, таы мы объявили время жизни в нашем типе, но на самом деле его не используем.
Мы *могли бы* пойти по пути PhantomData, но я хочу оставить это для двусвязного списка из следующей главы, где такое решение *на самом деле* пригодится.

> We're in an interesting situation where we actually don't need PhantomData.
> *I think*.
> I'm just going to claim that and trust that it's true, and if miri yells at us at the end I'll concede the point and we'll do the PhantomData thing.

Мы находимся в интересной ситуации, когда нам на самом деле не нужен тип PhantomData.
*Я думаю*.
Я просто собираюсь заявить об этом и верю, что это правда, а если miri обругает наш код, я уступлю и попробую прикрутить PhantomData.

> What we're actually going to do is put the references back in these Iterator types and be happy we get to use references in some places still.
> I think that's sound because there's still a kind of proper nesting when you use an iterator: you create the iterator, use safe references for a while, and then discard the iterator. 

Что мы хотим сделать на самом деле, это вернуть ссылки обратно в эти типы итераторов и радоваться тому, что ссылки всё ещё полезны в некоторых сценариях.
Я думаю, здесь всё правильно, потому что здесь всё ещё присутствует своего рода правильная вложенность: вы создаёте итератор, которое время используете безопасные ссылки, а затем удаляете итератор.

> Only once the iterator is gone can you access the list and call things like `push` and `pop` which need to mess with the tail pointer and Boxes.
> Now, during the iteration we *are* going to be dereferencing a bunch of raw pointers, so there is a kind of mixing there, but we should be able to think of those references as reborrows of the unsafe pointers.

Только после завершения итерации вы можете вернуть доступ к списку и вызывать `push` и `pop`, которые будут взаимодействовать с указателем на хвост и боксами.
А сейчас, во время итерации мы *будем* разыменовывать груду сырых указателей, поэтому здесь будет в каком-то смысле смешивание, но мы должны думать об этих ссылках, как о повторно заимствованных небезопасных указателях.

> *I'm* not even 100% convinced but I just wanna give it a try and see!

*Я* даже не уверена на 100%, но я хочу просто попробовать и проверить!

```rust ,ignore
pub struct IntoIter<T>(List<T>);

pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }

    pub fn iter(&self) -> Iter<'_, T> {
        unsafe {
            Iter { next: self.head.as_ref() }
        }
    }

    pub fn iter_mut(&mut self) -> IterMut<'_, T> {
        unsafe {
            IterMut { next: self.head.as_mut() }
        }
    }
}
```

> If we're going to be storing references, we need to upgrade our raw pointers to options-of-references.
> We *could* check if the pointer is null, but this is one of the incredibly narrow cases where I *think* it's ok to use the nasty [ptr::as_ref](https://doc.rust-lang.org/std/primitive.pointer.html#method.as_ref-1) and [ptr::as_mut](https://doc.rust-lang.org/std/primitive.pointer.html#method.as_mut) methods.

Если мы будем хранить ссылки, нам надо преобразовать сырые указатели в Options для ссылок.
Мы *могли бы* проверять указатели на null, но это один из тех редких случаев, когда я *допускаю* использование неприятнрых методов [ptr::as_ref](https://doc.rust-lang.org/std/primitive.pointer.html#method.as_ref-1) и [ptr::as_mut](https://doc.rust-lang.org/std/primitive.pointer.html#method.as_mut).

> I *usually* recommend avoiding these methods like the plague because they do some surprising and nasty stuff and they're inherently reintroducing references when my whole "easy rule" is to avoid doing that!

*Обычно* я рекомендую избегать этих методов, как чумы, потому что они делают неожиданные и неприятные вещи и по сути повторно вводят ссылки, тогда как моё «простое правило» — не делать этого!

> Those methods come with a lot of warnings, but the most interesting is this:

Эти методы сопровождаются множеством предупреждений, но самое интересное вот это:

> > You must enforce Rust’s aliasing rules, since the returned lifetime `'a` is arbitrarily chosen and does not necessarily reflect the actual lifetime of the data. In particular, for the duration of this lifetime, the memory the pointer points to must not get accessed (read or written) through any other pointer.

> Необходимо соблюдать правила Rust о псевдонимах, поскольку возвращаемое время жизни `'a` выбирается произвольно и не отражает реальное время жизни данных.
В частности, в течение этого времени жизни, доступ к этой памяти (чтение или запись) не должен осуществляться ни через один другой указатель

> Hey look it's the thing we talked about for 25 pages!
> I have already asserted we're *definitely* going to be fine to use references here, so aliasing solved!
> The other evil part is the signature:

Смотрите, это то, о чём мы говорили 25 страниц!
Я уже утверждала, что использование ссылок здесь *определённо* допустима, так что проблема псевдонимов решена!
Но есть и другая проблема, это сигнатура:

```rust ,ignore
pub unsafe fn as_mut<'a>(self) -> Option<&'a mut T>
```

> Do you see how that lifetime isn't attached to the input at all, because `self` is by-value?
> Yeah that's what we call an "unbounded lifetime" and it's nasty stuff.
> It's willing to pretend to be as large as we ask it to be, even `'static`!
> The way you *deal* with that is by putting it somewhere that *is* bounded, which usually just means "return this from a function as soon as possible so that the function signature limits it".

Вы видите, что время жизни вообще не привязано к входному значению, потому что `self` передаётся по значению?
Да, это то, что мы называем «неограниченным временем жизни» и это очень неприятная штука.
Оно готово претворяться настолько длинным, насколько нам нужно, даже «статическим»!

Способ *справиться* с этим — поместить его куда-нибудь, где оно *будет* ограничено, что обычно означает «верните его из функции как можно быстрее, так что теперь сигнатура функции будет его ограничивать».

> Boy I'm nervous about this but we're gonna keep pushing through!
> Let's steal some iterator impls from the stack:

Боже, я нервничаю по этому поводу, но мы продолжим двигаться вперёд!
Давайте позаимствуем несколько реализаций итератора из стека:

```rust ,ignore
impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        self.0.pop()
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        unsafe {
            self.next.map(|node| {
                self.next = node.next.as_ref();
                &node.elem
            })
        }
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        unsafe {
            self.next.take().map(|node| {
                self.next = node.next.as_mut();
                &mut node.elem
            })
        }
    }
}
```

> Moment of truth time...

Время момента истины...

```text
cargo test

running 15 tests
test fifth::test::basics ... ok
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::iter ... ok
test third::test::basics ... ok

test result: ok. 15 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out;
```

```text
MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri test

running 15 tests
test fifth::test::basics ... ok
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 15 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

> YES!!!
> Take that **NARRATOR**!
> Sometimes I don't make mistakes!

ДА!!!
Вот тебе, **ГОЛОС ЗА КАДРОМ**!
Иногда и я не ошибаюсь!

> > **NARRATOR**: but wasn't the whole point that the mistakes are there to teach the reader.

> **ГОЛОС ЗА КАДРОМ:** но разве вся суть не в том, что ошибки нужны, чтобы научить читателя?

> YEAH WELL SOMETIMES THE LESSON IS THAT I'M RIGHT AND EVERYONE SHOULD LISTEN TO ME WHEN I SAY THINGS ABOUT UNSAFE CODE BECAUSE I HAVE SPENT FAR TOO MUCH TIME THINKING ABOUT THE SOUNDNESS OF ITERATOR IMPLEMENTATIONS?!
> OK?!
> OK.

ДА, НО ИНОГДА ВЫВОД В ТОМ, ЧТО Я ПРАВ И ВСЕ ДОЛЖНЫ ПРИСЛУШИВАТЬСЯ КО МНЕ, КОГДА Я ГОВОРЮ ПРО НЕБЕЗОПАСНЫЙ КОД, ПОТОМУ ЧТО Я ПРОВЕЛА МНОГО ВРЕМЕНИ В РАЗМЫШЛЕНИЯХ О НАДЁЖНОСТИ РЕАЛИЗАЦИИ ИТЕРАТОРОВ?
ЛАДЫ?!
ЛАДЫ.

> Anyway here's `peek` and `peek_mut`.

Наконец, вот `peek` и `peek_mut`.

```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    unsafe {
        self.head.as_ref()
    }
}

pub fn peek_mut(&mut self) -> Option<&mut T> {
    unsafe {
        self.head.as_mut()
    }
}
```

> I'm not even gonna test them because I never make mistakes anymore.

Я даже не буду их тестировать, потому что больше я не ошибаюсь.

> > **NARRATOR**: `cargo build`

> **NARRATOR**: `cargo build`


```text
error[E0308]: mismatched types
  --> src\fifth.rs:66:13
   |
25 | impl<T> List<T> {
   |      - this type parameter
...
64 |     pub fn peek(&self) -> Option<&T> {
   |                           ---------- expected `Option<&T>` 
   |                                      because of return type
65 |         unsafe {
66 |             self.head.as_ref()
   |             ^^^^^^^^^^^^^^^^^^ expected type parameter `T`, 
   |                                found struct `fifth::Node`
   |
   = note: expected enum `Option<&T>`
              found enum `Option<&fifth::Node<T>>`

```

> FINE.

ПРЕКРАСНО.

```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    unsafe {
        self.head.as_ref().map(|node| &node.elem)
    }
}

pub fn peek_mut(&mut self) -> Option<&mut T> {
    unsafe {
        self.head.as_mut().map(|node| &mut node.elem)
    }
}
```

> I guess I am going to *continue* to make mistakes, so we're going to be extra careful and add a new test I'm going to call "miri food": something that just messes around and mixes up our APIs a bunch to help miri catch our mistakes.

Полагаю, я всё-таки *буду* время от времени ошибаться, так что мы будем особенно осторожны и добавим новый тест, который я назову «приманка для miri»: что-то, что просто перемешает наши API и поможет miri искать наши ошибки.

```rust ,ignore
#[test]
fn miri_food() {
    let mut list = List::new();

    list.push(1);
    list.push(2);
    list.push(3);

    assert!(list.pop() == Some(1));
    list.push(4);
    assert!(list.pop() == Some(2));
    list.push(5);

    assert!(list.peek() == Some(&3));
    list.push(6);
    list.peek_mut().map(|x| *x *= 10);
    assert!(list.peek() == Some(&30));
    assert!(list.pop() == Some(30));

    for elem in list.iter_mut() {
        *elem *= 100;
    }

    let mut iter = list.iter();
    assert_eq!(iter.next(), Some(&400));
    assert_eq!(iter.next(), Some(&500));
    assert_eq!(iter.next(), Some(&600));
    assert_eq!(iter.next(), None);
    assert_eq!(iter.next(), None);

    assert!(list.pop() == Some(400));
    list.peek_mut().map(|x| *x *= 10);
    assert!(list.peek() == Some(&5000));
    list.push(7);

    // Drop it on the ground and let the dtor exercise itself
    // Ну а здесь пусть запуститься деструктор
}
```


```text
cargo test

running 16 tests
test fifth::test::basics ... ok
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test fifth::test::miri_food ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test fourth::test::peek ... ok
test second::test::into_iter ... ok
test second::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::iter ... ok
test second::test::iter ... ok
test third::test::basics ... ok

test result: ok. 16 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out



MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri test

running 16 tests
test fifth::test::basics ... ok
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test fifth::test::miri_food ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test fourth::test::peek ... ok
test second::test::into_iter ... ok
test second::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::iter ... ok
test second::test::iter ... ok
test third::test::basics ... ok

test result: ok. 16 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

> Perfect.

Идеально.
