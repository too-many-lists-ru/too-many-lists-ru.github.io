> # Testing Stacked Borrows

# Тестирование стековых заимствований

> > TL;DR of the previous section's (simplified) memory model for Rust:
> >
> > * Rust conceptually handles reborrows by maintaining a "borrow stack"
> > * Only the one on the top of the stack is "live" (has exclusive access)
> > * When you access a lower one it becomes "live" and the ones above it get popped
> > * You're not allowed to use pointers that have been popped from the borrow stack
> > * The borrowchecker ensures safe code code obeys this
> > * Miri theoretically checks that raw pointers obey this at runtime

> Подытожим, что мы (упрощённо) знаем о подели памяти Rust, опираясь на предыдущие разделы:> 
>
> * Концептуально, в Rust обработка повторных заимствований осуществляется путём «стека заимствований»
> * Единственный указатель на вершине стека считается «живым» (имеет эксклюзивный доступ)
> * При доступе через указатель в глубине стека, он становится «живым», а указатели выше него удаляются из стека
> * Вы не можете использовать указатели, которые были удалены из стека заимствований
> * Анализатор зависимостей гарантирует, что безопасный код соответствуем этим требованиям
> * Miri в теории проверяет, что сырые указатели соответствуют этим правилам во время выполнения

> That was a lot of theory and ideas -- let's move on to the true heart and soul of this book: riting some bad code and getting our tools to scream at us.
> We're going to go through a *ton* of examples to try to see if our mental model makes sense, and to try to get an intuitive feel for stacked borrows.

До этого было много теории и идей — теперь давайте перейдём к истинному сердцу и душе этой книги: написанию плохого кода и тому, как заставить наши инструменты ругаться на этот код.
Мы собираемся пройти через *груды* примеров, чтобы попытаться понять, работает ли наша ментальная модель, и попытаться прочувствовать стековые заимствования на интуитивном уровне.

> > **NARRATOR:** Catching Undefined Behaviour in practice is a hairy business.
> > After all, you're dealing with situations that the compiler literally *assumes* don't happen.
> >
> > If you're lucky, things will "seem to work" today, but they'll be a ticking time bomb for a Smarter Compiler or slight change to the code.
> > If you're *really* lucky things will reliably crash so you can just catch the mistake and fix it.
> > But if you're unlucky, then things will be broken in weird and baffling ways.
> >
> > Miri tries to work around this by getting rustc's most naive and unoptimized view of the program and tracking extra state as it interprets.
> > As far as "sanitizers" go, this is a fairly deterministic and robust approach but it will never be *perfect*.
> > You need your test program to actually have an execution with that UB, and for a large enough program it's very easy to introduce all sorts of non-determinism (HashMaps use RNG by default!).
> >
> > We can never take miri approving of our program's execution as an absolute certain statement there's no UB.
> > It's also possible for miri to *think* something's UB when it really isn't.
> > But if we have a mental model of how things work, and miri seems to agree with us, that's a good sign that we're on the right track.

> **ГОЛОС ЗА КАДРОМ:** обнаружение Неопределённого Поведения на практике — хлопотное дело.
> Помимо прочего, вы имеете дело с ситуациями, которые компилятор буквально *полагает* невозможными.
> 
> Если вы счастливчик, программа будет «казаться работающей» сегодня, но она будет содержать бомбу замедленного действия для Более Умного Компилятора или небольшого изменения в коде.
> Если вы *настоящий* счастливчик, программа гарантированно не будет работать, так что вы сможете найти ошибку и исправить её.
> Но если вы не счастливчик, программа будет ломаться странными и непонятными образами.
> 
> Miri пытается работать обойти эту проблему, получая от rustc самое простое и не оптимизированное представление программы и следя за дополнительным состоянием при её интерпретации.
> Что касается «средств динамического анализа» (sanitizers), то это довольно детерминированный и надёжный подход, он он никогда не будет *совершенным*.
> Ваша тестовая программа, фактически, должна дойти до точки с UB, хотя во многих программах пышным цветом цветёт разного рода недетерминированное выполнение (скажем, HashMap по умолчанию использует датчик случайных чисел!).
> 
> Ни при каких обстоятельствах мы не должны считать, что одобрение miri нашей программы означает полное отсутствие в ней UB.
> В то же время, бывает, что miri *думает*, что нашла UB, но на самом деле это не так.
> Но если у нас есть ментальная модель того, как работают указатели, и miri с нами согласна, это хороший знак, что мы на правильном пути.

> # Basic Borrows

# Базовые заимствования

> In previous sections we saw that the borrowchecker didn't like this code:

В предыдущих разделах мы видели, что анализатору заимствований (borrowchecker) не нравился такой код:

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

> Let's see what happens when we replace `ref2` with `*mut`:

Давайте посмотрим, что случится, если мы заменим `ref2` на `*mut`:

```rust ,ignore
unsafe {
    let mut data = 10;
    let ref1 = &mut data;
    let ptr2 = ref1 as *mut _;

    // ORDER SWAPPED!
    // ПОРЯДОК ИЗМЕНИЛСЯ!
    *ref1 += 1;
    *ptr2 += 2;

    println!("{}", data);
}
```

```text
cargo run
   Compiling miri-sandbox v0.1.0
    Finished dev [unoptimized + debuginfo] target(s) in 0.71s
     Running `target\debug\miri-sandbox.exe`
13
```

> Rustc seems perfectly happy with this: no warnings and the program produced the result we expected!
> Now let's look at what miri (in strict mode) thinks of it: 

Кажется, rustc всё устраивает: никаких предупреждений и программа выдала ожидаемый нами результат!
Теперь давайте взглянем, что об этом думает miri (в строгом режиме):

```text
MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

    Finished dev [unoptimized + debuginfo] target(s) in 0.00s
     Running cargo-miri.exe target\miri

error: Undefined Behavior: no item granting read access 
to tag <untagged> at alloc748 found in borrow stack.

 --> src\main.rs:9:9
  |
9 |         *ptr2 += 2;
  |         ^^^^^^^^^^ no item granting read access to tag <untagged> 
  |                    at alloc748 found in borrow stack.
  |
  = help: this indicates a potential bug in the program: 
    it performed an invalid operation, but the rules it 
    violated are still experimental
 
```

> Nice!
> Our intuitive model of how things work held up: although the compiler couldn't catch the issue for us, miri did.

Отлично!
Наша интуитивная модель подтвердилась: хотя компилятор не смог обнаружить проблему, miri смогла.

> Let's try something more complicated, the `&mut -> *mut -> &mut -> *mut` case we alluded to before:

Давайте попробуем что-то более сложное, преобразование `&mut -> *mut -> &mut -> *mut`, о котором мы упоминали ранее:

```rust ,ignore
unsafe {
    let mut data = 10;
    let ref1 = &mut data;
    let ptr2 = ref1 as *mut _;
    let ref3 = &mut *ptr2;
    let ptr4 = ref3 as *mut _;

    // Access the first raw pointer first
    // Сначала обращаемся к первому сырому указателю
    *ptr2 += 2;

    // Then access things in "borrow stack" order
    // Затем обращаемся к переменным в порядке «стека заимствований»
    *ptr4 += 4;
    *ref3 += 3;
    *ptr2 += 2;
    *ref1 += 1;

    println!("{}", data);
}
```

```text
cargo run
22

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

error: Undefined Behavior: no item granting read access 
to tag <1621> at alloc748 found in borrow stack.

  --> src\main.rs:13:5
   |
13 |     *ptr4 += 4;
   |     ^^^^^^^^^^ no item granting read access to tag <1621> 
   |                at alloc748 found in borrow stack.
   |
```

> Wow yep!
> In strict mode miri can "tell apart" the two raw pointers and have using the second one invalidate the first one.
> Let's see if everything works when we remove the first use that messes everything up:

Да, да!
В строгом режиме miri смогла «различить» два сырых указателя, обнаружив, что использование второго ведёт к порче первого.
Давайте посмотрим, будет ли всё работать, когда мы удалим первое использование, которое всё ломает:

```rust ,ignore
unsafe {
    let mut data = 10;
    let ref1 = &mut data;
    let ptr2 = ref1 as *mut _;
    let ref3 = &mut *ptr2;
    let ptr4 = ref3 as *mut _;

    // Access things in "borrow stack" order
    // Обращаемся к переменным в порядке «стека заимствований»
    *ptr4 += 4;
    *ref3 += 3;
    *ptr2 += 2;
    *ref1 += 1;

    println!("{}", data);
}
```

```text
cargo run
20

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
20
```

> NICE.

ОТЛИЧНО.

> Yeah I'm pretty sure at this point we can all get PhD's in programming language memory model design and implementation.
> Who even *needs* compilers, this stuff is *easy*.

Я почти уверена, что к настоящему моменту все мы можем получить докторскую степень по проектированию и реализации моделей памяти в языках программирования.
Кому *вообще* нужны компиляторы, тут же всё *просто*.

> > **NARRATOR:** it was not, but I'm proud of you nonetheless.

> **ГОЛОС ЗА КАДРОМ:** это не так, но я всё равно тобой горжусь.


> # Testing Arrays

# Тестирование массивов

> Let's mess with some arrays and pointer offsets (`add` and `sub`).
> This should work, right?

Давайте поэкспериментируем с массивами и смещениями указателей (сложением и вычитанием).
Это должно работать, правильно?

```rust ,ignore
unsafe {
    let mut data = [0; 10];
    let ref1_at_0 = &mut data[0];           // Reference to 0th element
    let ptr2_at_0 = ref1_at_0 as *mut i32;  // Ptr to 0th element
    let ptr3_at_1 = ptr2_at_0.add(1);       // Ptr to 1st element
                                            // Ссылка на 0-й элемент
                                            // Указатель на 0-й элемент
                                            // Указатель на 1-й элемент

    *ptr3_at_1 += 3;
    *ptr2_at_0 += 2;
    *ref1_at_0 += 1;

    // Should be [3, 3, 0, ...]
    // Должно быть [3, 3, 0, ...]
    println!("{:?}", &data[..]);
}
```

```text
cargo run
[3, 3, 0, 0, 0, 0, 0, 0, 0, 0]

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

error: Undefined Behavior: no item granting read access 
to tag <1619> at alloc748+0x4 found in borrow stack.
 --> src\main.rs:8:5
  |
8 |     *ptr3_at_1 += 3;
  |     ^^^^^^^^^^^^^^^ no item granting read access to tag <1619>
  |                     at alloc748+0x4 found in borrow stack.
```

> *Rips up gradschool application*

*Рвёт заявление в докторантуру*

> What happened?
> We're using the borrow stack perfectly fine!
> Does something weird happen when we go `ptr -> ptr`?
> What if we just copy the pointer so they all go to the same location:

Что случилось?
Мы совершенно прекрасно используем стек заимствований!
Происходит ли что-то странное, когда мы преобразуем `ptr -> ptr`?
Что если мы просто скопируем указатель, чтобы оба указателя указывали на одно и то же место:

```rust
unsafe {
    let mut data = [0; 10];
    let ref1_at_0 = &mut data[0];           // Reference to 0th element
    let ptr2_at_0 = ref1_at_0 as *mut i32;  // Ptr to 0th element
    let ptr3_at_0 = ptr2_at_0;              // Ptr to 0th element
                                            // Ссылка на 0-й элемент
                                            // Указатель на 0-й элемент
                                            // Указатель на 0-й элемент

    *ptr3_at_0 += 3;
    *ptr2_at_0 += 2;
    *ref1_at_0 += 1;

    // Should be [6, 0, 0, ...]
    // Должно быть [6, 0, 0, ...]
    println!("{:?}", &data[..]);
}
```

```text
cargo run
[6, 0, 0, 0, 0, 0, 0, 0, 0, 0]

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
[6, 0, 0, 0, 0, 0, 0, 0, 0, 0]
```

> Nope, that works fine.
> Maybe we're getting lucky, let's just make a real big mess of pointers:

Нет, это прекрасно работает.
Может быть, нам просто повезло, давайте просто перетасуем указатели как следует:

```rust
unsafe {
    let mut data = [0; 10];
    let ref1_at_0 = &mut data[0];            // Reference to 0th element
    let ptr2_at_0 = ref1_at_0 as *mut i32;   // Ptr to 0th element
    let ptr3_at_0 = ptr2_at_0;               // Ptr to 0th element
    let ptr4_at_0 = ptr2_at_0.add(0);        // Ptr to 0th element
    let ptr5_at_0 = ptr3_at_0.add(1).sub(1); // Ptr to 0th element
                                            // Ссылка на 0-й элемент
                                            // Указатель на 0-й элемент
                                            // Указатель на 0-й элемент
                                            // Указатель на 0-й элемент
                                            // Указатель на 0-й элемент

    // An absolute jumbled hash of ptr usages
    // Абсолютно беспорядочная мешанина использования указателей
    *ptr3_at_0 += 3;
    *ptr2_at_0 += 2;
    *ptr4_at_0 += 4;
    *ptr5_at_0 += 5;
    *ptr3_at_0 += 3;
    *ptr2_at_0 += 2;
    *ref1_at_0 += 1;

    // Should be [20, 0, 0, ...]
    // Должно быть [20, 0, 0, ...]
    println!("{:?}", &data[..]);
}
```


```text
cargo run
[20, 0, 0, 0, 0, 0, 0, 0, 0, 0]

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
[20, 0, 0, 0, 0, 0, 0, 0, 0, 0]
```

> Nope!
> Miri is actually *way* more permissive when it comes to raw pointers that are derived from other raw pointers.
> They all share the same "borrow" (or miri calls it, a *tag*).

Нет! На самом деле, miri *действует* более снисходительно, когда речь заходит о прямых указателях, производных от других сырых указателей.
Все они разделяют одно и то же «заимствование» (или, как miri его называет, *маркер*).

> Once you start using raw pointers they can freely split into their own tiny angry men and mess with themselves.
> This is ok because the compiler understands that and won't optimize the reads and writes the same it does with references.

Как только вы начинаете использовать сырые указатели, они провоцируют появление сердитых человечков, которые начинают мешать.
Но это нормально, потому что компилятор понимает это, и не оптимизирует чтение и запись также, как делает это со ссылками. 

> > **NARRATOR:** If the code is simple enough, the compiler can keep track of all the derived pointers and still optimize things where possible, but it's going to be a lot more brittle than the reasoning it can use for references.

> **ГОЛОС ЗА КАДРОМ:** Если код достаточно простой, компилятор может продолжать отслеживать все производные указатели и продолжать оптимизацию, когда возможно, но всё это будет более хрупким, чем рассуждения, которые можно может использовать для ссылок.

> So what's the *real* problem?

Так в чём же *реальная* проблема?

> Even though `data` is one "allocation" (local variable), `ref1_at_0` is only borrowing the first element.
> Rust allows borrows to be broken up so that they only apply to particular parts of the allocation!
> Let's try it out:

Даже хотя у `data` одно «выделение» (локальная переменная), `ref1_at_0` заимствует только первый элемент.
Rust позволяет разбивать заимствования на части, чтобы они применялись к разным частям выделенной памяти!
Давайте попробуем:

```rust ,ignore
unsafe {
    let mut data = [0; 10];
    let ref1_at_0 = &mut data[0];           // Reference to 0th element
    let ref2_at_1 = &mut data[1];           // Reference to 1th element
    let ptr3_at_0 = ref1_at_0 as *mut i32;  // Ptr to 0th element
    let ptr4_at_1 = ref2_at_1 as *mut i32;   // Ptr to 1th element
                                            // Ссылка на 0-й элемент
                                            // Ссылка на 1-й элемент
                                            // Указатель на 0-й элемент
                                            // Указатель на 1-й элемент

    *ptr4_at_1 += 4;
    *ptr3_at_0 += 3;
    *ref2_at_1 += 2;
    *ref1_at_0 += 1;

    // Should be [3, 3, 0, ...]
    // Должно быть [3, 3, 0, ...]
    println!("{:?}", &data[..]);
}
```

```text
error[E0499]: cannot borrow `data[_]` as mutable more than once at a time
 --> src\main.rs:5:21
  |
4 |     let ref1_at_0 = &mut data[0];           // Reference to 0th element
  |                     ------------ first mutable borrow occurs here
5 |     let ref2_at_1 = &mut data[1];           // Reference to 1th element
  |                     ^^^^^^^^^^^^ second mutable borrow occurs here
6 |     let ptr3_at_0 = ref1_at_0 as *mut i32;  // Ptr to 0th element
  |                     --------- first borrow later used here
  |
  = help: consider using `.split_at_mut(position)` or similar method 
    to obtain two mutable non-overlapping sub-slices
```

> Shoot!
> Rust doesn't track array indices to prove these borrows are disjoint, but it does give us `split_at_mut` to break a slice into multiple parts in a way that is safe to assume works:

Блин!
Rust не отслеживает индексы массива, чтобы доказать, что эти заимствования не пересекаются, но он предоставляет нам `split_at_mut` чтобы разбить срез на несколько частей, способом, который, можно с уверенность предположить, работает:

```rust
unsafe {
    let mut data = [0; 10];

    let slice1 = &mut data[..];
    let (slice2_at_0, slice3_at_1) = slice1.split_at_mut(1); 
    
    let ref4_at_0 = &mut slice2_at_0[0];    // Reference to 0th element
    let ref5_at_1 = &mut slice3_at_1[0];    // Reference to 1th element
    let ptr6_at_0 = ref4_at_0 as *mut i32;  // Ptr to 0th element
    let ptr7_at_1 = ref5_at_1 as *mut i32;  // Ptr to 1th element
                                            // Ссылка на 0-й элемент
                                            // Ссылка на 1-й элемент
                                            // Указатель на 0-й элемент
                                            // Указатель на 1-й элемент

    *ptr7_at_1 += 7;
    *ptr6_at_0 += 6;
    *ref5_at_1 += 5;
    *ref4_at_0 += 4;

    // Should be [10, 12, 0, ...]
    // Должно быть [10, 12, 0, ...]
    println!("{:?}", &data[..]);
}
```

```text
cargo run
[10, 12, 0, 0, 0, 0, 0, 0, 0, 0]

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
[10, 12, 0, 0, 0, 0, 0, 0, 0, 0]
```

> Hey, that works!
> Slices properly tell the compiler and miri "hey I'm taking a huge loan on all of the memory in my range", so they know all of the elements can be mutated.

Да, так работает!
Срезы сообщают компилятору и miri: «Эй, я владею большим куском памяти в этом диапазоне», поэтому они знают, что все элементы могут быть изменены.

> Also note that operations like `split_at_mut` being allowed tells us that borrows can be less of a *stack* and more of a *tree*, because we can break one big borrow into a bunch of disjoint smaller ones, and everything still works.

Также имейте в виду, что операции наподобие `split_at_mut` позволяют нам говорить, что заимствования организуются не в виде *стека*, а в виде *дерева*, потому что мы можем разбить одно большой заимствование на несколько непересекающихся маленьких, и всё будет работать.

> (I think in the actual stacked borrows model everything's still stacks because the stacks are conceptually tracking permissions for each byte of the program..?)

(Я думаю, что в настоящей стековой модели заимствований всё продолжает оставаться стеками, потому что стеки концептуально отслеживают права доступа к каждому байт программы?..)

<!-- под правами имеются в виду чтение, запись, эксклюзивный доступ -->

> What if we *directly* turn a slice into a pointer?
> Will that pointer have access to the full slice?

А что, если мы *напрямую* преобразуем срез в указатель?
Будет ли этот указатель иметь доступ к полному срезу?

```rust
unsafe {
    let mut data = [0; 10];

    let slice1_all = &mut data[..];         // Slice for the entire array
    let ptr2_all = slice1_all.as_mut_ptr(); // Pointer for the entire array
                                            // Срез всего массива
                                            // Указатель на весь массив
    
    let ptr3_at_0 = ptr2_all;               // Pointer to 0th elem (the same)
    let ptr4_at_1 = ptr2_all.add(1);        // Pointer to 1th elem
    let ref5_at_0 = &mut *ptr3_at_0;        // Reference to 0th elem
    let ref6_at_1 = &mut *ptr4_at_1;        // Reference to 1th elem
                                            // Указатель на 0-й элемент (та же)
                                            // Указатель на 1-й элемент
                                            // Ссылка на 0-й элемент
                                            // Ссылка на 1-й элемент

    *ref6_at_1 += 6;
    *ref5_at_0 += 5;
    *ptr4_at_1 += 4;
    *ptr3_at_0 += 3;

    // Just for fun, modify all the elements in a loop
    // (Could use any of the raw pointers for this, they share a borrow!)
    // Просто для смеха, поменяем все элементы в цикле
    // (Для этого можем использовать любой из сырых указателей, они разделяют заимствование!)
    for idx in 0..10 {
        *ptr2_all.add(idx) += idx;
    }

    // Safe version of this same code for fun
    // Безопасная версия того же самого кода, для развлечения
    for (idx, elem_ref) in slice1_all.iter_mut().enumerate() {
        *elem_ref += idx; 
    }

    // Should be [8, 12, 4, 6, 8, 10, 12, 14, 16, 18]
    // Должно быть [8, 12, 4, 6, 8, 10, 12, 14, 16, 18]
    println!("{:?}", &data[..]);
}
```


```text
cargo run
[8, 12, 4, 6, 8, 10, 12, 14, 16, 18]

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
[8, 12, 4, 6, 8, 10, 12, 14, 16, 18]
```


> Nice!
> Pointers aren't just integers: they have a range of memory associated with them, and with Rust we're allowed to narrow that range!

Прекрасно!
Указатели — это не просто целые числа: у них есть область памяти, ассоциированная с ними, и с помощью Rust мы можем сузить эту область!

> # Testing Shared References

# Тестирование разделяемых ссылок

> In all of these examples I have been very carefully only using mutable references and doing read-modify-write operations (`+=`) to keep things as simple as possible.

Во всех этих примерах я очень осторожно использовала только изменяемые ссылки и делала операции чтения-изменения-записи (`+=`), чтобы оставлять вещи настолько простыми, насколько это возможно.

> But Rust has shared references that are read-only and can be freely copied, how should those work?
> Well we've seen that raw pointers can be freely copied and we can handle that by saying they "share" a single borrow.
> Maybe we think of shared references the same way?

Но в Rust есть разделяемые ссылки, которые предназначены только для чтения и могут быть свободно скопированы, как должны работать они?
Ну, мы видели, что сырые указатели могут быть свободно скопированы и мы можем обработать это, сказал, что они «разделяют» единственное заимствование.
Можно ли таким же образом думать о разделяемых ссылках?

> Let's test that out with a function that reads a value (`println!` can be a little magical with auto-ref/deref stuff, so I'm wrapping it in a function to make sure we're testing what we want to be):

Давайте проверим это с помощью функции, которая читает значение (макрос `println! может быть немного волшебным, если речь идёт о таких вещах, как автоматическое получение и разыменование ссылок, так что я завернула его в функцию, чтобы быть уверенной, что мы тестируем то, что нужно):

```rust ,ignore
fn opaque_read(val: &i32) {
    println!("{}", val);
}

unsafe {
    let mut data = 10;
    let mref1 = &mut data;
    let sref2 = &mref1;
    let sref3 = sref2;
    let sref4 = &*sref2;

    // Random hash of shared reference reads
    // Случайная мешанина чтений разделяемых ссылок
    opaque_read(sref3);
    opaque_read(sref2);
    opaque_read(sref4);
    opaque_read(sref2);
    opaque_read(sref3);

    *mref1 += 1;

    opaque_read(&data);
}
```

```text
cargo run

warning: unnecessary `unsafe` block
 --> src\main.rs:6:1
  |
6 | unsafe {
  | ^^^^^^ unnecessary `unsafe` block
  |
  = note: `#[warn(unused_unsafe)]` on by default

warning: `miri-sandbox` (bin "miri-sandbox") generated 1 warning

10
10
10
10
10
11
```

> Oh yeah we forgot to do anything with raw pointers, but at least we can see that it's fine for all the shared references to be used interchangeably.
> Now let's mix in some raw pointers:

Ох, да, мы забыли что-нибудь сделать с сырыми указателями, но по крайней мере мы можем видеть, что взаимозаменяемо использовать разделяемые указатели вполне нормально.
Теперь давайте добавим несколько сырых указателей:

```rust ,ignore
fn opaque_read(val: &i32) {
    println!("{}", val);
}

unsafe {
    let mut data = 10;
    let mref1 = &mut data;
    let ptr2 = mref1 as *mut i32;
    let sref3 = &mref1;
    let ptr4 = sref3 as *mut i32;

    *ptr4 += 4;
    opaque_read(sref3);
    *ptr2 += 2;
    *mref1 += 1;

    opaque_read(&data);
}
```

```text
cargo run

error[E0606]: casting `&&mut i32` as `*mut i32` is invalid
  --> src\main.rs:11:16
   |
11 |     let ptr4 = sref3 as *mut i32;
   |                ^^^^^^^^^^^^^^^^^
```

> Oh whoops, we were actually messing around with `& &mut` instead of `&`!
> Rust is very good at papering over that when it doesn't matter.
> Let's properly reborrow it with `let sref3 = &*mref1`:


Ой, извините, мы на самом деле работали с `& &mut` вместо `&`!
Rust очень хорошо умеет скрывать такие вещи, когда они не имеют значения.
Давайте сделаем правильное повторное заимствование: `let sref3 = &*mref1`:

```text
cargo run

error[E0606]: casting `&i32` as `*mut i32` is invalid
  --> src\main.rs:11:16
   |
11 |     let ptr4 = sref3 as *mut i32;
   |                ^^^^^^^^^^^^^^^^^
```

> Nope, Rust still doesn't like that!
> You can only cast a shared reference to a `*const` which can only read.
> But what if we just... do... this...?

Нет, Rust всё ещё не в восторге от этого!
Вы можете привести разделяемую ссылку только к `*const`, который может быть только прочитан.
Но что, если мы просто... сделаем... так...?

```rust ,ignore
    let ptr4 = sref3 as *const i32 as *mut i32;
```

```text
cargo run

14
17
```

> WHAT.
> OK SURE FINE?
> Great cast system there Rust.
> It's almost like the `*const` is a pretty useless type that only really exists to describe C APIs and to vaguely suggest correct usage (it is, it does).
> What does miri think?

ЧТО?
ПРОСТО РАБОТАЕТ?
Отличная система приведения типов, Rust.
Выглядит так, будто `*const` — практически бесполезный тип, который на самом деле нужен только для описания C API и для того, чтобы подсказывать правильные сценарии использования (на самом деле, так и есть).
А что по этому поводу думает miri?

```text
MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

error: Undefined Behavior: no item granting write access to 
tag <1621> at alloc742 found in borrow stack.
  --> src\main.rs:13:5
   |
13 |     *ptr4 += 4;
   |     ^^^^^^^^^^ no item granting write access to tag <1621>
   |                at alloc742 found in borrow stack.
```

> Alas, though we can get around the compiler complaining with a double cast, it doesn't actually make this operation *allowed*.
> When we take the shared reference, we're promising not to modify the value. 

К сожалению, хотя мы можем избавиться от жалоб компилятора с помощью двойного приведения, это не делает операцию *разрешённой".
Когда мы получаем разделяемую ссылку, мы обещаем не модифицировать значение.

> This is important because that means when the shared borrow is popped off the borrow stack, the mutable pointers below it *can* assume the memory hasn't changed.
> There may have been some tiny angry men *reading* the memory (so writes had to be comitted) but they weren't able to modify it and the mutable pointers can assume the last value they wrote is still there!

Это важно, потому что это значит, что когда мы удаляем разделяемое заимствование из стека заимствований, изменяемые указатели перед ним *могут* быть уверены в том, что память не изменилась.
Может быть, какие-то сердитые человечки и читали память (потому что прав на запись у них нет было), но у них не было возможности её изменить и изменяемые указатели могут предполагать, что последнее значение, которое они записали, всё ещё там!

> **Once a shared reference is on the borrow-stack, everything that gets pushed on top of it only has read permissions.**

**Как только разделяемая ссылка оказывается в стеке заимствований, всё, что вставляется после неё, обладает только правами на чтение.**

> We can however do this:

Однако мы можем сделать так:

```rust
fn opaque_read(val: &i32) {
    println!("{}", val);
}

unsafe {
    let mut data = 10;
    let mref1 = &mut data;
    let ptr2 = mref1 as *mut i32;
    let sref3 = &*mref1;
    let ptr4 = sref3 as *const i32 as *mut i32;

    opaque_read(&*ptr4);
    opaque_read(sref3);
    *ptr2 += 2;
    *mref1 += 1;

    opaque_read(&data);
}
```

> Note how it was still "fine" to create a mutable raw pointer as long as we only actually read from it!

Обратите внимание что создание изменяемого сырого указателя считается «нормальным», если, фактически, мы только читаем его данные!

```text
cargo run
10
10
13

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
10
10
13
```

> And just to be sure, let's check that a shared reference gets popped like normal:

И на всякий случай давайте убедимся, что разделяемая ссылка удаляется <!-- из стека заимствований --> как надо:

```rust ,ignore
fn opaque_read(val: &i32) {
    println!("{}", val);
}

unsafe {
    let mut data = 10;
    let mref1 = &mut data;
    let ptr2 = mref1 as *mut i32;
    let sref3 = &*mref1;

    *ptr2 += 2;
    opaque_read(sref3); // Read in the wrong order?
                        // Читаем в неправильном порядке?
    *mref1 += 1;

    opaque_read(&data);
}
```

```text
cargo run
12
13

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

error: Undefined Behavior: trying to reborrow for SharedReadOnly 
at alloc742, but parent tag <1620> does not have an appropriate 
item in the borrow stack

  --> src\main.rs:13:17
   |
13 |     opaque_read(sref3); // Read in the wrong order?
   |                 ^^^^^ trying to reborrow for SharedReadOnly 
   |                       at alloc742, but parent tag <1620> 
   |                       does not have an appropriate item 
   |                       in the borrow stack
   |
```

> Hey, we even got a slightly different error message about SharedReadOnly instead of some specific tag.
> That makes sense: once there's *any* shared references, basically everything else is just a big SharedReadOnly soup so there's no need to distinguish any of them!

Эй, мы даже получили немного другое сообщение об ошибке про SharedReadOnly вместо сообщения о конкретном маркере <!-- конкретной переменной -->.
Это логично: как только появляется *несколько* разделяемых ссылок, всё превращается в одну большую груду SharedReadOnly и нет нужны различать конкретные ссылки!

> # Testing Interior Mutability

# Тестирование внутренней изменчивости

> Remember that really horrible chapter of the book where we tried to make a linked list with RefCell and Rc and everything was even worse than usual when trying to write this godforsaken linked lists?

Помните ту по настоящему ужасную главу книги, где мы пытались написать связный список с помощью RefCell и Rc и всё было даже хуже, чем обычно, когда пытаешься написать эти хреновы связные списки?

> We've been insisting shared references can't be used for mutation but that chapter was all about how you could actually mutate through shared references with *interior mutability*.
> Let's try the nice and simple [std::cell::Cell](https://doc.rust-lang.org/std/cell/struct.Cell.html) type:

Всё это время мы настаиваем, что разделяемые ссылки нельзя использовать для изменения, но вся так глава была о том, что на самом деле мегять значение разделяемых ссылок можно с помощью *внутренней изменчивости*.
Давайте попробуем простой и удобный тип [std::cell::Cell](https://doc.rust-lang.org/std/cell/struct.Cell.html):

```rust
use std::cell::Cell;

unsafe {
    let mut data = Cell::new(10);
    let mref1 = &mut data;
    let ptr2 = mref1 as *mut Cell<i32>;
    let sref3 = &*mref1;

    sref3.set(sref3.get() + 3);
    (*ptr2).set((*ptr2).get() + 2);
    mref1.set(mref1.get() + 1);

    println!("{}", data.get());
}
```

> Ah, such a beautiful mess.
> It will be lovely to see miri spit on it.

Ах, какая великолепная мешанина!
Будет здорово увидеть, как miri её обругает.
<!-- в оригинале "наплюёт на неё", но в русском плевать на значит игнорировать, здесь смысл обратный -->


```text
cargo run
16

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
16
```

> Wait, really?
> *That's* fine?
> Why?
> How?
> What even is a *Cell*?

Подождите, правда?
*Тут* всё нормально?
Почему?
Как?
Что вообще такое эта ячейка *Cell*?

> *Smashes the padlock on the stdlib*

*Вскрывает замок на стандартной библиотеке*

```rust ,ignore
pub struct Cell<T: ?Sized> {
    value: UnsafeCell<T>,
}
```

> What the heck is `UnsafeCell`?

Что это за ерундовина, `UnsafeCell`?

> *Smashes another padlock just to really show the stdlib we mean business*

*Вскрывает другой замок, просто чтобы показать стандартной библиотеке, что мы настроены серьёзно*

```rust ,ignore
#[lang = "unsafe_cell"]
#[repr(transparent)]
#[repr(no_niche)]
pub struct UnsafeCell<T: ?Sized> {
    value: T,
}
```

> Oh it's wizard magic.
> Ok.
> I guess.
> `#[lang = "unsafe_cell"]` is literally just saying UnsafeCell is UnsafeCell.
> Let's stop breaking locks and check the actual documentation of [std::cell::UnsafeCell](https://doc.rust-lang.org/std/cell/struct.UnsafeCell.html).

А, это магия.
Понятно.
Наверное.
`#[lang = "unsafe_cell"]` — это буквально способ сказать, что UnsafeCell — это UnsafeCell.
Давайте прекратим вкрывать замки и проверим актуальную документацию на [std::cell::UnsafeCell](https://doc.rust-lang.org/std/cell/struct.UnsafeCell.html).

> > The core primitive for interior mutability in Rust.
> >
> > If you have a reference `&T`, then normally in Rust the compiler performs optimizations based on the knowledge that `&T` points to immutable data.
> > Mutating that data, for example through an alias or by transmuting an `&T` into an `&mut T`, is considered undefined behavior.
> > `UnsafeCell<T>` opts-out of the immutability guarantee for `&T`: a shared reference `&UnsafeCell<T>` may point to data that is being mutated.
> > This is called “interior mutability”.

> Базовый примитив для <!-- реализации --> внутренней изменчивости в Rust.
> 
> Если у вас есть ссылка `&T`, тогда обычно компилятор Rust выполняет оптимизации, основанные на том, что `&T` указывает на неизменяемые данные.
> Изменение этих данных, например, через псевдоним или после преобразования `&T` в `&mut T` ведёт к неопределённому поведению.
> `UnsafeCell<T>` отключает гарантию неизменяемости для `&T`: разделяемая ссылка `&UnsaveCell<T>` может указывать на данные, которые могут измениться.
> Это называется «внутренней изменчивостью».

> Oh it *really is* just wizard magic.

А, так это *на самом деле* просто магия.

> UnsafeCell basically tells the compiler "hey listen, we're gonna get goofy with this memory, don't make any of the usual aliasing assumptions about it".
> Like putting up a big "CAUTION: TINY ANGRY MEN CROSSING" sign.

По сути своей, UnsafeCell говорит компилятору: «Эй, послушай, мы собираемся валять дурака с этой памятью, не делай никаких обычнрых предположений по поводу её».
Это как повесить большой знак «ВНИМАНИЕ: ПЕРЕХОД ДЛЯ СЕРДИТЫХ ЧЕЛОВЕЧКОВ».

> Let's see how adding UnsafeCell makes miri happy:

Давайте взглянем, как добавление UnsafeCell делает miri счастливым:

```rust ,ignore
use std::cell::UnsafeCell;

fn opaque_read(val: &i32) {
    println!("{}", val);
}

unsafe {
    let mut data = UnsafeCell::new(10);
    let mref1 = data.get_mut();      // Get a mutable ref to the contents
    let ptr2 = mref1 as *mut i32;    // Получаем изменяемиую ссылку на содержимое
    let sref3 = &*ptr2;

    *ptr2 += 2;
    opaque_read(sref3);
    *mref1 += 1;

    println!("{}", *data.get());
}
```

```text
cargo run
12
13

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

error: Undefined Behavior: trying to reborrow for SharedReadOnly
at alloc748, but parent tag <1629> does not have an appropriate
item in the borrow stack

  --> src\main.rs:15:17
   |
15 |     opaque_read(sref3);
   |                 ^^^^^ trying to reborrow for SharedReadOnly 
   |                       at alloc748, but parent tag <1629> does
   |                       not have an appropriate item in the
   |                       borrow stack
   |

```

> Wait, what?
> We spoke the magic words!
> What am I going to do with all this federally approved ritual-enhancing goat blood?

Подождите, что?
Мы же сказали волшебные слова?
Что мне теперь делать со всей этой сертифицированной жертвенной кровью с повышенной ритуальностью?

> Well, we did, but then we completely discarded the spell by using `get_mut` which peeks inside the UnsafeCell and makes a proper `&mut i32` to it anyway!

Ну, мы так и сделали, но затем полностью отказались от заклинания, использовав `get_mut`, который заглядывает в UnsafeCell и превращает его в полноценный `&mut i32`.

> Think about it: if the compiler had to assume `&mut i32` *could* be looking inside an `UnsafeCell`, then it would never be able to make any assumptions about aliasing at all!
> Everything could be full of tiny angry men.

Подумайте об этом: если компилятор предполагает, что `&mut i32` *может* заглядывать в `UnsafeCell`, тогда он никогда не сможет делать никаких предположений о псевдономизации вообще!
Всё должно быть полным сердитых человечков.

> So what we need to do is keep the `UnsafeCell` in our pointer types so that the compiler understands what we're doing.

Что нам надо сделать: оставить `UnsafeCel` в наших типах указателей, чтобы компилятор понимал, что мы хотим сделать.

```rust
use std::cell::UnsafeCell;

fn opaque_read(val: &i32) {
    println!("{}", val);
}

unsafe {
    let mut data = UnsafeCell::new(10);
    let mref1 = &mut data;              // Mutable ref to the *outside*
    let ptr2 = mref1.get();             // Get a raw pointer to the insides
    let sref3 = &*mref1;                // Get a shared ref to the *outside*
                                        // Изменяемая ссылка на *весь объект*
                                        // Получаем сырой указатель на содержимое
                                        // Получаем разделяемую ссылку на *весь объект*

    *ptr2 += 2;                         // Mutate with the raw pointer
    opaque_read(&*sref3.get());         // Read from the shared ref
    *sref3.get() += 3;                  // Write through the shared ref
    *mref1.get() += 1;                  // Mutate with the mutable ref
                                        // Меняем значение по сырому указателю
                                        // Читаем из разделяемой ссылки
                                        // Пишем через разделяемую ссылку
                                        // Меняем через изменяемую ссылку

    println!("{}", *data.get());
}
```


```text
cargo run
12
16

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
12
16
```

> It works!
> I won't have to throw out all this blood after all.

Заработало!
В итоге мне не придётся выливать всю эту кровь.

> Actually, hey wait.
> We're still being a bit goofy with the order here.
> We made ptr2 first, and then made sref3 from the mutable pointer.
> And then we used the raw pointer before the shared pointer.
> That all seems... wrong.

На самом деле, эй, подождите.
Мы всё ещё немного не разобрались с порядком.
Сначала мы создали pt2, а затем создали sref3 из изменяемого указателя.
А потом мы использовали сырой указатель перед разделяемым указателем.
Это кажется... неправильным.

> Actually wait we did that with the Cell example too.
> HMMM.

На самом деле подождите, мы же делали это и в примере с Cell.
Хммм.

> We're forced to conclude one of two things:

Мы должны заключить одну из двух вещей:

> * Miri is imperfect and this is actually still UB.
> * Our simplified model is in fact an oversimplication.

* Miri несовершенна и на самом деле у нас всё ещё UB.
* Наша упрощённая модель на самом деле оказалсь чрезмерно упрощённой.

> I'd put my money on the second one, but just to be safe let's make a version that's definitely airtight in our simplified model of stacked borrows:

Я бы поставила на второй вариант, но просто чтобы быть уверенными давайте сделаем версию, которой будет абсолютно нажёдной в нашей упрощённой модели стековых заимствований:

```rust
use std::cell::UnsafeCell;

fn opaque_read(val: &i32) {
    println!("{}", val);
}

unsafe {
    let mut data = UnsafeCell::new(10);
    let mref1 = &mut data;
    // These two are swapped so the borrows are *definitely* totally stacked
    // Меняем эти два местами, так что зависимости *определённо* полностью в порядке стека
    let sref2 = &*mref1;
    // Derive the ptr from the shared ref to be super safe!
    // Создаём ptr из разделяемой ссылки для максимальной безопасности!
    let ptr3 = sref2.get();             

    *ptr3 += 3;
    opaque_read(&*sref2.get());
    *sref2.get() += 2;
    *mref1.get() += 1;

    println!("{}", *data.get());
}
```

```text
cargo run
13
16

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
13
16
```

> Now, one reason why the first implementation we had *might* actually be correct is because if you *really* think about it `&UnsafeCell<T>` really is no different from `*mut T` as far as aliasing is concerned.
> You can infinitely copy it and mutate through it!

Одна из причин, почему первая реализация, которая у нас была, *могла* на самомделе быть корректной, потому что если *на самом деле* задуматься, `&UnsafeCell<T>`, на самом деле не отличается от `*mut T`, если вести речь о псевдонимах.
Вы можете бесконечно копировать его и менять его!

> So in some sense we just created two raw pointers and used them interchangeably like normal.
> It's *a little* sketchy that both were derived from the mutable reference, so maybe the second one's creation should still pop the first one off the borrow stack, but that's not really necessary since we're not *actually* accessing the contents of the mutable reference, just copying its address.

Так что мы всего лишь создали два сырых указателя и использовали их по очереди, как обычно.
*Немного* подозрительно, что оба указателя получены из изменяемой ссылки, так что, кажется при создании второго надо удалять из стека заимствований первый, но это на самом деле не ужо, поскольку мы *на самом деле* не обращаемся к содержимому изменяемой ссылки, а просто копируем его адрес.

> A line like `let sref2 = &*mref1` is a tricksy thing.
> *Syntactically* it looks like we're dereferencing it, but dereferencing on it's own isn't actually a *thing*?
> Consider `&my_tuple.0`: you aren't actually doing anything to `my_tuple` or `.0`, you're just using them to refer to a location in memory and putting `&` in front of it that says "don't load this, just write the address down".

Строка `let sref2 = &*mref1` — довольно хитрая штука.
*Синтаксически* кажется, что мы разыменовываем указатель, но разыменование само по себе — это не что-то *реальное*.
Сравните с `&my_tuple.0`: вы на самом деле ничего ни с `my_tuple`, ни с `.0`, вы просто используете выражение, чтобы получить адрес в памяти, а, записав перед ним `&` как бы говорите: «не загружай содержимое, просто запомни адрес».

> `&*` is the same thing: the `*` is just saying "hey let's talk about the location this pointer points to" and the `&` is just saying "now write that address down".
> Which is of course the same value the original pointer had.
> But the type of the pointer has changed, because, uh, types!

`&*` — это то же самое: `*` говорит «давай обсудим местоположение, на которое указывает этот указатель», а `&` говорит «теперь запиши этот адрес».
И, конечно же, речь идёт о том же значении, что и у исходного указателя.
Но тип указателя изменился, потому что... ну, типы!

> That said, if you do `&**` then you are in fact loading a value with the first `*`! `*` is weird!

С другой стороны, если вы пишете `&**`, то фактически загружаете с первой `*`!
Эта `*` такая странная!

> > **NARRATOR:** No one cares that you know the word "lvalue", *Jonathan*.
> > In Rust we call them *places*, which is totally different and *so* much cooler?

> **ГОЛОС ЗА КАДРОМ:** Никого не волнует, что вам знаком слово «л-значение», *Джонатан*.
> В Rust мы называем эти штуки *местами*, что полностью отлично и *гораздо* круче?

> # Testing Box

# Тестирование Box

> Hey remember why we started this extremely long aside?
> You don't?
> Weird.

Эй, помните, почему мы начали это очень долгое отступление?
Нет?
Странно.

> Well it was because we mixed Box and raw pointers.
> Box is *kind of* like `&mut`, because it claims unique ownership of the memory it points to.
> Let's test that claim out:

Ну, это было потому, что мы смешали Box и сырой указатель.
Box — это *что-то вроде* `&mut`, поскеольку он заявляет об единоличном владении памятью, на которую указывает.
Давайте проверим это утверждение:

```rust ,ignore
unsafe {
    let mut data = Box::new(10);
    let ptr1 = (&mut *data) as *mut i32;

    *data += 10;
    *ptr1 += 1;

    // Should be 21
    // Должно быть 21
    println!("{}", data);
}
```

```text
cargo run
21

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

error: Undefined Behavior: no item granting read access 
       to tag <1707> at alloc763 found in borrow stack.

 --> src\main.rs:7:5
  |
7 |     *ptr1 += 1;
  |     ^^^^^^^^^^ no item granting read access to tag <1707> 
  |                at alloc763 found in borrow stack.
  |
```

> Yep, miri hates that.
> Let's check that doing things in the right order is ok:

Да, miri это не нриавится.
Давайте проверим, что выполнение действий в правильном порядке не приводит к ошибкам:

```rust
unsafe {
    let mut data = Box::new(10);
    let ptr1 = (&mut *data) as *mut i32;

    *ptr1 += 1;
    *data += 10;

    // Should be 21
    // Должно быть 21
    println!("{}", data);
}
```

```text
cargo run
21

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
21
```

> Yep!

Так!

> Whelp that's all folks, we're finally done talking and thinking about stacked borrows!

Что ж, на этом всё, мы, наконец, закончили обсуждать и думать о стековых заимствованиях!

> ...wait how do we solve this problem with Box?
> Like, sure we can write toy programs like this but we need to store the Box somewhere and hold onto our raw pointers for a potentially long time.
> Surely stuff will get mixed up and invalidated?

...хотя, как нам решить проблему с Box?
Конечно, мы можем писать похожие игрушечные программы, но нам надо хранить где-то Box и удерживать значения сырых указателей в течение потенциально долгого времени.
Несомненно, всё перепутается и стнет недействительным?

> Great question!
> To answer that we'll finally be returning to our true calling: writing some god damn linked lists.

Хороший вопрос!
Чтобы на него ответить, мы наконец вернёмся к нашей истинной задаче: написанию хреновых связных списков.

> Wait, I need to write linked lists again?
> Let's not be hasty folks.
> Be reasonable.
> Just hold on I'm sure there's some other interesting issues for me to discu&mdash;

Подождите, мне снова предстоит писать связные списки?
Давайте не будем торопиться, друзья.
Будем благоразумны.
Я уверена, что у меня есть несколько интересных вопросов для обсуждения.