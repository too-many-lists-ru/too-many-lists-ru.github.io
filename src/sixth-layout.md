> # Layout

# Представление

> Let us begin by first studying the structure of our enemy.
> A Doubly-Linked List is conceptually simple, but that's how it decieves and manipulates you.
> It's the same kind of linked list we've looked at over and over, but the links go both ways.
> Double the links, double the evil.

Давайте начнём с изучения структуры нашего врага.
Двусвязный список концептуально прост, но именно так он обманывает вас и манипулирует вами.
Это тот же самый вид связного списка, с которым мы сталкивались снова и снова, но ссылки идут в обе стороны.
Двойные ссылки, двойное зло.

> So rather than this (gonna drop the Some/None stuff to keep it cleaner):

Так что вместо этого (уберу Some/None для большей ясности):

```text
... -> (A, ptr) -> (B, ptr) -> ...
```

> We have this:

Мы имеем:

```text
... <-> (ptr, A, ptr) <-> (ptr, B, ptr) <-> ...
```

> This lets you traverse the list from either direction, or seek back and forth with a [cursor](https://doc.rust-lang.org/std/collections/struct.LinkedList.html#method.cursor_back_mut).

Это позволяет обходить список в любом направлении, или искать вперёд и назад с помощью [курсора](https://doc.rust-lang.org/std/collections/struct.LinkedList.html#method.cursor_back_mut).

> In exchange for this flexibility, every node has to store twice as many pointers, and every operation has to fix up way more pointers.
> It's a significant enough complication that it's a lot easier to make a mistake, so we're going to be doing a lot of testing.

Взамен на эту гибкость, каждый узел вынужден хранить в два раза больше указателей, и каждая операция требует изменения большего количество указателей.
Это достаточно значимое усложнение, при котором гораздо проще допустить ошибку, так что нам придётся написать много дополнительных тестов.

> You might have also noticed that I intentionally haven't drawn the *ends* of the list.
> This is because this is one of the places where there are genuinely defensible options for our implementation.
> We *definitely* need our implementation to have two pointers: one to the start of the list, and one to the end of the list.

Вы должно быть заметили, что я умышленно не говорил о *концах* списка.
Это потому, что это одно из тех мест, где есть действительно обоснованные варианты для нашей реализации.
Нам *определённо* нужно, чтобы наша реализация имела два указателя: один на начало списка и один — на конец.

> There are two notable ways to do this in my mind: "traditional" and "dummy node".

В моей голове есть два известных способа это сделать: «традиционный» и «фиктивный узел».

> The traditional approach is the simple extension of how we did a Stack &mdash; just store the head and tail pointers on the stack:

Традиционный подход — это простое расширение того способа, который мы использовали для Stack — просто хранить указатели на голову и хвост стека:

```text
[ptr, ptr] <-> (ptr, A, ptr) <-> (ptr, B, ptr)
  ^                                        ^
  +----------------------------------------+
```

> This is fine, but it has one downside: corner cases.
> There are now two edges to our list, which means twice as many corner cases.
> It's easy to forget one and have a serious bug.

Это прекрасно, но у этого есть обратная сторона: граничные случаи.
А теперь у нас два конца в нашем списке, что означает в два раза больше граничных случаев.
Легко забыть о каком-то и получить серьёзную ошибку.

> The dummy node approach attempts to smooth out these corner cases by adding an extra node to our list which contains no data but links the two ends together into a ring:

Подход с фиктивным узлом пытается сгладить эти граничные случаи путём добавления внешнего узла в наш список, в котором нет данных, а есть только ссылки на оба конца, что превращает структуру в кольцо:

```text
[ptr] -> (ptr, ?DUMMY?, ptr) <-> (ptr, A, ptr) <-> (ptr, B, ptr)
           ^                                                 ^
           +-------------------------------------------------+ 
```

> By doing this, every node *always* has actual pointers to a previous and next node in the list.
> Even when you remove the last element from the list, you just end up stitching the dummy node to point at itself:

После этого каждый узел *всегда* имеет действительные указатели на предыдущий и следующий узлы в списке.
Даже когда вы удаляете последний элементы из списка, в итоге вы просто замыкаете фиктивный узел сам на себя:

```text
[ptr] -> (ptr, ?DUMMY?, ptr) 
           ^             ^
           +-------------+
```

> There is a part of me that finds this *very* satisfying and elegant.
> Unfortunately, it has a couple practical problems:

Есть часть меня, которая находит это решение *вполне* удовлетворительным и элегантным.
К сожалению, у него есть пара практических проблем:

> Problem 1: An extra indirection and allocation, especially for the empty list, which must include the dummy node.
> Potential solutions include:

Проблема 1: Дополнительные уровень косвенности и выделение памяти, особенно для пустого списка, который должен включать фиктивный узел.
Потенциальные решения включают:

> * Don't allocate the dummy node until something is inserted: simple and effective, but it adds back some of the corner cases we were trying to avoid by using dummy pointers!
> * Use a static copy-on-write empty singleton dummy node, with some really clever scheme that lets the Copy-On-Write checks piggy-back on normal checks: look I'm really tempted, I really do love that shit, but we can't go down that dark path in this book.
>   Read [ThinVec's sourcecode](https://docs.rs/thin-vec/0.2.4/src/thin_vec/lib.rs.html#319-325) if you want to see that kind of perverted stuff.
> * Store the dummy node on the stack - not practical in a language without C++-style move-constructors.
>   I'm sure there's something weird thing we could do here with [pinning](https://doc.rust-lang.org/std/pin/index.html) but we're not gonna.

* Не создавать фиктивный узел до тех пор, пока что-то не будет вставлено: просто и эффективно, но это возвращает некоторые из граничных случаев, от которых мы пытались избавиться, используя фиктивные указатели!
* Использовать статический одиночный фиктивный узел с копированием при записи и с каким-нибудь хитроумным методом, позволяющим совместить проверки Copy-On-Write с обычными проверками: послушайте, меня это очень соблазняет, мне очень нравится это дерьмо, но мы не станем идти по этому тёмному пути в этой книге.
  Читайте [исходный код ThinVec](https://docs.rs/thin-vec/0.2.4/src/thin_vec/lib.rs.html#319-325), если хотите познакомиться с подобной извращённой штукой.
* Хранить фиктивный узел на стеке — не практично в языке, где не поддерживаются 
конструкторы перемещения (move constructors) из C++.
  Я уверена, можно было бы придумать странное решение а [закреплением](https://docs.rs/thin-vec/0.2.4/src/thin_vec/lib.rs.html#319-325) (pin), но мы этого делать не будем.

> Problem 2: What *value* is stored in the dummy node?
> Sure if it's an integer it's fine, but what if we're storing a list full of `Box`?
> It may be impossible for us to initialized this value!
> Potential solutions include:

Проблема 2: Какое *значение* хранить в фиктивном узле?
Ладно бы, речь шла о целом числе, но что если в нашем списке хранятся сплошь экземпляры `Box`?
Мы бы не могли инициализировать такое значение!
Потенциальные решения включают:

> * Make every node store `Option<T>`: simple and effective, but also bloated and annoying.
> * Make every node store [`MaybeUninit<T>`](https://doc.rust-lang.org/std/mem/union.MaybeUninit.html).
>   Horrifying and annoying.
> * *Really* careful and clever inheritance-style type punning so the dummy node doesn't include the data field.
>   This is also tempting but it's extremely dangerous and annoying.
>   Read [BTreeMap's source](https://doc.rust-lang.org/1.55.0/src/alloc/collections/btree/node.rs.html#49-104) if you want to see that kind of perverted stuff.

* Хранить в каждом узле `Option<T>`: просто и эффективно, в то же время неэкономно и раздражающе.
* Хранить в каждом узле [`MaybeUninit<T>`](https://doc.rust-lang.org/std/mem/union.MaybeUninit.html).
  Не только раздражающе, но и ужасающе.
* *По настоящему* осторожно и хитроумно, в стиле наследования сделать так, чтобы фиктивный узел не хранил данных.
  Тоже заманчиво, но крайне опасно и раздражающе.
  Читайте [исходный код BTreeMap](https://doc.rust-lang.org/1.55.0/src/alloc/collections/btree/node.rs.html#49-104) если хотите познакомиться с подобной извращённой штукой.

> The problems really outweigh the convenience for a language like Rust, so we're going to stick to the traditional layout.
> We'll be using the same basic design as we did for the unsafe queue in the previous chapter:

Для такого языка, как Rust проблемы действительно перевешивают удобство, так что мы будем придерживаться традиционного представления.
Будем использовать тот же базовый дизайн, как и в случае с небезопасной очередью из предыдущей главы:

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

> (Now that we have reached the doubly-linked-deque, we have finally earned the right to call ourselves LinkedList, for this is the True Linked List.)

(Теперь, когда мы добрались до двусвязного дека, мы наконец заслужили право назвать структуру LinkedList, поскольку в данном случае речь идёт об истинном связном списке.)

> This isn't quite a *true* production-quality layout yet.
> It's *fine* but there's magic tricks we can do to tell Rust what we're doing a bit better.
> To do that we're going to need to go... deeper.

Это ещё не совсем представление *настоящего* продуктового уровня.
Оно *неплохое*, но есть волшебные приёмы, помогающие лучше объяснить Rust, что мы хотим сделать.
Для этого нам придётся двигаться... глубже.