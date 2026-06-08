> # Layout

# Представление

> So what's a singly-linked queue like?
> Well, when we had a singly-linked stack we pushed onto one end of the list, and then popped off the same end.
> The only difference between a stack and a queue is that a queue pops off the *other* end.
> So from our stack implementation we have:

Так на что похожа односвязная очередь?
Ну, когда у нас был односвязный стек, мы вставляли элементы с одной стороны списка, а затем удаляли с той же самой стороны.
Единственное отличие между стеком и очередью в том, что в очереди элементы удаляются с *другой* стороны.
Поэтому из нашей реализации стека мы получаем:

```text
список на входе:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)

вставка X:
[Some(ptr)] -> (X, Some(ptr)) -> (A, Some(ptr)) -> (B, None)

удаление:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)
```

> To make a queue, we just need to decide which operation to move to the end of the list: push, or pop?
> Since our list is singly-linked, we can actually move *either* operation to the end with the same amount of effort.

Чтобы сделать очередь нам достаточно решить, какую операцию перенести на другой конец списка: вставку или удаление?
Поскольку у нас односвязный список, мы на самом деле можем перекинуть на другую сторону *любую* операцию с одинаковыми усилиями.

> To move `push` to the end, we just walk all the way to the `None` and set it to Some with the new element.

Чтобы перенести в конец вставку, мы должны пробежаться по списку до `None` и заменить его на `Some` с новым элементом.

```text
список на входе:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)

вставка X в конец:
[Some(ptr)] -> (A, Some(ptr)) -> (B, Some(ptr)) -> (X, None)
```

> To move `pop` to the end, we just walk all the way to the node *before* the None, and `take` it:

Чтобы перенести в конец удаление, мы должны пробежаться по списку до узла *перед* `None` и изъять узел.

```text
список на входе:
[Some(ptr)] -> (A, Some(ptr)) -> (B, Some(ptr)) -> (X, None)

удаление с конца:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)
```

> We could do this today and call it quits, but that would stink!
> Both of these operations walk over the *entire* list.
> Some would argue that such a queue implementation is indeed a queue because it exposes the right interface.
> However I believe that performance guarantees are part of the interface.
> I don't care about precise asymptotic bounds, just "fast" vs "slow".
> Queues guarantee that push and pop are fast, and walking over the whole list is definitely *not* fast.

Мы могли бы сделать задачу прямо сейчас и назвать это решением, но это было бы ужасно!
Обе эти операции пробегаются по *всему* списку.
Некоторые могли бы возразить, что такая реализация вполне является нормальна, поскольку полностью соответствует интерфейсу.
Однако, я уверена, что гарантии производительности также являются частью интерфейса.
Меня не волнуют точные асимптотические границы, просто «быстро» или «медленно».
Гарантии очереди в том, что вставка и удаление быстрые, а обход всего списка — определённо *не* быстрый.

> One key observation is that we're wasting a ton of work doing *the same thing* over and over.
> Can we "cache" all that work and reuse it?
> Why, yes!
> We can store a pointer to the end of the list, and just jump straight to there!

Одно ключевое наблюдение в том, что мы делаем гору работы, повторяя *одну и ту же вещь* снова и снова.
Можем ли мы «кешировать» всю эту работу и переиспользовать её?
Да, разумеется!
Мы можем сохранить указатель на конец списка и просто туда перепрыгивать!

> It turns out that only one inversion of `push` and `pop` works with this.
> To invert `pop` we would have to move the "tail" pointer backwards, but because our list is singly-linked, we can't do that efficiently.
> If we instead invert `push` we only have to move the "head" pointer forward, which is easy.

Оказывается, что с этой идеей работает только один из вариантов инверсии вставки и удаления.
Чтобы инвертировать `pop`, мы должны переместить указатель на хвост назад, но поскольку наш список односвязный, мы не можем сделать это эффективно.
Если вместо этого мы инвертируем `push`, нам надо будет перемещать вперёд указатель на голову, что легко.

> Let's try that:

Давайте попробуем:

```rust ,ignore
use std::mem;

pub struct List<T> {
    head: Link<T>,
    tail: Link<T>, // НОВАЯ СТРОКА!
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_tail = Box::new(Node {
            elem: elem,
            // When you push onto the tail, your next is always None
            // Когда вставляете в конец списка, следующий узел всегда None
            next: None,
        });

        // swap the old tail to point to the new tail
        // теперь указатель на хвост указывает на новый хвост
        let old_tail = mem::replace(&mut self.tail, Some(new_tail));

        match old_tail {
            Some(mut old_tail) => {
                // If the old tail existed, update it to point to the new tail
                // Если старый хвост существовал, обновляем его, чтобы он указывал на новый хвост
                old_tail.next = Some(new_tail);
            }
            None => {
                // Otherwise, update the head to point to it
                // В противном случае, обновляем голову, чтобы указывала на него
                self.head = Some(new_tail);
            }
        }
    }
}
```

> I'm going a bit faster with the impl details now since we should be pretty comfortable with this sort of thing.
> Not that you should necessarily expect to produce this code on the first try.
> I'm just skipping over some of the trial-and-error we've had to deal with before.
> I actually made a ton of mistakes writing this code that I'm not showing, but you can only see me leave off a `mut` or `;` so many times before it stops being instructive.
> Don't worry, we'll see plenty of *other* error messages!

Я собираюсь быть быстрее в реализации деталей, поскольку к этому моменту мы уже должны были привыкнуть к вещам такого сорта.
Сейчас вы неизбежно должны ожидать, что этот код запустится с первой попытки.
Я опускаю некоторые пробы и ошибки, которые показывала раньше.
Я на самом деле допустила гору ошибок, которые не показываю, поскольку пропуски `mut` или `;` довольно быстро перестают быть поучительным.
Не переживайте, мы столкнёмся с множеством *других* сообщений об ошибках!

```text
> cargo build

error[E0382]: use of moved value: `new_tail`
  --> src/fifth.rs:38:38
   |
26 |         let new_tail = Box::new(Node {
   |             -------- move occurs because `new_tail` has type `std::boxed::Box<fifth::Node<T>>`, which does not implement the `Copy` trait
...
33 |         let old_tail = mem::replace(&mut self.tail, Some(new_tail));
   |                                                          -------- value moved here
...
38 |                 old_tail.next = Some(new_tail);
   |                                      ^^^^^^^^ value used here after move
```

> Shoot! <!-- эвфемизм для shit -->

Да что ж такое!

> > use of moved value: `new_tail`

> use of moved value: `new_tail`

Использование перемещённого значения. <!-- подумать над переводом: вообще-то речь про то, что значение кто-то извлёк, забрал, так что может быть "извлечённого"? -->

> Box doesn't implement Copy, so we can't just assign it to two locations.
> More importantly, Box *owns* the thing it points to, and will try to free it when it's dropped.
> If our `push` implementation compiled, we'd double-free the tail of our list!
> Actually, as written, our code would free the old_tail on every push.
> Yikes! 🙀

Box не реализует Copy, так что мы просто не можем присвоить его двум разным переменным.
Что важнее, Box *владеет* значением, на которое указывает, и постарается освободить его, когда будет удаляться.
Если бы наша реализация `push` компилировалась, мы бы дважды освобождали хвост нашего списка!
На самом деле, как написано, наш код мог бы освобождать `old_tail` при каждом вызове `push`.
Брррр!

> Alright, well we know how to make a non-owning pointer.
> That's just a reference!

Ладно, мы знаем, когда создать указатель без владения.
Это просто ссылка!

```rust ,ignore
pub struct List<T> {
    head: Link<T>,
    tail: Option<&mut Node<T>>, // НОВАЯ СТРОКА!
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_tail = Box::new(Node {
            elem: elem,
            // When you push onto the tail, your next is always None
            // Когда вставляете в конец списка, следующий узел всегда None
            next: None,
        });

        // Put the box in the right place, and then grab a reference to its Node
        // Помещаем бокс в правильное место и затем получем ссылку на его узел
        let new_tail = match self.tail.take() {
            Some(old_tail) => {
                // If the old tail existed, update it to point to the new tail
                // Если старый хвост существовал, обновляем его, чтобы он указывал на новый хвост
                old_tail.next = Some(new_tail);
                old_tail.next.as_deref_mut()
            }
            None => {
                // Otherwise, update the head to point to it
                // В противном случае, обновляем голову, чтобы указывала на него
                self.head = Some(new_tail);
                self.head.as_deref_mut()
            }
        };

        self.tail = new_tail;
    }
}
```

> Nothing too tricky here.
> Same basic idea as the previous code, except we're using some of that implicit return goodness to extract the tail reference from wherever we stuff the actual Box.

Никакого особого трюкачества.
Та же базовая идея, что и в предыдущем коде, за исключением то, что мы используем преимущества неявного возврата, чтобы получить ссылку на хвост прямо оттуда, куда мы только что поместили Box.

```text
> cargo build

error[E0106]: missing lifetime specifier
 --> src/fifth.rs:3:18
  |
3 |     tail: Option<&mut Node<T>>, // NEW!
  |                  ^ expected lifetime parameter
```

> Oh right, we need to give lifetimes to references in types.
> Hmm... what's the lifetime of this reference?
> Well, this seems like IterMut, right?
> Let's try what we did for IterMut, and just add a generic `'a`:

Да, правильно, мы ведь должны указывать время жизни ссылок в типах.
Хмм... какое время жизни у этой ссылки?
Ну, это всё похоже на IterMut, верно?
Давайте попробуем, что мы делали в IterMut и просто добавим обобщённый `'a`:

```rust ,ignore
pub struct List<'a, T> {
    head: Link<T>,
    tail: Option<&'a mut Node<T>>, // НОВЫЙ КОД!
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<'a, T> List<'a, T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_tail = Box::new(Node {
            elem: elem,
            // When you push onto the tail, your next is always None
            // Когда вставляете в конец списка, следующий узел всегда None
            next: None,
        });

        // Put the box in the right place, and then grab a reference to its Node
        // Помещаем бокс в правильное место и затем получем ссылку на его узел
        let new_tail = match self.tail.take() {
            Some(old_tail) => {
                // If the old tail existed, update it to point to the new tail
                // Если старый хвост существовал, обновляем его, чтобы он указывал на новый хвост
                old_tail.next = Some(new_tail);
                old_tail.next.as_deref_mut()
            }
            None => {
                // Otherwise, update the head to point to it
                // В противном случае, обновляем голову, чтобы указывала на него
                self.head = Some(new_tail);
                self.head.as_deref_mut()
            }
        };

        self.tail = new_tail;
    }
}
```

```text
cargo build

error[E0495]: cannot infer an appropriate lifetime for autoref due to conflicting requirements
  --> src/fifth.rs:35:27
   |
35 |                 self.head.as_deref_mut()
   |                           ^^^^^^^^^^^^
   |
note: first, the lifetime cannot outlive the anonymous lifetime #1 defined on the method body at 18:5...
  --> src/fifth.rs:18:5
   |
18 | /     pub fn push(&mut self, elem: T) {
19 | |         let new_tail = Box::new(Node {
20 | |             elem: elem,
21 | |             // When you push onto the tail, your next is always None
...  |
39 | |         self.tail = new_tail;
40 | |     }
   | |_____^
note: ...so that reference does not outlive borrowed content
  --> src/fifth.rs:35:17
   |
35 |                 self.head.as_deref_mut()
   |                 ^^^^^^^^^
note: but, the lifetime must be valid for the lifetime 'a as defined on the impl at 13:6...
  --> src/fifth.rs:13:6
   |
13 | impl<'a, T> List<'a, T> {
   |      ^^
   = note: ...so that the expression is assignable:
           expected std::option::Option<&'a mut fifth::Node<T>>
              found std::option::Option<&mut fifth::Node<T>>


```

> Woah, that's a really detailed error message.
> That's a bit concerning, because it suggests we're doing something really messed up.
> Here's an interesting part:

Ого, вот действительно подробное сообщение об ошибке.
Оно немного настораживает, потому что утверждает, что мы делаем что-то по настоящему неправильное.
Вот интересная часть:


> > the lifetime must be valid for the lifetime `'a` as defined on the impl

> the lifetime must be valid for the lifetime `'a` as defined on the impl

Время жизни должно быть действительно в течение времени жизни `'a`, заданного в блоке impl

> We're borrowing from `self`, but the compiler wants us to last as long as `'a`, what if we tell it `self` *does* last that long..?

Мы заимствуем `self`, но компилятор хочет, чтобы мы существовали столько же, сколько и `'a`, а что, если мы скажем ему, что `self` действительно существует столько же?

```rust ,ignore
    pub fn push(&'a mut self, elem: T) {
```

```text
cargo build

warning: field is never used: `elem`
 --> src/fifth.rs:9:5
  |
9 |     elem: T,
  |     ^^^^^^^
  |
  = note: #[warn(dead_code)] on by default
```

> Oh, hey, that worked!
> Great!

О, здорово, так работает!
Отлично!

> Let's just do `pop` too:

Давайте также сделаем `pop`:

```rust ,ignore
pub fn pop(&'a mut self) -> Option<T> {
    // Grab the list's current head
    // Забираем текущую голову списка
    self.head.take().map(|head| {
        let head = *head;
        self.head = head.next;

        // If we're out of `head`, make sure to set the tail to `None`.
        // Если новый список пуст, то и хвост должен быть установлен в `None`.
        if self.head.is_none() {
            self.tail = None;
        }

        head.elem
    })
}
```

> And write a quick test for that:

И напишем быстрый <!-- небольшой --> тест для этого <!-- для нашего кода -->.

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;
    #[test]
    fn basics() {
        let mut list = List::new();

        // Check empty list behaves right
        // Проверяем, что пустой список ведёт себя правильно
        assert_eq!(list.pop(), None);

        // Populate list
        // Заполняем список
        list.push(1);
        list.push(2);
        list.push(3);

        // Check normal removal
        // Проверяем обычное удаление
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), Some(2));

        // Push some more just to make sure nothing's corrupted
        // Вставляем новые значения, просто чтобы проверить, что ничего не сломается
        list.push(4);
        list.push(5);

        // Check normal removal
        // Проверяем обычное удаление
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(4));

        // Check exhaustion
        // Проверяем граничный случай
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), None);
    }
}
```

```text
cargo test

error[E0499]: cannot borrow `list` as mutable more than once at a time
  --> src/fifth.rs:68:9
   |
65 |         assert_eq!(list.pop(), None);
   |                    ---- first mutable borrow occurs here
...
68 |         list.push(1);
   |         ^^^^
   |         |
   |         second mutable borrow occurs here
   |         first borrow later used here

error[E0499]: cannot borrow `list` as mutable more than once at a time
  --> src/fifth.rs:69:9
   |
65 |         assert_eq!(list.pop(), None);
   |                    ---- first mutable borrow occurs here
...
69 |         list.push(2);
   |         ^^^^
   |         |
   |         second mutable borrow occurs here
   |         first borrow later used here

error[E0499]: cannot borrow `list` as mutable more than once at a time
  --> src/fifth.rs:70:9
   |
65 |         assert_eq!(list.pop(), None);
   |                    ---- first mutable borrow occurs here
...
70 |         list.push(3);
   |         ^^^^
   |         |
   |         second mutable borrow occurs here
   |         first borrow later used here


....

** WAY MORE LINES OF ERRORS **

** ГОРАЗДО БОЛЬШЕ СТРОК С ОШИБКАМИ **

....

error: aborting due to 11 previous errors
```

🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀

> Oh my goodness.

Да боже ты мой.

> The compiler's not wrong for vomiting all over us.
> We just committed a cardinal Rust sin: we stored a reference to ourselves *inside ourselves*.
> Somehow, we managed to convince Rust that this totally made sense in our `push` and `pop` implementations (I was legitimately shocked we did).

Компилятор совершенно прав, костеря наш код.
Мы только что совершили смертный грех языка Rust: мы сохранили ссылку на себя *внутри самих себя*.
Нам как-то удалось убедить компилятор, что это действительно имеет смысл в наших реализациях `push` и `pop` (я искренне удивилась, что нам это удалось).


> The reason this *sort of* works is that Rust doesn't really have the notion of a pointer into yourself at all.
> Each part of the code is *technically* correct in isolation (we *can* call push and pop *once*) but then the absurdity of what we created takes affect and everything just *locks up*. 

Причина, по которой это *вроде бы* работает в том, что в Rust на самом деле вообще нет понятия указателя на самого себя.
Каждая часть кода *технически* корректна в изоляции (мы *можем* вызывать `push` и `pop` *один раз*), но затем вступает в силу абсурдность того, что мы сделали и всё просто *ломается*.

> I'm sure there is *some* use for what we've written, but as far as *I'm* concerned it's just syntatically valid *gibberish*.
> We're saying we contain something with lifetime `'a`,  and that `push` and `pop` borrows *self* for that lifetime. 
> That's *weird* but Rust can look at each part of our code *individually* and it doesn't see any rules being broken.

Я уверена, что у написанного нами есть *какое-то* применение, но, насколько *мне* известно, это всего лишь синтаксически корректная *тарабарщина*.
Мы говорим, что у нас есть что-то с временем жизни `'a`, и что `push` и `pop` заимствуют *self* на это время жизни.
Это *странно*, но Rust может посмотреть на каждую часть нашего кода *по отдельности* и ни видит никаких нарушений <!-- правил -->.

> But as soon as we try to actually *use* the list, the compiler quickly goes "yep you've borrowed `self` mutably for `'a`, so you can't use `self` anymore until the end of `'a`" but *also* "because you contain `'a`, it must be valid for the entire list's existence".

Но как только мы пытаемся действительно *использовать* список, компилятор быстро нам сообщает: «так, вы заимствовали `self` изменяемым образом на время `'a`, так что вы не можете использовать `self` до конца `'a`», но *также*: «поскольку вы содержите `'a`, он должен быть действителен в течение всего времени существования списка».

> It's *nearly* a contradiction but there *is* one solution: as soon as you `push` or `pop`, the list "pins" itself in place and can't be accessed anymore.
> It has swallowed its own proverbial tail, and ascended to a world of dreams.

Это *почти* противоречие, но *есть* одно решение: сразу после вызова `push` или `pop` список закрепляется на месте и к нему больше нельзя получить доступ.
Образно говоря, он проглатывает свой хвост и возносится в мир грёз.

> > **NARRATOR**: it didn't exist when this book was first written, but Rust actually [formalized the notion of a *pin* into something useful][pin]!
> > This was probably the most complex addition to the language since *the borrowchecker*.
> > We don't *want* our list to be pinned though!
> >
> > Pins *are* necessary and useful for async-await/futures/coroutines because the compiler needs to be able to bundle up all the local variables of a function into some kind of struct and store them somewhere until the future/coroutine is ready to be resumed.
> > Since local variables can reference other local variables, and we want that to *work*, these structs can end up containing references to themselves!
> >
> > So to `await` or `yield` Rust needs a way to be able to properly describe and manipulate pinned values.
> > Thankfully all of this stuff is *largely* just hidden away in automatic compiler machinery and no one actually has to think about `Pin` (or even *Futures*) under normal circumstances.
> > The main exception is that this stuff is very important for the folks building and designing async *runtimes* like tokio.
> >
> > We will not be implementing an async runtime in this book.
> > I know my friends know all sorts of "cool" (messed up) *tricks* you can do with `Pin`, but from what I can tell, I'd be happier to just not know them.
> > I will continue to tell myself that Pinned types aren't real and they can't hurt me. 

> **ГОЛОС ЗА КАДРОМ:** с тех пор, как была написана эта книга, Rust, фактически, [формализовал *закрепление* и нашёл ему применение][pin]!
> Возможно, это было самое сложное расширение языка со времён *анализатор заимствований* (borrow checker).
> Но нам, в любом случае, не надо закреплять наш список!
> 
> Закрепления нужны и полезны для async/await, футур, сопрограмм, поскольку компилятору надо собирать локальные переменные функции в некоторое подобие структуры и хранить их где-то, пока футура/сопрограмма не будет готова к возобновлению.
> Поскольку локальные переменные могут ссылаться на другие локальные переменные, и мы хотим, чтобы всё *работало*, эти структуры в конечном счёте могут содержать ссылки на самих себя!
> 
> Таким образом, для `await` или `yield` Rust нуждается в способе корректного описания и манипулирования закреплёнными значениями.
> К счастью, *в основном* всё это скрыто глубоко в недрах автоматики компилятора и на самом деле при обычных обстоятельствах никому не приходится думать про `Pin` (или даже про *футуры*).
> Главное исключение из этого правила: разработчики и проектировщики асинхронных *библиотек*, таких как tokio.
> 
> > We will not be implementing an async runtime in this book.
> > I know my friends know all sorts of "cool" (messed up) *tricks* you can do with `Pin`, but from what I can tell, I'd be happier to just not know them.
> > I will continue to tell myself that Pinned types aren't real and they can't hurt me. 
> В этой книге мы не будем реализовывать асинхронную библиотеку.
> Я знаю, что мои друзья знают все виды «крутых» (и странных) *трюков*, которые можно провернуть с `Pin`, но мне было бы лучше просто не знать об этом вовсе.
> Продолжу убеждать себя, что закреплённые типы не существуют и не могут мне навредить.

> Our `pop` implementation hints at why storing a reference to ourselves *inside* ourselves could be really dangerous:

Наша реализация `pop` подсказывает, почему хранение ссылки на себя *внутри* себя может быть действительно опасным:

```rust ,ignore
// ...
if self.head.is_none() {
    self.tail = None;
}
```

> What if we forgot to do this?
> Then our tail would point to some node *that had been removed from the list*.
> Such a node would be instantly freed, and we'd have a dangling pointer which Rust was supposed to protect us from!

Что, если мы забудем это сделать?
Тогда наш хвост будет указывать на какой-то узел, *который уже удалён из списка*.
Этот узел может быть мгновенно освобождён и мы получим висящий указатель, от чего Rust должен был нас защитить!

> And indeed Rust is protecting us from that kind of danger.
> Just in a very... **roundabout** way.

И действительно Rust защищает нас от подобной опасности.
Просто очень... **окольным** путём.

> So what can we do?
> Go back to `Rc<RefCell>>` hell?

Так что же нам делать?
Возвращаться в ад `Rc<RefCell>>`?

> Please.
> No.

Спасибо.
Не надо.

> No, instead we're going to go off the rails and use *raw pointers*.
> Our layout is going to look like this:

Нет, вместо этого мы отойдём от правил и воспользуемся *сырыми указателями*.
Наша структура теперь будет выглядеть так.

```rust ,ignore
pub struct List<T> {
    head: Link<T>,
    tail: *mut Node<T>, // ОПАСНО!
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```


> And that's that.
> None of this wimpy reference-counted-dynamic-borrow-checking nonsense!
> Real.
> Hard.
> Unchecked.
> Pointers.

И это всё.
Никакого детского лепета про подсчёт ссылок и динамическую проверку заимствований!
Реальные.
Сложные.
Непроверяемые.
Указатели.

> > **NARRATOR:** This implementation was in fact still dangerously wrong, but it wasn't yet time to learn that lesson.
> > The next section will learn that the hard way, as usual.

> **ГОЛОС ЗА КАДРОМ:** эта реализация была фактически опасно неправильной, но ещё не настало время для этого урока.
> В следующей части, как обычно, усвоим его на собственном горьком опыте.

> Let's be C everyone.
> Let's be C all day.

Давайте все станем сишниками.
Давайте все будем сишниками дни напролёт.

<!--
Игра слов, сложно перевести напрямую.
 -->

> I'm home.
> I'm ready.

Я дома.
Я готова.

> Hello `unsafe`.

Привет, `unsafe`.

> > **NARRATOR:** Wow, just incredible hubris from the author here.

> **ГОЛОС ЗА КАДРОМ:** какая невероятная самонадеянность со стороны автора.

[pin]: https://doc.rust-lang.org/std/pin/index.html
