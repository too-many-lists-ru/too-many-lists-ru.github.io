> # Arc

# Arc

> One reason to use an immutable linked list is to share data across threads.
> After all, shared mutable state is the root of all evil, and one way to solve that is to kill the *mutable* part forever.

Одна из причин использовать иммутабельный связный список — разделение данных между потоками.
Помимо прочего, разделяемое мутабельное состояние — корень всех зол, и один из способов справиться с этим — навсегда избавиться от *мутабельного*.

> Except our list isn't thread-safe at all.
> In order to be thread-safe, we need to fiddle with reference counts *atomically*.
> Otherwise, two threads could try to increment the reference count, *and only one would happen*.
> Then the list could get freed too soon!

Впрочем, наш список вовсе не является потокобезопасным.
Чтобы считаться потокобезопасным, нужно решить проблему *атомарного* подсчёта ссылок.
В противном случае два потока могут попытаться увеличить счётчик ссылок *и только одному это удастся*.
Тогда этот список будет освобождён слишком скоро!

> In order to get thread safety, we have to use *Arc*.
> Arc is completely identical to Rc except for the fact that reference counts are modified atomically.
> This has a bit of overhead if you don't need it, so Rust exposes both.
> All we need to do to make our list is replace every reference to Rc with `std::sync::Arc`.
> That's it.
> We're thread safe.
> Done!

Чтобы получить потокобезопасность, мы должны использовать *Arc*.
Arc полностью идентичен Rc за исключением того факт, что счётчики ссылок модифицируются атомарно.
Это влечёт небольшие накладные расходы, если оно вам не надо, так что Rust предоставляет оба варианта.
Всё, что нам надо с делать с нашим списком, это заменить каждое упоминание Rc на `std::sync::Arc`.
Это всё.
Мы потокобезопасны.
Сделано!

> But this raises an interesting question: how do we *know* if a type is thread-safe or not?
> Can we accidentally mess up?

Но это влечёт интересный вопрос: откуда мы *знаем*, потокобезопасен тип или нет?
Можем ли мы случайно ошибиться?

> No!
> You can't mess up thread-safety in Rust!

Нет!
В Rust невозможно сломать потокобезопасность!

> The reason this is the case is because Rust models thread-safety in a first-class way with two traits: `Send` and `Sync`.

Причина в том, что Rust моделирует потокобезопасность путём объектов первого сорта <!-- на самом высоком уровне --> с помощью двух типажей: `Send` и `Sync`.

> A type is *Send* if it's safe to *move* to another thread.
> A type is *Sync* if it's safe to *share* between multiple threads.
> That is, if `T` is Sync, `&T` is Send.
> Safe in this case means it's impossible to cause *data races*, (not to be mistaken with the more general issue of *race conditions*).

Тип относится к *Send* (отправляемый), если его можно безопасно *передать* в другой поток.
Тип относится к *Sync* (синхронный), если его можно безопасно *разделить* между многими потоками.
То есть <!-- неочевидно, я бы написал "есть правило:" -->, если `T` реализует Sync, то 
`&T` реализует Send.
Безопасность в данном случае означает, что невозможно устроить *гонку данных* (что не надо путать с более общий случаем, *состоянием гонки*).

> These are marker traits, which is a fancy way of saying they're traits that provide absolutely no interface.
> You either *are* Send, or you aren't.
> It's just a property *other* APIs can require.
> If you aren't appropriately Send, then it's statically impossible to be sent to a different thread!
> Sweet!

Это типажи-маркеры, что является умным способом сказать, что они не предоставляют ни одного метода.
Вы просто либо *являетесь* Send, либо не являтесь.
Это всего лишь свойство, которое могут требовать *другие* API.
Если вы не являетесь полноценным Send, вас нельзя отправить в другой поток статически!
Идеально!

> Send and Sync are also automatically derived traits based on whether you are totally composed of Send and Sync types.
> It's similar to how you can only implement Copy if you're only made of Copy types, but then we just go ahead and implement it automatically if you are.

Send и Sync также автоматически реализуются для типов, состоящими только из полей Send и Sync.
Это похоже на то, что вы можете реализовать Copy только если состоите исключительно из полей, реализующих Copy, но теперь мы пошли ещё дальше и реализовали всё за вас.

> Almost every type is Send and Sync.
> Most types are Send because they totally own their data.
> Most types are Sync because the only way to share data across threads is to put them behind a shared reference, which makes them immutable!

Почти все типы — одновременно Send и Sync.
Большинство типов относятся к Send, потому что они полностью владеют своими данными.
Большинсвто типов относятся к Sync, потому что единственный способ разделить данные между потоками, это спрятать их за разделяемой ссылкой, которая делает их иммутабельными!

> However there are special types that violate these properties: those that have *interior mutability*.
> So far we've only really interacted with *inherited mutability* (AKA external mutability): the mutability of a value is inherited from the mutability of its container.
> That is, you can't just randomly mutate some field of a non-mutable value because you feel like it.

Однако существуют особые типы, которые нарушают эти свойства, они обладают *внутренней мутабельностью*.
До сих пор мы сталкивались только с *унаследованной мутабельностью* (или *внешней мутабельностью*): мутабельность значения наследуется от мутабельность её контейнера.
То есть, вы не можете просто взять и поменять какое-то поле немутабельного значения <!-- объекта -->, просто потому, что вам так захотелось.

> Interior mutability types violate this: they let you mutate through a shared reference.
> There are two major classes of interior mutability: cells, which only work in a single-threaded context; and locks, which work in a multi-threaded context.
> For obvious reasons, cells are cheaper when you can use them.
> There's also atomics, which are primitives that act like a lock.

Типы с внутренней мутабельностью нарушают это правило: оин позволяют менять значения полей через разделяемую ссылку.
Есть два важных класса внутренней мутебельности: ячейке, которые работают только в однопоточном контексте и блокировки, которые работают в многопоточном контексте.
По очевидным причинам, ячейки дешевле <!-- менее ресурсоёмки? -->, если вы можете их использовать.
Есть также атомарные объекты <!-- надесь, что да -->, то есть примитивы, работающие, как блокировки.

> So what does all of this have to do with Rc and Arc?
> Well, they both use interior mutability for their *reference count*.
> Worse, this reference count is shared between every instance!
> Rc just uses a cell, which means it's not thread safe.
> Arc uses an atomic, which means it *is* thread safe.
> Of course, you can't magically make a type thread safe by putting it in Arc.
> Arc can only derive thread-safety like any other type.

Так какое отношение всё это имеет к Rc и Arc?
Ну, оба они внутренне мутабельны в отношении их *счётчков ссылок*.
Хуже того, этот счётчик ссылок разделяется между всеми экземплярами!
Rc просто использует ячейку, что означает, что он не является потокобезопасным.
Arc использует атомарные объекты, что означает, что он *является* потокобезопасным.
Естественно, вы не можете магически сделать тип потокобезопасным, просто поместив его в Arc.
Arc может только наследовать потокобезопасность, как любой другой тип.

> I really really really don't want to get into the finer details of atomic memory models or non-derived Send implementations.
> Needless to say, as you get deeper into Rust's thread-safety story, stuff gets more complicated.
> As a high-level consumer, it all *just works* and you don't really need to think about it.

Я очень-очень-очень не хочу углубляться в тонкости атомарных моделей памяти или не-унаследованной реализации Send.
Излишне упоминать, что по мере погружения в историю потокобезопасности Rust, вещи становятся сложнее.
Для пользователя высокого уровня всё это *просто работает* и вам на самом деле не надо об этом думать.