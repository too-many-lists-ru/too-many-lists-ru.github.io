> # An Introduction To Cursors

# Введение в Курсоры

> OK!!!
> We now have a LinkedList that's on par with std's 1.0 implementation!
> Which of course means that our LinkedList is *still completely useless*.
> We've taken the enormous performance penalty of implementing a Deque as a linked list, **and we don't have any of the APIs that make it actually useful**. 

Здорово!!!
Теперь у нас есть LinkedList не уступающий по качеству реализации из стандартной библиотеки версии 1.0!
Что, конечно, означает, что наш LinkedList *всё ещё совершенно бесполезен*.
Мы смирились с огромными потерями производительности, реализуя дек как связный список, **и у нас нет ни одного API, который бы сделал его полезным**.

> Here's how we do against the "killer apps" of linked lists:

Вот как у нас обстоят дела с «киллер-фичами» связных списков:

> * 🚫 Getting to do [weird intrusive stuff](https://docs.rs/linked-hash-map/latest/linked_hash_map/)
> * 🚫 Getting to do [weird lockfree stuff](https://doc.rust-lang.org/std/sync/mpsc/)
> * 🚫 Getting to store [Dynamically Sized Types](https://doc.rust-lang.org/nomicon/exotic-sizes.html#dynamically-sized-types-dsts)
> * 🌟 O(1) push/pop without [amortization](https://en.wikipedia.org/wiki/Amortized_analysis) (if you are willing to believe that malloc is O(1))
> * 🚫 O(1) list splitting
> * 🚫 O(1) list splicing

> * 🚫 Возможность делать [странные интрузивные штуки](https://docs.rs/linked-hash-map/latest/linked_hash_map/)
> * 🚫 Возможность делать [странные неблокирующие штуки](https://doc.rust-lang.org/std/sync/mpsc/)
> * 🚫 Возможность хранить [типы переменного размера](https://doc.rust-lang.org/nomicon/exotic-sizes.html#dynamically-sized-types-dsts)
> * 🌟 push/pop за O(1) без [амортизации](https://en.wikipedia.org/wiki/Amortized_analysis) (если вы, конечно, верите, что malloc выполняется за O(1))
> * 🚫 Разделение списка за O(1)
> * 🚫 Слияние списков за O(1)

> Well... 1 out of 6 is... better than nothing!
> Do you see why I wanted to rip this thing out of std?

Ну... 1 из 6 это... лучше, чем ничего!
Понимаете, почему я хотел выдрать эту штуку из стандартной библиотеки?

> We're not going to make our list support "weird" stuff, because that's all adhoc and domain-specific.
> But the splitting and splicing thing, now that's something we can do!

Мы не станем делать поддержку «странных» штук, потому что это всё зависит от предметной области и обычно делается для решения конкретной задачи (adhoc).
Но вот разделение и слияние — совсем другое дело!

> But here's the problem: actually *reaching* the k<sup>th</sup> element in a LinkedList takes O(k) time, so how can we *possibly* do arbitrary splits and merges in O(1)?
> Well, the trick is that you don't have an API like `split_at(index)` -- you make a system where the user can statefully iterate to a position in the list and make O(1) modifications at that point!

Но здесь есть проблема: на самом деле *получение* k-того элемента в связном списке требует времени O(k), так как же мы *можем* произвольно разделить или слить список за O(1)?
Ну, трюк в том, что вы не предоставляете API наподобие `split_at(index)` — вы создаёте систему, в которой пользователь может перебрать элементы до нужной позиции в списке, а затем, в этой точке, модифицировать его за O(1)!

> Hey, we already have iterators!
> Can we use them for this?
> Kind of... but one of their super-powers gets in the way.
> You may recall that the way that we write out the lifetimes for by-ref iterators means that the references they return *aren't* tied to the iterator.
> This lets us repeatedly call `next` and hold onto the elements:

Эй, а ведь у нас уже есть итераторы!
Можем ли мы использовать их для перебора?
Вроде бы... но мешает одна из их супер-способностей.
Возможно, вы помните, что способ, которым мы указываем время жизни итераторов, возвращающих ссылку на элемент, работает так, что они возвращают ссылку, которая *не привязана* к итератору.
Это позволяет нам повторно вызывать `next` и хранить элементы:

```rust ,ignore
let mut list = ...;
let iter = list.iter_mut();
let elem1 = list.next();
let elem2 = list.next();

if elem1 == elem2 { ... }
```

> If the returned references borrowed the iterator, then this code wouldn't work at all.
> The compiler would just complain about the second call to `next`!
> This flexibility is great, but it puts some implicit constraints on us:

Если бы возвращаемые ссылки заимствовали итератор, этот код вообще бы не работал.
Компилятор бы просто жаловался на второй вызов `next`!
Эта гибкость великолепна, о она накладывает на нас некоторые неявные ограничения:

> * By-Mutable-Ref Iterators can never go backwards and yield an element again, because the user would be able to get two `&mut`'s to the same element, breaking fundamental rules of the language.
> * By-Ref Iterators can't have extra methods which could possibly modify the underlying collection in a way that would invalidate any reference that has already been yielded.

* Итераторы, возвращающие изменяемую ссылку, не могут двигаться назад и возвращать элемент повторно, поскольку тогда пользовать мог бы получить две ссылки `&mut` на один и тот же элемент, что ломает фундаментальные правила языка.
* Итераторы, возвращающие разделяемую ссылку, не должны иметь других методов, которые могут модифицировать нижележащую коллекцию способом, который сделает уже полученные ссылки невалидными.

> Unfortunately, both of these things are *exactly* what we want our LinkedList API to do!
> So we can't just use iterators, we need something new: *Cursors*.

К сожалению, обе эти штуки — это *именно* то, что мы хотим сделать с нашим LinkedList API!
Так что мы не можем просто использовать итераторы, нам потребуется что-то новое: *Курсоры*.

> Cursors are exactly like the little blinking `|` you get when you're editing some text on a computer.
> It's a position in a sequence (the text) that you can move around (with the arrow keys), and whenever you type the edits happen at that point.

Курсоры очень похожи на маленькие мигающие `|`, которые вы видите, редактируя текст на компьютере.
По сути, они — это позиция в последовательности (тексте), которую вы можете двигать (с помощью стрелок), и которая показывает место, где происходят изменения.

> See if I just

Смотрите, если я просто

> press

нажму

> enter

Enter

> the whole

весь

> text

текст

> gets broken in half.

сломается напополам.

> Sorry you're standing behind me and watching me type this right?
> So that totally makes sense, right?
> Right.

Извините, вы сейчас стоит за моей спиной и смотрите, как я это печатаю, правда?
Ну, теперь то логика понятна, правда?
Правда.

> Now if you've ever had the misfortune of having a keyboard with an "insert" key and actually pressed it, you know that there's actually technically two interpretations of cursors: they can either lie between elements (characters) or *on* elements.
> I'm pretty sure no one has ever pressed "insert" on purpose in their life, and that it exists purely as a Suffering Button, so it's pretty obvious which one is Better and Right: cursors go between elements!

Если вам не посчастливилось иметь клавиатуру с клавишей «Insert» и время от времени её нажимать, вы знаете, что технически есть две интерпретации курсоров: они могут находиться между элементами (символами) или *над* элементами.
Я почти уверена, что никто никогда в жизни целенаправленно не нажимал «Insert», и что она существует исключительно, как Кнопка Страдания, потому что совершенно очевидно, какая интерпретация лучше и правильней: курсоры находятся между элементами!

> Pretty rock-solid logic right there, I don't think anyone can disagree with me.
Довольно убедительная логика, я думаю, никто не станет со мной спорить.

> Sorry what?
> There was an [RFC in 2018 to add Cursors to Rust's LinkedList](https://github.com/rust-lang/rfcs/blob/master/text/2570-linked-list-cursors.md)?

Простите, что?
В [2018 был RFC по добавлению Курсоров в LinkedList библиотеки Rust](https://github.com/rust-lang/rfcs/blob/master/text/2570-linked-list-cursors.md)?

> > With a Cursor one can seek back and forth through a list and get the current element.
> > With a CursorMut One can seek back and forth and get mutable references to elements, and it can insert and delete elements before and behind the current element (along with performing several list operations such as splitting and splicing).

> С помощью Cursor моно перемещаться по списку вперёд и назад и получать текущий элемент.
> С помощью CursorMut можно перемещаться по списку вперёд и назад, получать изменяемые ссылки на элементы, а также вставлять и удалять элементы до/после текущего (наряду с несколькими операциями, такими как разделение и слияние).

> *Current element*?
> This cursor is *on* elements, not between them!
> I can't believe they didn't accept my totally rock-solid argument!
> So yeah you can just go use the Cursor in std... wait, it's [2022, and Rust 1.60 still has Cursor marked as unstable](https://doc.rust-lang.org/1.60.0/std/collections/linked_list/struct.CursorMut.html)?

*Текущий элемент*?
Такой курсор может быть только *над* элементом, а не между ними!
Не могу поверить, что они не вняли моей убедительной логике!
Так что да, вы можете просто взять Cursor из стандартной библиотеки... подождите, в [2022 в Rust 1.60 Cursor всё ещё помечен, как нестабильный](https://doc.rust-lang.org/1.60.0/std/collections/linked_list/struct.CursorMut.html)?

> Hey wait:

Эй, подождите:

> > Cursors always rest between two elements in the list, and index in a logically circular way.
> > To accommodate this, there is a "ghost" non-element that yields None between the head and tail of the list.

> Курсоры всегда располагаются между двумя элементами в списке и нумеруются циклически.
> Чтобы это работало, между концом и началом списка существует специальный «псевдоэлемент», который возвращает None.

> HEY WAIT.
> This is the opposite of what the RFC says???
> But wait all the docs on the methods still refer to "current" elements... wait hold on, where have I seen this ghost stuff before.
Oh wait, didn't I do that in [my old linked-list fork](https://docs.rs/linked-list/0.0.3/linked_list/struct.Cursor.html) where I prototyped?

ЭЙ, ПОДОЖДИТЕ.
Это ведь противоречит тому, что написано в RFC???
Но, подождите, все эти доку по методам продолжаются ссылаться на «текущие» элементы... так, минутку, я ведь где уже видела этот псевдоэлемент раньше.
О, подождите. разве я не писала его [в своём старом прототипе связного списка](https://docs.rs/linked-list/0.0.3/linked_list/struct.Cursor.html)?

> > Cursors always rest between two elements in the list, and index in a logically circular way.
> > To accomadate this, there is a "ghost" non-element that yields None between the head and tail of the List.

> Курсоры всегда располагаются между двумя элементами в списке и нумеруются циклически.
> Чтобы это работало, между концом и началом списка существует специальный «псевдоэлемент», который возвращает None.

> Hold up what the fuck.
> This isn't a gag, I am actually trying to Read The Docs right now.
> Did std actually RFC a different design from the one I proposed in 2015, but then copy-paste the docs from my prototype???
> Is std meta-shitposting me for writing a book about how much I hate LinkedList?????
> Like yeah I built that prototype to demonstrate the concept so that people would let me add it to std and make LinkedList not useless but, qu'est-ce que le fuck??????????????

Минут, что хрень здесь творится?
Я не шучу, я действительно прямо сейчас пытаюсь Читать Доки.
Стандартная библиотека взяла дизайн, отличный от того, что я предложила в 2015, а затем просто скопи-пастила документацию из моего прототипа???
Она что, троллит меня за то, что я пишу книгу о том, как сильно я ненавижу связные списки???
Да, я написала этот прототип, чтобы продемонстрировать концепцию, так, чтобы люди позволили мне добавить его в стандартную библиотеку и сделать LinkedList не таким бесполезным, но, простите мой французский, это что за херня?

> Ok you know what, clearly std is blessing my design as the objectively superior one, so we're going to do my design.
> Also that's nice because this entire chapter is me actually literally rewriting that library from scratch, so not changing the API sounds Good To Me!

Ладно, знаете, очевидно, что стандартная библиотека одобрила мой дизайн, как объективно лучший, так что именно его мы и будем использовать.
А, кроме того, всю главу я буду переписывать с нуля свою же библиотеку, что меня вполне устраивает!

> Here's the full top-level docs I wrote:

Вот полный текст документации верхнего уровня, которую я написала:

> > A Cursor is like an iterator, except that it can freely seek back-and-forth, and can safely mutate the list during iteration.
> > This is because the lifetime of its yielded references are tied to its own lifetime, instead of just the underlying list.
> > This means cursors cannot yield multiple elements at once.
> >
> > Cursors always rest between two elements in the list, and index in a logically circular way.
> > To accomadate this, there is a "ghost" non-element that yields None between the head and tail of the List.
> >
> > When created, cursors start between the ghost and the front of the list.
> > That is, next will yield the front of the list, and prev will yield None.
> > Calling prev again will yield the tail.

> Курсоры похожи на итераторы, за тем исключением, что могут перемещаться вперёд-и-назад, и безопасно менять список в процессе перемещения.
> Это возможно благодаря тому, что время жизни возвращаемых ссылок совпадает с временем жизни курсоров, а не нижележащих списков.
> Это значит, что курсоры не могут вернуть несколько элементов за раз.
>
> Курсоры всегда располагаются между двумя элементами в списке и нумеруются циклически.
> Чтобы это работало, между концом и началом списка существует специальный «псевдоэлемент», который возвращает None.
>
> Будучи созданными, курсоры находятся между псевдоэлементом и передним элементом списка.
> Таким образом, вызов next вернёт передний элемент списка, а вызов prev вернёт None.
> Повторный вызов prev вернёт хвост.

> Cute, even though we concluded that the whole "sentinel-node" thing was more trouble than it's worth, we're still going to end up with semantics that "pretend" there's a sentinel node so that the cursor can wrap around to the other side of the list.

Мило, хотя мы вроде бы договорились, что от «сторожевого узла» больше вреда, чем пользы, в итоге мы всё равно пришли к семантике, которая «притворяется», что сторожевой узел существует, что позволяет нам переворачиваться на другую сторону списка.

> *Skims over my old APIs some more*

*Ещё раз пробегает свои старые API глазами*

```rust ,ignore
fn splice(&mut self, other: &mut LinkedList<T>)
```

> > Inserts the entire list's contents right after the cursor.

> Вставляет содержимое целого списка прямо за курсором.

> Oh yeah, this is coming back to me.
> I wrote this when I was really mad about combinatoric explosion, and was trying to come up with a way for there to only be one copy of each operation.
> Unfortunately this is... semantically problematic.
> See, when the user wants to splice one list into another, they might want the cursor to end up *before* the splice or *after it*.
> The inserted list can be arbitrarily large, so it's a genuine issue for us to only allow for one and expect the user to walk over the entire inserted list!

О, да, начинаю вспомнить.
Я писала это в жутком расстройстве из-за комбинаторного взрыва и пыталась придумать, как сделать так, чтобы была только одна копия каждой операции.
К сожалению, это... семантически проблематично.
Смотрите, когда пользовать хочет вставить один список в другой, он может хотеть,чтобы курсор остался *перед* вставляемым списком или *за* ним.
Вставляемый список может быть произвольно большим, поэтому разрешать только один способ и ожидать, что пользователь переберёт весь вставленный список, чтобы добраться до другого конца — это подлинная проблема.

> We're gonna have to rework this design from the ground up after all.
> What does our Cursor type need?
> Well it needs to:

В конце концов, нам придётся заново разработать весь дизайн.
Что нужно нашему типу Cursor?
Ну, ему нужны:

> * point "between" two elements
> * as a nice little feature, keep track of what "index" is next
> * update the list itself to modify front/back/len. 

* указывать *между* двумя элементами
* отслеживать «индекс» следующего элемента — в качестве удобной небольшой функции
* обновлять сам список, чтобы менять значения front/back/len

> How do you point between two elements?
> Well, you don't.
> You just point at the "next" element.
> So, yeah even though we're exposing "cursor goes in-between" semantics, we're really implementing it as "cursor is on", and just pretending everything happens before or after that point.

Как вы можете указывать между двумя элементами?
Ладно, вы не можете.
Вы просто указываете на «следующий» элемент.
Так что да, несмотря на то. что мы подразумеваем семантику «курсор между элементами», на
самом деле мы реализуем «курсор над элементом» и просто претворяемся, что всё происходит перед этой точкой, или за ней.

> But there's a reason!
> The splice use-case wants to let the user choose whether they end up before or after the list, but this is... *horribly* complicated to express with the std API!
> They have splice_after and splice_before, but neither changes the cursor's position, so really you'd need splice_after_before and splice_after_after...

Но для этого есть причина!
В сценарии вставки пользователь может выбрать, должен ли курсор оказаться за вставляемым списком или перед ним... но всё это *ужасно* трудно выразить с помощью стандартного API!
У них есть splice_after и splice_before, но они не меняют позицию курсора, поэтому нам потребуются также splice_after_before и splice_after_after...

> Wait no I'm being silly.
> In the std API you can just choose the node you want to end up on, and then use splice_after/before as appropriate.

Подождите, я шучу.
В стандартном API вы можете выбрать узел, где хотите оказаться и затем просто вызывать splice_after/splice_before в зависимости от того, что вам нужно.

> *squints*

*прищуривается*

> Wait is the std API actually good.

Подождите, а стандартный API действительно хорош.

> *skims through the code*

*пробегает код глазами*

> Ok the std API is actually good.

Правда, стандартный API действительно хорош.

> Alright screw it, we're going to [implement the RFC](https://github.com/rust-lang/rfcs/blob/master/text/2570-linked-list-cursors.md).
> Or at least the interesting parts of it.

Ладно, к чёрту всё, будем [реализовывать RFC](https://github.com/rust-lang/rfcs/blob/master/text/2570-linked-list-cursors.md).
Или по крайней мере самые интересные части.

> I have my quibbles with some of the terminology std uses, but cursors are always going to be a bit brain-melty: `iter().next_back()`  gets you `back()`, so that's good, but then each subsequent `next_back()` is actually bringing you *closer to the front* and indeed, every pointer we follow is a "front" pointer!
> If I think about this seeming-paradox too much it hurts my brain, so, I can certainly respect going for different terminology to avoid this.

У меня есть некоторые претензии к терминологии, которую использует стандартная библиотека, но курсоры всегда немного сбивают с толку: `iter().next_back()` означает по сути `back()` (то есть назад), что в целом правильно, но каждый вызов `next_back()` делает нас *ближе к переднему элементу*, что значит, что мы двигаемся не назад, а вперёд!
Когда я начинаю слишком много думать об этом кажущемся парадоксе, у меня болит голова, так что я могу понять необходимость в другой терминологии, чтобы избежать таких проблем.

> The std API talks about operations before "before" (towards the front) and "after" (towards the back), and instead of `next` and `next_back`, it... calls things `move_next` and `move_prev`.
> HRM.
> Ok so they're getting into a bit of the iterator terminology, but at least `next` doesn't evoke front/back, and helps you orient how things behave compared to the iterators.

Стандартный API операции описываются, как «before» (в сторону переднего элемента) и «after» (в сторону заднего элемента), и вместо `next`/`back_next`... есть вызовы `move_next` и `move_prev`.
ГРМ.
Хорошо, они позаимствовали немного терминологии итераторов, но, в конце концов `next` не вызывает неверных ассоциаций по отношению к front/back и помогает понять, как всё работает по сравнению с итераторами.

> We can work with this.

С этим можно работать.
