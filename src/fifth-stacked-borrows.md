> # Attempting To Understand Stacked Borrows

# Пытаемся разобраться в стековом заимствовании

> In the previous section we tried running our unsafe singly-linked queue under miri, and it said we had broken the rules of *stacked borrows*, and linked us some documentation.

В предыдущем разделе мы попытались запустить нашу небезопасную односвязную очередь под miri, и он сказал, что мы нарушили правила *стековых заимствований* и дал нам ссылку на документацию.

> Normally I'd give a guided tour of the docs, but we're not really the target audience of that documentation.
> It's more designed for compiler developers and academics who are working on the semantics of Rust. 

Обычно мне бы следовало сделать обзор доки, но на самом деле мы не являемся целевой аудиторией этой документации.
Она спроектирована больше для разработчиков компилятора и академиков, которые работают над семантикой Rust.

> So I'm going to just give you the high level *idea* of "stacked borrows", and then give you a simple strategy for following the rules.

Так что я собираюсь просто дать вам высокоуровневую *идею* «стековых заимствований», а затем дать вам простую стратегию следования правилам.

> > **NARRATOR:** Stacked borrows are still "experimental" as a semantic model for Rust, so breaking these rules may not actually mean your program is "wrong".
> > But unless you literally work on the compiler, you should just fix your program when miri complains.
> > Better safe than sorry when it comes to Undefined Behaviour.

> **ГОЛОС ЗА КАДРОМ:** многоуровневые заимствования остаются «экспериментальными» в качестве семантической модели Rust, так что нарушение этих правил не означает, что у вас «неправильная» программа.
> Но если вы буквально не разработчик компилятора, вам следует просто исправить вашу программу, когда miri жалуется.
> Это намного безопаснее, чем сожалеть, когда наступит Неопределённое Поведение.

> # The Motivation: Pointer Aliasing

# Мотивация: псевдономизация указателей

> Before we get into *what* rules we've broken, it will help to understand *why* the rules exist in the first place.
> There are a few different motivating problems, but I think the most important one is *pointer aliasing*.

Перед тем, как узнать, *какие* правила мы нарушили, прежде всего будет полезно разобраться, *почему* эти правила вообще существуют.
Есть несколько различных обуславливающих проблем, но я думаю, что самой важной является *псевдономизация указателей*.

> We say two pointers *alias* when the pieces of memory they point to overlap.
> Just as someone who "goes by an alias" can be referred to by two different names, that overlapping piece of memory can be referred to by two different pointers.
> This can lead to problems.

Мы говорим, что два указателя являются *псевдонимами*, когда области памяти, на которые они указывают, перекрываются.
Точно также, как кого-то «известного под псевдонимом» можно называть двумя различными именами, эта перекрывающая область памяти может быть доступна под двум различным указателям.
Это может приводить к проблемам.

> The compiler uses information about pointer aliasing to optimize accesses to memory, so if the information it has is *wrong* then the program will be miscompiled and do random garbage. 

Компилятор использует информацию о псевдономизации указателя, чтобы оптимизировать доступ к памяти, так что если информация, на которую он опирается *ошибочная*,тогда программа будет скомпилирована неправильно, и будет производить случайный мусор.

> > **NARRATOR:** Practically speaking, aliasing is more concerned with memory accesses than the pointers themselves, and only really matters when one of the accesses is mutating.
> > Pointers are emphasized because they're a convenient thing to attach rules to.

> **ГОЛОС ЗА КАДРОМ:** на практике, псевдономизация больше связана с доступом к памяти, чем с самими указателями, и имеет значение только тогда, когда один из указателей является изменяющим.
> Внимание именно к указателям связано с тем, что к ним удобно прикреплять правила.

> To understand why pointer aliasing information is important, let's consider *The Parable of the Tiny Angry Man*. 

Чтобы понять, почему информация о псевдономизации указателей важна, давайте рассмотрим *Притчу о сердитом человечке*.

----

> Michiel was looking through their bookshelf one day when they saw a book they didn't remember.
> They pulled it from the bookcase and looked at the cover. 

Однажды Михил смотрел на свою книжную полку и увидел незнакомую книгу.
Он снял её с полки и посмотрел на обложку.

> "Oh yes, my old copy of *War and Peace*, a book I definitely have read.
> I loved the part with all the Peace."

«Ах, да, мой старый экземпляр *Войны и мира*, книга, которую я определённо читал.
Люблю часть, посвящённую миру.»

> Suddenly there was a knock at the door.
> Michiel returned the book to its shelf and opened the door -- it was their sworn nemesis **Hamslaw**.
> As Hamslaw prepared a devastating remark about Michiel's clearly inferior codegolfing skills, they sensed an opening:

Внезапно в дверь постучали.
Михил вернул книгу на полку и открыл дверь — там стояла его непримиримая противница Хамслава.

> "Hey Hamslaw, have you ever read War and Peace?"

«Привет, Хамслава, ты когда-нибудь читала Войну и мир?»

> "Pfft, no one's *actually* read War and Peace."

«Пффф, *на самом деле* никто не читал Войну и мир.»

> "Well I have, look it's right there in my bookcase, which *obviously* means I've read it."

«Ну, а я читал, смотри, она у меня прямо на полке, что *очевидно* означает, что я её читал.»

> Hamslaw couldn't believe it.
> Her face shifted from its usual smug demeanor to an iron mask of rage and determination.
> Hamslaw pushed Michiel aside and power-walked to the book shelf, cleaving the tome from its resting place with the fury of a thousand Valkyries.
> She turned the ancient text over in her hands, and the instant she saw the cover she began to shake.

Хамслава не могла в это поверить.
Её лицо сменило обычное самодовольное выражение на железную маску ярости и решимости.
Оттолкнув Михила в сторону, Хамслава решительным шагом направилась к книжной полке, с яростью тысячи Валькирий вырвала том с его законного места.
Она перевернула в руках древний текст и, увидев обложку, задрожала.

> Michiel prepared to boast of their clearly unparalleled brilliance, but was interrupted by the sudden laughter of Hamslaw.

Михил уже был готов насладиться своим очевидным превосходством, но его прервал внезапный смеха Хамславы.

> "This isn't War and Peace, this is War and *Feet*!"

«Это не Война и мир, это Война из *дыр*!

<!-- Пытался передать рифму из английского текста -->

> Tears were rolling down Hamslaw's face.
> This was clearly the greatest moment of her life.

Слёзы текли по лицу Хамславы.
Несомненно, это был лучший момент в её жизни.

> "N-no! I just looked at it!"

«Н-нет! Я же только что смотрел!»

> They grabbed the book from Hamslaw and checked the cover.
> Indeed, the word "Peace" had been scratched out and replaced with "Feet".
> Michiel was mortified.
> This was clearly the worst moment of their life.

Он вырвал книгу из рук Хамславы и посмотрел на обложку.
Действительно, слова «и мир» были зачёркнуты и исправлены на «из дыр».
Михил в ужасе застыл.
Несомненно, это был худший момент в его жизни.

> They fell to their knees and stared blankly at the bookcase.
> How could this have happened?
> They had checked the cover only a moment ago!

Он упал на колени и беспомощно уставился на книжный шкаф.
Как это могло произойти?
Он же видел обложку мгновенье назад!

> And then they saw a bit of motion in the bookcase.
> It was a tiny man.
> A tiny many with the angriest scowl Michiel had ever seen.
> The tiny man flipped Michiel off and mouthed the words "no one will believe you" and disappeared back between the books.

Тут он заметил какое-то движение в шкафу.
Это был человечек.
Человечек с самым сердитым выражением лица, которое Михил когда-либо видел.
Он показал Михилу средний палец и, сказав «тебе никто не поверит», скрылся между книгами.

> Michiel's plan *had* been perfect, but they had failed to account for the possibility of a tiny angry man with a sharpie and the desire for destruction.
> They thought they knew what the cover of the book said, and they thought that no one could have possibly changed it.
> But alas, they were wrong.

План Михила *был* идеальным, но не учитывал появления сердитого человечка с маркером и жаждой разрушения.
Он думал, что знает, что написано на обложке и считал, что никто не может этого изменить.
Но, увы, он ошибался.

> Hamslaw was already working on a zine commemorating her incredible victory &mdash; Michiel's reputation at the local Internet Cafe would never recover.

А Хамслава в красках описала свою невероятную победу в стенгазете, так что репутация Михила в местном Интернет-кафе так никогда и не восстановилась.

----

> No one wants to be like Michiel, but no one wants to live in constant fear of the tiny angry man either.
> We want to know when the tiny angry man could be playing tricks on us.
> When he is, we will be very careful and paranoid about checking everything before we use it.
> But when the tiny angry man is gone, we want to be able to remember things.

Никто бы не захотел оказаться на месте Михила, но никто бы и не хотел жить в постоянном страхе сердитого человечка.
Нам бы хотелось знать, когда этот сердитый человечек может над нами подшутить.
В этот момент мы были бы очень осторожны, и параноидально проверяли всё перед использованием.
Но в другое время мы бы хотели доверять собственной памяти.

> That's the (very simplified) crux of pointer aliasing: when can the compiler assume it's safe to "remember" (cache) values instead of loading them over and over?
> To know that, the compiler needs to know whenever there *could* be little angry men mutating the memory behind your back.

Таким образом (очень упрощённо) ключевой момент псевдономизации указателей: когда компилятор может предположить, что безопасно «запомнить» (кешировать) значения, вместо того, чтобы загружать их снова и снова.
Чтобы это узнать, компилятору надо знать про все ситуации, когда сердитый человечек *мог бы* изменить память за вашей спиной.

> > **NARRATOR:** the compiler also uses this information to cache stores, which just means it can avoid committing things to memory if it thinks no one will notice.
> > In this case the problem is still tiny angry men, but they only need to read the memory for it to be a problem.

> **ГОЛОС ЗА КАДРОМ:** компилятор также использует эту информацию для кеширования хранений, что означает, что он избегает отправки данные в память, если думает, это этого никто не заметит.
> В этом случае проблема всё ещё в сердитых человечках, но им надо прочитать память, чтобы это стало проблемой.

> # Safe Stacked Borrows

# Безопасные стековые заимствования

> Ok so we want the compiler to have good pointer aliasing information, can we do that?
> Well, seemingly Rust is *designed* for it.
> Mutable references aren't aliased by definition, and although shared references *can* alias eachother, they can't mutate.
> Perfect
> Ship it!

Ладно, значит мы хотим, чтобы у компилятора была полная информация о псевдономизации указателей, можем ли мы её предоставить?
Ну, выглядит так, что Rust как раз для этого и *спроектирован*.
Изменяемые ссылки по определению не могут иметь псевдонимов, а разделяемые ссылки, хотя и *могут* быть псевдонимами друг друга, не могут изменяться.
Прекрасно.
Отправляй в продакшн!

> Except it's more complicated than that.
> We can "reborrow" mutable pointers like this:

Но на самом деле всё гораздо сложнее.
Мы можем «повторно заимствовать» изменяемые указатели, как здесь:

```rust
let mut data = 10;
let ref1 = &mut data;
let ref2 = &mut *ref1;

*ref2 += 2;
*ref1 += 1;

println!("{}", data);
```

> The compiles and runs fine.
> What's the deal? 

Компилируется и запускается без проблем.
Почему?

> Well we can see what's going on by swapping the two uses:

Ну, мы можем увидеть, что происходит, поменяв местами две строки:

```rust ,ignore
let mut data = 10;
let ref1 = &mut data;
let ref2 = &mut *ref1;

// ORDER SWAPPED!
// ПОРЯДОК ИЗМЕНИЛСЯ!
*ref1 += 1;
*ref2 += 2;

println!("{}", data);
```

```text
error[E0503]: cannot use `*ref1` because it was mutably borrowed
 --> src/main.rs:6:5
  |
4 |     let ref2 = &mut *ref1;
  |                ---------- borrow of `*ref1` occurs here
5 |     
6 |     *ref1 += 1;
  |     ^^^^^^^^^^ use of borrowed `*ref1`
7 |     *ref2 += 2;
  |     ---------- borrow later used here

For more information about this error, try `rustc --explain E0503`.
error: could not compile `playground` due to previous error
```

> It's suddenly a compiler error!

Внезапно теперь мы получаем ошибку компилятора!

> When we reborrow a mutable pointer, the original pointer can't be used anymore until the borrower is done with it (no more uses). 

Когда мы заимствуем изменяемый указатель повторно, оригинальный указатель не может быть больше использован, пока заёмщик не закончит с ним работать (больше не будет использований).

> In the code that works, there's a nice little nesting of the uses.
> We reborrow the pointer, use the new pointer for a while, and then stop using it before using the older pointer again.
> In the code that *doesn't* work, that doesn't happen.
> We just interleave the uses arbitrarily.

В коде, который работает присутствует удобная вложенность использований.
Мы заимствуем указатель повторно, используем новый указатель в течение какого-то времени, и затем перестаём использовать его до того, как снова использовать старый указатель.
В коде, который *не* работает, этого не происходит.
Мы просто чередуем использования не упорядочено.
<!-- в оригинале arbitrarily == произвольно, но здесь речь не о случайном порядке, а о таком, когда использования не сгруппированы, не упорядочены -->

> This is how we can have reborrows and still have aliasing information: all of our reborrows clearly nest, so we can consider only one of them "live" at any given time.

Именно так мы и можем иметь несколько заимствований и в то же время иметь информацию о псевдонимизации: все наши заимствования явным образом сложены, так что мы можем только один из них считать «живым» в каждый момент времени.

> Hey, you know what's a great way to represent cleanly nested things?
> A stack.
> A stack of borrows.

Эй, а вы знаете, как лучше всего представить явным образом вложенные штуки?
Стек.
Стек заимствований.

> Oh hey it's *Stacked Borrows*!

Ага, вот и *стек заимствований*!

> Whatever's at the top of the borrow stack is "live" and knows it's effectively unaliased.
> When you reborrow a pointer, the new pointer is pushed onto the stack, becoming *the* live pointer.
> When you use an older pointer it's brought back to life by popping everything on the borrow stack above it.
> At this point the pointer "knows" it was reborrowed and that the memory might have been modified, but that it once more has exclusive access -- no need to worry about little angry men.

Объект, находящийся на вершине стека, является «живым» и знает, что у него фактически нет псевдонимов.
Когда вы заимствуете указатель повторно, новый указатель вставляется в начало стека, становясь *новым* живым указателем.
Когда вы используете старый указатель, он возвращается к жизни путём удаления из стека всего, что выше него.
В этой точке указатель «знает», что был заимствован и что память могла быть изменена, но теперь он снова имеет эксклюзивный доступ — нет надобности беспокоиться о сердитом человечке.

> So it's actually *always* ok to access a reborrowed pointer, because we can always pop everything above it.
> The real trouble is accessing a pointer that has already been popped off of the borrow stack -- then you've messed up.

Дак что в действительности это *всегда* нормально обращаться к повторно заимствованному указателю, поскольку мы всегда можем удалить всё, что выше него.
Реальная проблема возникает при доступе к указателю, который уже был удалён из стека заимствований — тогда у вас всё испортилось.

> Thankfully the design of the borrowchecker ensures that safe Rust programs follow these rules, as we saw in the above example, but the compiler generally views this problem "backwards" from the stacked borrows perspective.
> Instead of saying using `ref1` invalidates `ref2`, it insists that `ref2` *must* be valid for all its uses, and that `ref1` is the one messing things up by going out of turn.

К счастью, конструкция анализатора заимствований гарантирует, что безопасные программы на Rust следуют этим правилам, как мы видели в примере выше, но компилятор обычно видит эту проблему «с другой стороны», с точки зрения стековых заимствований.
Вместо того, что утверждать, что использование `ref1` ломает `ref2`, он настаивает, что именно `ref2` *должен* быть корректным в течение всего времени использования, и что `ref` является той самой штукой, которая всё портит, действуя вне очереди.

> Hence "cannot use `*ref1` because it was mutably borrowed".
> It's the same result (especially with non-lexical lifetimes), but framed in a way that's probably more intuitive.

Следовательно, «нельзя использовать `*ref1`, поскольку он был заимствован изменяемым образом».
Это тот же самый результат (особенно с не-лексическими временами жизни), но, возможно, оформленный в более интуитивном виде.

> But the borrowchecker can't help us when we start using unsafe pointers!

Но анализатор заимствований не может нам помочь, когда мы начинаем использовать небезопасные указатели!

> # Unsafe Stacked Borrows

# Небезопасные стековые заимствования

> So we want to somehow have a way for unsafe pointers to participate in this stacked borrows system, even though the compiler can't track them properly.
> And we also want the system to be fairly permissive so that it's not *too* easy to mess it up and cause UB.

Таким образом мы хотим каким-то образом заставить небезопасные указатели участвовать в этой системе стековых заимствований, даже если компилятор не может их корректно отслеживать.
И также мы хотим, чтобы система была достаточно гибкой, чтобы её было не *слишком* легко сломать и вызвать UB.

> That's a hard problem, and I don't know how to solve it, but the folks who worked on Stacked Borrows came up with something plausible, and miri tries to implement it.

Это трудная проблема, и я не знаю, как её решить, но ребята, работавшие над стековыми заимствованиями, придумали что-то, что внушает доверие, и miri пытается это реализовать.

> The very high-level concept is that when you convert a reference (or any other safe pointer) into an raw pointer it's *basically* like taking a reborrow.
> So now the raw pointer is allowed to do whatever it wants with that memory, and when the reborrow expires it's just like when that happens with normal reborrows.

В самом общем смысле, когда вы преобразуете ссылку (или любой другой безопасный указатель) в сырой указатель это *по сути* выглядит как повторное заимствование.
Так что теперь сырой указатель может делать с этой памятью, что захочет, и когда срок заимствования истекает, всё происходит так же, как и при обычном повторном заимствовании.

> But the question is, when does that reborrow expire?
> Well, probably a good time to expire it is when you start using the original reference again.
> Otherwise things aren't a nice nested stack.

Весь вопрос в том, когда истекает срок повторного заимствования?
Ну, возможно, лучшее время для завершения заимствования — это когда вы начинаете использовать оригинальную ссылку.
В противном случае всё не будет выглядеть как аккуратный вложенный стек.

> But wait, you can turn a raw pointer *into* a reference!
> And you can copy raw pointers!
> What if you go `&mut -> *mut -> &mut -> *mut` and then access the first `*mut`?
> How the heck do the stacked borrows work then?

Но, подождите, вы можете превратить сырой указатель *в* ссылку!
И вы можете копировать сырые указатели!
Что будет, если вы сделаете `&mut -> *mut -> &mut -> *mut`, а затем обратитесь к первому `*mut`?
Как, блин, стековые заимствования работают тогда?

> I genuinely don't know!
> That's why things are complicated.
> In fact they're *extra* complicated because stacked borrows are *trying* to be more permissive and let more unsafe code work the way you'd expect it to.
> This is why I run things under miri to try to help me catch mistakes.

Честно говоря, я не знаю!
Вот почему всё так сложно.
Фактически, всё *ещё* сложнее, потому что стековые заимствования *пытаются* быть более снисходительными и позволять более небезопасному коду работать способом, который вы от него ожидаете.
Вот поэтому я запускаю miri, чтобы попытаться выявить ошибки.

> In fact, this messiness is why there is an extra-experimental extra-strict mode of miri: `-Zmiri-tag-raw-pointers`.

Фактически, эта неразбериха является причиной появления экстра-экспериментального экстра-строгого режима miri: `-Zmiri-tag-raw-pointers`.

> To enable it, we need to pass it via a MIRIFLAGS environment variable like this:

Чтобы включить этот режим, надо передать флаг через переменную окружения MIRIFLAGS, как здесь:

```text
MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri test
```

> Or like this on Windows, where you need to just set the variable globally:

Или, как в Windows, где нужно просто глобально установить переменную:

```text
$env:MIRIFLAGS="-Zmiri-tag-raw-pointers"
cargo +nightly-2022-01-21 miri test
```

> We'll generally be trying to conform to this extra-strict mode just to be *extra* confident in our work.
> It's also in some sense "simpler", so it's actually better for messing around and getting an intuition for stacked borrows.

В целом, мы будем придерживаться этого экстра-строгого режима, просто чтобы быть *экстра* уверенными в нашей работе.
Он также в некотором смысле «проще», поэтому лучше подходит для экспериментов и формирования интуитивного понимания стековых заимствований.

> # Managing Stacked Borrows

# Управление стековыми заимствованиями 

> So when using raw pointers we're going to try to stick to a heuristic that's simple and blunt and will hopefully have a large margin of error: 

Поэтому при использовании сырых указателей мы будем стараться придерживаться простой и прямолинейной эвристики, которая, я надеюсь, имеет большую толерантность к ошибкам:

> **Once you start using raw pointers, try to ONLY use raw pointers.**

**Как только вы стали использовать сырые указатели, старайтесь использовать ТОЛЬКО сырые указатели.**

> This makes it as unlikely as possible to accidentally lose the raw pointer's "permission" to access the memory.

Это сильно снижает возможность непредумышленной потери «права» сырого указателя на доступ к памяти.

> > **NARRATOR:** this is oversimplified in two regards:
> > 1. Safe pointers often assert more properties than just aliasing: the memory is allocated, it's aligned, it's large enough to fit the type of the pointee, the pointee is properly initialized, etc.
> >    So it's even more dangerous to wildly throw them around when things are in a dubious state.
> > 2. Even if you stay in raw pointer land, you can't just wildly alias any memory.
> >    Pointers are conceptually tied to specific "allocations" (which can be as granular as a local variable on the stack), and you're not supposed to take a pointer from one allocation, offset it, and then access memory in a different allocation.
> >    If this was allowed, there would *always* be the threat of tiny angry men *everywhere*.
> >    This is part of the reason "pointers are just integers" is a *problematic* viewpoint.

> **ГОЛОС ЗА КАДРОМ:** у этого упрощения есть два аспекта:
> 1. У безопасных указателей часто есть другие свойства, помимо псевдономизации: память выделена, выровнена, её достаточно для хранения объекта указывания, объект указывания инициализирован и т. д.
>    Так что это более опасно — разбрасывать их везде, когда они находятся в нестабильном состоянии.
> 2. Даже если вы остаётесь в рамках использования сырых указателей, вы просто не можете использовать псевдонимы для доступа к любой памяти.
>    Указатели концептуально привязаны к определённым «областям выделенной памяти» (которые могут быть такими же мелкими, как и локальная переменная на стеке), и поэтому вы не можете брать указатель из одной области, прибавлять к ему смещение и и обращаться к другой области.
>    Если бы это было возможно, угроза сердитых человечков была бы *всегда* и *везде*.
>    Именно по этой причине точка зрения «указатели — всего лишь целые числа» является проблематичной.

> Now, we still want safe references in our *interface*, because we want to build a nice *safe abstraction* so the user of our list doesn't have to know or worry about. 

Теперь, мы всё ещё хотим иметь безопасные ссылки в нашем *интерфейсе*, потому что мы хотим строить красивые *безопасные абстракции*, так чтобы пользователю нашего списка не нужно было ни о чём беспокоиться.

> So what we're going to do is:

Итак, что мы будем делать:

> 1. At the start of a method, use the input references to get our raw pointers
> 2. Do our best to only use unsafe pointers from this point on
> 3. Convert our raw pointers back to safe pointers at the end if needed

1. В начале метода использовать входные ссылки, чтобы получить наши сырые указатели
2. С этого момента стараться использовать только сырые указатели
3. В конце, если нужно, преобразовывать наши сырые указатели в в безопасные указатели

> But the fields of our types are private so we're going to keep those *entirely* as raw pointers.

Но поля наших типов приватны, поэтому мы будем хранить *их все* в виде сырых указателей.

> In fact, part of the big mistake we made was continuing to use Box!
> Box has a special annotation in it that tells the compiler "hey this is a lot like `&mut`, because it uniquely owns that pointer".
> Which is true!

Фактически, часть большой ошибки, которую мы совершили была в том, что мы продолжили использовать Box!
Box имеет специальную аннотацию, которая говорит компилятору «эта штука очень похожа на `&mut`, потому что эксклюзивно владеет указателем».
Что правда!

> But the raw pointer we were keeping to the end of the list was pointing into a Box, so whenever we access the Box normally we're probably invalidating that raw pointer's "reborrow"! ☠

Но сырой указатель, в котором мы хранили конец списка, указывал на Box, поэтому всякий раз, когда мы обращаемся к Box обычным образом, мы, возможно, ломаем повторное заимствование этого сырого указателя!

> In the next section we'll return to our true form and hit our heads against a bunch of examples.

В следующем разделе мы вернёмся в нашему привычному формату и поломаем головы над целым ворохом примеров.
