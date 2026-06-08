> # A Production-Quality Unsafe Doubly-Linked Deque

# Небезопасный двусвязный дек продуктового уровня

> We finally made it.
> My greatests nemesis: **[std::collections::LinkedList][linked-list], the Doubly-Linked Deque**. 

Мы, наконец, добрались.
Мой величайший враг: **[std::collections::LinkedList][linked-list], двусвязный дек**.

> The one that I tried and failed to destroy.

То, что я безуспешно пыталась уничтожить.

> Our story begins as 2014 was coming to a close and we were rapidly approaching the release of Rust 1.0, Rust's first stable release.
> I had found myself in the role of caring for `std::collections`, or as we affectionately called it in those times, libcollections.

Наша история начинается в 2014 году, когда мы стремительно приближались к выпуску Rust 1.0, первой стабильной версии Rust.
Я была ответственна за `std::collections` или, как мы ласково называли тогда эту библиотеку, libcollections.

> libcollections had spent years as a dumping ground for everyone's Cute Ideas and Vaguely Useful Things.
> This was all well and good when Rust was a fledgling experimental language, but if my children were going to escape the nest and be stabilized, they would have to prove their worth.

Библиотека годами служила свалкой для интересных идей и мало-мальски полезных вещей.
Это было здорово, пока Rust оставался экспериментальным языком, но когда мои дети собираются вырваться из гнезда и обрести свободу, им приходится доказывать свою состоятельность.

> Until then I had encouraged and nurtured them all, but it was now time for them to face judgement for their failings.

До той поры я поддерживала их все, но пришло время для них предстать перед судом за их прегрешения.

> I sunk my claws into the bedrock and carved tombstones for my most foolish children.
> A grisly monument that I placed in the town square for all to see:

Я вонзила когти в твердь и вырезала надгробные плиты для самых неразумных моих детей.
Жуткий памятник, который я установила на городской площади, чтобы все могли его видеть:

> **[Kill TreeMap, TreeSet, TrieMap, TrieSet, LruCache and EnumSet](https://github.com/rust-lang/rust/pull/19955)**


**[Удалить TreeMap, TreeSet, TrieMap, TrieSet, LruCache и EnumSet](https://github.com/rust-lang/rust/pull/19955)**

> Their fates were sealed, for my word was absolute.
> The other collections were horrified by my brutality, but they were not yet safe from their mother's wrath.
> I soon returned with two more tombstones:

Их судьбы были предрешены, ибо моё слово было нерушимо.
Другие коллекции были в ужасе от моей жестокости, но они ещё не были в безопасности от гнева своей матери.
Вскоре я вернулась ещё с двумя надгробиями:

> **[Deprecate BitSet and BitVec](https://github.com/rust-lang/rust/pull/26034)**

**[Объявить устаревшими BitSet и BitVec](https://github.com/rust-lang/rust/pull/26034)**

> The Bit twins were more cunning than their fallen comrades, but they lacked the strength to escape me.
> Most thought my work done, but I soon took one more: 

Битовые близнецы оказались хитрее своих павших товарищей, но и им не хватило сил сбежать от меня.
Большинство думало, что моя работа выполнена, но вскоре я поймала ещё одного:

> **[Deprecate VecMap](https://github.com/rust-lang/rust/pull/26734)**

**[Объявить устаревшим VecMap](https://github.com/rust-lang/rust/pull/26734)**

> VecMap had tried to survive through stealth &mdash; it was so small and inoffensive!
> But that wasn't enough for the libcollections I saw in my vision of the future.

VecMap пытался выжить, полагаясь на свою незаметность — он был таким маленьким и безобидным!
Но этого оказалось недостаточно для libcollections, которую я себе представляла в будущем.

> I surveyed the land and saw what remained:

Я оглянулась и увидела тех, кто остались:

> * Vec and VecDeque - hearty and simple, the heart of computing.
> * HashMap and HashSet - powerful and wise, the brain of computing.
> * BTreeMap and BTreeSet - awkward but necessary, the liver of computing.
> * BinaryHeap - crafty and dextrous, the ankle of computing.

* Vec и VecDeque — простые и надёжные, сердце вычислений.
* HashMap и HashSet — мощные и требующие знаний, мозг вычислений.
* BTreeMap и BTreeSet — неуклюжие, но нужные, печень вычислений.
* BinaryHeap — хитрая и ловкая, лодыжка вычислений.

> I nodded in contentment.
> Simple and effective.
> My work was don&mdash;

Я удовлетворённо кивнула.
Просто и эффективно.
Моя работа была сделана.

> No, [DList](https://github.com/rust-lang/rust/blob/0a84308ebaaafb8fd89b2fd7c235198e3ec21384/src/libcollections/dlist.rs), it can't be!
> I thought you died in that tragic garbage collection incident!
> The one which was definitely an accident and not intentional at all!

Нет, [DList](https://github.com/rust-lang/rust/blob/0a84308ebaaafb8fd89b2fd7c235198e3ec21384/src/libcollections/dlist.rs), этого не может быть!
Я думала, ты погиб в том трагическом инциденте со сборкой мусора!
Единственный, кто определённо появился в результате случайности, а не какого-то намерения!

> They had faked their death and taken on a new name, but it was still them: LinkedList, the shadowy and untrustworthy schemer of computing. 

Он инсценировал свою смерть и взял себе новое имя, но это всё ещё были он: связный список, тёмный и ненадёжный интриган вычислений.

> I spread word of their misdeeds to all that would hear me, but hearts were unmoved.
> LinkedList was a silver-tongued devil who had convinced everyone around me that it was some sort of fundamental and natural datastructure of computing.
> It had even convinced C++ that it was [*the* list](https://en.cppreference.com/w/cpp/container/list)!

Я рассказывала о его злодеяниях всем, кто готов был слушать, но их сердца оставались непоколебимыми.
Связный список оказался красноречивым дьяволом, который убедил всех вокруг, что является фундаментальной и естественной структурой данных в вычислениях.
Он даже убедил C++, что бы [*тем самым* списком](https://en.cppreference.com/w/cpp/container/list)!

> "How could you have a standard library without a *LinkedList*?"

«Как стандартная библиотека может быть без *связного списка*?»

> Easily!
> Trivially!

Легко!
Просто!

> "It's non-trivial unsafe code, so it makes sense to have it in the standard library!"

«Это нетривиальный небезопасный код, поэтому вполне логично включить его в стандартную библиотеку!»

> So are GPU drivers and video codecs, libcollections is minimalist!

То же самое можно сказать про драйверы GPU и видео кодеки.
Библиотека libcollections должна быть минималистична!

> But alas, LinkedList had gathered too many allies and grown too strong while I was distracted with its kin.

Но, увы, связный список обзавёлся слишком большим числом сторонников и стал слишком сильным, пока я занималась его братьями.

> I fled to my laboratory and tried to devise some sort of [evil clone](https://github.com/contain-rs/linked-list) or [enhanced cyborg replicant](https://github.com/contain-rs/blist) that could rival and destroy it, but my grant funding ran out because my research was "too murderously evil" or somesuch nonsense.

Я сбежала в свою лабораторию и попыталась создать некоего [злого клона](https://github.com/contain-rs/linked-list) или [продвинутого кибернетического репликанта](https://github.com/contain-rs/blist), который мог бы соперничать с ним и уничтожить его, но мой грант приостановили, потому что мои исследования были признаны «очевидно кровожадными» или что-то в этом роде.

> LinkedList had won.
> I was defeated and forced into exile.

Связный список победил.
Я потерпела поражение и отправилась в изгнание.

> But you're here now.
> You've come this far.
> Surely now you can understand the depths of LinkedList's debauchery!
> Come, I will you show you everything you need to know to help me destroy it once and for all &mdash; everything you need to know to implement an unsafe production-quality Doubly-Linked Deque.

Но сейчас вы здесь.
Вы проделали весь этот путь.
И, конечно, сейчас вы понимаете всю глубину распущенности связного списка!
Пойдёмте, я покажу вам всё, что нужно знать, чтобы помочь мне уничтожить его раз и навсегда, всё, что нужно знать, чтобы реализовать небезопасный двусвязный дек продуктового уровня.

> How production-quality?
> Well we're going to completely rewrite my ancient Rust 1.0 linked-list crate, the one that is objectively better than the one in std.
> The one with Cursors on stable Rust, from 2015!
> Something the 2022 stdlib still doesn't have!

Действительно продуктового уровня?
Ну, я собираюсь полностью переписать мой старую реализацию связного списка из Rust 1.0, ту самую, которая объективно лучше реализации из стандартной библиотеки.
Ту, в которой были Курсоры, из стабильной версии Rust 2015 года!
Ту, которой всё ещё нет в стандартной библиотеке 2022 года!

[linked-list]: https://github.com/rust-lang/rust/blob/master/library/alloc/src/collections/linked_list.rs
