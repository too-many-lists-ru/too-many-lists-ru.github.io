> # Basic Data Layout

# Базовое представление данных
<!-- layout -- план, макет, как всё должно выглядеть в своей сути  -->

> Alright, so what's a linked list?
> Well basically, it's a bunch of pieces of data on the heap (hush, kernel people!) that point to each other in sequence.
> Linked lists are something procedural programmers shouldn't touch with a 10-foot pole, and what functional programmers use for everything.
> It seems fair, then, that we should ask functional programmers for the definition of a linked list.
> They will probably give you something like the following definition:

Ладно, так что же такое связный список?
Ну, в своей основе это набор кусочков данных в куче (успокойтесь, писатели ядра!), которые последовательно указывают друг на друга.
Связные списки — это что-то такое, что процедурные программисты не должны трогать 10-тиметровой палкой, а функциональные программисты используют вообще для всего.
Кажется справедливым в этом случае что мы должны спросить у функциональных программистов определение связного списка.
И они, возможно, дадут вам что-то похожее на такое определение:

<!-- hush можно перевести, как "ш-ш-ш" или "т-с-с", а kernel people -- это люди, которые пишут ядро или по крайней мере, разбираются, как оно устроено -->

```haskell
List a = Empty | Elem a (List a)
```

> Which reads approximately as "A List is either Empty or an Element followed by a List".
> This is a recursive definition expressed as a *sum type*, which is a fancy name for "a type that can have different values which may be different types".
> Rust calls sum types `enum`s!
> If you're coming from a C-like language, this is exactly the enum you know and love, but in overdrive.
> So let's transcribe this functional definition into Rust!

Что читается приблизительно как «Список — это либо Пустота, либо Элемент за которым следует Список».
Это рекурсивное определение, выраженное через *тип-сумму*, что является причудливым способом сказать «тип, который может хранить ралличные значения, которые могут быть различных типов».
В языке Rust *типы-суммы* называются перечислениями (`enum`s)!
Если вы пришли из языка, похожего на C, то речь идёт о знакомых и любимых вами `enum`, но разогнанные. <!-- overdirve -- ускоренный, разогнанный, можно написать "на максималках" -->
Что ж, давайте перепишем это фнукциональное определение на Rust!

> For now we'll avoid generics to keep things simple.
> We'll only support storing signed 32-bit integers:

Для простоты первое время мы не будем использовать обобщённые типы данных.
Мы будем поддерживать только хранение знаковых 32-битных целых чисел:

```rust ,ignore
// in first.rs
// в first.rs

// pub says we want people outside this module to be able to use List
// pub говорит, что мы хотим, чтобы авторы других модулей могли использовать List
pub enum List {
    Empty,
    Elem(i32, List),
}
```

> *phew*, I'm swamped.
> Let's just go ahead and compile that:

*Уф*, как сложно.
Давайте просто попробуем скомилировать этот код:

```text
> cargo build

error[E0072]: recursive type `first::List` has infinite size
 --> src/first.rs:4:1
  |
4 | pub enum List {
  | ^^^^^^^^^^^^^ recursive type has infinite size
5 |     Empty,
6 |     Elem(i32, List),
  |               ---- recursive without indirection
  |
  = help: insert indirection (e.g., a `Box`, `Rc`, or `&`) at some point to make `first::List` representable
```

(Перевод: рекурсивный тип `first::List` имеет бесконечный размер. Рекурсия без косвенного доступа. Добавьте в нужном месте косвенный доступ, т. е. `Box`, `Rc` или `&`, чтобы сделать `first::List` представимым.)

> Well.
> I don't know about you, but I certainly feel betrayed by the functional programming community.

Ну.
Не знаю, как вы, но я определённо чувствую себя преданным всеми сообществом функциональных программистов. <!-- всеми функциональными программистами -->

> If we actually check out the error message (after we get over the whole betrayal thing), we can see that rustc is actually telling us exactly how to solve this problem:

Если мы действительно подумаем над сообщеним об ошибке (после того, как переживём всю эту историю с предательством), мы увидим, что rustc на самом деле точно подсказывает нам, как решить эту проблему:

> > insert indirection (e.g., a `Box`, `Rc`, or `&`) at some point to make `first::List` representable

> добавьте в нужном месте косвенный доступ, т. е. `Box`, `Rc` или `&`, чтобы сделать `first::List` представимым

> Alright, `box`.
> What's that?
> Let's google `rust box`...

Так, `box`.
Что это?
Давайте погуглим `rust box`...

> > [std::boxed::Box - Rust](https://doc.rust-lang.org/std/boxed/struct.Box.html)

> [std::boxed::Box - Rust](https://doc.rust-lang.org/std/boxed/struct.Box.html)


> Lesse here...

Посмотрим, что пишут...

<!-- lesse значит let's see в https://www.urbandictionary.com/define.php?term=Lesse -->

> > `pub struct Box<T>(_);`
> >
> > A pointer type for heap allocation.
> > See the [module-level documentation](https://doc.rust-lang.org/std/boxed/) for more.

> `pub struct Box<T>(_);`
>
> Тип-указатель для размещения в куче.
> Дополнительную информацию см. в документации к модулю [boxed](https://doc.rust-lang.org/std/boxed/).


> *clicks link*

*щёлкает по ссылке*

> > `Box<T>`, casually referred to as a 'box', provides the simplest form of heap allocation in Rust. Boxes provide ownership for this allocation, and drop their contents when they go out of scope.
> >
> > Examples
> >
> > Creating a box:
> >
> > `let x = Box::new(5);`
> >
> > Creating a recursive data structure:
> >

> `Box<T>`, обычно называемый «боксом», представляет собой простейшую форму аллокации <!-- объекта --> в куче в языке Rust.
> Боксы обеспечивает владение этой аллокацией, и уничтожают содержимое, когда они <!-- боксы --> выходят из области видимости.
>
> Примеры
>
> Создание бокса
>
> `let x = Box::new(5);`
>
> Создание рекурсивной структуры данных:
>
```rust
#[derive(Debug)]
enum List<T> {
    Cons(T, Box<List<T>>),
    Nil,
}
```
>
```rust ,ignore
fn main() {
    let list: List<i32> = List::Cons(1, Box::new(List::Cons(2, Box::new(List::Nil))));
    println!("{:?}", list);
}
```
> >
> > This will print `Cons(1, Box(Cons(2, Box(Nil))))`.
> >
> > Recursive structures must be boxed, because if the definition of Cons looked like this:
> >
> > `Cons(T, List<T>),`
> >
> > It wouldn't work.
> > This is because the size of a List depends on how many elements are in the list, and so we don't know how much memory to allocate for a Cons.
> > By introducing a Box, which has a defined size, we know how big Cons needs to be.

>
> Этот код напечатает `Cons(1, Box(Cons(2, Box(Nil))))`.
>
> Рекурсивные структуры должны быть помещены в бокс, потому что если определение выглядит так:
>
> `Cons(T, List<T>),`
>
> Оно не будет работать.
> Это потому, что размер List зависит от того, сколько элементов в списке, так что мы не знаем, сколько памяти выделить для Cons.This is because the size of a List depends on how many elements are in the list, and so we don't know how much memory to allocate for a Cons.
> Введя Box, который имеет определённый размер, мы понимаем, насколько большим будет Cons.

> Wow, uh.
> That is perhaps the most relevant and helpful documentation I have ever seen.
> Literally the first thing in the documentation is *exactly what we're trying to write, why it didn't work, and how to fix it*.

Ого! Хм.
Кажется, это самая релевантная и полезная документация, которую я когда-либо читал.
Буквально, первая вещь в этой документации о том, *что мы пытаемся написать, почему оно не работало и как его исправить*.

> Dang, docs rule.

Блин, доки рулят.

> Ok, let's do that:

Ладно, давайте напишем так:

```rust ,ignore
pub enum List {
    Empty,
    Elem(i32, Box<List>),
}
```

```text
> cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

> Hey it built!

Ага, оно собралось!

> ...but this is actually a really foolish definition of a List, for a few reasons.

...но на самом деле это достаточно дурацкое определение списка по нескольким причинам.

> Consider a list with two elements:

Представим список из двух элементов.

```text
[] = Стек
() = Куча

[Elem A, ptr] -> (Elem B, ptr) -> (Empty, *junk*)
```

> There are two key issues:

Есть два ключевых вопроса:

> * We're allocating a node that just says "I'm not actually a Node"
> * One of our nodes isn't heap-allocated at all.

* Мы аллоцировали узел, который как бы говорит «На самом деле я не Узел»
* Одни из наших улове вообще не размещён в куче.

> On the surface, these two seem to cancel each-other out.
> We heap-allocate an extra node, but one of our nodes doesn't need to be heap-allocated at all.
> However, consider the following potential layout for our list:

На первый взгляд кажется, что эта пара нейтрализует друг друга.
Мы размещаем в куче дополнительный узел, но один из наших узлов вообще не должен быть в куче.
Однако, рассмотрим такой потенциальный сценарий для нашего списка:

```text
[ptr] -> (Elem A, ptr) -> (Elem B, *null*)
```

> In this layout we now unconditionally heap allocate our nodes.
> The key difference is the absence of the *junk* from our first layout.
> What is this junk?
> To understand that, we'll need to look at how an enum is laid out in memory.

В таком сценарии мы размещаем в куче все наши узлы без исключения.
Ключевое отличие в отсутствии *барахла* <!-- мусора? --> из нашего первого сценария.
Что это за барахло?
Что это понять, нам надо выяснить, как наш `enum` размещается в памяти.

> In general, if we have an enum like:

В целом, если у нас такой `enum`:

```rust ,ignore
enum Foo {
    D1(T1),
    D2(T2),
    ...
    Dn(Tn),
}
```

> A Foo will need to store some integer to indicate which *variant* of the enum it represents (`D1`, `D2`, .. `Dn`).
> This is the *tag* of the enum.
> It will also need enough space to store the *largest* of `T1`, `T2`, .. `Tn` (plus some extra space to satisfy alignment requirements).

`Foo` будет хранить какое-то целое число, чтобы обозначить, какой из *вариантов* перечисления представлен (`D1`, `D2`, .. `Dn`).
Это — метка перечисления.
Ему также требуется достаточно памяти, чтобы сохранить *наибольший* из типов `T1`, `T2`, .. `Tn` (плюс немного дополнительного пространства, чтобы удовлетворить требования выравнивания).

> The big takeaway here is that even though `Empty` is a single bit of information, it necessarily consumes enough space for a pointer and an element, because it has to be ready to become an `Elem` at any time.
> Therefore the first layout heap allocates an extra element that's just full of junk, consuming a bit more space than the second layout.

Основной вывод здесь в том, что хотя `Empty` хранит один бит информации, он в любом случае резервирует место для указателя и элемента, поскольку в любой момент может превратиться в `Elem`.
Поэтому в первом сценарии куча хранит дополнительный элемент с барахлом <!-- невостребованным местом --> внутри, требуя немного больше места, чем во втором сценарии.

> One of our nodes not being allocated at all is also, perhaps surprisingly, *worse* than always allocating it.
> This is because it gives us a *non-uniform* node layout.
> This doesn't have much of an appreciable effect on pushing and popping nodes, but it does have an effect on splitting and merging lists.

Тот факт, что один из наших узлов вообще не аллоцирован, как ни странно, *хуже*, чем всегда аллоцировать его.
Это потому что приводит нас к *неунифицированному* размещению узлов.
Это не оказывает существенного влияния на вставку и удалени узлов, но влияет на разделение и слияние списков.

> Consider splitting a list in both layouts:

Сравним разделение списка в обоих сценариях.

```text
сценарий 1:

[Elem A, ptr] -> (Elem B, ptr) -> (Elem C, ptr) -> (Empty *junk*)

переносим C в другой список:

[Elem A, ptr] -> (Elem B, ptr) -> (Empty *junk*)
[Elem C, ptr] -> (Empty *junk*)
```

```text
сценарий 2:

[ptr] -> (Elem A, ptr) -> (Elem B, ptr) -> (Elem C, *null*)

переносим C в другой список:

[ptr] -> (Elem A, ptr) -> (Elem B, *null*)
[ptr] -> (Elem C, *null*)
```

> Layout 2's split involves just copying B's pointer to the stack and nulling the old value out.
> Layout 1 ultimately does the same thing, but also has to copy C from the heap to the stack.
> Merging is the same process in reverse.

Во втором случае разделение сводится к копированию указателя на B в стек и обнулении старого значения.
В первом случае в целом происходит то же самое, но кроме всег прочего приходиться копировать C из кучи в стек.
Слияние по сути то же самое, но в обратном порядке.

> One of the few nice things about a linked list is that you can construct the element in the node itself, and then freely shuffle it around lists without ever moving it.
> You just fiddle with pointers and stuff gets "moved".
> Layout 1 trashes this property.

Одно из немногих преимуществ связных списков заключается в том, что вы можете сконструировать элемент непосредственно в узле, и затем свободно перемещать его по списку, оставляя в то же время в одном и том же месте памяти.
Вы играете с указателями <!-- так же обманывать, мухлевать --> и элементы «перемещаются».
Первый сценарий портит это свойство.

> Alright, I'm reasonably convinced Layout 1 is bad.
> How do we rewrite our List?
> Well, we could do something like:

Ладно, я почти уверен, что первый сценарий плох.
Как нам переписать список?
Ну, мы могли бы сделать что-то вроде этого:

```rust ,ignore
pub enum List {
    Empty,
    ElemThenEmpty(i32),
    ElemThenNotEmpty(i32, Box<List>),
}
```

> Hopefully this seems like an even worse idea to you.
> Most notably, this really complicates our logic, because there is now a completely invalid state: `ElemThenNotEmpty(0, Box(Empty))`.
> It also *still* suffers from non-uniformly allocating our elements.

Надеюсь, вы сразу решили, что эта идея ещё хуже.
В первую очередь, это сразу усложняет нашу логику, потому что теперь существует совершенно недопустимый вариант: `ElemThenNotEmpty(0, Box(Empty))`.
Кроме всего прочего, мы *всё ещё* не избавились от неунифицированного размещения элементов.

> However it does have *one* interesting property: it totally avoids allocating the Empty case, reducing the total number of heap allocations by 1.
> Unfortunately, in doing so it manages to waste *even more space*!
> This is because the previous layout took advantage of the *null pointer optimization*.

Однако у этого варианта есть *одно* интересное свойство: он позволяет полностью избавиться от аллокации узла Empty, сокращая общее число аллокаций в куче на 1.
К сожалению, при этом расходуется *ещё больше места*!
Это связано с тем, что в прошлом сценарии использовалась *оптимизация указателей на null*.

> We previously saw that every enum has to store a *tag* to specify which variant of the enum its bits represent.
> However, if we have a special kind of enum:

Мы видели ранее, что каждое перечисление содержит *метку*, чтобы знать, какой из вариантов перечисления представлен.
Однако, если у нас есть специальный вид перечисления:

```rust,ignore
enum Foo {
    A,
    B(ContainsANonNullPtr),
}
```

> the null pointer optimization kicks in, which *eliminates the space needed for the tag*.
> If the variant is A, the whole enum is set to all `0`'s.
> Otherwise, the variant is B.
> This works because B can never be all `0`'s, since it contains a non-zero pointer.
> Slick !

срабатывает оптимизация указателя на null, которая *избавляется от места, выделяемого под метку*.
В случае варианта A все байты перечисления равны 0.
В противном случае мы имеем вариант B.
Это работает, потому что B никогда не может состоять из одних 0, поскольку содержит ненулевой указатель.
Ловко!

> Can you think of other enums and types that could do this kind of optimization?
> There's actually a lot!
> This is why Rust leaves enum layout totally unspecified.
> There are a few more complicated enum layout optimizations that Rust will do for us, but the null pointer one is definitely the most important!
> It means `&`, `&mut`, `Box`, `Rc`, `Arc`, `Vec`, and several other important types in Rust have no overhead when put in an `Option`!
> (We'll get to most of these in due time.)

Не приходят ли вам в голову другие перечисления и типы, где могла бы работать такого рода оптимизация?
На самом деле их очень много!
По этой причине Rust не регламентирует способ хранения перчислений.
Есть ещё несколько сложных оптимизаций способа хранения, которые Rust делает для нас, но указатель на null безусловно один из самых важных!
Это значит, что `&`, `&mut`, `Box`, `Rc`, `Arc`, `Vec` и некоторые другие важные типы Rust не требуют дополнительной памяти, если поместить их в тип `Option`!
(Большинство из них мы обсудим позже.)

> So how do we avoid the extra junk, uniformly allocate, *and* get that sweet null-pointer optimization?
> We need to better separate out the idea of having an element from allocating another list.
>  To do this, we have to think a little more C-like: structs!

Итак, как измбежать лишнего барахла, унифицированно аллоцировать память *и* добиться заветной оптимизации указателя на null?
Нам надо лучше разграничить идеи наличия элемента и аллоцирования последующего списка.
Чтобы это сделать, мы должны думать немного в стиле C: структуры!

> While enums let us declare a type that can contain *one* of several values, structs let us declare a type that contains *many* values at once.
> Let's break our List into two types: A List, and a Node.

В то время, как пречисления позволяют нам объявлять типы, которые содержат *одно* из нескольких значений, структуры позволяют нам объявлять типы, которые одновременно содержат *много* значений.
Давайте разделим наш List на два типа: List и Node.

> As before, a List is either Empty or has an element followed by another List.
> By representing the "has an element followed by another List" case by an entirely separate type, we can hoist the Box to be in a more optimal position:

Как и раньше, Список может быть либо Пустым или содержать элемент, за которым следует другой Список.
Представив вариант «содержать элемент, за которым  следует другой Список" в виде отдельного типа, мы можем довести Box по более оптимального положения:

```rust ,ignore
struct Node {
    elem: i32,
    next: List,
}

pub enum List {
    Empty,
    More(Box<Node>),
}
```

> Let's check our priorities:

Давайте сверим наши приоритеты:

> * Tail of a list never allocates extra junk: check!
> * `enum` is in delicious null-pointer-optimized form: check!
> * All elements are uniformly allocated: check!

* Хвост списка никогда не аллоцирует дополнительное пространство: есть!
* `enum` хранится в заветной форме оптимизированного указателя на null: есть!
* Все элементы хранятся унифицированно: есть!

> Alright!
> We actually just constructed exactly the layout that we used to demonstrate that our first layout (as suggested by the official Rust documentation) was problematic.

Прекрасно!
Мы сумели сконструировать точно такой способ представления, который использовали, чтобы продемострировать, что наш первый способ был проблематичным (в соответствии с официальной документацией Rust).

```text
> cargo build

warning: private type `first::Node` in public interface (error E0446)
 --> src/first.rs:8:10
  |
8 |     More(Box<Node>),
  |          ^^^^^^^^^
  |
  = note: #[warn(private_in_public)] on by default
  = warning: this was previously accepted by the compiler but
    is being phased out; it will become a hard error in a future release!
```

:(

> Rust is mad at us again.
> We marked the `List` as public (because we want people to be able to use it), but not the `Node`.
> The problem is that the internals of an `enum` are totally public, and we're not allowed to publicly talk about private types.
> We could make all of `Node` totally public, but generally in Rust we favour keeping implementation details private.
> Let's make `List` a struct, so that we can hide the implementation details:

Rust снова спятил из-за нас.
Мы сделали `List` публичным (поскольку мы хотим позволить людям использовать его), но не `Node`.
Проблема в том, что внутренности `enum` полностью публичны и нам не разрешено публично обсуждать приватные типы.
Нам бы следовало сделать `Node` полностью публичным, но в целом в Rust мы предпочитаем оставлять детали реализации приватными.
Давайте сделаем `List` структурой, чтобы мы могли спрятать детали реализации:

```rust ,ignore
pub struct List {
    head: Link,
}

enum Link {
    Empty,
    More(Box<Node>),
}

struct Node {
    elem: i32,
    next: Link,
}
```

> Because `List` is a struct with a single field, its size is the same as that field.
> Yay zero-cost abstractions!

Поскольку `List` — это структура с единственным полем, её размер будет таким же, как у этого поля.
Даёшь абстракции с нулевой стоимостью!

```text
> cargo build

warning: field is never used: `head`
 --> src/first.rs:2:5
  |
2 |     head: Link,
  |     ^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: variant is never constructed: `Empty`
 --> src/first.rs:6:5
  |
6 |     Empty,
  |     ^^^^^

warning: variant is never constructed: `More`
 --> src/first.rs:7:5
  |
7 |     More(Box<Node>),
  |     ^^^^^^^^^^^^^^^

warning: field is never used: `elem`
  --> src/first.rs:11:5
   |
11 |     elem: i32,
   |     ^^^^^^^^^

warning: field is never used: `next`
  --> src/first.rs:12:5
   |
12 |     next: Link,
   |     ^^^^^^^^^^

```

> Alright, that compiled!
> Rust is pretty mad, because as far as it can tell, everything we've written is totally useless: we never use `head`, and no one who uses our library can either since it's private.
> Transitively, that means Link and Node are useless too.
> So let's solve that!
> Let's implement some code for our List!

Здорово, код компилируется!
Rust довольно сердит, потому что, насколько он может судить, всё, что мы написали, совершенно бесполезно: мы не используем `head` и никто из тех, кто пользуется нашей библиотекой, тоже не сможет этого сделать, потому что это поле приватное.
Транзитивно, это также значит, что List и Node тоже бесполезны.
Так давайте с этим разберёмся!
Напишем немного кода для нашего Списка!