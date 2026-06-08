> # Drop and Panic Safety

# Деструктор и устойчивость к панике

> So hey, did you notice this comment:

А вы заметили этот комментарий:

```rust
// Note that we don't need to mess around with `take` anymore
// because everything is Copy and there are no dtors that will
// run if we mess up... right? :) Riiiight? :)))
// Обратите внимание, что мы больше не должны беспокоиться
// о `take`, потому что всё теперь Copy и нет деструкторов,
// которые будут запущены, если мы что-то напутаем... правда? :)
// Праааавда? :)))
```

> Is it right? 

Это правда?

> Sorry did you forget the book you're reading?
> Of course it's wrong! (Sort Of.)

Простите, вы забыли, какую книгу читаете?
Конечно, нет! (В каком-то смысле.)

> Let's look at the inner body of pop_front again:

Давайте снова заглянем внутрь pop_front:

```rust ,ignore
// Bring the Box back to life so we can move out its value and
// Drop it (Box continues to magically understand this for us).
// Возвращаем к жизни Box, так что мы можем извлечь его значение
// и уничтожить его (Box магическим образом делает это за нас).
let boxed_node = Box::from_raw(node.as_ptr());
let result = boxed_node.elem;

// Make the next node into the new front.
// Делаем следующий узел новой головой.
self.front = boxed_node.back;
if let Some(new) = self.front {
    // Cleanup its reference to the removed node
    // Убираем её ссылку на удалённый узел.
    (*new.as_ptr()).front = None;
} else {
    // If the front is now null, then this list is now empty!
    // Если голова теперь равна null, значит, список теперь пустой!
    debug_assert!(self.len == 1);
    self.back = None;
}

self.len -= 1;
result
// Box gets implicitly freed here, knows there is no T.
// Здесь Box неявным образом освобождается, потому что больше нет T.
```

> Do you see the bug?
> Horrifyingly, it's actually this line:

Видите ошибку?
Ужасно, но на самом деле она в этой строчке:

```rust ,ignore
debug_assert!(self.len == 1);
```

> *Really*?
> Our friggin' integrity check for tests is a bug??
> Yes!!!
> Well, if we implement our collection right it *shouldn't* be, but it can turn something benign like "oh we are doing a bad job of keeping len up to date" into *An Exploitable Memory Safety Bug*!
> Why?
> Because it can panic!
> Most of the time you don't have to think or worry about panics, but once you start writing *really* unsafe code and playing fast and loose with "invariants", you need to become hyper-vigilant about panics!

*Серьёзно*?
Наша хренова проверка целостности, сделанная для тестов — ошибка?
Да!!!
Впрочем, если мы реализовали нашу коллекцию корректно, она *не должна* возникнуть, но может случиться так, что наша доброкачественная штука вроде «мы плохо справились с обновлением len» превратится в *Уязвимость, вызванную ошибкой безопасности памяти*!
Почему?
Потому что он может запаниковать!
Большую часть времени вам не надо думать или беспокоиться о панике, но однажды вы начинаете писать *поистине* небезопасный код и вольно обращаетесь с «инвариантностью», и вам приходится проявлять повышенную бдительность из-за паники!

> We've gotta talk about [*exception safety*](https://doc.rust-lang.org/nightly/nomicon/exception-safety.html) (AKA panic safety, AKA unwind safety, ...).

Нам нужно поговорить об [*устойчивости к исключениям*](https://doc.rust-lang.org/nightly/nomicon/exception-safety.html) (ИЛИ устойчивости к панике, ИЛИ устойчивости к раскрутке стека...).

> So here's the deal: by default, panics are *unwinding*.
> Unwinding is just a fancy way to say "make every single function immediately return".
> You might think "well, if *everyone* returns then the program is about to die, so why care about it?", but you'd be wrong!

Итак, дело в том, что по умолчанию паника приводит к *раскрутке* стека (unwinding).
Раскрутка — это просто красивый способ сказать «заставить каждую функцию немедленно завершиться».
Вы можете подумать «Ладно, если *всё* завершается, значит программа тоже завершается, так что зачем об этом беспокоиться?», но вы не правы!

> We have to care for two reasons: destructors run when a function returns, and the unwind can be *caught*.
> In both cases, code can keep running after a panic, so we need to be very careful and make sure our unsafe collections are always in *some* kind of coherent state whenever a panic could occur, because each panic is an implicit early return!

Мы должны беспокоиться по двум причинам: когда функция завершается, запускаются деструктор, и раскрутка может быть *перехвачена*.
В обоих случаях, код может продолжать выполняться после паники, так что нам надо быть очень осторожными и убедиться, что наши небезопасные коллекции всегда находятся в *каком-то* когерентном состоянии независимо от того, возникла ли паника, потому что каждая паника неявно означает ранний возврат (early return)!

> Let's think about what state our collection is in when we get to that line:

Давайте подумаем, в каком состоянии находится наша коллекция, когда мы доходим до этой строки:

> We have our boxed_node on the stack, and we've extracted the element from it.
> If we were to return at this point, the Box would be dropped, and the node would be freed.
> Do you see it now..?
> self.back is still pointing at that freed node!
> Once we implement the rest of our collection and start using self.back for things, this could result in a use-after-free!
> Yikes!

У нас есть наш boxed_node на стеке, и мы извлекли из него элемент.
Если бы вы завершили функцию в этой точке, Box должен был быть уничтожен, и узел должен был быть освобождён.
Теперь вы видите?..
self.back всё ещё указывает на освобождённый узел!
Как только мы реализуем оставшиеся функции и начнём использовать self.back для разных вещей, это может привести к использованию-после-освобождения (use-after-free)!
Кошмар!

> Interestingly, this line has similar problems, but it's much safer:

Интересно, но в этой строке встречается похожая проблема, но гораздо более безопасная:

```rust ,ignore
self.len -= 1;
```

> By default in debug builds Rust checks for underflows and overflows and will panic when they happen.
> Yes, every arithmetic operation is a panic-safety hazard!
> This one is *better* because it happens after we've repaired all of our invariants, so it won't cause memory-safety issues... as long as we don't trust len to be right, but then, if we underflow it's definitely wrong, so we were dead either way!
> The debug assert is in some sense *worse* because it can escalate a minor issue into a critical one!

По умолчанию в отладочных сборках Rust проверяет выход за нижнюю и верхнюю границы, и паникует, когда они происходят.
Да, любая арифметическая операция представляет собой угрозу для устойчивости к панике!
Этот случай *лучше*, потому что он происходит после того, как мы восстановили все наши инварианты, так что он не может привести к проблемам с безопасностью памяти...
<!-- лучше разбить рассуждение на несколько предложений, а то сложно...-->
при условии, что мы не доверяем значению len,
но тогда, если мы вышли за нижнюю границу, ошибка уже была, так что программа упала бы в любом случае!
В каком смысле debug_assert *хуже*, потому что превращает неважную проблему в критическую!

> I've brought up the term "invariants" a few times, and that's because it's a really useful concept for panic-safety!
> Basically, to an outside observer of our collection, there are certain property we're always upholding.
> For a LinkedList, one of those is that any node that is reachable in our list is still allocated and initialized.

Я уже несколько раз употребляла термин «инвариант», а всё потому, что это действительно полезная концепция для разговора про устойчивость к панике!
В каком-то смысле, для стороннего наблюдателя у нашей коллекции есть определённые свойства, которые мы всегда соблюдаем.
Для связного списка одним из таких свойств является следующее: для любого достижимого узла в нашем списке выделена и проинициализирована память.

> *Inside* the implementation we have a bit more flexibility to break invariants *temporarily* as long as we make sure to repair them *before anyone notices*.
> This is actually one of the "killer apps" of Rust's ownership and borrowing system for collections: if an operation requires an `&mut Self`, then we are *guaranteed* that we have exclusive access to our collection and that it's fine for us to temporarily break invariants, safe in the knowledge that no one can sneakily mess with it.

*Внутри* реализации у нас есть немного больше гибкости, чтобы *временно* нарушить инварианты до тех пор, пока мы уверены, что восстановим их *до того, как кто-нибудь заметит*.
По сути, это одно из «главных преимуществ» системы владения и заимствования в Rust для коллекций: если операции нужен параметр `&mut Self`, тогда мы *гарантированно* получаем эксклюзивный доступ к нашей коллекции и мы прекрасно может временно нарушить инварианты, зная, что никто не сможет незаметно в это вмешаться.

> Perhaps the greatest expression of this is [Vec::drain](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.drain), which actually lets you completely smash a core invariant of Vec and start moving values out from the *front* or even *middle* of a Vec.
> The reason this is *sound* is because the Drain iterator that we return holds an `&mut` to the Vec, and so all access is gated behind it!
> No one can observe the Vec until the Drain iterator goes away, and then it's destructor can "repair" the Vec before anyone can notice, it's perfe--

Возможно, наилучшим воплощением этой идеи является [Vec::drain](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.drain), который, фактически, позволяем вам полностью нарушить основной инвариант Vec и начать извлекать значения не только из *начала* Vec, но даже из *середины*.
Причина, по которой это *правильно* заключается в том, что итератор Drain, который мы возвращаем, хранит `&mut ` на Vec, поэтому любой доступ к нему ограничен!
Никто не сможет пользоваться Vec до тех пор, пока итератор Drain не завершится, и тогда деструктор «восстановит» Vec до того, как кто-либо это заметит, что идеа—

> [It's not perfect](https://doc.rust-lang.org/nightly/nomicon/leaking.html#drain).
> Unfortunately, you [can't rely on destructors in code you don't control to run](https://doc.rust-lang.org/std/mem/fn.forget.html), and so even with Drain we need to do a little extra work to make our type always preserved invariants, but in a kind of goofy way: [we just set the Vec's len to 0 at the start](https://doc.rust-lang.org/std/mem/fn.forget.html), so if anyone leaks the Drain, then they will have a *safe* Vec... but they will have also lost a bunch of data.
> You leak me?
> I leak you!
> An eye for an eye!
> True justice!

Нет, [это не идеально](https://doc.rust-lang.org/nightly/nomicon/leaking.html#drain).
К сожалению вы [не можете полагаться на выполнение деструкторов в коде, который вы не контролируете](https://doc.rust-lang.org/std/mem/fn.forget.html), так что даже с Drain нам нужно делать небольшую дополнительную работу, чтобы инварианты нашего типа всегда сохранялись, но в несколько нелепой манере: [в начале мы просто сбрасываем длину Vec в 0](https://doc.rust-lang.org/std/mem/fn.forget.html), так что если у кого-то утечёт Drain, он получит *безопасный* Vec... потеряв при этом все данные.
Ты устроил утечку мне?
Я устрою утечку тебе!
Око за око!
Истинная справедливость!

> For a situation where you *can* actually use destructors for panic-safety, check out the [BinaryHeap::sift_up case study](https://doc.rust-lang.org/nightly/nomicon/exception-safety.html#binaryheapsift_up).

Чтобы узнать, в какой ситуации вы действительно *можете* использовать деструкторы для устойчивости при панике, познакомьтесь с [учебным примером использования BinaryHeap::sift_up](https://doc.rust-lang.org/nightly/nomicon/exception-safety.html#binaryheapsift_up).

> Anyway, we won't be needing all of this fancy stuff for our LinkedLists, we just need to be a bit more vigilant about where we break our invariants, what we trust/require to be correct, and to avoid introducing unnecessary unwinds in the middle of hairy tasks.

Нам в любом случае не потребуется вся эти навороченная ерунда для наших связных списков, нам всего лишь надо быть немного внимательнее по отношению к тому, где мы нарушаем наши инварианты, чему мы верим/чего требуем, чтобы обеспечивать корректность и избежать ненужной раскрутки стека в середине сложных задач.

> In this case, we have two options to make our code a bit more robust:

В этом случае у нас есть две возможности сделать наш код надёжнее:

> * Use operations like Option::take a lot more aggressively, because they are more "transactional" and have a tendency to preserve invariants.
> * Kill the debug_asserts and trust ourselves to write better tests with dedicated "integrity check" functions that won't run in user code ever.

* Более агрессивно использовать такие операции, как Option::take, потому что они более «транзакционные» и имеют тенденцию сохранять инварианты.
* Удалить все debug_assert и доверять себе в написании лучших тестов со специальными функциями «проверки целостности», которые никогда не будут выполняться в пользовательском коде.

> In principle I like the first option, but it doesn't actually work great for a doubly-linked list, because everything is doubly-redundantly encoded.
> Option::take wouldn't fix the problem here, but moving the debug_assert down a line would.
> But really, why make things harder for ourselves?
> Let's just remove those debug_asserts, and make sure anything can panic is at the start or end of our methods, where our invariants should be known to hold.

В принципе мне нравится первый вариант, но на самом деле он не сработает для двусвязного списка, потому что нам одновременно требуются два значения, а не одно.
Option::take не решит нашей проблемы, а вот перенос debug_assert на строку ниже — решит.
Но, правда, зачем усложнять себе жизнь?
Давайте просто удалим все debug_assert и убедимся, что любая паника может возникнуть только в начале или конце наших методов, где, как известно, выполняются все наши инварианты.

> (In this way it's perhaps more accurate to think of them as *preconditions* and *postconditions* but you really should endeavour to treat them as invariants as much as possible!)

(В этом случае, про них, возможно, следует думать, как о *предусловиях* и *постусловиях*, но на самом деле к ним правильнее относиться, как к инвариантам!)

> Here's our full implementation now:

Вот наша полная текущая реализация:

```rust
use std::ptr::NonNull;
use std::marker::PhantomData;

pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<T>,
}

type Link<T> = Option<NonNull<Node<T>>>;

struct Node<T> {
    front: Link<T>,
    back: Link<T>,
    elem: T, 
}

impl<T> LinkedList<T> {
    pub fn new() -> Self {
        Self {
            front: None,
            back: None,
            len: 0,
            _boo: PhantomData,
        }
    }

    pub fn push_front(&mut self, elem: T) {
        // SAFETY: it's a linked-list, what do you want?
        unsafe {
            let new = NonNull::new_unchecked(Box::into_raw(Box::new(Node {
                front: None,
                back: None,
                elem,
            })));
            if let Some(old) = self.front {
                // Put the new front before the old one
                (*old.as_ptr()).front = Some(new);
                (*new.as_ptr()).back = Some(old);
            } else {
                // If there's no front, then we're the empty list and need 
                // to set the back too.
                self.back = Some(new);
            }
            // These things always happen!
            self.front = Some(new);
            self.len += 1;
        }
    }

    pub fn pop_front(&mut self) -> Option<T> {
        unsafe {
            // Only have to do stuff if there is a front node to pop.
            self.front.map(|node| {
                // Bring the Box back to life so we can move out its value and
                // Drop it (Box continues to magically understand this for us).
                let boxed_node = Box::from_raw(node.as_ptr());
                let result = boxed_node.elem;

                // Make the next node into the new front.
                self.front = boxed_node.back;
                if let Some(new) = self.front {
                    // Cleanup its reference to the removed node
                    (*new.as_ptr()).front = None;
                } else {
                    // If the front is now null, then this list is now empty!
                    self.back = None;
                }

                self.len -= 1;
                result
                // Box gets implicitly freed here, knows there is no T.
            })
        }
    }

    pub fn len(&self) -> usize {
        self.len
    }
}
```

> What can panic here?
> Well, knowing that honestly requires you to be a bit of a Rust expert, but thankfully, I am!

Что здесь может вызвать панику?
Честно говоря, чтобы ответить на этот вопрос, надо быть экспертом по Rust, но, к счастью, я им являюсь!

> The only places I can see in this code that *possibly* can panic (barring some absolute fuckery where someone recompiles the stdlib with debug_asserts enabled, but this is not something you should ever do) are `Box::new` (for out-of-memory conditions) and the len arithmetic.
> All of that stuff is at the very end or very start of our methods, so yep, we're being nice and safe!

Единственные места, которые я вижу, где код *может* вызвать панику (за исключением совершенно безумной перекомпиляции стандартной библиотеки с включенными debug_assert, но этого никогда не следует делать) — это `Box::new` (в случае нехватки памяти) и арифметические операции с len.
Всё это находится или в самом конце или в самом начале наших методов, так что да, у нас всё красиво и безопасно!

> ...were you surprised by `Box::new` being able to panic?
> Panics will get you like that!
> Try to preserve those invariants so you don't need to worry about it!

...удивились, что `Bos::new` может вызывать панику?
Паника может достать вас самым непредсказуемым образом!
Постарайтесь держать свои инварианты в целости, чтобы вас это не беспокоило!