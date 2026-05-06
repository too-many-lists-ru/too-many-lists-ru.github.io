> # Layout

# Представление данных

> Alright, back to the drawing board on layout.

Ладно, возвращаемся к чертёжной доске, чтобы нарисовать представление.

> The most important thing about a persistent list is that you can manipulate the tails of lists basically for free:

Важнейшая вещь <!-- факт --> об устойчивых списках заключается в том, что вы свободно можете манипулировать хвостами списков:

> For instance, this isn't an uncommon workload to see with a persistent list:

Например, такое поведение не является чем то необычным при работе с устойчивым списком:

```text
list1 = A -> B -> C -> D
list2 = tail(list1) = B -> C -> D
list3 = push(list2, X) = X -> B -> C -> D
```

> But at the end we want the memory to look like this:

Но в итоге мы хотим, чтобы память выглядела так:
<!-- Скорее, "И", а не "Но" -->

```text
list1 -> A ---+
              |
              v
list2 ------> B -> C -> D
              ^
              |
list3 -> X ---+
```

> This just can't work with Boxes, because ownership of `B` is *shared*.
> Who should free it?
> If I drop list2, does it free B?
> With boxes we certainly would expect so!

Это просто не будет работать с Box, потому что владение `B` *разделяется*.
Кто должне его особождать?
Если удалить list2, должно ли это привести к освобождению B?
В случае боксов, мы определённо должны ли бы этого ожидать!

> Functional languages &mdash; and indeed almost every other language &mdash; get away with this by using *garbage collection*.
> With the magic of garbage collection, B will be freed only after everyone stops looking at it.
> Hooray!

Функиональным языкам — да, по сути, всем языкам — это сходит с рук за счёт *сборки мусора*.
Благодаря магии сборки мусора, B будет освобождён только после того, как все перестанут на него смотреть <!-- использовать его значение -->.
Ура!

> Rust doesn't have anything like the garbage collectors these languages have.
> They have *tracing* GC, which will dig through all the memory that's sitting around at runtime and figure out what's garbage automatically.
> Instead, all Rust has today is *reference counting*.
> Reference counting can be thought of as a very simple GC.
> For many workloads, it has significantly less throughput than a tracing collector, and it completely falls over if you manage to build cycles.
> But hey, it's all we've got!
> Thankfully, for our usecase we'll never run into cycles (feel free to try to prove this to yourself &mdash; I sure won't).

В Rust нет ничего похожего на сборщики мусора, которые есть в этих языках.
У них есть *отслеживающий* сборщик мусора, который перелопатит всю помять во время выполнения и найдёт, что <!-- можно --> убрать автоматически.
Вместо этого, всё, что есть сегодня в Rust — это *подсчёт ссылок*.
О подсчёте ссылок можно думать, как об очень простом сборщике мусора.
Во многих сценариях его производительность значительно ниже, чем у отслеживающих сборщиков и он полностью выходит из строя, если вы суммете создать циклы.
Но — эй, это всё, что у нас есть!
К счастью, в наших сценарив мы никогда не зайдём в циклы <!-- не создадим --> (можете попробовать доказать это сами себе — я точно не буду).

> So how do we do reference-counted garbage collection?
> `Rc`!
> Rc is just like Box, but we can duplicate it, and its memory will *only* be freed when *all* the Rc's derived from it are dropped.
> Unfortunately, this flexibility comes at a serious cost: we can only take a shared reference to its internals.
> This means we can't ever really get data out of one of our lists, nor can we mutate them.

Так как же нам сделать сборку мусора, осносванную на подсчёте ссылок?
`Rc`!
Rc — то же самое, что и Box, но мы можем дублировать его, и его память будет освобождена *только* тогда, когда все производные Rc <!-- все его дубли --> будут уничтожены.
К сожалению, эта гибкость ведёт к серьёзному ограничению: мы можем только получить только разделяемую ссылку на внутренний объект.
Это озачает, что мы не можем даже ни по настоящему забрать данные из наших списков, ни изменить их.

> So what's our layout gonna look like?
> Well, previously we had:

Так что на что же должно быть похоже наше представление?
Ну, раньше у нас было:

```rust ,ignore
pub struct List<T> {
    head: Link<T>,
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

> Can we just change Box to Rc?

Можем ли мы просто заменить Box на Rc?

```rust ,ignore
// in third.rs
// в third.rs

pub struct List<T> {
    head: Link<T>,
}

type Link<T> = Option<Rc<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

```text
cargo build

error[E0412]: cannot find type `Rc` in this scope
 --> src/third.rs:5:23
  |
5 | type Link<T> = Option<Rc<Node<T>>>;
  |                       ^^ not found in this scope
help: possible candidate is found in another module, you can import it into scope
  |
1 | use std::rc::Rc;
  |
```

> Oh dang, sick burn.
> Unlike everything we used for our mutable lists, Rc is so lame that it's not even implicitly imported into every single Rust program.
> *What a loser*.

Блин какая едкая насмешка.
В отличие от всего, что мы использовали для мутабельных списков, Rc настолько отстойный, что даже неявно не импотируется ни в какую программу на Rust.
*Что за неудачник*.

```rust ,ignore
use std::rc::Rc;
```

```text
cargo build

warning: field is never used: `head`
 --> src/third.rs:4:5
  |
4 |     head: Link<T>,
  |     ^^^^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: field is never used: `elem`
  --> src/third.rs:10:5
   |
10 |     elem: T,
   |     ^^^^^^^

warning: field is never used: `next`
  --> src/third.rs:11:5
   |
11 |     next: Link<T>,
   |     ^^^^^^^^^^^^^
```

> Seems legit.
> Rust continues to be *completely* trivial to write.
> I bet we can just find-and-replace Box with Rc and call it a day!

Выглядит правильно.
Rust продолжает быть *полность* тривиальным для написания.
Держу пари, мы просто заменим Box на Rc, и на этом закончим!

...

> No.
> No we can't.

Нет.
У нас не выйдет.