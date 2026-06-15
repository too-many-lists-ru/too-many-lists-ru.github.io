# Реализация Курсоров

Мы займёмся типом CursorMut из стандартной библиотеки, поскольку неизменяемая версия на самом деле не так интересна.
Как и в оригинальном дизайне, у неё есть «псевдоэлемент», который содержит None, чтобы обозначить начало/конец списка, и через который можно «пройти», чтобы перепрыгнуть на другую сторону списка.
Для реализации нам потребуются:

* Указатель на текущий узел
* Указатель на список
* Текущий индекс

Так, а каково должно быть значение индекса, когда мы указываем на «псевдоэлемент»?

*хмурит брови*... *проверяет стандартную библиотеку*... *отвергает решение из стандартной библиотеки*...

Ладно, вполне логично, что `index` Курсора возвращает `Option<usize>`.
Стандартная реализация делает кучу ненужных вещей, чтобы избежать хранения индекса, как Option, но... у нас же связный список, это вполне нормально.
Так же в стандартной библиотеке есть функции cursor_front/cursor_back, которые передвигают курсор на передний/задний элементы, что кажется интуитивным, пока речь не заходит о пустом списке.

Вы можете написать такой код, какой захотите, но я собираюсь выкинуть весь повторяющийся мусор, все граничные случае и сделать простой метод `cursor_mut`, который начинается в псевдоэлементе, и который можно передвигать вперёд/назад, чтобы получить нужную позицию.
И, если, очень хотите, можете написать свой cursor_front.

Приступим:

```rust ,ignore
pub struct CursorMut<'a, T> {
    cur: Link<T>,
    list: &'a mut LinkedList<T>,
    index: Option<usize>,
}
```

Всё довольно просто: одно поле на каждый пункт нашего списка!
Теперь напишем метод `cursor_mut`:

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

Поскольку мы стартуем на псевдоэлементе, мы можем проинициализировать все поля значением None: красиво и и просто!
Далее, перемещение:

```rust ,ignore
impl<'a, T> CursorMut<'a, T> {
    pub fn index(&self) -> Option<usize> {
        self.index
    }

    pub fn move_next(&mut self) {
        if let Some(cur) = self.cur {
            unsafe {
                // Мы на реальном элементе, двигаемся к следующему (в сторону заднего)
                self.cur = (*cur.as_ptr()).back;
                if self.cur.is_some() {
                    *self.index.as_mut().unwrap() += 1;
                } else {
                    // Мы попали на псевдоэлемент, убираем индекс
                    self.index = None;
                }
            }
        } else if !self.list.is_empty() {
            // Мы на псевдоэлементе и у нас есть реальный передний элемент, так что перемещаемся на него!
            self.cur = self.list.front;
            self.index = Some(0)
        } else {
            // Мы на псевдоэлементе, но никаких других элементов нет... ничего не делаем.
        }
    }
}
```

Итак, есть 4 интересные варианта:

* Обычный вариант
* Обычный вариант, но мы перемещаемся к псевдоэлементу
* Вариант псевдоэлемента, когда мы перемещаемся к переднему элементу
* Вариант псевдоэлемента, но список пуст, так что ничего не делаем

В move_prev точно такая же логика, но мы меняем front/back местами и инвертируем  изменение индекса:

```rust ,ignore
pub fn move_prev(&mut self) {
    if let Some(cur) = self.cur {
        unsafe {
            // Мы на реальном элементе, двигаемся к предыдущему (в сторону переднего)
            self.cur = (*cur.as_ptr()).front;
            if self.cur.is_some() {
                *self.index.as_mut().unwrap() -= 1;
            } else {
                // Мы попали на псевдоэлемент, убираем индекс
                self.index = None;
            }
        }
    } else if !self.list.is_empty() {
        // Мы на псевдоэлементе и у нас есть реальный задний элемент, так что перемещаемся на него!
        self.cur = self.list.back;
        self.index = Some(self.list.len - 1)
    } else {
        // Мы на псевдоэлементе, но никаких других элементов нет... ничего не делаем.
    }
}
```

Добавим методы для просмотра элементов вокруг курсора: current, peek_next и peek_prev.
**Очень важное примечание:** эти методы должны заимствовать наш курсор через `&mut self` и результаты должны быть привязаны к этому заимствованию.
Мы не можем позволить пользователю получить несколько копий изменяемой ссылки и мы не можем позволить ему использовать любой из методов insert/remove/split/splice, пока у него есть такая ссылка!

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

Даже думать не пришлось, всю работу сделали методы Option и (опущенные) ошибки компилятора.
Я была настроена скептически по отношению к `Option<NonNull>`, но, будь я проклята, этот тип позволил мне писать код на автопилоте.
Я потратила слишком много времени на написание коллекций на основе массивов, где вам никогда не приходится использовать Option, так что мне есть, с чем сравнивать!
(`(*node.as_ptr())`, конечно, выглядит кринжово, но, с другой стороны, для вас это всего лишь обычные указатели Rust...)

Далее у нас есть выбор: мы можем сразу заняться split и splice, центральной темой этого API, или мы можем сделать небольшой шажок, реализовав insert/remove для одного элемента.
У меня такое чувство, что нам придётся реализовать insert/remove через split и splice, так что... давайте попробуем и посмотрим, как ляжет карта (честно говоря, понятия не имею, пока пишу эти строки).

# Метод split

> First up, split_before and split_after, which return everything before/after the current element as a LinkedList (stopping at the ghost element, unless you're at the ghost, in which case we just return the whole List and the cursor now points to an empty list):

Итак, split_before и split_after возвращают всё до/после текущего элемента в виде LinkedList (и останавливаются на псевдоэлементе, а если они сразу находятся на псевдоэлементе, то возвращают весь список, а курсор после этого указывает на пустой список).

*прищуривается* ладно, здесь и правда есть нетривиальная логика, так что давайте разобьём её на части.

Я вижу 4 потенциально интересных варианта для split_before:

* Обычный вариант
* Обычный вариант, но предыдущий элемент является псевдоэлементом
* Вариант псевдоэлемента, где мы возвращаем весь список, а оставляем пустой
* Вариант псевдоэлемента, но список пустой, так что ничего не делаем и возвращаем пустой список

Начнём с граничных вариантов.
Третий вариант, мне кажется, самый простой.

```rust
mem::replace(self.list, LinkedList::new())
```

Так?
Список становится пустым, мы возвращаем весь предыдущий список и теперь наши поля хранят None, так что нам нечего исправлять.
Прекрасно.
О, кстати, этот метод закрывает и четвёртый вариант!

Так, теперь обычные варианты... ладно, здесь мне потребуется несколько ASCII-диаграмм.
В самом общем случае, у нас есть что-то такое:

```text
list.front -> A <-> B <-> C <-> D <- list.back
                          ^
                         cur
```

А мы хотим получить вот это:

```text
list.front -> C <-> D <- list.back
              ^
             cur

return.front -> A <-> B <- return.back
```

Так что нам надо разорвать связь между cur и prev, и... боже, сколько всего надо менять.
Ладно, мне просто надо разбить весь путь на отдельные шаги, чтобы быть уверенной, что я нигде не ошиблась.
Будет чуток многословно, но я по крайней мере ничего не пропущу:

```rust ,ignore
pub fn split_before(&mut self) -> LinkedList<T> {
    if let Some(cur) = self.cur {
        // Указываем на реальный элемент, так что список не пустой.
        unsafe {
            // Текущее состояние
            let old_len = self.list.len;
            let old_idx = self.index.unwrap();
            let prev = (*cur.as_ptr()).front;
            
            // Во что должен превратиться self
            let new_len = old_len - old_idx;
            let new_front = self.cur;
            let new_back = self.list.back;
            let new_idx = Some(0);

            // Что мы должны вернуть
            let output_len = old_len - new_len;
            let output_front = self.list.front;
            let output_back = prev;

            // Разрываем связь между cur и prev
            if let Some(prev) = prev {
                (*cur.as_ptr()).front = None;
                (*prev.as_ptr()).back = None;
            }

            // Создаём результат:
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
        // Мы на псевдолементе, просто меняем наш список на пустой.
        // Никаких других изменений состояния не нужно.
        std::mem::replace(self.list, LinkedList::new())
    }
}
```

Обратите внимание, что конструкция if-let помогает справиться с «обычным вариантом, когда предыдущий элемент — это псевдоэлемент»:

```rust ,ignore
if let Some(prev) = prev {
    (*cur.as_ptr()).front = None;
    (*prev.as_ptr()).back = None;
}
```

Если *вы* хотите, можете рассмотреть весь код, как единое целое и сделать такие оптимизации:

* заменить два обращения к `(*cur.as_ptr()).front` на вызов `(*cur.as_ptr()).front.take()`
* заметить, что new_back не выполняет никакой работы, поэтому переменную можно удалить

Насколько я могу судить, оставшийся код должен работать.
Узнаем, когда напишем тесты.
(копи-пастим метод split_after)

Я больше не хочу совершать ошибки, я хочу писать самый надёжный код, который могу.
Вот как я *на самом деле* разрабатывала коллекции: для начала разбивала задачу на тривиальные шаги и варианты, пока они не становились понятными и не помещались у меня в голове.
Затем писала тонну тестов, чтобы быть уверенной, что не напортачила.

Поскольку большая часть моей работы с коллекциями *крайней небезопасна*, я, в целом, не могла полагаться на то, что компилятор обнаружит ошибки, а miri в то время ещё небыло!
Поэтому мне приходилось изучать проблему, пока у меня не начинала болеть голова и стараться изо всех сил, чтобы Никогда Никогда Никогда не совершать ошибок.

Не пишите на Rust небезопасный код!
Безопасный Rust гораздо лучше!!!

# Вставка

> Just one more boss to fight, splice_before and splice_after, which I expect to be the corner-casiest one of them all.
> The two functions *take in* a LinkedList and grafts its contents into outrs.
> Our list could be empty, their list could be empty, we've got ghosts to deal with... *sigh* let's just take it one step at a time with splice_before.

Остался последний босс, с которым нужно сразиться — splice_before и splice_after, которые, похоже, будут содержать больше всего граничных случаев.
Эти функции *принимают на вход* один LinkedList и вставляют его содержимое в наш список.
Наш список может быть пустым, их список может быть пустым, нужно что-то решать с псевдоэлементами...
*вздыхает* Давайте просто действовать шаг за шагом на примере splice_before.

> * If their list is empty, we don't need to do anything. 
> * If our list is empty, then our list just becomes their list.
> * If we're pointing at the ghost, then this appends to the back (change list.back)
> * If we're pointing at the first element (0), this this appends to the front (change list.front)
> * In the general case, we do a whole lot of pointer fuckery.

* Если их список пустой, мы ничего не делаем
* Если наш список пустой, тогда их список становится нашим списком
* Если мы указываем на псевдоэлемент, их список добавляется сзади (изменяя list.back)
* Если мы указываем на первый элемент (0), их список добавляется впереди (изменяя list.front)
* В других случаях, мы очень много возимся с указателями.

> The general case is this:

Общий случай:

```text
input.front -> 1 <-> 2 <- input.back

 list.front -> A <-> B <-> C <- list.back
                     ^
                    cur
```

> Becoming this:

Превращается в:

```text
list.front -> A <-> 1 <-> 2 <-> B <-> C <- list.back
```

> Ok?
> Ok.
> Let's write that out... *TAKES A HUGE BREATH AND PLUNGES IN*:

Так?
Так.
Будем писать...
***ГЛУБОКО ВЗДЫХАЕТ И ПИШЕТ*:

```rust ,ignore
    pub fn splice_before(&mut self, mut input: LinkedList<T>) {
        unsafe {
            if input.is_empty() {
                // Пустой входной список, ничего не делаем
            } else if let Some(cur) = self.cur {
                if let Some(0) = self.index {
                    // We're appending to the front, see append to back
                    // Добавляем спереди, сравните с добавлением сзади
                    (*cur.as_ptr()).front = input.back.take();
                    (*input.back.unwrap().as_ptr()).back = Some(cur);
                    self.list.front = input.front.take();

                    // Index moves forward by input length
                    // Индекс увеличивается на длину входного списка
                    *self.index.as_mut().unwrap() += input.len;
                    self.list.len += input.len;
                    input.len = 0;
                } else {
                    // General Case, no boundaries, just internal fixups
                    // Общий случай, никаких граничных случаев:
                    // просто правим приватные поля
                    let prev = (*cur.as_ptr()).front.unwrap();
                    let in_front = input.front.take().unwrap();
                    let in_back = input.back.take().unwrap();

                    (*prev.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(prev);
                    (*cur.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(cur);

                    // Index moves forward by input length
                    // Индекс увеличивается на длину входного списка
                    *self.index.as_mut().unwrap() += input.len;
                    self.list.len += input.len;
                    input.len = 0;
                }
            } else if let Some(back) = self.list.back {
                // We're on the ghost but non-empty, append to the back
                // We can either `take` the input's pointers or `mem::forget`
                // it. Using take is more responsible in case we do custom
                // allocators or something that also needs to be cleaned up!
                // Мы на псевдоэлементе и список не пуст, добавляем сзади.
                // Мы можем вызывать для входных указателей либо `take`,
                // либо `mem::forget`. Take корректнее, если мы используем
                // собственный аллокатор или что-то, что тоже требует очистки!
                (*back.as_ptr()).back = input.front.take();
                (*input.front.unwrap().as_ptr()).front = Some(back);
                self.list.back = input.back.take();
                self.list.len += input.len;
                // Not necessary but Polite To Do
                // Не обязательно, но вежливо
                input.len = 0;
            } else {
                // We're empty, become the input, remain on the ghost
                // Наш список пуст, заменяем его на входной, остаёмся на псевдоэлементе
                *self.list = input;
            }
        }
    }
```

> Ok this one is genuinely horrendous, and really is feeling that `Option<NonNull>` pain now.
> But there's a lot of cleanups we can do.
> For one, we can pull this code out to the very end, because we always want to do it.
> I don't *love*  (although sometimes it's a noop, and setting `input.len` is more a matter of paranoia about future extensions to the code):

Да, этот код по настоящему ужасен и я ощущаю всю боль от использования `Option<NonNull>`.
Но многое можно исправить.
Во-первых, эти две строки можно вынести в самый конец, поскольку они нужным всегда.
(Да, в некоторых сценариях они не обязательны, но ничего не ломают, а установка `input.len` вызвана, скорее, паранойей):

```rust ,ignore
self.list.len += input.len;
input.len = 0;
```

> > Use of moved value: `input`

> Использование перемещённого значения: `input`

> Ah, right, in the "we're empty" case we're moving the list.
> Let's replace that with a swap:

Ах, да, в случае «наш список пуст» мы перемещаем `list`.
Заменим эту строку на вызов swap:

```rust ,ignore
// We're empty, become the input, remain on the ghost
// Наш список пуст, заменяем его на входной, остаёмся на псевдоэлементе
std::mem::swap(self.list, &mut input);
```

> In this case the writes will be pointless, but, they still work (we could probably also early-return in this branch to appease the compiler).

В этом случае операторы, меняющие значения `self.list.len` и `input.len`, не делают ничего, так значения, уже корректны, но они ничего и не ломают (и, в принципе, мы могли бы сделать в этом месте ранний возврат из функции).

> This unwrap is just a consequence of me thinking about the cases backwards, and can be fixed by making the if-let ask the right question:

Вызов unwrap вызван тем, что я рассматривала варианты в неудобном порядке и его можно убрать, если мы правильно зададим вопрос в операторе if-let:

```rust ,ignore
if let Some(0) = self.index {

} else {
    let prev = (*cur.as_ptr()).front.unwrap();
}
```

> Adjusting the index is duplicated inside the branches, so can also be hoisted out:

Корректировка индекса повторяется в обоих ветках, так что её тоже можно вынести наружу:

```rust
*self.index.as_mut().unwrap() += input.len;
```

> Ok, putting that all together we get this:

Итак, сложив всё воедино, получаем:

```rust
if input.is_empty() {
    // Input is empty, do nothing.
    // Пустой входной список, ничего не делаем
} else if let Some(cur) = self.cur {
    // Both lists are non-empty
    // Оба списка не пустые
    if let Some(prev) = (*cur.as_ptr()).front {
        // General Case, no boundaries, just internal fixups
        // Общий случай, никаких граничных случаев:
        // просто правим приватные поля
        let in_front = input.front.take().unwrap();
        let in_back = input.back.take().unwrap();

        (*prev.as_ptr()).back = Some(in_front);
        (*in_front.as_ptr()).front = Some(prev);
        (*cur.as_ptr()).front = Some(in_back);
        (*in_back.as_ptr()).back = Some(cur);
    } else {
        // We're appending to the front, see append to back below
        // Добавляем спереди, сравните с добавлением сзади
        (*cur.as_ptr()).front = input.back.take();
        (*input.back.unwrap().as_ptr()).back = Some(cur);
        self.list.front = input.front.take();
    }
    // Index moves forward by input length
    // Индекс увеличивается на длину входного списка
    *self.index.as_mut().unwrap() += input.len;
} else if let Some(back) = self.list.back {
    // We're on the ghost but non-empty, append to the back
    // We can either `take` the input's pointers or `mem::forget`
    // it. Using take is more responsible in case we do custom
    // allocators or something that also needs to be cleaned up!
    // Мы на псевдоэлементе и список не пуст, добавляем сзади.
    // Мы можем вызывать для входных указателей либо `take`,
    // либо `mem::forget`. Take корректнее, если мы используем
    // собственный аллокатор или что-то, что тоже требует очистки!
    (*back.as_ptr()).back = input.front.take();
    (*input.front.unwrap().as_ptr()).front = Some(back);
    self.list.back = input.back.take();

} else {
    // We're empty, become the input, remain on the ghost
    // Наш список пуст, заменяем его на входной, остаёмся на псевдоэлементе
    std::mem::swap(self.list, &mut input);
}

self.list.len += input.len;
// Not necessary but Polite To Do
// Не обязательно, но вежливо
input.len = 0;

// Input dropped here
// input освобождается здесь
```

> Alright this still sucks, but mostly because of -- nope ok just spotted a bug:

Ладно, это всё ещё отстой, но в основном из-за... нет, так, только что нашла ошибку:

```rust
    (*back.as_ptr()).back = input.front.take();
    (*input.front.unwrap().as_ptr()).front = Some(back);
```

> We `take` input.front and then unwrap it on the next line! *sigh* and we do the same thing in the equivalent mirror case.
> We would have caught this instantly in tests, but, we're trying to be Perfect now, and I'm just kinda doing this live, and this is the exact moment where I saw it.
> This is what I get for not being my usual tedious self and doing things in phases.
> More explicit!

Мы вызываем `take` у `input.front` и затем вызываем `unwrap` буквально на следующей строке!
*вздыхает* и мы делаем то же самое в эквивалентном зеркальном методе.
Мы бы нашли эту ошибку с помощью тестов, но сейчас я пытаюсь быть Идеальной и пишу код в режиме реального времени, и я увидела эту ошибку только что.
Вот что бывает, когда выполняешь работу не так, как привыкла, не в том порядке.
Больше ясности в коде!

```rust
// We can either `take` the input's pointers or `mem::forget`
// it. Using `take` is more responsible in case we ever do custom
// allocators or something that also needs to be cleaned up!
// Мы можем вызывать для входных указателей либо `take`,
// либо `mem::forget`. Take корректнее, если мы используем
// собственный аллокатор или что-то, что тоже требует очистки!
if input.is_empty() {
    // Input is empty, do nothing.
    // Пустой входной список, ничего не делаем
} else if let Some(cur) = self.cur {
    // Both lists are non-empty
    // Оба списка не пустые
    let in_front = input.front.take().unwrap();
    let in_back = input.back.take().unwrap();

    if let Some(prev) = (*cur.as_ptr()).front {
        // General Case, no boundaries, just internal fixups
        // Общий случай, никаких граничных случаев:
        // просто правим приватные поля
        (*prev.as_ptr()).back = Some(in_front);
        (*in_front.as_ptr()).front = Some(prev);
        (*cur.as_ptr()).front = Some(in_back);
        (*in_back.as_ptr()).back = Some(cur);
    } else {
        // No prev, we're appending to the front
        // Нет предыдущего элемента, добавляем спереди
        (*cur.as_ptr()).front = Some(in_back);
        (*in_back.as_ptr()).back = Some(cur);
        self.list.front = Some(in_front);
    }
    // Index moves forward by input length
    // Индекс увеличивается на длину входного списка
    *self.index.as_mut().unwrap() += input.len;
} else if let Some(back) = self.list.back {
    // We're on the ghost but non-empty, append to the back
    // Мы на псевдоэлементе и список не пуст, добавляем сзади.
    let in_front = input.front.take().unwrap();
    let in_back = input.back.take().unwrap();

    (*back.as_ptr()).back = Some(in_front);
    (*in_front.as_ptr()).front = Some(back);
    self.list.back = Some(in_back);
} else {
    // We're empty, become the input, remain on the ghost
    // Наш список пуст, заменяем его на входной, остаёмся на псевдоэлементе
    std::mem::swap(self.list, &mut input);
}

self.list.len += input.len;
// Not necessary but Polite To Do
// Не обязательно, но вежливо
input.len = 0;

// Input dropped here
// input освобождается здесь
```

> Alright now this, this I can tolerate.
> The only complaints I have are that we don't dedupe in_front/in_back (probably we could rejig our conditions but eh whatever).
> Really this is basically what you would write in C but with `Option<NonNull>` gunk making it tedious.
> I can live with that.
> Well no we should just make raw pointers better for this stuff.
> But, out of scope for this book.

Ладно, с таким кодом уже можно смириться.
Единственный недостаток в том, что мы два раза инициализируем in_front/in_back (возможно мы могли бы поправить наши условия, но не суть).
Подобный код мы бы написали и на C, но `Option<NonNull>` делает его многословным.
С этим можно жить.
Ладно, нам надо сделать сырые указатели более удобными для решения таких задач.
Но это выходит за рамки этой книги.

> Anyway, I am absolutely exhausted after that, so, `insert` and `remove` and all the other APIs can be left as an excercise to the reader. 

В любом случае, я абсолютно измучена после этого, так что `insert`, `remove` и все остальные методы оставляю в качестве упражнения читателю.

> Here's the final code for our Cursor with my attempt at copy-pasting the combinatorics.
> Did I get it right?
> I'll only find out when I write the next chapter and test this monstrosity!

Вот финальный код нашего курсора вместе с копи-пастой комбинаторных методов.
Всё ли здесь правильно?
Это я узнаю только после написания следующего раздела и тестирования этого монстра!

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
                // Мы на реальном элементе, двигаемся к следующему (в сторону заднего)
                self.cur = (*cur.as_ptr()).back;
                if self.cur.is_some() {
                    *self.index.as_mut().unwrap() += 1;
                } else {
                    // Мы попали на псевдоэлемент, убираем индекс
                    self.index = None;
                }
            }
        } else if !self.list.is_empty() {
            // Мы на псевдоэлементе и у нас есть реальный передний элемент, так что перемещаемся на него!
            self.cur = self.list.front;
            self.index = Some(0)
        } else {
            // Мы на псевдоэлементе, но никаких других элементов нет... ничего не делаем.
        }
    }

    pub fn move_prev(&mut self) {
        if let Some(cur) = self.cur {
            unsafe {
                // Мы на реальном элементе, двигаемся к предыдущему (в сторону переднего)
                self.cur = (*cur.as_ptr()).front;
                if self.cur.is_some() {
                    *self.index.as_mut().unwrap() -= 1;
                } else {
                    // Мы попали на псевдоэлемент, убираем индекс
                    self.index = None;
                }
            }
        } else if !self.list.is_empty() {
            // Мы на псевдоэлементе и у нас есть реальный задний элемент, так что перемещаемся на него!
            self.cur = self.list.back;
            self.index = Some(self.list.len - 1)
        } else {
            // Мы на псевдоэлементе, но никаких других элементов нет... ничего не делаем.
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
        // У нас есть что-то такое:
        //
        //     list.front -> A <-> B <-> C <-> D <- list.back
        //                               ^
        //                              cur
        //
        // А мы хотим получить вот это:
        //
        //     list.front -> C <-> D <- list.back
        //                   ^
        //                  cur
        //
        //     return.front -> A <-> B <- return.back
        //
        if let Some(cur) = self.cur {
            // Указываем на реальный элемент, так что список не пустой.
            unsafe {
                // Текущее состояние
                let old_len = self.list.len;
                let old_idx = self.index.unwrap();
                let prev = (*cur.as_ptr()).front;
                
                // Во что должен превратиться self
                let new_len = old_len - old_idx;
                let new_front = self.cur;
                let new_back = self.list.back;
                let new_idx = Some(0);

                // Что мы должны вернуть
                let output_len = old_len - new_len;
                let output_front = self.list.front;
                let output_back = prev;

                // Разрываем связь между cur и prev
                if let Some(prev) = prev {
                    (*cur.as_ptr()).front = None;
                    (*prev.as_ptr()).back = None;
                }

                // Создаём результат:
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
            // Мы на псевдолементе, просто меняем наш список на пустой.
            // Никаких других изменений состояния не нужно.
            std::mem::replace(self.list, LinkedList::new())
        }
    }

    pub fn split_after(&mut self) -> LinkedList<T> {
        // У нас есть что-то такое:
        //
        //     list.front -> A <-> B <-> C <-> D <- list.back
        //                         ^
        //                        cur
        // 
        //
        // А мы хотим получить вот это:
        // 
        //     list.front -> A <-> B <- list.back
        //                         ^
        //                        cur
        //
        // 
        //     return.front -> C <-> D <- return.back
        //
        if let Some(cur) = self.cur {
            // Указываем на реальный элемент, так что список не пустой.
            unsafe {
                // Текущее состояние
                let old_len = self.list.len;
                let old_idx = self.index.unwrap();
                let next = (*cur.as_ptr()).back;
                
                // Во что должен превратиться self
                let new_len = old_idx + 1;
                let new_back = self.cur;
                let new_front = self.list.front;
                let new_idx = Some(old_idx);

                // Что мы должны вернуть
                let output_len = old_len - new_len;
                let output_front = next;
                let output_back = self.list.back;

                // Разрываем связь между cur и next
                if let Some(next) = next {
                    (*cur.as_ptr()).back = None;
                    (*next.as_ptr()).front = None;
                }

                // Создаём результат:
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
            // Мы на псевдолементе, просто меняем наш список на пустой.
            // Никаких других изменений состояния не нужно.
            std::mem::replace(self.list, LinkedList::new())
        }
    }

    pub fn splice_before(&mut self, mut input: LinkedList<T>) {
        // Наше:
        //
        // input.front -> 1 <-> 2 <- input.back
        //
        //  list.front -> A <-> B <-> C <- list.back
        //                      ^
        //                     cur
        //
        // Превращается в:
        //
        // list.front -> A <-> 1 <-> 2 <-> B <-> C <- list.back
        //
        unsafe {
            // Мы можем вызывать для входных указателей либо `take`,
            // либо `mem::forget`. Take корректнее, если мы используем
            // собственный аллокатор или что-то, что тоже требует очистки!
            if input.is_empty() {
                // Пустой входной список, ничего не делаем
            } else if let Some(cur) = self.cur {
                // Оба списка не пустые
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                if let Some(prev) = (*cur.as_ptr()).front {
                    // Общий случай, никаких граничных случаев:
                    // просто правим приватные поля
                    (*prev.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(prev);
                    (*cur.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(cur);
                } else {
                    // Нет предыдущего элемента, добавляем спереди
                    (*cur.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(cur);
                    self.list.front = Some(in_front);
                }
                // Индекс увеличивается на длину входного списка
                *self.index.as_mut().unwrap() += input.len;
            } else if let Some(back) = self.list.back {
                // Мы на псевдоэлементе и список не пуст, добавляем сзади.
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                (*back.as_ptr()).back = Some(in_front);
                (*in_front.as_ptr()).front = Some(back);
                self.list.back = Some(in_back);
            } else {
                // Наш список пуст, заменяем его на входной, остаёмся на псевдоэлементе
                std::mem::swap(self.list, &mut input);
            }

            self.list.len += input.len;
            // Не обязательно, но вежливо
            input.len = 0;
            
            // input освобождается здесь
        }        
    }

    pub fn splice_after(&mut self, mut input: LinkedList<T>) {
        // Наше:
        //
        // input.front -> 1 <-> 2 <- input.back
        //
        // list.front -> A <-> B <-> C <- list.back
        //                     ^
        //                    cur
        //
        //
        // Превращается в :
        //
        // list.front -> A <-> B <-> 1 <-> 2 <-> C <- list.back
        //                     ^
        //                    cur
        //
        unsafe {
            // Мы можем вызывать для входных указателей либо `take`,
            // либо `mem::forget`. Take корректнее, если мы используем
            // собственный аллокатор или что-то, что тоже требует очистки!
            if input.is_empty() {
                // Пустой входной список, ничего не делаем
            } else if let Some(cur) = self.cur {
                // Оба списка не пустые
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                if let Some(next) = (*cur.as_ptr()).back {
                    // Общий случай, никаких граничных случаев:
                    // просто правим приватные поля
                    (*next.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(next);
                    (*cur.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(cur);
                } else {
                    // Нет следующего элемента, добавляем сзади
                    (*cur.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(cur);
                    self.list.back = Some(in_back);
                }
                // Индекс не меняется
            } else if let Some(front) = self.list.front {
                // Мы на псевдоэлементе и список не пуст, добавляем спереди.
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                (*front.as_ptr()).front = Some(in_back);
                (*in_back.as_ptr()).back = Some(front);
                self.list.front = Some(in_front);
            } else {
                // Наш список пуст, заменяем его на входной, остаёмся на псевдоэлементе
                std::mem::swap(self.list, &mut input);
            }

            self.list.len += input.len;
            // Не обязательно, но вежливо
            input.len = 0;
            
            // input освобождается здесь
        }        
    }
}
```
