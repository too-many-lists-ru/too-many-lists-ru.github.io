> # Implementing Cursors

# Реализация Курсоров

> Ok so we're only going to bother with std's CursorMut because the immutable version isn't actually interesting.
> Just like my original design, it has a "ghost" element that contains None to indicate the start/end of the list, and you can "walk over it" to wrap around to the other side of the list.
> To implement it, we're going to need:

Ну что ж, мы займёмся именно CursorMut из стандартной библиотеки, поскольку неизменяемая версия наа самом деле не так интересна.
Как и в оригинальном дизайне, у неё есть «псевдоэлемент», который содержит None, чтобы обозначить начало/конец списка, и вы можете «перейти через него», чтобы перепрыгнуть на другую сторону списка.
Для реализации нам потребуются:

> * A pointer to the current node
> * A pointer to the list
> * The current index

* Указатель на текущий узел
* Указатель на список
* Текущий индекс

> Wait what's the index when we point at the "ghost"? 

Так, а каково должно быть значение индекса, когда мы указываем на «псевдоэлемент»?

> *furrows brow* ... *checks std* ... *dislikes std's answer*

*хмурит брови*... *проверяет стандартную библиотеку*... *не нравится решение из стандартной библиотеки*...

> Ok so quite reasonably `index` on a Cursor returns an `Option<usize>`.
> The std implementation does a bunch of junk to avoid storing it as an Option but... we're a linked list, it's fine.
> Also std has the cursor_front/cursor_back stuff which starts the cursor on the front/back elements, which feels intuitive, but then has to do something weird when the list is empty.

Ладно, вполне логично, что `index` Курсора возвращает `Option<usize>`.
Стандартная реализация делает кучу ненужных вещей, чтобы избежать хранения индекса, как Option, но... у нас же связный список, это вполне нормально.
Так же в стандартной библиотеке есть функции cursor_front/cursor_back, которые передвигают курсор на передний/задний элементы, что кажется интуитивным, пока речь не заходит о пустом списке.

> You can implement that stuff if you want, but I'm going to cut down on all the repetitive gunk and corner cases and just make a bare `cursor_mut` method that starts at the ghost, and people can use move_next/move_prev to get the one they want (and then you can wrap that up as cursor_front if you really want).

Вы можете реализовать такой подход, какой хотите, но я собираюсь выкинуть весь повторяющийся мусор, все граничные случае и написать простой метод `cursor_mut`, который начинается в псевдоэлементе, а люди могут двигать его вперёд/назад для получения позиции, которая им нужна (и они смогут написать свою реализацию cursor_front, если им нужно).

> Let's get cracking:

Давайте приступим:

```rust ,ignore
pub struct CursorMut<'a, T> {
    cur: Link<T>,
    list: &'a mut LinkedList<T>,
    index: Option<usize>,
}
```

> Pretty straight-forward, one field for each item of our bulleted list!
> Now the `cursor_mut` method:

Всё довольно просто: одно поле на каждый пункт нашего списка!
Теперь метод `cursor_mut`:

```rust ,ignore
impl<T> LinkedList<T> {
    pub fn cursor_mut(&mut self) -> CursorMut<T> {
        CursorMut { 
            list: self, 
            cur: None, 
            index: None,
        }
    }
}
```

> Since we're starting at the ghost, we can just start with everything as None, nice and simple!
> Next, movement:

Поскольку мы стартуем на псевдоэлементе, мы можем проинициализировать все поля значением None, красивои и просто!
Далее, перемещение:


```rust ,ignore
impl<'a, T> CursorMut<'a, T> {
    pub fn index(&self) -> Option<usize> {
        self.index
    }

    pub fn move_next(&mut self) {
        if let Some(cur) = self.cur {
            unsafe {
                // We're on a real element, go to its next (back)
                // Мы на реальном элементе, двигаемся к следующему (в сторону заднего)
                self.cur = (*cur.as_ptr()).back;
                if self.cur.is_some() {
                    *self.index.as_mut().unwrap() += 1;
                } else {
                    // We just walked to the ghost, no more index
                    // Мы попали на псевдоэлемент, убираем индекс
                    self.index = None;
                }
            }
        } else if !self.list.is_empty() {
            // We're at the ghost, and there is a real front, so move to it!
            // Мы на псевдоэлементе и у нас есть реальный передний элемент, так что перемещаемся на него!
            self.cur = self.list.front;
            self.index = Some(0)
        } else {
            // We're at the ghost, but that's the only element... do nothing.
            // Мы на псевдоэлементе, но никаких других элементов нет... ничего не делаем.
        }
    }
}
```

> So there's 4 interesting cases:

Итак, есть 4 интересные варианта:

> * The normal case
> * The normal case, but we reach the ghost
> * The ghost case, where we go to the front of the list
> * The ghost case, but the list is empty, so do nothing

* Обычный вариант
* Обычный вариант, но мы перемещаемся к псевдоэлементу
* Вариант псевдоэлемента, когда мы перемещаемся к переднему элементу
* Вариант псевдоэлемента, но список пуст, так что ничего не делаем

> move_prev is the exact same logic, but with front/back inverted and the indexing changes inverted:

В move_prev точно такая же логика, но мы меняем front/back местами и инвертируем  изменение индекса:

```rust ,ignore
pub fn move_prev(&mut self) {
    if let Some(cur) = self.cur {
        unsafe {
            // We're on a real element, go to its previous (front)
            // Мы на реальном элементе, двигаемся к предыдущему (в сторону переднего)
            self.cur = (*cur.as_ptr()).front;
            if self.cur.is_some() {
                *self.index.as_mut().unwrap() -= 1;
            } else {
                // We just walked to the ghost, no more index
                // Мы попали на псевдоэлемент, убираем индекс
                self.index = None;
            }
        }
    } else if !self.list.is_empty() {
        // We're at the ghost, and there is a real back, so move to it!
        // Мы на псевдоэлементе и у нас есть реальный задний элемент, так что перемещаемся на него!
        self.cur = self.list.back;
        self.index = Some(self.list.len - 1)
    } else {
        // We're at the ghost, but that's the only element... do nothing.
        // Мы на псевдоэлементе, но никаких других элементов нет... ничего не делаем.
    }
}
```

> Next let's add some methods to look at the elements around the cursor: current, peek_next, and peek_prev.
> **A Very Important Note:** these methods must borrow our cursor by `&mut self`, and the results must be tied to that borrow.
> We cannot let the user get multiple copies of a mutable reference, and we cannot let them use any of our insert/remove/split/splice APIs while holding onto such a reference!

Далее добавим несколько методов для просмотра элементов вокруг курсора: current, peek_next и peek_prev.
**Очень важное примечание:** эти методы должны заимствовать наш курсор через `&mut self` и результаты должны быть привязаны к этому заимствованию.
Мы не можем позволить пользователю получить несколько копий изменяемой ссылки и мы не можем позволить им использовать любой из методов insert/remove/split/splice, пока у него есть такая ссылка!

> Thankfully, this is the default assumption rust makes when you use lifetime elision, so, we will just do the right thing by default!

К счастью, неявное выведение времени жизни работает в Rust именно так, как нужно, поэтому мы по умолчанию пишем правильный код!

```rust ,ignore
pub fn current(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur.map(|node| &mut (*node.as_ptr()).elem)
    }
}

pub fn peek_next(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur
            .and_then(|node| (*node.as_ptr()).back)
            .map(|node| &mut (*node.as_ptr()).elem)
    }
}

pub fn peek_prev(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur
            .and_then(|node| (*node.as_ptr()).front)
            .map(|node| &mut (*node.as_ptr()).elem)
    }
}
```

> Head empty, Option methods and (omitted) compiler errors do all thinking now.
> I was skeptical about the `Option<NonNull>` stuff, but, god damn it really just lets me autopilot this code.
> I've spent way too much time writing array-based collections where you never get to use Option, wow this is nice!
> (`(*node.as_ptr())` is still miserable but, that's just Rust's raw pointers for you...)

Даже думать не пришлось, всю работу сделали методы Option и (опущенные) ошибки компилятора.
Я была настроена скептически по отношению к `Option<NonNull>`, но, будь я проклята, этот тип позволил мне писать код на автопилоте.
Я потратила слишком много времени на написание коллекций на основе массивов, где вам никогда не приходится использовать Option, так что мне есть, с чем сравнивать!
(`(*node.as_ptr())`, конечно, выглядит кринжово, но, с другой стороны, для вас это всего лишь обычные указатели Rust...)

> Next we have a choice: we can either jump right to split and splice, the entire point of these APIs, or we can take a baby-step with single element insert/remove.
> I have a feeling we're just going to want to implement insert/remove in terms of split and splice so... let's just do those first and see where the cards fall (genuinely have no idea as I type this).

Далее у нас есть выбор: мы можем сразу заняться split и splice, центральной темой этого API, или мы можем сделать небольшой шажок, реализовав insert/remove для одного элемента.
У меня такое чувство, что нам просто надо будет реализовать insert/remove через split и splice, так что... давайте просто попробуем и посмотрим, как ляжет карта (честно говоря, понятия не имею, пока пишу эти строки).

> # Split

# Метод split

> First up, split_before and split_after, which return everything before/after the current element as a LinkedList (stopping at the ghost element, unless you're at the ghost, in which case we just return the whole List and the cursor now points to an empty list):

Прежде всего, split_before и split_after, которые возвращают всё до/после текущего элемента в виде LinkedList (и останавливаются на псевдоэлементе, а если они сразу находятся на псевдоэлементе, то возвращают весь список, а курсор после этого указывает на пустой список):

> *squints* ok this one is actually some non-trivial logic so we're going to have to talk it out one step at a time.

*прищуривается* ладно, здесь и правда есть нетривиальная логика, так что давайте разобьём её на части.

> I see 4 potentially interesting cases for split_before:

Я вижу 4 потенциально интересных варианта для split_before:

> * The normal case
> * The normal case, but prev is the ghost
> * The ghost case, where we return the whole list and become empty
> * The ghost case, but the list is empty, so do nothing and return the empty list

* Обычный вариант
* Обычный вариант, но предыдущий элемент является псевдоэлементом
* Вариант псевдоэлемента, где мы возвращаем весь список, а оставляем пустой
* Вариант псевдоэлемента, но список пустой, так что мы ничего не делаем и возвращаем пустой список

> Let's start with the corner cases.
> The third case I believe is just

Начнём с граничных вариантов.
Третий вариант, мне кажется, самый простой.

```rust
mem::replace(self.list, LinkedList::new())
```

> Right?
> We become empty, we return the whole list, and our fields were already None, so nothing to update.
> Nice.
> Oh hey, this also Does The Right Thing on the fourth case too!

Так?
Список становится пустым, мы возвращаем весь предыдущий список и теперь наши поля хранят None, так что нам нечего исправлять.
Прекрасно.
О, кстати, этот метод закрывает и четвёртый вариант!

> So now the normal cases... ok I'm going to need some ASCII diagrams for this.
> In the most general case, we have something like this:

Так, теперь обычные варианты... ладно, здесь мне потребуется несколько ASCII-диаграмм.
В самом общем случае, у нас есть что-то такое:

```text
list.front -> A <-> B <-> C <-> D <- list.back
                          ^
                         cur
```

> And we want to produce this:

А мы хотим получить вот это:

```text
list.front -> C <-> D <- list.back
              ^
             cur

return.front -> A <-> B <- return.back
```

> So we need to break the link between cur and prev, and... god so much needs to change.
> Ok I just need to break this up into steps so I can convince myself it makes sense.
> This will be a bit over-verbose but I can at least make sense of it:

Так что нам надо разорвать связь между ruc и prev, и... боже, сколько всего надо менять.
Ладно, мне просто надо разбить весь путь на отдельные шаги, чтобы быть уверенной, что я нигде не ошиблась.
Будет чуток многословно, но я по крайней мере ничего не пропущу:

```rust ,ignore
pub fn split_before(&mut self) -> LinkedList<T> {
    if let Some(cur) = self.cur {
        // We are pointing at a real element, so the list is non-empty.
        // Указываем на реальный элемент, так что список не пустой.
        unsafe {
            // Current state
            // Текущее состояние
            let old_len = self.list.len;
            let old_idx = self.index.unwrap();
            let prev = (*cur.as_ptr()).front;
            
            // What self will become
            // Во что должен превратиться self
            let new_len = old_len - old_idx;
            let new_front = self.cur;
            let new_back = self.list.back;
            let new_idx = Some(0);

            // What the output will become
            // Что мы должны вернуть
            let output_len = old_len - new_len;
            let output_front = self.list.front;
            let output_back = prev;

            // Break the links between cur and prev
            // Разрываем связь между cur и prev
            if let Some(prev) = prev {
                (*cur.as_ptr()).front = None;
                (*prev.as_ptr()).back = None;
            }

            // Produce the result:
            // Создаём результат
            self.list.len = new_len;
            self.list.front = new_front;
            self.list.back = new_back;
            self.index = new_idx;

            LinkedList {
                front: output_front,
                back: output_back,
                len: output_len,
                _boo: PhantomData,
            }
        }
    } else {
        // We're at the ghost, just replace our list with an empty one.
        // No other state needs to be changed.
        // Мы на псевдолементе, просто меняем наш список на пустой.
        // Никаких других изменений состояния не нужно.
        std::mem::replace(self.list, LinkedList::new())
    }
}
```

> Note that this if-let is handling the "normal case, but prev is the ghost" situation:

Обратите внимание, что конструкция if-let помогает справиться с «обычным вариантом, когда предыдущий элемент — это псевдоэлемент»:

```rust ,ignore
if let Some(prev) = prev {
    (*cur.as_ptr()).front = None;
    (*prev.as_ptr()).back = None;
}
```

> If *you* want to, you can squash that all together and apply optimizations like:

Если *вы* хотите, вы можете рассмотреть весь код, как единое целое и сделать такие оптимизации:

> * fold the two accesses to `(*cur.as_ptr()).front` as just `(*cur.as_ptr()).front.take()` 
> * note that new_back is a noop, and just remove both

* заменить два обращения к `(*cur.as_ptr()).front` на вызов `(*cur.as_ptr()).front.take()`
* заметить, что new_back не выполняет никакой работы, и её можно совсем удалить

> As far as I can tell, everything else just incidentally Does The Right Thing otherwise.
> We'll see when we write tests!
> (copy-paste to make split_after)

Насколько я могу судить, весь остальной код должен работать.
Узнаем, когда напишем тесты.
(копи-пастим метод split_after)

> I am done Making Mistakes and I am just going to try to write the most foolproof code I can.
> This is how I *actually* write collections: just break things down into trivial steps and cases until it can fit in my head and seems foolproof.
> Then write a ton of tests until I'm convinced I didn't manage to mess it up still.

Я больше не хочу допускать ошибки, я просто хочу постараться написать самый надёжный код, который могу.
Вот как я *на самом деле* пишу коллекции: для начала разбивают задачу на тривиальные шаги и варианты, пока они не становятся понятными не помещаются в моей голове.
Затем пишу тонну тестов, пока не буду уверена, что всё-таки не напортачила.

> Because most of the collections work I've done is *extremely unsafe* I don't generally get to rely on the compiler catching mistakes, and miri didn't exist back in the day!
> So I just need to squint at a problem until my head hurts and try my hardest to Never Ever Ever Make A Mistake.

Поскольку большая часть моей работы с коллекциями *крайней небезопасна*, я, в целом, не могу полагаться на то, что компилятор обнаружит ошибки, а miri в то время ещё небыло!
Поэтому мне приходится изучать проблему, пока у меня не заболит голова и стараться изо всех сил, чтобы Никогда Никогда Никогда не делать ошибок.

> Don't write Unsafe Rust Code!
> Safe Rust is so much better!!!!

Не пишите на Rust небезопасный код!
Безопасный Rust гораздо лучше!!!




> # Splice

> Just one more boss to fight, splice_before and splice_after, which I expect to be the corner-casiest one of them all.
> The two functions *take in* a LinkedList and grafts its contents into outrs.
> Our list could be empty, their list could be empty, we've got ghosts to deal with... *sigh* let's just take it one step at a time with splice_before.

> * If their list is empty, we don't need to do anything. 
> * If our list is empty, then our list just becomes their list.
> * If we're pointing at the ghost, then this appends to the back (change list.back)
> * If we're pointing at the first element (0), this this appends to the front (change list.front)
> * In the general case, we do a whole lot of pointer fuckery.

> The general case is this:

```text
input.front -> 1 <-> 2 <- input.back

 list.front -> A <-> B <-> C <- list.back
                     ^
                    cur
```

Becoming this:

```text
list.front -> A <-> 1 <-> 2 <-> B <-> C <- list.back
```

> Ok?
> Ok.
> Let's write that out... *TAKES A HUGE BREATH AND PLUNGES IN*:

```rust ,ignore
    pub fn splice_before(&mut self, mut input: LinkedList<T>) {
        unsafe {
            if input.is_empty() {
                // Input is empty, do nothing.
            } else if let Some(cur) = self.cur {
                if let Some(0) = self.index {
                    // We're appending to the front, see append to back
                    (*cur.as_ptr()).front = input.back.take();
                    (*input.back.unwrap().as_ptr()).back = Some(cur);
                    self.list.front = input.front.take();

                    // Index moves forward by input length
                    *self.index.as_mut().unwrap() += input.len;
                    self.list.len += input.len;
                    input.len = 0;
                } else {
                    // General Case, no boundaries, just internal fixups
                    let prev = (*cur.as_ptr()).front.unwrap();
                    let in_front = input.front.take().unwrap();
                    let in_back = input.back.take().unwrap();

                    (*prev.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(prev);
                    (*cur.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(cur);

                    // Index moves forward by input length
                    *self.index.as_mut().unwrap() += input.len;
                    self.list.len += input.len;
                    input.len = 0;
                }
            } else if let Some(back) = self.list.back {
                // We're on the ghost but non-empty, append to the back
                // We can either `take` the input's pointers or `mem::forget`
                // it. Using take is more responsible in case we do custom
                // allocators or something that also needs to be cleaned up!
                (*back.as_ptr()).back = input.front.take();
                (*input.front.unwrap().as_ptr()).front = Some(back);
                self.list.back = input.back.take();
                self.list.len += input.len;
                // Not necessary but Polite To Do
                input.len = 0;
            } else {
                // We're empty, become the input, remain on the ghost
                *self.list = input;
            }
        }
    }
```

> Ok this one is genuinely horrendous, and really is feeling that `Option<NonNull>` pain now.
> But there's a lot of cleanups we can do.
> For one, we can pull this code out to the very end, because we always want to do it.
> I don't *love*  (although sometimes it's a noop, and setting `input.len` is more a matter of paranoia about future extensions to the code):

```rust ,ignore
self.list.len += input.len;
input.len = 0;
```

> > Use of moved value: `input`

> Ah, right, in the "we're empty" case we're moving the list.
> Let's replace that with a swap:

```rust ,ignore
// We're empty, become the input, remain on the ghost
std::mem::swap(self.list, &mut input);
```

> In this case the writes will be pointless, but, they still work (we could probably also early-return in this branch to appease the compiler).

> This unwrap is just a consequence of me thinking about the cases backwards, and can be fixed by making the if-let ask the right question:

```rust ,ignore
if let Some(0) = self.index {

} else {
    let prev = (*cur.as_ptr()).front.unwrap();
}
```

> Adjusting the index is duplicated inside the branches, so can also be hoisted out:

```rust
*self.index.as_mut().unwrap() += input.len;
```

> Ok, putting that all together we get this:

```rust
if input.is_empty() {
    // Input is empty, do nothing.
} else if let Some(cur) = self.cur {
    // Both lists are non-empty
    if let Some(prev) = (*cur.as_ptr()).front {
        // General Case, no boundaries, just internal fixups
        let in_front = input.front.take().unwrap();
        let in_back = input.back.take().unwrap();

        (*prev.as_ptr()).back = Some(in_front);
        (*in_front.as_ptr()).front = Some(prev);
        (*cur.as_ptr()).front = Some(in_back);
        (*in_back.as_ptr()).back = Some(cur);
    } else {
        // We're appending to the front, see append to back below
        (*cur.as_ptr()).front = input.back.take();
        (*input.back.unwrap().as_ptr()).back = Some(cur);
        self.list.front = input.front.take();
    }
    // Index moves forward by input length
    *self.index.as_mut().unwrap() += input.len;
} else if let Some(back) = self.list.back {
    // We're on the ghost but non-empty, append to the back
    // We can either `take` the input's pointers or `mem::forget`
    // it. Using take is more responsible in case we do custom
    // allocators or something that also needs to be cleaned up!
    (*back.as_ptr()).back = input.front.take();
    (*input.front.unwrap().as_ptr()).front = Some(back);
    self.list.back = input.back.take();

} else {
    // We're empty, become the input, remain on the ghost
    std::mem::swap(self.list, &mut input);
}

self.list.len += input.len;
// Not necessary but Polite To Do
input.len = 0;

// Input dropped here
```

> Alright this still sucks, but mostly because of -- nope ok just spotted a bug:

```rust
    (*back.as_ptr()).back = input.front.take();
    (*input.front.unwrap().as_ptr()).front = Some(back);
```

> We `take` input.front and then unwrap it on the next line! *sigh* and we do the same thing in the equivalent mirror case. We would have caught this instantly in tests, but, we're trying to be Perfect now, and I'm just kinda doing this live, and this is the exact moment where I saw it.
> This is what I get for not being my usual tedious self and doing things in phases.
> More explicit!

```rust
// We can either `take` the input's pointers or `mem::forget`
// it. Using `take` is more responsible in case we ever do custom
// allocators or something that also needs to be cleaned up!
if input.is_empty() {
    // Input is empty, do nothing.
} else if let Some(cur) = self.cur {
    // Both lists are non-empty
    let in_front = input.front.take().unwrap();
    let in_back = input.back.take().unwrap();

    if let Some(prev) = (*cur.as_ptr()).front {
        // General Case, no boundaries, just internal fixups
        (*prev.as_ptr()).back = Some(in_front);
        (*in_front.as_ptr()).front = Some(prev);
        (*cur.as_ptr()).front = Some(in_back);
        (*in_back.as_ptr()).back = Some(cur);
    } else {
        // No prev, we're appending to the front
        (*cur.as_ptr()).front = Some(in_back);
        (*in_back.as_ptr()).back = Some(cur);
        self.list.front = Some(in_front);
    }
    // Index moves forward by input length
    *self.index.as_mut().unwrap() += input.len;
} else if let Some(back) = self.list.back {
    // We're on the ghost but non-empty, append to the back
    let in_front = input.front.take().unwrap();
    let in_back = input.back.take().unwrap();

    (*back.as_ptr()).back = Some(in_front);
    (*in_front.as_ptr()).front = Some(back);
    self.list.back = Some(in_back);
} else {
    // We're empty, become the input, remain on the ghost
    std::mem::swap(self.list, &mut input);
}

self.list.len += input.len;
// Not necessary but Polite To Do
input.len = 0;

// Input dropped here
```

> Alright now this, this I can tolerate.
> The only complaints I have are that we don't dedupe in_front/in_back (probably we could rejig our conditions but eh whatever).
> Really this is basically what you would write in C but with `Option<NonNull>` gunk making it tedious.
> I can live with that.
> Well no we should just make raw pointers better for this stuff.
> But, out of scope for this book.

> Anyway, I am absolutely exhausted after that, so, `insert` and `remove` and all the other APIs can be left as an excercise to the reader. 

> Here's the final code for our Cursor with my attempt at copy-pasting the combinatorics.
> Did I get it right?
> I'll only find out when I write the next chapter and test this monstrosity!


```rust ,ignore
pub struct CursorMut<'a, T> {
    list: &'a mut LinkedList<T>,
    cur: Link<T>,
    index: Option<usize>,
}

impl<T> LinkedList<T> {
    pub fn cursor_mut(&mut self) -> CursorMut<T> {
        CursorMut { 
            list: self, 
            cur: None, 
            index: None,
        }
    }
}

impl<'a, T> CursorMut<'a, T> {
    pub fn index(&self) -> Option<usize> {
        self.index
    }

    pub fn move_next(&mut self) {
        if let Some(cur) = self.cur {
            unsafe {
                // We're on a real element, go to its next (back)
                self.cur = (*cur.as_ptr()).back;
                if self.cur.is_some() {
                    *self.index.as_mut().unwrap() += 1;
                } else {
                    // We just walked to the ghost, no more index
                    self.index = None;
                }
            }
        } else if !self.list.is_empty() {
            // We're at the ghost, and there is a real front, so move to it!
            self.cur = self.list.front;
            self.index = Some(0)
        } else {
            // We're at the ghost, but that's the only element... do nothing.
        }
    }

    pub fn move_prev(&mut self) {
        if let Some(cur) = self.cur {
            unsafe {
                // We're on a real element, go to its previous (front)
                self.cur = (*cur.as_ptr()).front;
                if self.cur.is_some() {
                    *self.index.as_mut().unwrap() -= 1;
                } else {
                    // We just walked to the ghost, no more index
                    self.index = None;
                }
            }
        } else if !self.list.is_empty() {
            // We're at the ghost, and there is a real back, so move to it!
            self.cur = self.list.back;
            self.index = Some(self.list.len - 1)
        } else {
            // We're at the ghost, but that's the only element... do nothing.
        }
    }

    pub fn current(&mut self) -> Option<&mut T> {
        unsafe {
            self.cur.map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn peek_next(&mut self) -> Option<&mut T> {
        unsafe {
            self.cur
                .and_then(|node| (*node.as_ptr()).back)
                .map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn peek_prev(&mut self) -> Option<&mut T> {
        unsafe {
            self.cur
                .and_then(|node| (*node.as_ptr()).front)
                .map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn split_before(&mut self) -> LinkedList<T> {
        // We have this:
        //
        //     list.front -> A <-> B <-> C <-> D <- list.back
        //                               ^
        //                              cur
        // 
        //
        // And we want to produce this:
        // 
        //     list.front -> C <-> D <- list.back
        //                   ^
        //                  cur
        //
        // 
        //    return.front -> A <-> B <- return.back
        //
        if let Some(cur) = self.cur {
            // We are pointing at a real element, so the list is non-empty.
            unsafe {
                // Current state
                let old_len = self.list.len;
                let old_idx = self.index.unwrap();
                let prev = (*cur.as_ptr()).front;
                
                // What self will become
                let new_len = old_len - old_idx;
                let new_front = self.cur;
                let new_back = self.list.back;
                let new_idx = Some(0);

                // What the output will become
                let output_len = old_len - new_len;
                let output_front = self.list.front;
                let output_back = prev;

                // Break the links between cur and prev
                if let Some(prev) = prev {
                    (*cur.as_ptr()).front = None;
                    (*prev.as_ptr()).back = None;
                }

                // Produce the result:
                self.list.len = new_len;
                self.list.front = new_front;
                self.list.back = new_back;
                self.index = new_idx;

                LinkedList {
                    front: output_front,
                    back: output_back,
                    len: output_len,
                    _boo: PhantomData,
                }
            }
        } else {
            // We're at the ghost, just replace our list with an empty one.
            // No other state needs to be changed.
            std::mem::replace(self.list, LinkedList::new())
        }
    }

    pub fn split_after(&mut self) -> LinkedList<T> {
        // We have this:
        //
        //     list.front -> A <-> B <-> C <-> D <- list.back
        //                         ^
        //                        cur
        // 
        //
        // And we want to produce this:
        // 
        //     list.front -> A <-> B <- list.back
        //                         ^
        //                        cur
        //
        // 
        //    return.front -> C <-> D <- return.back
        //
        if let Some(cur) = self.cur {
            // We are pointing at a real element, so the list is non-empty.
            unsafe {
                // Current state
                let old_len = self.list.len;
                let old_idx = self.index.unwrap();
                let next = (*cur.as_ptr()).back;
                
                // What self will become
                let new_len = old_idx + 1;
                let new_back = self.cur;
                let new_front = self.list.front;
                let new_idx = Some(old_idx);

                // What the output will become
                let output_len = old_len - new_len;
                let output_front = next;
                let output_back = self.list.back;

                // Break the links between cur and next
                if let Some(next) = next {
                    (*cur.as_ptr()).back = None;
                    (*next.as_ptr()).front = None;
                }

                // Produce the result:
                self.list.len = new_len;
                self.list.front = new_front;
                self.list.back = new_back;
                self.index = new_idx;

                LinkedList {
                    front: output_front,
                    back: output_back,
                    len: output_len,
                    _boo: PhantomData,
                }
            }
        } else {
            // We're at the ghost, just replace our list with an empty one.
            // No other state needs to be changed.
            std::mem::replace(self.list, LinkedList::new())
        }
    }

    pub fn splice_before(&mut self, mut input: LinkedList<T>) {
        // We have this:
        //
        // input.front -> 1 <-> 2 <- input.back
        //
        // list.front -> A <-> B <-> C <- list.back
        //                     ^
        //                    cur
        //
        //
        // Becoming this:
        //
        // list.front -> A <-> 1 <-> 2 <-> B <-> C <- list.back
        //                                 ^
        //                                cur
        //
        unsafe {
            // We can either `take` the input's pointers or `mem::forget`
            // it. Using `take` is more responsible in case we ever do custom
            // allocators or something that also needs to be cleaned up!
            if input.is_empty() {
                // Input is empty, do nothing.
            } else if let Some(cur) = self.cur {
                // Both lists are non-empty
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                if let Some(prev) = (*cur.as_ptr()).front {
                    // General Case, no boundaries, just internal fixups
                    (*prev.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(prev);
                    (*cur.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(cur);
                } else {
                    // No prev, we're appending to the front
                    (*cur.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(cur);
                    self.list.front = Some(in_front);
                }
                // Index moves forward by input length
                *self.index.as_mut().unwrap() += input.len;
            } else if let Some(back) = self.list.back {
                // We're on the ghost but non-empty, append to the back
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                (*back.as_ptr()).back = Some(in_front);
                (*in_front.as_ptr()).front = Some(back);
                self.list.back = Some(in_back);
            } else {
                // We're empty, become the input, remain on the ghost
                std::mem::swap(self.list, &mut input);
            }

            self.list.len += input.len;
            // Not necessary but Polite To Do
            input.len = 0;
            
            // Input dropped here
        }        
    }

    pub fn splice_after(&mut self, mut input: LinkedList<T>) {
        // We have this:
        //
        // input.front -> 1 <-> 2 <- input.back
        //
        // list.front -> A <-> B <-> C <- list.back
        //                     ^
        //                    cur
        //
        //
        // Becoming this:
        //
        // list.front -> A <-> B <-> 1 <-> 2 <-> C <- list.back
        //                     ^
        //                    cur
        //
        unsafe {
            // We can either `take` the input's pointers or `mem::forget`
            // it. Using `take` is more responsible in case we ever do custom
            // allocators or something that also needs to be cleaned up!
            if input.is_empty() {
                // Input is empty, do nothing.
            } else if let Some(cur) = self.cur {
                // Both lists are non-empty
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                if let Some(next) = (*cur.as_ptr()).back {
                    // General Case, no boundaries, just internal fixups
                    (*next.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(next);
                    (*cur.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(cur);
                } else {
                    // No next, we're appending to the back
                    (*cur.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(cur);
                    self.list.back = Some(in_back);
                }
                // Index doesn't change
            } else if let Some(front) = self.list.front {
                // We're on the ghost but non-empty, append to the front
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                (*front.as_ptr()).front = Some(in_back);
                (*in_back.as_ptr()).back = Some(front);
                self.list.front = Some(in_front);
            } else {
                // We're empty, become the input, remain on the ghost
                std::mem::swap(self.list, &mut input);
            }

            self.list.len += input.len;
            // Not necessary but Polite To Do
            input.len = 0;
            
            // Input dropped here
        }        
    }
}
```
