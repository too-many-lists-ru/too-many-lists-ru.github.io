> # Learn Rust With Entirely Too Many Linked Lists

# Прорва связных списков, чтобы выучить Rust

> > Got any issues or want to check out all the final code at once?
> > [Everything's on Github!][github]

> Есть какие-то проблемы или хотите сразу получить весь финальный код?

> > **NOTE**: The current edition of this book is written against Rust 2018, which was first released with rustc 1.31 (Dec 8, 2018).
> > If your rust toolchain is new enough, the Cargo.toml file that `cargo new` creates should contain the line `edition = "2018"` (or if you're reading this in the far future, perhaps some even larger number!).
> > Using an older toolchain is possible, but unlocks a secret **hardmode**, where you get extra compiler errors that go completely unmentioned in the text of this book.
> > Wow, sounds like fun!

> **ПРИМЕЧАНИЕ**: Текущая редакция этой книги написана для Rust 2018, который был впервые выпущен вместе с rustc 1.31 (8 декабря 2018).
> Если у вас достаточно новый инструментарий Rust, файл Carto.toml, который создаст <!-- команда --> `cargo new`, должен содержать строку `edition = "2018"` (или, если вы читаете это в далёком будущем, возможно, какое-то ещё большее число!).
> Использование более старого инструментария возможно, но разблокирует секретный **режим повышенной сложности**, в котором вы получаете от компилятора дополнительные ошибки, совершенно не упомянутые в этой книге.
> Ух ты, звучит весело!

> I fairly frequently get asked how to implement a linked list in Rust.
> The answer honestly depends on what your requirements are, and it's obviously not super easy to answer the question on the spot.
> As such I've decided to write this book to comprehensively answer the question once and for all.

Меня довольно часто спрашивают, как реализовать в Rust односвязный список.
Ответ определённо зависит от того, каковы ваши требования, и очевидно не так просто ответить на этот вопрос сразу.
Так что я решил написать эту книгу, чтобы всесторонне ответить на вопрос раз и навсегда.

> In this series I will teach you basic and advanced Rust programming entirely by having you implement 6 linked lists.
> In doing so, you should learn:

В этом цикле я научу вас вместе и базовому, и продвинутому программированию на Rust, с помощью реализации 6 связных списков.
При этом вы изучите:

> * The following pointer types: `&`, `&mut`, `Box`, `Rc`, `Arc`, `*const`, `*mut`, `NonNull`(?)
> * Ownership, borrowing, inherited mutability, interior mutability, Copy
> * All The Keywords: struct, enum, fn, pub, impl, use, ...
> * Pattern matching, generics, destructors
> * Testing, installing new toolchains, using `miri`
> * Unsafe Rust: raw pointers, aliasing, stacked borrows, UnsafeCell, variance

* Следующие типы указателей: `&`, `&mut`, `Box`, `Rc`, `Arc`, `*const`, `*mut`, `NonNull`(?)
* Владение, заимствование, наследуемую мутабельность, внутреннюю мутабельность, <!-- семантику? типаж? --> копирования
* Все ключевые слова: struct, enu, fn, pub, impl, use, ...
* Сопоставление с образцом, обобщения, деструкторы
* Тестирование, установку новых инструментов, использование `miri`
* Небезопасный Rust: сырые указатели, псевдонимы, многоуровневые заимствования, вариантность

> Yes, linked lists are so truly awful that you deal with all of these concepts in making them real.

Да, связные списки именно так ужасны, что вам придётся изучить все эти концепции, чтобы сделать их реальными.

> Everything's in the sidebar (may be collapsed on mobile), but for quick reference, here's what we're going to be making:

Полное содержание книги есть в боковой панели (может быть свёрнута на смартфоне), но для быстрой справки вот что мы собираемся сделать:

> 1. [A Bad Singly-Linked Stack](first.md)
> 2. [An Ok Singly-Linked Stack](second.md)
> 3. [A Persistent Singly-Linked Stack](third.md)
> 4. [A Bad But Safe Doubly-Linked Deque](fourth.md)
> 5. [An Unsafe Singly-Linked Queue](fifth.md)
> 6. [TODO: An Ok Unsafe Doubly-Linked Deque](sixth.md)
> 7. [Bonus: A Bunch of Silly Lists](infinity.md)

1. [Неправильный односвязный список](first.md)
2. [Правильный односвязный список](second.md)
3. [Устойчивый односвязный стек](third.md)
4. [Неправильный, но безопасный двухсвязный дек](fourth.md)
5. [Небезопасная односвязная очередь](fifth.md)
6. [TODO: Правильный небезопасный двухсвязный дек](sixth.md)
7. [Бонус: Куча глупых списков](infinity.md)

> Just so we're all the same page, I'll be writing out all the commands that I feed into my terminal.
> I'll also be using Rust's standard package manager, Cargo, to develop the project.
> Cargo isn't necessary to write a Rust program, but it's *so much* better than using rustc directly.
> If you just want to futz around you can also run some simple programs in the browser via [play.rust-lang.org][play].

Чтобы мы понимали друг друга, я буду записывать все команды, которые я ввожу в свой терминал.
Я таже буду пользовать стандартный расптовский пакетный менеджер Cargo для разработки проекта.
Cargo не является необходимым для написания программы на Rust, но он *намного* лучше, чем ручной вызов rustc.
Если вы просто хотите поэкспериментировать, вы также можете запустить некоторые простые программы в браузере через [play.rust-lang.org][play].

> In later sections, we'll be using "rustup" to install extra Rust tooling.
> I strongly recommend [installing all of your Rust toolchains using rustup](https://www.rust-lang.org/tools/install).

В следующих разделах мы будем использовать "rustup", чтобы установить дополнительные инструменты Rust.
Я настоятельно рекомендую [устанавливать весь ваш инструментарий Rust, используя rustup](https://www.rust-lang.org/tools/install).

> Let's get started and make our project:

Давайте начнём и создадим наш проект:

```text
> cargo new --lib lists
> cd lists
```

> We'll put each list in a separate file so that we don't lose any of our work.

Мы будем помещать каждый список в отдельный файл, так что мы ничего не потеряем из нашей работы.

> It should be noted that the *authentic* Rust learning experience involves writing code, having the compiler scream at you, and trying to figure out what the heck that means.
> I will be carefully ensuring that this occurs as frequently as possible.
> Learning to read and understand Rust's generally excellent compiler errors and documentation is *incredibly* important to being a productive Rust programmer.

Следует заметить, что *аутентичный* способ изучения Rust состоит из написания кода, на который ругается компилятор и попыток понять, какого хрена все эти ругательства означают.
<!-- heck -- эвфемизм от hell, можно переводить эвфемизмом -->
Я будут тщательно следить за тем, чтобы это случалось как можно чаще.
Умение читать и понимать в целом превосходные ошибки компилятора Rust и документацию — *невероятно* важны для того, чтобы стать продуктивных программистом на Rust.

> Although actually that's a lie.
> In writing this I encountered *way* more compiler errors than I show.
> In particular, in the later chapters I won't be showing a lot of the random "I typed (copy-pasted) bad" errors that you expect to encounter in every language.
> This is a *guided tour* of having the compiler scream at us.

Хотя в дествительности это ложь.
При написании этого текста, я встретил *значительно* больше ошибок компилятора, чем показываю вам.
В частности, в последующих главах я не стану показывать множество случайных ошибок "Я опечатался или плохо скопи-пастил", которые вы ожидаете встретить в каждом языке.
Это — *экскурсия с гидом* о том, как компилятор будет на нас ругаться.

> We're going to be going pretty slow, and I'm honestly not going to be very serious pretty much the entire time.
> I think programming should be fun, dang it!
> If you're the type of person who wants maximally information-dense, serious, and formal content, this book is not for you.
> Nothing I will ever make is for you.
> You are wrong.

Мы собираемся двигаться досточно медленно, и я честно не собираюсь быть слишком серьёзным всё это время.
Я, блин, думаю, что программирование должно быть весёлым!
<!-- dang -- эвфемизм от damn, можно переводить эвфемизмом -->
Если вы — тот тип читателя, которому нужен максильно инфрмативный, серьёзный и формальный текст <!-- в оригинале: содержание -->, то эта книга не для вас.
Вообще всё, что я когда-либо напишу — не для вас.
Вы неправы.

> # An Obligatory Public Service Announcement

# Обязательно общественное обращение <!-- юридический термин -->

> Just so we're totally 100% clear: I hate linked lists.
> With a passion.
> Linked lists are terrible data structures.
> Now of course there's several great use cases for a linked list:

Просто чтобы всё было предельно ясным: я ненавижу связные списки.
Со страстью.
Связные списки — ужасные структуры данных.
Хотя, конечно, есть несколько отличных вариантов использования <!-- сценариев --> для связных списков:

> * You want to do *a lot* of splitting or merging of big lists.
>   *A lot*.
> * You're doing some awesome lock-free concurrent thing.
> * You're writing a kernel/embedded thing and want to use an intrusive list.
> * You're using a pure functional language and the limited semantics and absence of mutation makes linked lists easier to work with.
> * ... and more!

* Вы хотите сделать *множество* разделений и слияний больших списков.
  *Множество*.
* Вы делаете какую-то удивительную неблокирующую конкурентную штуку.
* Вы пишите системную/низкоуровневую штуку и хотите использовать интрузивные списки.
* Вы используете чистый функциональный язык и ограниченная семантика вкупе с иммутабельностью делают связные списки простейшим решением.
* ...ну и так далее!

> But all of these cases are *super rare* for anyone writing a Rust program.
> 99% of the time you should just use a Vec (array stack), and 99% of the other 1% of the time you should be using a VecDeque (array deque).
> These are blatantly superior data structures for most workloads due to less frequent allocation, lower memory overhead, true random access, and cache locality.

Но все эти варианты встречаются *супер редко* для любого, кто пишет на Rust.
В 99% случаев вы должны просто использовать Vec (массив-стек), а 99% из оставшегося 1% вы должны использовать VecDeque (массив-дек).
Это безусловно лучшие структуры данных для большинства рабочих задач из-за редких аллокаций, меньших накладных расходов на память, истинно произвольного доступа и локальности кэша.

> Linked lists are as *niche* and *vague* of a data structure as a trie.
> Few would balk at me claiming a trie is a niche structure that your average programmer could happily never learn in an entire productive career -- and yet linked lists have some bizarre celebrity status.
> We teach every undergrad how to write a linked list.
> It's the only niche collection [I couldn't kill from std::collections][rust-std-list].
> It's [*the* list in C++][cpp-std-list]!

Связные списки — такая же *нишевая* и *смутная* структура данных, как и префиксное дерево.
Мало кто станет возражать против моего утверждения, что префиксное дерево — нишевая структура, с которой средний программист к счастью никогда в своей карьере не столкнётся; и в то же время у связных списков какая-то необъяснимая <!-- причудливая, эксцентричная --> популярность.
Мы учим каждого студента как писать связный список.
Это единственная нишевая коллекция, которую [я не смог удалить из std::collections][rust-std-list].
Это — [*тот самый* список из C++][cpp-std-list]!
<!-- речь о том, что связные списки есть в любой стандартной библиотеке, хотя, по мнению автора, их там быть не должно -->

> We should all as a community say *no* to linked lists as a "standard" data structure.
> It's a fine data structure with several great use cases, but those use cases are *exceptional*, not common.

Мы, как сообщество, должны сказать *нет* связным спискам как «стандартной» структуре данных.
Это прекрасная структура данных с несколькими прекрасными сценариями использования, но эти сценарии — *исключения*, а не правила.

> Several people apparently read the first paragraph of this PSA and then stop reading.
> Like, literally they'll try to rebut my argument by listing one of the things in my list of *great use cases*.
> The thing right after the first paragraph!

По-видимому, некоторые люди прочитают первый параграф этого обращения <!-- PSA -- Public Service Announcement -- название раздела --> и бросят чтение.
Они, типа, <!-- like в начале предложение -- попытка передать мусорное слово в разговорной речи --> попытаются опровергнуть мою аргументацию, слово в слово процитировав один из пунктов моего списка *отличных сценариев*.
Который, как вы помните, следует сразу за первым параграфом!

> Just so I can link directly to a detailed argument, here are several attempts at counter-arguments I have seen, and my response to them.
> Feel free to skip to [the first chapter](first.md) if you just want to learn some Rust!

Просто чтобы у меня под рукой была ссылка на подробную дискуссию, я собрал в одном месте несколько контр-аргументов и свои ответы на них.
Можете смело переходить к [первой главе](first.md), если вы хотите просто выучить немного языка Rust!

> ## Performance doesn't always matter

## Производительность не всегда имеет значение

> Yes!
> Maybe your application is I/O-bound or the code in question is in some cold case that just doesn't matter.
> But this isn't even an argument for using a linked list.
> This is an argument for using *whatever at all*.
> Why settle for a linked list?
> Use a linked hash map!

Да!
Возможно, ваше приложение занимается в основном вводом и выводом, или настолько второстепенное, что время его работы действительно не имеет значения. <!-- some cold case -- буквально "висяк", проект, которым никто не хочет заниматься >
Но это не аргумент в пользу связного списка.
Это аргумент в пользу *чего угодно*.
Зачем ограничиваться связным списком?
Используйте связный ассоциативный массив!
<!-- Составная структура из стандартной библиотеки Java, обычно на русский язык не переводится. Может быть, имеет смысл заменить на что-то другое, что имеет устойчивый перевод. Да вот хотя бы префиксное дерево. -->

> If performance doesn't matter, then it's *surely* fine to apply the natural default of an array.

Если производительность не имеет значения, тогда *безусловно* в качестве структуры данных прекрасно подойдёт и массив.

> ## They have O(1) split-append-insert-remove if you have a pointer there

## У них разделение-добавление-вставку-удаление за O(1), если у вас там указатель

> Yep!
> Although as [Bjarne Stroustrup notes][bjarne] *this doesn't actually matter* if the time it takes to get that pointer completely dwarfs the time it would take to just copy over all the elements in an array (which is really quite fast).

Точняк! <!-- Немного сленговое Да -->
Хотя, как [заметил Бьёрн Страуструп][bjarne], *это на самом деле неважно*, если время получения этого указателя значительно превышает время, которое требуется для простого копирования элементов массива (что на практике довольно быстро).
<!-- completely dwarfs -- точный смысл: первое число гигантское по сравнению со вторым, или второе число карликовое по сравнению с первым. Можно обыграть. -->

> Unless you have a workload that is heavily dominated by splitting and merging costs, the penalty *every other* operation takes due to caching effects and code complexity will eliminate any theoretical gains.

Если только основная работа программы — не слияние и разделение, стоимость *любых других операций* сведёт на нет всю теоретическую выгоду из-за особенностей работы кэширования и сложности кода.

> *But yes, if you're profiling your application to spend a lot of time in splitting and merging, you may have gains in a linked list*.

*Но — да, если вы планируете посвятить своё приложение слиянию и разделению, связные списки могут принести вам пользу <!-- выгоду -->*.

> ## I can't afford amortization

## Я не могу позволить себе амортизированную сложность

> You've already entered a pretty niche space -- most can afford amortization.
> Still, arrays are amortized *in the worst case*.
> Just because you're using an array, doesn't mean you have amortized costs.
> If you can predict how many elements you're going to store (or even have an upper-bound), you can pre-reserve all the space you need.
> In my experience it's *very* common to be able to predict how many elements you'll need.
> In Rust in particular, all iterators provide a `size_hint` for exactly this case.

Вы уже оказались в достаточно нишевой зоне, поскольку многие могут позволить себе амортизированную сложность.
Даже для массивов амортизированная сложность — это *худший случай*.
Вам придётся платить амортизированную цену <!-- за операцию вставки --> просто потому, что вы используете массив.
Если вы можете предсказать, столько элементов планируете хранить (или даже если у вас есть только верхняя граница), вы можете зарезервировать столько памяти, сколько вам надо.
По моему опыту, достаточно *часто* можно предсказать, сколько элементов вам потребуется.
В частности, в Rust все итераторы предоставляют `size_hing` как раз на этот случай.

> Then `push` and `pop` will be truly O(1) operations.
> And they're going to be *considerably* faster than `push` and `pop` on linked list.
> You do a pointer offset, write the bytes, and increment an integer.
> No need to go to any kind of allocator.

В этом случае операции `push` и `pop` действительно будут выполняться за O(1).
И они окажутся значительно быстрее, чем `push` и `pop` на связных списках.
Вы смещаете указатель, пишите байты и увеличиваете целое число.
И нет необходимости выделять память.

> How's that for low latency?

Как вам такая низкая задержка? <!-- Можно: как вам такая скорость? -->

> *But yes, if you can't predict your load, there are worst-case latency savings to be had!*

*Но — да, если вы мне можете предсказать количество элементов, связные списки работают быстро даже в худшем случае!*

> ## Linked lists waste less space

## Связные списки экономят место

> Well, this is complicated.
> A "standard" array resizing strategy is to grow or shrink so that at most half the array is empty.
> This is indeed a lot of wasted space.
> Especially in Rust, we don't automatically shrink collections (it's a waste if you're just going to fill it back up again), so the wastage can approach infinity!

Так, это сложно.
«Стандартная» стратегия при изменении размера массива — следить, чтобы при увеличении или уменьшении пустой оставалась по меньшей мере половина массива.
При этом действительно тратится много места.
Особенно в Rust, где мы не уплотняем коллекции автоматически (это не имеет смысла, если вы собираетесь снова их заполнять), так что потери могут стремиться к бесконечности!

> But this is a worst-case scenario.
> In the best-case, an array stack only has three pointers of overhead for the entire array.
> Basically no overhead.

Но это худший случай.
В лучшем случае массив-стек имеет всего три дополнительных указателя.
Практически никаких накладных расходов.

> Linked lists on the other hand unconditionally waste space per element.
> A singly-linked list wastes one pointer while a doubly-linked list wastes two.
> Unlike an array, the relative wasteage is proportional to the size of the element.
> If you have *huge* elements this approaches 0 waste.
> If you have tiny elements (say, bytes), then this can be as much as 16x memory overhead (8x on 32-bit)!

С другой стороны, связные списки гарантированно расходуют место для каждого элемента.
Односвязные списки тратят один указатель, а двухсвязные — два.
В отличие от массива, относительные расходы обратно пропорциоальны размеру элементов.
Если элементы *огромные*, накладные расходы близки к 0.
Если элементы крошечные (скажем, байты), накладные расходы будут больше в 16 раз (в 8 — на 32-хбитных машинах)!

> Actually, it's more like 23x (11x on 32-bit) because padding will be added to the byte to align the whole node's size to a pointer.

А, на самом деле — в 23 раза (в 11 за 32-хбитные машинах) из-за выравнивания, которое будет добавлено, чтобы выровнять каждый узел по размеру указателя.

> This is also assuming the best-case for your allocator: that allocating and deallocating nodes is being done densely and you're not losing memory to fragmentation.

При этом предполагается лучший случай для выделения памяти: выделяемые и возвращаемые узлы находятся близко друг к другу, и память не терятся из-за фрагментации.

> *But yes, if you have huge elements, can't predict your load, and have a decent allocator, there are memory savings to be had!*

*Но — да, если у вас огромные элементы, вы не можете предсказать их количество и фрагментация не очень высокая — можно добиться экономии памяти!*

> ## I use linked lists all the time in &lt;functional language&gt;

## Я всё время использую связные списки в «функциональных языках»

> Great!
> Linked lists are super elegant to use in functional languages because you can manipulate them without any mutation, can describe them recursively, and also work with infinite lists due to the magic of laziness.

Великолепно!
Связные списки супер элегантны для использования в функциональных языках, поскольку вы можете манипулировать ими без всякой мутации, можете описывать их рекурсивно, а также, благодрая магии ленивых вычислений, работать с бесконечными списками.

> Specifically, linked lists are nice because they represent an iteration without the need for any mutable state.
> The next step is just visiting the next sublist.

Связные списки особенно хороши, поскольку они позволяют представить итерацию без необходимости изменять состояние программы.
Следующий шаг — просто перейти к следующему подсписку.

> Rust mostly does this kind of thing with [iterators][].
> They can be infinite and you can map, filter, reverse, and concatenate them just like a functional list, and it will all be done just as lazily!

В Rust подобные вещи делаются с помощью [итераторов][].
Они могут быть бесконечными, вы можете выполнять с ними операции `map`, `filter`, `reverse` и `concatenate`, как и с обычными функциональными списками и, помимо всего прочего, они ещё и ленивые!

> Rust also lets you easily talk about sub-arrays with *[slices][]*.
> Your usual head/tail split in a functional language is [just `slice.split_at_mut(1)`][split].
> For a long time, Rust had an experimental system for pattern matching on slices which was super cool, but the feature was simplified when it was stabilized.
> Still, [basic slice patterns][slice-pats] are neat!
> And of course, slices can be turned into iterators!

Кроме того, Rust позволяет вам легко работать с под-массивами с помощью *[срезов][]*.
Обычно в функцинальных языках списки делят на голову и хвост, [точно также, как это делает `slice.split_at_mut(1)`][split].
На протяжении долгого времени в Rust существовала экспериментальная система сопоставления с образцом для срезов, которая была очень крутой, но она была упрощена во время стабилизации.
Впрочем, [образцы для срезов][slice-pats] весьма изящны!
И, конечно, срезы можно превратить в итераторы!

> *But yes, if you're limited to immutable semantics, linked lists can be very nice*.

*Но — да, если вы ограничены иммутабельной семантикой, связные списки могут быть очень удобными*.

> Note that I'm not saying that functional programming is necessarily weak or bad.
> However it *is* fundamentally semantically limited: you're largely only allowed to talk about how things *are*, and not how they should be *done*.
> This is actually a *feature*, because it enables the compiler to do tons of [exotic transformations][ghc] and potentially figure out the *best* way to do things without you having to worry about it.
> However this comes at the cost of being *able* to worry about it.
> There are usually escape hatches, but at some limit you're just writing procedural code again.

Заметьте, я не утверждаю, что функциональное программирование обязательно бедное или плохое.
Однако, речь *идёт* о фундаментальном семантическом ограничении: вам позволено говорить о том, что вещи из себя *представляют*, но не позволено о том, что они должны *делать*.
В действительности это *фича*, которая позволяет компилятору выполнить тонны [экзотических преобразований][ghc] и может быть даже найти *лучший* способ решения задачи, снимая с вас необходимость думать об этом.
Однако это происходит ценой потери *возможность* думать об этом.
Как правило, есть обходные пути, но время от времени вам всё равно приходиться писать обычный процедурный код.

> Even in functional languages, you should endeavour to use the appropriate data structure for the job when you actually need a data structure.
> Yes, singly-linked lists are your primary tool for control flow, but they're a really poor way to actually store a bunch of data and query it.

Даже в функциональных языках следует стремиться использовать подходящую структуру данных для задач, где действительно нужна какая-то структура данных.
Да, односвязные списки — основной способ управлять потоком выполнения, но это действительно плохой способ хранить большой объём данных и искать в нём значения.

> ## Linked lists are great for building concurrent data structures!

## Связные списки отлично подходят для реализации потокобезопасных структур данных

> Yes!
> Although writing a concurrent data structure is really a whole different beast, and isn't something that should be taken lightly.
> Certainly not something many people will even *consider* doing.
> Once one's been written, you're also not really choosing to use a linked list.
> You're choosing to use an MPSC queue or whatever.
> The implementation strategy is pretty far removed in this case!

Да!
Хотя создание потокобезопасных структур данных совершенно из другой оперы, и к нему нельзя относиться легкомысленно.
Несомненно, не так уж много людей *разбираются* в теме.
Как только у вас есть готовая структура, вы больше не выбираете связный список.
Вы выбираете MPSC-очередь или что-то похожее.
И стратегия реализации в этом случае будет очень сильно отличаться!

> *But yes, linked lists are the defacto heroes of the dark world of lock-free concurrency.*

*Но — да, связные списки — де факто герои тёмного мира неблокирующей конкуррентности.*

> ## Mumble mumble kernel embedded something something intrusive.

## Тык-пык ядро, встраиваемые системы туда-сюда интрузивный

> It's niche.
> You're talking about a situation where you're not even using your language's *runtime*.
> Is that not a red flag that you're doing something strange?

Это ниши.
Вы говорите о ситуациях, где нельзя использовать даже *стандартную библиотеку* языка.
Разве это не красный флаг, что вы делаете что-то странное?

> It's also wildly unsafe.

Это также крайне небезопасно.

> *But sure. Build your awesome zero-allocation lists on the stack.*

*Но — ладно. Пишите свои потрясающие списки без выделения памяти в куче.*

> ## Iterators don't get invalidated by unrelated insertions/removals

## Итераторы продолжают работать после вставки/удаления элементов до или после итератора

> That's a delicate dance you're playing.
> Especially if you don't have a garbage collector.
> I might argue that your control flow and ownership patterns are probably a bit too tangled, depending on the details.

Вы играете в очень деликатную игру.
Особенно если у вас нет сборщика мусора.
Не вдаваясь в детали, могу предоположить, что ваш код, вероятно, несколько запутан.

> *But yes, you can do some really cool crazy stuff with cursors.*

*Но — да, с помощью курсоров можно делать действительно крутые и безумные штуки.*

> ## They're simple and great for teaching!

## Они простые и хорошо подходят для обучения!

> Well, yeah.
> You're reading a book dedicated to that premise.
> Well, singly-linked lists are pretty simple.
> Doubly-linked lists can get kinda gnarly, as we'll see.

Ну, да.
Вы как раз читаете книгу, основанную на этой предпосылке.
Односвязные списки и правда достаточно просты.
А вот двусвязные списки, как мы увидим, могут быть довольно жуткими.

> # Take a Breath

# Вдохните-выдохните

> Ok.
> That's out of the way.
> Let's write a bajillion linked lists.

Что ж.
С этим разобрались.
Теперь давайте напишем хреналлиард связных списков.

> [On to the first chapter!](first.md)

> [Вперёд, к первой главе!](first.md)

[rust-std-list]: https://doc.rust-lang.org/std/collections/struct.LinkedList.html
[cpp-std-list]: http://en.cppreference.com/w/cpp/container/list
[github]: https://github.com/rust-unofficial/too-many-lists
[bjarne]: https://www.youtube.com/watch?v=YQs6IC-vgmo
[slices]: https://doc.rust-lang.org/std/primitive.slice.html
[split]: https://doc.rust-lang.org/std/primitive.slice.html#method.split_at_mut
[slice-pats]: https://doc.rust-lang.org/edition-guide/rust-2018/slice-patterns.html
[iterators]: https://doc.rust-lang.org/std/iter/trait.Iterator.html
[ghc]: https://wiki.haskell.org/GHC_optimisations#Fusion
[play]: https://play.rust-lang.org/
