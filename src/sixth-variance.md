> # Variance and PhantomData

# Вариантность и PhantomData

> It's going to be annoying to punt on this now and fix it later, so we're going to do the Hardcore Layout stuff now.

Будет обидно откладывать это на потом, поэтому мы прямо сейчас займёмся Хардкорным Представлением.

> There are five terrible horsemen of making unsafe Rust collections:

Есть пять ужасных всадников создания небезопасных коллекций в Rust:

> 1. [Variance](https://doc.rust-lang.org/nightly/nomicon/subtyping.html)
> 2. [Drop Check](https://doc.rust-lang.org/nightly/nomicon/dropck.html)
> 3. [NonNull Optimizations](https://doc.rust-lang.org/nightly/std/ptr/struct.NonNull.html)
> 4. [The isize::MAX Allocation Rule](https://doc.rust-lang.org/nightly/nomicon/vec/vec-alloc.html)
> 5. [Zero-Sized Types](https://doc.rust-lang.org/nightly/nomicon/vec/vec-zsts.html)

1. [Вариантность](https://doc.rust-lang.org/nightly/nomicon/subtyping.html)
2. [Дроп-чек](https://doc.rust-lang.org/nightly/nomicon/dropck.html)
3. [Оптимизации нулевого указателя](https://doc.rust-lang.org/nightly/std/ptr/struct.NonNull.html)
4. [Правило выделения isize::MAX](https://doc.rust-lang.org/nightly/nomicon/vec/vec-alloc.html)
5. [Типы нулевой длины](https://doc.rust-lang.org/nightly/nomicon/vec/vec-zsts.html)

> Mercifully, the last 2 aren't going to be a problem for us. 

Слава небесам, последние два не являются для нас проблемой.

> The third we *could* make into our problem but it's more trouble than it's worth -- if you've opted into a LinkedList you've already given up the battle on memory-effeciency 100-fold already.

Мы *могли бы* сделать проблемой третьего, но овчинка не стоит выделки — если вы выбрали связный список, вы уже стократно проиграли битву за эффективное использование памяти.

> The second is something that I used to insist was really important and that std messes around with, but the defaults are safe, the ways to mess with it are unstable, and you need to try *so very hard* to ever notice the limitations of the defaults, so, don't worry about it.

Второго я когда-то настойчиво считала важным, поскольку стандартная библиотека применяет разные трюки для него <!-- для дроп-чека -->, но настройки по умолчанию безопасны, а способы настройки нестабильны, и вам надо *очень постараться*, чтобы заметить ограничения этих настроек, так что не беспокойтесь об этом.

> That just leaves us with Variance.
> To be honest, you can probably punt on this one too, but I still have my pride as a Collections Person, so we're going to Do The Variance Thing.

Остаётся Вариантность.
Честно говоря, на этом, наверное, тоже можно было не настаивать, но я всё ещё горжусь своей работой в качестве человека, разработавшего std::collections, так что мы всё-таки займёмся Штукой под названием Вариантность.

> So, surprise: Rust has subtyping.
> In particular, `&'big T` is a *subtype* of `&'small T`.
> Why?
> Well because if some code needs a reference that lives for some particular region of the program, it's usually perfectly fine to give it a reference that lives for *longer*.
> Like, intuitively that's just true, right?

Так вот, сюрприз: в Rust есть подтипы.
В частности, `&'big T` — это *подтип* `&'small T`.
<!-- у big больше время жизни, чем у small -->
Почему?
Ну, потому что если какому-то коду нужна ссылка, которая живёт в течение какого-то участка программы, вполне допустимо дать ему ссылку, которая живут *дольше*.
Интуитивно это же очевидно, да?

> Why is this important?
> Well imagine some code that takes two values with the same type:

Почему это важно?
Ну, представьте какой-то код, который получает два значения одного и того же типа:

```rust ,ignore
fn take_two<T>(_val1: T, _val2: T) { }
```

> This is some deeply boring code, and so we should expect it to work with T=&u32 fine, right?

Это глубоко скучный код, поэтому мы должны ожидать, что он будет работать при T=&u32, ведь так?

```rust
fn two_refs<'big: 'small, 'small>(
    big: &'big u32, 
    small: &'small u32,
) {
    take_two(big, small);
}

fn take_two<T>(_val1: T, _val2: T) { }
```

> Yep, that compiles fine!

Да, компилируется без проблем!

> Now let's have some fun and wrap it in, oh, I don't know, `std::cell::Cell`:

А теперь давайте немного повеселимся и завернём его, ну, я не знаю, в `std::cell::Cell`:

```rust ,compilefail
use std::cell::Cell;

fn two_refs<'big: 'small, 'small>(
    // NOTE: these two lines changed
    // ОБРАТИТЕ ВНИМАНИЕ: эти две строки изменились
    big: Cell<&'big u32>, 
    small: Cell<&'small u32>,
) {
    take_two(big, small);
}

fn take_two<T>(_val1: T, _val2: T) { }
```

```text
error[E0623]: lifetime mismatch
 --> src/main.rs:7:19
  |
4 |     big: Cell<&'big u32>, 
  |               ---------
5 |     small: Cell<&'small u32>,
  |                 ----------- these two types are declared with different lifetimes...
6 | ) {
7 |     take_two(big, small);
  |                   ^^^^^ ...but data from `small` flows into `big` here
```

> Huh???
> We didn't touch the lifetimes, why's the compiler angry now!?

Что???
Мы же не трогали время жизни, почему компилятор сейчас ругается?

> Ah well, the lifetime "subtyping" stuff must be really simple, so it falls over if you wrap the references in anything, see look it breaks with Vec too:

Ну что же, штука со временем жизни «подтипов» должна быть очень простой, поэтому она даётся сбой, если завернуть ссылки во что-то ещё, смотрите, она ломается и с Vec:

```rust
fn two_refs<'big: 'small, 'small>(
    big: Vec<&'big u32>, 
    small: Vec<&'small u32>,
) {
    take_two(big, small);
}

fn take_two<T>(_val1: T, _val2: T) { }
```

```text
    Finished dev [unoptimized + debuginfo] target(s) in 1.07s
     Running `target/debug/playground`
```

> See it doesn't compile eith-- wait what???
> Vec is magic??????

Видите, этот код тоже не компилиру... — подождите, что???
Vec что, магический???

> Well, yes.
> But also, no.
> The magic was inside us all along, and that magic is ✨*Variance*✨.

Ну, да.
Но, в то же время, нет.
Эта магия всегда была с нами и магия эта — ✨*Вариантность*✨.

> Read the [nomicon's chapter on subtyping](https://doc.rust-lang.org/nightly/nomicon/subtyping.html) if you want all the gorey details, but basically subtyping *isn't* always safe.
> In particular it's not safe when mutable references are involved because you can use things like `mem::swap` and suddenly oops dangling pointers!

Прочитайте [главу про подтипы в Растономиконе](https://doc.rust-lang.org/nightly/nomicon/subtyping.html), если вам нужны кровавые подробности, но, вкратце: подтипы *не всегда* безопасны.
В частности, небезопасно когда перемешиваются изменяемые ссылки, потому что вы можете вызывать что-то вроде `mem::swap` и внезапно получить висячие указатели!

> Things that are "like mutable references" are *invariant* which means they block subtyping from happening on their generic parameters.
> So for safety, `&mut T` is invariant over T, and `Cell<T>` is invariant over T because `&Cell<T>` is basically just `&mut T` (because of interior mutability).

Штуки, которые выглядят «как изменяемые ссылки» являются *инвариантными*, что означает, что они блокируют подтипы для своих обобщённых параметров.
Поэтому, в целях безопасности, `&mut T` — инвариантен относительно T, и `Cell<T>` — инвариантен относительно T потому что `&Cell<T>` по сути является просто `&mut T` (из-за внутренней изменчивости).

> Almost everything that isn't invariant is *covariant*, and that just means that subtyping "passes through" it and continues to work normally (there are also contravariant types that make subtyping go backwards but they are really rare and no one likes them so I won't mention them again).

Почти всё, что не является инвариантным, *ковариантно* и это значит, что подтипы можно использовать везде, где можно использовать типы и всё будет работать (есть также контравариантные типы, где типы можно использовать вместо подтипов, но они встречаются редко и никому не нравятся, так что я больше не буду про них говорить).

> Collections generally contain a mutable pointer to their data, so you might expect them to be invariant too, but in fact, they don't need to be!
> Because of Rust's ownership system, `Vec<T>` is semantically equivalent to `T`, and that means it's safe for it to be covariant!

Обычно коллекции содержат изменяемый указатель на свои данные, так что вы можете ожидать, что они тоже инвариантные, но на самом деле, им не обязательно быть таковыми!
Из-за системы владения Rust, `Vec<T>` семантически эквивалентен `T` и это значит, что он безопасен для того, чтобы быть ковариантным!

> Unfortunately, this definition is invariant:

К сожалению, это определение инвариантно:

```rust
pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
}

type Link<T> = *mut Node<T>;

struct Node<T> {
    front: Link<T>,
    back: Link<T>,
    elem: T, 
}
```

> But how is Rust actually deciding the variance of things?
> Well in the good-old-days before 1.0 we messed around with just letting people specify the variance they wanted and... it was an absolute train-wreck!
> Subtyping and variance is really hard to wrap your head around, and core developers genuinely disagreed on basic terminology!
> So we moved to a "variance by example" approach: the compiler just looks at your fields and copies their variances.
> If there's any kind of disagreement, then invariance always wins, because that's safe.

Но как на самом деле Rust принимает решение о вариантности штук?
Ну, в старые добрые времена до версии 1.0 мы игрались с тем, чтобы просто позволить людям указывать нужный им вид вариантности... это был полный провал!
Подтипы и вариантность — действительно сложные понятия и разработчики ядра искренне не могли договориться даже о базовой терминологии!
Поэтому мы договорились о «вариантности на примере»: компилятор просто смотрит на ваши поля и копирует их вариантность.
Если возникают какие-нибудь противоречия, всегда побеждает инвариантность, потому что это безопасно.

> So what's in our type definitions that Rust is getting mad about?
> `*mut`!

Так что же такого есть в наших определениях, что так не нравится Rust?
`*mut`!

> Raw pointers in Rust really just try to let you do whatever, but they have exactly one safety feature: because most people have no idea that variance and subtyping are a thing in Rust, and being *incorrectly* covariant would be horribly dangerous, `*mut T` is invariant, because there's a good chance it's being used "as" `&mut T`.

Сырые указатели в Rust на самом деле пытаются позволить вам делать всё, что угодно, но они имеют ровно одну безопасную черту: из-за того, что большинство людей не имеют представления о вариантности и подтипах, и из-за того, что *некорректная* ковариантность может привести к ужасным последствиям, `*mut T` является инвариантным, потому что есть большая вероятность, что он будет использован «как» `&mut T`.

> This is extremely annoying for Exactly Me as a person who has spent a lot of time writing collections in Rust.
> This is why when I made [std::ptr::NonNull](https://doc.rust-lang.org/std/ptr/struct.NonNull.html), I added this little piece of magic:

Это крайне раздражает меня, потратившую много времени на реализую коллекций в Rust.
Именно поэтому, когда я делала [std::ptr::NonNull](https://doc.rust-lang.org/std/ptr/struct.NonNull.html), я добавил этот кусочек магии:

> > Unlike `*mut T`, `NonNull<T>` was chosen to be covariant over `T`.
> > This makes it possible to use `NonNull<T>` when building covariant types, but introduces the risk of unsoundness if used in a type that shouldn’t actually be covariant.

> В отличие от `*mut T`, `NotNull<T>` был выбран, чтобы быть ковариантным относительно `T`.
> Благодаря этому можно использовать `NotNull<T>` для построения ковариантных типов, что повышает риск возникновения ошибок, если используется с типом, который в действительности не должен быть ковариантным.

> But hey, it's interface is built around `*mut T`, what's the deal!
> Is it just magic?!
> Let's look:

Но, эй, его интерфейс построен вокруг `*mut T`, почему всё работает?
Опять магия?
Давайте взглянем:

```rust
pub struct NonNull<T> {
    pointer: *const T,
}


impl<T> NonNull<T> {
    pub unsafe fn new_unchecked(ptr: *mut T) -> Self {
        // SAFETY: the caller must guarantee that `ptr` is non-null.
        // БЕЗОПАСНОСТЬ: вызывающая сторона должна гарантировать, что `ptr` не равен null.
        unsafe { NonNull { pointer: ptr as *const T } }
    }
}
```

> NOPE.
> NO MAGIC HERE!
> NonNull just abuses the fact that `*const T` is covariant and stores that instead, casting back and forth between `*mut T` at the API boundary to make it "look like" it's storing a `*mut T`.
> That's the whole trick!
< That's how collections in Rust are covariant!
> And it's miserable!
> So I made the Good Pointer Type do it for you!
> You're welcome!
> Enjoy your subtyping footgun!

НЕТ.
НИКАКОЙ МАГИИ!
NonNull просто злоупотребляет тем, что что `*const T` ковариантен и хранит его вместо `*mut T`, преобразуя значения туда и обратно на границе API, чтобы создать впечатление, будто он хранит `*mut T`.
Весь трюк в этом!
Вот почему коллекции в Rust ковариантны.
И это ужасно!
Поэтому я заставил Хороший Тип Указателя делать всю грязную работу!
Пожалуйста!
Наслаждайтесь своими подтипами!

> The solution to all your problems is to use NonNull, and then if you want to have nullable pointers again, use `Option<NonNull<T>>`.
> Are we really going to bother doing that..?

Решение для всех ваших программ в том, чтобы использовать NonNull, а если вам потребуются нулевые указатели, использовать `Option<NonNull<T>>`.
Мы действительно будем этим заниматься?

> Yep!
> It sucks, but we're making *production grade linked lists* so we're going to eat all our vegetables and do things the hard way (we could just use bare `*const T` and cast everywhere, but I genuinely want to see how painful this is... for Ergonomics Science).

Ага!
Это хреново, но мы пишем *связные списки продуктового уровня*, поэтому будем героически разбираться в деталях и писать сложный код (мы могли бы просто использовать `*const T` и везде приводить типы, но я искренне хочу узнать, насколько это больно... в Интересах Науки).

> So here's our final type definitions:

Так что вот наше окончательное определение типа:

```rust
use std::ptr::NonNull;

// !!!This changed!!!
// !!!Изменилось!!!
pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
}

type Link<T> = Option<NonNull<Node<T>>>;

struct Node<T> {
    front: Link<T>,
    back: Link<T>,
    elem: T, 
}
```

> ...wait nope, one last thing.
> Any time you do raw pointer stuff, you should add a Ghost to protect your pointers:

...подождите, нет, одна последняя деталь.
Всякий раз, используя сырые указатели, вы должны добавить Привидение, чтобы защитить указатели:

```rust ,ignore
use std::marker::PhantomData;

pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    /// We semantically store values of T by-value.
    /// Мы семантически храним значения типа T по значению.
    _boo: PhantomData<T>,
}
```

> In this case I don't think we *actually* need [PhantomData](https://doc.rust-lang.org/std/marker/struct.PhantomData.html), but any time you *do* use NonNull (or just raw pointers in general), you should always add it to be safe and make it clear to the compiler and others what you *think* you're doing.

В этом случае я не думаю, что нам действительно нужен [PhantomData](https://doc.rust-lang.org/std/marker/struct.PhantomData.html), но всякий раз, когда вы *используете* NonNull (или просто в целом сырые указатели), вы должны всегда добавлять его для безопасности, чтобы компилятору и всем другим было понятно, что вып *подумали* над тем, что делаете.

> PhantomData is a way for us to give the compiler an extra "example" field that *conceptually* exists in your type but for various reasons (indirection, type erasure, ...) doesn't.
> In this case we're using NonNull because we're claiming our type behaves "as if" it stored a value T, so we add a PhantomData to make that explicit.

PhantomData это способ предоставить компилятору «пример» поля, которое *концептуально* присутствует в вашем типе, но <!-- *физически* --> по разным причинам (косвенный доступ, стирание типов...) — нет.
В данном случае мы используем NonNull, поэтому можем утверждать, что наш тип ведёт себя так, как будто хранит значения T, и мы добавляем PhantomData, чтобы явно это выразить.

> The stdlib actually has other reasons to do this because it has access to the accursed [Drop Check overrides](https://doc.rust-lang.org/nightly/nomicon/dropck.html), but that feature has been reworked so many times that I don't actually know if the PhantomData thing *is* a thing for it anymore.
> I'm still going to cargo-cult it for all eternity, because Drop Check Magic is burned into my brain!

Раньше в стандартной библиотеке были причины массово использовать PhantomData из-за [сложных и опасных трюков в деструкторах](https://doc.rust-lang.org/nightly/nomicon/dropck.html), но эти правила менялись столько раз, что я уже не уверена в необходимости PhantomData.
Впрочем, за годы я уже выработала привычку, так что буду следовать карго-культу и ставить PhantomData, даже если он по существующим правилам не обязателен!

> (Node literally stores a T, so it doesn't have to do this, yay!)

(Кстати, узел на самом деле хранит значение типа T, так что маркер для LinkedList не обязателен!)

> ...ok for real we're done with layout now!
> On to actual basic functionality!

...ладно, с представлением мы закончили!
Переходим к основным функциям!