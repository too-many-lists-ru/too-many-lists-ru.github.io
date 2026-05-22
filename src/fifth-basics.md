> # Basics

# Основы

> > **NARRATOR:** This section has a looming fundamental error in it, because that's the whole point of the book.
> > However once we start using `unsafe` it's possible to do things wrong and still have everything compile and *seemingly* work.
> > The fundamental mistake will be identified in the next section.
> > Don't actually use the contents of this section in production code!

> **ГОЛОС ЗА КАДРОМ:** эта глава содержит предчувствие фундаментальной ошибки, про которую, собственно, и написана эта книга.
> Однако, как только мы стали использовать `unsafe`, стало возможно делать неправильные вещи, при чём всё будет компилироваться и даже *на первый взгляд* работать.
> Фундаментальная ошибка будет идентифицирована в следующей главе.
> Не используйте содержимое этой главы в продуктовом годе!

> Alright, back to basics.
> How do we construct our list?

Ладно, вернёмся к основам.
Как мы конструируем наш список?

> Before we just did:

Раньше мы просто делали:

```rust ,ignore
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }
}
```

> But we're not using Option for the `tail` anymore:

Но теперь мы не можем использовать Option для `tail`:

```text
> cargo build

error[E0308]: mismatched types
  --> src/fifth.rs:15:34
   |
15 |         List { head: None, tail: None }
   |                                  ^^^^ expected *-ptr, found 
   |                                       enum `std::option::Option`
   |
   = note: expected type `*mut fifth::Node<T>`
              found type `std::option::Option<_>`
```

> We *could* use an Option, but unlike Box, `*mut` *is* nullable.
> This means it can't benefit from the null pointer optimization.
> Instead, we'll be using `null` to represent None.

Мы *могли бы* использовать Option, но, в отличие от Box, `*mut` *может* принимать значение `null`.
Это значит, что мы не получаем преимуществ от оптимизации указателей на `null`.
Вместо этого мы используем `null`, чтобы представить вариант `None`.

> So how do we get a null pointer?
> There's a few ways, but I prefer to use `std::ptr::null_mut()`.
> If you want, you can also use `0 as *mut _`, but that just seems so *messy*.

Ну, а как нам получить указатель на `null`?
Есть несколько способов, но я предпочитаю вызывать `std::ptr::null_mut()`.
Если хотите, можете писать `0 as *mut _`, но это выглядит таким *неряшливым*.

```rust ,ignore
use std::ptr;

// defns...
// определения...

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: ptr::null_mut() }
    }
}
```

```text
cargo build

warning: field is never used: `head`
 --> src/fifth.rs:4:5
  |
4 |     head: Link<T>,
  |     ^^^^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: field is never used: `tail`
 --> src/fifth.rs:5:5
  |
5 |     tail: *mut Node<T>,
  |     ^^^^^^^^^^^^^^^^^^

warning: field is never used: `elem`
  --> src/fifth.rs:11:5
   |
11 |     elem: T,
   |     ^^^^^^^

warning: field is never used: `head`
  --> src/fifth.rs:12:5
   |
12 |     head: Link<T>,
   |     ^^^^^^^^^^^^^
```

> *shush* compiler, we will use them soon.

*Тише*, компилятор, совсем скоро мы используем все эти переменные.

> Alright, let's move on to writing `push` again.
> This time, instead of grabbing an `Option<&mut Node<T>>` after we insert, we're just going to grab a `*mut Node<T>` to the insides of the Box right away.
> We know we can soundly do this because the contents of a Box has a stable address, even if we move the Box around.
> Of course, this isn't *safe*, because if we just drop the Box we'll have a pointer to freed memory.

Ладно, давайте снова напишем `push`.
Теперь, вместо того, чтобы получить `Option<&mut Node<T>>` после вставки, мы сразу же получим `*mut Node<T>` их Box.
Мы знаем, мы можем без проблем это сделать, потому что содержимое Box имеет стабильный адрес, даже если мы перемещаем Box.
Конечно, это *не* безопасно, потому что если мы просто удалим Box, у нас останется указатель на освобождённую память.

> How do we make a raw pointer from a normal pointer?
> Coercions!
> If a variable is declared to be a raw pointer, a normal reference will coerce into it:

Как получить сырой указатель из обычного указателя?
Приведение типа!
Если переменная объявлена как сырой указатель, к нему можно привести обычную ссылку.

```rust ,ignore
let raw_tail: *mut _ = &mut *new_tail;
```

> We have all the info we need.
> We can translate our code into, approximately, the previous reference version:

У нас есть вся необходимая информация.
Мы можем переписать наш код так, чтобы он был более-менее похож на нашу эталонную реализацию.

```rust ,ignore
pub fn push(&mut self, elem: T) {
    let mut new_tail = Box::new(Node {
        elem: elem,
        next: None,
    });

    let raw_tail: *mut _ = &mut *new_tail;

    // .is_null checks for null, equivalent to checking for None
    // .is_null проверяет на null, эквивалент проверки на None
    if !self.tail.is_null() {
        // If the old tail existed, update it to point to the new tail
        // Если старый хвост существовал, обновляем его, чтобы он указывал на новый хвост
        self.tail.next = Some(new_tail);
    } else {
        // Otherwise, update the head to point to it
        // В противном случае, обновляем голову, чтобы указывала на него
        self.head = Some(new_tail);
    }

    self.tail = raw_tail;
}
```

```text
> cargo build

error[E0609]: no field `next` on type `*mut fifth::Node<T>`
  --> src/fifth.rs:31:23
   |
31 |             self.tail.next = Some(new_tail);
   |             ----------^^^^
   |             |
   |             help: `self.tail` is a raw pointer; 
   |             try dereferencing it: `(*self.tail).next`
```

> Huh?
> We have a pointer to a Node, why can't we get the `next` field?

Что?
У нас есть указатель на Node, почему нам недоступно поле `next`?

> Rust is kinda a jerk when you use raw pointers.
> To access the contents of a raw pointer, it insists that we manually deref them, because it's such an unsafe operation.
> So let's do that:

Когда используешь сырые указатели, Rust становится довольно грубым.
Для доступа к содержимому сырого указателя нужно ручное разыменования, поскольку это крайней небезопасная операция.
Ну, давайте, наконец, сделаем:

```rust ,ignore
*self.tail.next = Some(new_tail);
```

```text
> cargo build

error[E0609]: no field `next` on type `*mut fifth::Node<T>`
  --> src/fifth.rs:31:23
   |
31 |             *self.tail.next = Some(new_tail);
   |             -----------^^^^
   |             |
   |             help: `self.tail` is a raw pointer; 
   |             try dereferencing it: `(*self.tail).next`
```

> Uuuugh operator precedence.

Приоритет операторов, будь он неладен.

```rust ,ignore
(*self.tail).next = Some(new_tail);
```

```text
> cargo build

error[E0133]: dereference of raw pointer is unsafe and requires 
              unsafe function or block

  --> src/fifth.rs:31:13
   |
31 |             (*self.tail).next = Some(new_tail);
   |             ^^^^^^^^^^^^^^^^^ dereference of raw pointer
   |
   = note: raw pointers may be NULL, dangling or unaligned; 
     they can violate aliasing rules and cause data races: 
     all of these are undefined behavior
```

> THIS.
> SHOULDN'T.
> BE.
> THIS.
> HARD.

ЭТО.
НЕ ДОЛЖНО.
БЫТЬ.
НАСТОЛЬКО.
СЛОЖНЫМ.

> Remember how I said Unsafe Rust is like an FFI language for Safe Rust?
> Well, the compiler wants us to explicitly delimit where we're doing this FFI-ing.
> We have two options.
> First, we can mark our *entire* function as unsafe, in which case it becomes an Unsafe Rust function and can only be called in an `unsafe` context.
> This isn't great, because we want our list to be safe to use.
> Second, we can write an `unsafe` block inside our function, to delimit the FFI boundary.
> This declares the overall function to be safe.
> Let's do that one:

Помните, я говорила, что Небезопасный Rust — это своего рода FFI для Безопасного Rust?
Ну вот, компилятор требует, чтобы мы явно указывали, где именно применяем FFI.
У нас есть два варианта.
Во-первых, мы можем пометить всю нашу функцию ключевым словом `unsafe`, при этом она становится функцией Небезопасного Rust и может быть вызвана только в небезопасном контексте.
Это не очень здорово, потому что мы хотим, чтобы наш список можно было использовать в безопасном коде.
Во-вторых, мы можем написать блок `unsafe` внутри нашей функции, чтобы обозначить границы FFI.
При этом сама функция считается безопасной.
Что ж, давайте сделаем:


```rust ,ignore
pub fn push(&mut self, elem: T) {
    let mut new_tail = Box::new(Node {
        elem: elem,
        next: None,
    });

    let raw_tail: *mut _ = &mut *new_tail;

    if !self.tail.is_null() {
        // Hello Compiler, I Know I Am Doing Something Dangerous And
        // I Promise To Be A Good Programmer Who Never Makes Mistakes.
        // Привет, компилятор. Я знаю, что делаю кое-что опасное и
        // я обещаю быть хорошим программистом, который никогда не ошибается.
        unsafe {
            (*self.tail).next = Some(new_tail);
        }
    } else {
        self.head = Some(new_tail);
    }

    self.tail = raw_tail;
}
```

```text
> cargo build
warning: field is never used: `elem`
  --> src/fifth.rs:11:5
   |
11 |     elem: T,
   |     ^^^^^^^
   |
   = note: #[warn(dead_code)] on by default
```

> Yay!

Ага!

> It's kind've interesting that that's the *only* place we've had to write an unsafe block so far.
> We do raw pointer stuff all over the place, what's up with that?

Довольно интересно, что пока это *единственное* место, где нам пришлось написать небезопасный код.
Сырые указатели у нас повсюду, почему `unsafe` потребовался только здесь?

> It turns out that Rust is a massive rules-lawyer pedant when it comes to `unsafe`.
> We quite reasonably want to maximize the set of Safe Rust programs, because those are programs we can be much more confident in.
> To accomplish this, Rust carefully carves out a minimal surface area for unsafety.
> Note that all the other places we've worked with raw pointers has been *assigning* them, or just observing whether they're null or not.

Дело в том, что, когда речь заходит об `unsafe`, Rust становится настоящим педантом, помешанным на правилах.
Мы вполне обоснованно хотим максимизировать множество программ на Безопасном Rust, потому что в отношении них мы можем быть гораздо более уверены.
Чтобы этого достигнуть, Rust тщательно выделяет минимальную возможную небезопасную область.
Обратите внимание, что в других местах, где мы работали с сырыми указателями, мы либо *присваивали* им значение, либо просто проверяли их на null.

> If you never actually dereference a raw pointer *those are totally safe things to do*.
> You're just reading and writing an integer!
> The only time you can actually get into trouble with a raw pointer is if you actually dereference it.
> So Rust says *only* that operation is unsafe, and everything else is totally safe.

Если вы никогда не разыменовываете сырой указатель *все эти операции совершенно безопасны*.
Вы просто читаете и записываете целые числа!
Неприятности с сырым указателем могут возникнуть только при разыменовании.
Так что Rust считает небезопасной *только* эту операцию, а все остальные совершенно безопасны.

> Super.
> Pedantic.
> But technically correct.

Супер.
Педантично.
Но технически корректно.

> > **NARRATOR:** Somewhere on the other side of the world, a hardware engineer feels a shiver down her spine &mdash; someone must be insisting pointers are just integers again.
> > She looks down at her proposal for a new hardware pointer authentication scheme and sheds a single tear.
> > The compiler engineer next door feels nothing &mdash; they long ago learned to always wear a heavy sweater.

> **ГОЛОС ЗА КАДРОМ:** Где-то на другом конце света инженер-электронщик чувствует, как мурашки бегут по её коже — должно быть, кто-то снова утверждает, что указатели — это всего лишь целые числа.
> Она смотрит на своё предложение по новой схеме аутентификации аппаратных указателей и скупая слеза бежит по её щеке.
> Инженер-сборщик по соседству не чувствует ничего — он давно научился всегда носить тёплый свитер.

> Having only some of the pointer operations be *actually* unsafe raises an interesting problem: although we're supposed to delimit the scope of the unsafety with the `unsafe` block, it actually depends on state that was established outside of the block.
> Outside of the function, even!

Из-за того, что только некоторые операции *действительно* являются небезопасными, возникает интересная проблема: хотя предполагается, что небезопасная область ограничена блоком `unsafe`, в действительности, она зависит от состояния, заложенного вне этого блока.
Даже вне функции!

> This is what I call unsafe *taint*.
> As soon as you use `unsafe` in a module, that whole module is tainted with unsafety.
> Everything has to be correctly written in order to make sure all invariants are upheld for the unsafe code.

Это то, что я называю небезопасным *заражением*.
Как только вы используете `unsafe` в модуле, весь модуль заражается небезопасностью.
Всё должно быть написано корректно, чтобы гарантировать соблюдение всех инвариантов в небезопасном коде.

> This taint is manageable because of *privacy*.
> Outside of our module, all of our struct fields are totally private, so no one else can mess with our state in arbitrary ways.
> As long as no combination of the APIs we expose causes bad stuff to happen, as far as an outside observer is concerned, all of our code is safe!
> And really, this is no different from the FFI case.
> No one needs to care if some python math library shells out to C as long as it exposes a safe interface.

Проблема заражения решается благодаря *приватности*.
Вне нашего модуля все поля нашей структуры полностью приватны, поэтому никто не другой не может произвольно менять наше состояние.
До тех пор, пока никакая комбинация вызовов API не приводит к негативным последствиям, с точки зрения внешнего наблюдателя весь наш код безопасен!
И, по сути, это ничем не отличается от FFI.
Никого не волнует, когда математическая библиотека Python вызывает код на C, если она предоставляет безопасные интерфейс.

> Anyway, let's move on to `pop`, which is pretty much verbatim the reference version:

В любом случае, давайте напишем `pop`, который практически дословно повторяет эталонную версию:

```rust ,ignore
pub fn pop(&mut self) -> Option<T> {
    self.head.take().map(|head| {
        let head = *head;
        self.head = head.next;

        if self.head.is_none() {
            self.tail = ptr::null_mut();
        }

        head.elem
    })
}
```

> Again we see another case where safety is stateful.
> If we fail to null out the tail pointer in *this* function, we'll see no problems at all.
> However subsequent calls to `push` will start writing to the dangling tail!

Мы встретили ещё один сценарий, когда безопасность зависит от состояния.
Если нам не удастся обнулить указатель на хвост в *этой* функции, у нас не будет никаких проблем.
Однако последующие вызовы `push` начнут писать в висячий хвост!

> Let's test it out:

Давайте протестируем:

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;
    #[test]
    fn basics() {
        let mut list = List::new();

        // Check empty list behaves right
        assert_eq!(list.pop(), None);

        // Populate list
        list.push(1);
        list.push(2);
        list.push(3);

        // Check normal removal
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push(4);
        list.push(5);

        // Check normal removal
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), None);

        // Check the exhaustion case fixed the pointer right
        list.push(6);
        list.push(7);

        // Check normal removal
        assert_eq!(list.pop(), Some(6));
        assert_eq!(list.pop(), Some(7));
        assert_eq!(list.pop(), None);
    }
}
```

> This is just the stack test, but with the expected `pop` results flipped around.
> I also added some extra steps at the end to make sure that tail-pointer corruption case in `pop` doesn't occur.

Это всё тот же тест стека, но с инвертированным ожидаемыми результатами `pop`.
Я также добавила несколько дополнительных проверок в конец, чтобы убедиться, что при вызове `pop` указатель на хвост не повреждается.

```text
cargo test

running 12 tests
test fifth::test::basics ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test fourth::test::into_iter ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 12 passed; 0 failed; 0 ignored; 0 measured
```

> Gold Star!

Отличная работа!

> > **NARRATOR:** Here it comes...

> **ГОЛОС ЗА КАДРОМ:** Ну-ну.
