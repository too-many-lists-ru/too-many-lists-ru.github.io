> # Layout and Basics 2: Getting Raw

# Представление и основы 2: добавляем сырости

> > TL;DR on the previous three sections: randomly mixing safe pointers like `&`, `&mut`, and `Box` with unsafe pointers like `*mut` and `*const` is a recipe for Undefined Behaviour because the safe pointers introduce extra constraints that we aren't obeying with the raw pointers.

> Краткое содержание трёх предыдущих частей: смешение в случайном порядке указатели наподобие `&`, `& mut` и `Box` с небезопасными указателями наподобие `*mut` и `*const` — верный способ получить Неопределённое Поведение, потому что безопасные указатели вводят дополнительные ограничения, которым мы не обязаны подчиняться в случае сырых указателей.

> Oh god I need to write linked lists again.
> Fine.
> FINE.
> It's Fine.
> We're fine.

Господи, мне снова придётся писать связные списки.
Прекрасно.
ПРЕКРАСНО.
Всё прекрасно.
У нас всё прекрасно.

> We're gonna knock a lot of this section out real quick since we already discussed the design in the first try around, and everything we did *was* basically correct except for how we mixed together safe and unsafe pointers.

Мы быстро разберём большую часть этого раздела, поскольку мы уже обсудили дизайн в первой части и всё, что бы делали *было* в основном правильно, за исключением того, что мы смешивали безопасные и небезопасные указатели.

> # Layout

# Представление

> So in the new layout we're only going to only use raw pointers and everything will be perfect and we'll never make mistakes again.

Итак, в новом представлении вы собираемся использовать только сырые указатели и всё будет нормально, и мы никогда больше не будет ошибаться.

> Here's our old broken layout:

Вот наш старое проблемное представление:

```rust
pub struct List<T> {
    head: Link<T>,
    tail: *mut Node<T>, // INNOCENT AND KIND
                        // НЕВИННЫЙ И ДОБРЫЙ
}

type Link<T> = Option<Box<Node<T>>>; // THE REAL EVIL
                                     // НАСТОЯЩЕЕ ЗЛО

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

And here's our new layout:

А вот наше новое представление.

```rust
pub struct List<T> {
    head: Link<T>,
    tail: *mut Node<T>,
}

type Link<T> = *mut Node<T>; // MUCH BETTER
                             // ГОРАЗДО ЛУЧШЕ

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

> Remember: Option isn't as nice or useful when we're using raw pointers, so we're not using that anymore.
> In later sections we'll look at the `NonNull` type, but don't worry about that for now.

Запомните: Option не так удоен или полезен, когда мы используем сырые указатели, так что мы не будем больше его использовать.
Далее мы познакомимся с типом `NotNull`, но не беспокойтесь об этом сейчас.

> # Basics

# Основы

> List::new is basically the same.

List::new в основе остаётся тем же самым.

```rust ,ignore
use ptr;

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: ptr::null_mut(), tail: ptr::null_mut() }
    }
}
```

> Push is basically the s-

`push` в основе т-


```rust ,ignore
pub fn push(&mut self, elem: T) {
    let mut new_tail = Box::new(
```

> Wait we're not using Box anymore.
> How do we allocate memory without Box?

Подождите, мы же больше не используем Box.
Как нам выделить память без Box?

> Well, we *could* with `std::alloc::alloc`, but that's like bringing a katana into the kitchen.
> It'll get the job done but it's kinda overkill and unwieldy.

Ну, мы *могли бы* вызывать `std::alloc::alloc`, но в данном случае это как брать катану на кухню.
Свою работу мы сделаем, но это будет неудобно и избыточно.

> We want to *have* boxes, but, *not*.
> One completely wild but *maybe* viable option would be to do something like this:

Мы хотим *использовать* боксы, но, в то же время, *нет*.
Один совершенно безумный вариант, но *возможно* жизнеспособный — сделать что-то подобное:

```
struct Node<T> {
    elem: T,
    real_next: Option<Box<Node<T>>>,
    next: *mut Node<T>,
}
```

> With the idea that we create the Boxes and store them in our node, but then we take a raw pointer into them and only use that raw pointer until we're done with the Node and want to destroy it.
> Then we can `take` the Box out of `real_next` and drop it.
> I *think* that would conform to our very simplified stacked borrows model? 

С такой идеей, что мы создаём боксы и храним их в нашем узле, но затем мы получаем сырой указатель на их содержимое, и используем его до тех пор, пока мы не закончим с узлом и не захотим его удалить.
Тогда мы можем вызывать извлечь (`take`) Box из переменной `real_next` и уничтожить его.
Я *думаю*, это совместимо с нашей очень упрощённой моделью стековых заимствований?

> If you wanna try to make that, have "fun", but that just looks awful right?
> This isn't the chapter on Rc and RefCell, we're not gonna play this *game* anymore.
> We're gonna just make simple and clean stuff.

Я вы хотите попробовать эту идею, пожалуйста, но выглядит она уродливо, не правда ли?
Это не глава про Rc или RefCell и мы больше не будем играть в эту *игру*.
Сейчас мы собираемся делать простые и ясные вещи.

> So instead we're going to use the very nice [Box::into_raw][] function:

Так что вместо этого мы будем использовать очень милую функцию [Box::into_raw][].

> > ```rust ,ignore
> >   pub fn into_raw(b: Box<T>) -> *mut T
> > ```
> >
> > Consumes the Box, returning a wrapped raw pointer.
> >
> > The pointer will be properly aligned and non-null.
> >
> > After calling this function, the caller is responsible for the memory previously managed by the Box.
> > In particular, the caller should properly destroy T and release the memory, taking into account the memory layout used by Box.
> > The easiest way to do this is to convert the raw pointer back into a Box with the `Box::from_raw` function, allowing the Box destructor to perform the cleanup.
> >
> > Note: this is an associated function, which means that you have to call it as `Box::into_raw(b)` instead of `b.into_raw()`.
> > This is so that there is no conflict with a method on the inner type.
> >
> > **Examples**
> >
> > Converting the raw pointer back into a Box with Box::from_raw for automatic cleanup:
> >
> > ```
> >  let x = Box::new(String::from("Hello"));
> >  let ptr = Box::into_raw(x);
> >  let x = unsafe { Box::from_raw(ptr) };
> > ```

> ```rust ,ignore
>   pub fn into_raw(b: Box<T>) -> *mut T
> ```
> 
> Получает Box, возвращает внутренний сырой указатель.
> 
> Указатель будет правильно выровнен и не будет равен null.
> 
> После вызова этой функции вызывающая сторона отвечает за память, которой прежде управлял Box. 
> В частности, вызывающая сторона должна правильно уничтожить T и освободить память с учётом внутреннего устройства Box.
> Простейший способ это сделать — преобразовать сырой указатель обратно в Box с помощью функции `Box::from_raw`, позволив деструктору Box выполнить очистку.
> 
> Обратите внимание, что это ассоциированная функция, что означает, что вы должны вызывать её как `Box::into_raw(b)` вместо `b.into_raw()`.
> Это нужно, чтобы избежать конфликта с методом внутреннего типа.
>
> **Примеры**
>
> Конвертируем сырой указатель обратно в Box с помощью Box::from_raw для автоматической очистки:
>
> ```
> let x = Box::new(String::from("Hello"));
> let ptr = Box::into_raw(x);
> let x = unsafe { Box::from_raw(ptr) };
> ```

> Nice, that looks *literally* designed for our use case.
> It also matches the rules we're trying to follow: start with safe stuff, turn into into raw pointers, and then only convert back to safe stuff at the end (when we want to Drop it).

Прекрасно, это выглядит *буквально* разработанным для нашего случая.
Оно также совпадает с правилами, которым мы стараемся следовать: начинать с безопасной штуки, превращать её в сырые указатели, и затем преобразовывать их обратно в безопасную штуку (когда мы хотим вызывать для них `drop`).

> This is basically exactly like doing the weird `real_next` thing but without having to faff around storing the Box when it's the exact same pointer as the raw pointer anyway.

В основе своей это выглядит точно также, как и ужасный способ с `real_next`, но без необходимости возиться с сохранением Box, когда это всё равно тот же самый указатель, что и сырой указатель.

> Also now that we're just using raw pointers everywhere, let's not worry about keeping those `unsafe` blocks narrow: it's all unsafe now.
> (It always was, but it's nice to lie to yourself sometimes.)

Также теперь мы просто используем везде сырые указатели, давайте не беспокоиться про сохранение этих блоков `unsafe` небольшими: теперь все они небезопасные.
(Так всегда и было, но так иногда приятно обманывать самого себя.)


```rust ,ignore
pub fn push(&mut self, elem: T) {
    unsafe {
        // Immediately convert the Box into a raw pointer
        // Немедленно преобразуем Box в сырой указатель
        let new_tail = Box::into_raw(Box::new(Node {
            elem: elem,
            next: ptr::null_mut(),
        }));

        if !self.tail.is_null() {
            (*self.tail).next = new_tail;
        } else {
            self.head = new_tail;
        }

        self.tail = new_tail;
    }
}
```


> Hey that code's actually looking a lot cleaner now that we're sticking to raw pointers!

Эй, этот код на самом деле выглядит гораздо чище теперь, раз мы используем только сырые указатели!

> On to pop, which is also pretty similar to how we left it, although we've got to remember to use `Box::from_raw` to clean up the allocation:

Перейдём к `pop`, который тоже достаточно похожа на старую реализацию, хотя мы помним, что надо использовать `Box::from_raw` для очистки выделения:

```rust ,ignore
pub fn pop(&mut self) -> Option<T> {
    unsafe {
        if self.head.is_null() {
            None
        } else {
            // RISE FROM THE GRAVE
            // ВОССТАНЬ ИЗ МОГИЛЫ
            let head = Box::from_raw(self.head);
            self.head = head.next;

            if self.head.is_null() {
                self.tail = ptr::null_mut();
            }

            Some(head.elem)
        }
    }
}
```

> Our nice little `take`s and `map`s are dead, gotta just check and set `null` manually now.

Наши прекрасные маленькие `take` и `map` больше не нужны, просто проверяем и устанавливаем `null` теперь вручную.

> And while we're here, let's slap in the destructor.
> This time we'll implement it as just repeatedly popping, because it's cute and simple:

И, раз уже мы заговорили об этом, давайте добавим деструктор.
В это время мы реализуем его как простой повторяющийся вызов `pop`, потому что это легко и просто:

```rust ,ignore
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        while let Some(_) = self.pop() { }
    }
}
```


> Now, for the moment of truth:

А теперь настал момент истины:

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

> Good, but does miri agree?

Хорошо, но согласна ли miri?

```text
MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri test

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

> EYYYY!!!!!

ЙЕЕЕЕ!!!!!

> IT FRIGGIN WORKED!

ЭТА ХРЕНЬ СРАБОТАЛА!

> PROBABLY!

ВОЗМОЖНО!

> FAILING TO FIND UNDEFINED BEHAVIOUR IS NOT A PROOF THAT IT ISN'T THERE WAITING TO CAUSE PROBLEMS BUT THERE IS A LIMIT TO HOW RIGOROUS I AM WILLING TO BE FOR A JOKE BOOK ABOUT LINKED LISTS SO WE'RE GONNA CALL THIS A 100% MACHINE VERIFIED PROOF AND ANYONE WHO SAYS OTHERWISE CAN SUCK MY COQ!

НЕВОЗМОЖНО ДОКАЗАТЬ ОТСУТСТВИЕ НЕОПРЕДЕЛЁННОГО ПОВЕДЕНИЯ, ВПОЛНЕ ВОЗМОЖНО, ЧТО ОНО ГДЕ-ТО ТАМ ЕСТЬ, НО ЕСТЬ ПРЕДЕЛ ТОГО, НАСКОЛЬКО Я ГОТОВА БЫТЬ СТРОГОЙ В ШУТОЧНОЙ КНИГЕ О СВЯЗНЫХ СПИСКАХ, ТАК ЧТО МЫ БУДЕМ СЧИТАТЬ ЭТО ДОКАЗАТЕЛЬСТВО 100% ПРОВЕРЕННЫМ МАШИНОЙ И ЛЮБОЙ, КТО СКАЖЕТ ОБРАТНОЕ, МОЖЕТ ОБЛИЗАТЬ МОЕГО ПЕТУШКА!

<!-- Игра слов: COQ -- язык, используемый для доказательства корректности программ и одновременно петух, плюс, в английском созвучен со словом член -->

> ∴ QED □

Ч. т. д.

[Box::into_raw]: https://doc.rust-lang.org/std/boxed/struct.Box.html#method.into_raw
