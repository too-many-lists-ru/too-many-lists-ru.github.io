# Miri

*нервно смеётся* Все эти штуки с `unsafe` оказались такими простыми, я не понимаю, почему все утверждают обратное.
Наша программа прекрасно работает.

> **ГОЛОС ЗА КАДРОМ:** 🙂

...правда?

> **ГОЛОС ЗА КАДРОМ:** 🙂

Хорошо, мы написали небезопасный код и компилятор не может нам помочь с поиском ошибок.
Возможно, тесты сработали *случайно*, а на самом деле у нас там что-то недетерминированное.
Какое-то Неопределённое Поведение.

Но что мы можем сделать?
Мы выбрались из домика, в котором нас защищал rustc.
Теперь нам никто не поможет.

...Подождите, а что это за подозрительный тип в переулке?

*"Эй, парень, не хочешь поинтерпретировать немного Rust кода?"*

Ннннет...
Наверное.

*"Чувак, это невероятно, интерпретация может проверить, действительно ли исполнение твоей программы в динамике соответствует семантике модели памяти Rust. Крышу сносит..."*

Что?

*"Она проверяет, есть ли у тебя Неопределённое Поведение."*

Думаю, *один* раз я могу попробовать интерпретацию.

*"У тебя же установлен rustup, правда?"*

Конечно да, это же *та самая* утилита для поддержания инструментария Rust в актуальном состоянии!

```text
> rustup +nightly-2022-01-21 component add miri

info: syncing channel updates for 'nightly-2022-01-21-x86_64-pc-windows-msvc'
info: latest update on 2022-01-21, rust version 1.60.0-nightly (777bb86bc 2022-01-20)
info: downloading component 'cargo'
info: downloading component 'clippy'
info: downloading component 'rust-docs'
info: downloading component 'rust-std'
info: downloading component 'rustc'
info: downloading component 'rustfmt'
info: installing component 'cargo'
info: installing component 'clippy'
info: installing component 'rust-docs'
info: installing component 'rust-std'
info: installing component 'rustc'
info: installing component 'rustfmt'
info: downloading component 'miri'
info: installing component 'miri'
```

Что ты только что установил на мой компьютер!?

*"Отличную Вещь"*

> **ГОЛОС ЗА КАДРОМ:** Есть кое-какие тонкости, касающиеся версии инструментария:
> 
> Инструмент, который мы установили — `miri` — тесно взаимодействует с внутренностями rustc, поэтому доступен только в ночных сборках.
> 
> `+nightly-2022-01-21` указывает `rustup`, что мы хотим установить miri вместе с ночной сборкой инструментария за указанную дату.
> Я использовала конкретную дату, потому что иногда miri содержит ошибки и не компилируется несколько ночей подряд.
> rustup автоматически загрузит любой инструментарий, который мы укажем с помощью `+`, если он ещё не установлен.
>
> 2022-01-21 — это сборка, где, я точно знаю, miri работает, что можно проверить [на этой странице](https://rust-lang.github.io/rustup-components-history/).
> Если вы счастливчик, можете использовать просто `+nightly`.
> 
> Мы будем использовать синтаксис `+` при каждом вызове miri через `cargo miri` для указания версии совместимого инструментария.
> Если не хотите каждый раз писать версию, запустите [`rustup override set`](https://rust-lang.github.io/rustup/overrides.html)

```text
> cargo +nightly-2022-01-21 miri test

I will run `"cargo.exe" "install" "xargo"` to install 
a recent enough xargo. Proceed? [Y/n]
```

Э-Э-Э, ЧТО ЭТО ЗА XARGO?

*Не бери в голову, всё в порядке."*

```text
> y

    Updating crates.io index
  Installing xargo v0.3.24
...
    Finished release [optimized] target(s) in 10.65s
  Installing C:\Users\ninte\.cargo\bin\xargo-check.exe
  Installing C:\Users\ninte\.cargo\bin\xargo.exe
   Installed package `xargo v0.3.24` (executables `xargo-check.exe`, `xargo.exe`)

I will run `"rustup" "component" "add" "rust-src"` to install 
the `rust-src` component for the selected toolchain. Proceed? [Y/n]
```

Э-Э??

*"Кто же не любит иметь копию исходного кода Rust?"*

```text
> y

info: downloading component 'rust-src'
info: installing component 'rust-src'
```

*"Ну, вот, всё готово, а теперь самое интересное."*

```text
   Compiling lists v0.1.0 (C:\Users\ninte\dev\tmp\lists)
    Finished test [unoptimized + debuginfo] target(s) in 0.25s
     Running unittests (lists-5cc11d9ee5c3e924.exe)

error: Undefined Behavior: trying to reborrow for Unique at alloc84055, 
       but parent tag <209678> does not have an appropriate item in 
       the borrow stack

   --> \lib\rustlib\src\rust\library\core\src\option.rs:846:18
    |
846 |             Some(x) => Some(f(x)),
    |                  ^ trying to reborrow for Unique at alloc84055, 
    |                    but parent tag <209678> does not have an 
    |                    appropriate item in the borrow stack
    |
    = help: this indicates a potential bug in the program: 
      it performed an invalid operation, but the rules it 
      violated are still experimental
    = help: see https://github.com/rust-lang/unsafe-code-guidelines/blob/master/wip/stacked-borrows.md 
      for further information

    = note: inside `std::option::Option::<std::boxed::Box<fifth::Node<i32>>>::map::<i32, [closure@src\fifth.rs:31:30: 40:10]>` at \lib\rustlib\src\rust\library\core\src\option.rs:846:18

note: inside `fifth::List::<i32>::pop` at src\fifth.rs:31:9
   --> src\fifth.rs:31:9
    |
31  | /         self.head.take().map(|head| {
32  | |             let head = *head;
33  | |             self.head = head.next;
34  | |
...   |
39  | |             head.elem
40  | |         })
    | |__________^
note: inside `fifth::test::basics` at src\fifth.rs:74:20
   --> src\fifth.rs:74:20
    |
74  |         assert_eq!(list.pop(), Some(1));
    |                    ^^^^^^^^^^
note: inside closure at src\fifth.rs:62:5
   --> src\fifth.rs:62:5
    |
61  |       #[test]
    |       ------- in this procedural macro expansion
62  | /     fn basics() {
63  | |         let mut list = List::new();
64  | |
65  | |         // Check empty list behaves right
...   |
96  | |         assert_eq!(list.pop(), None);
97  | |     }
    | |_____^
 ...
error: aborting due to previous error
```

Ого.
Какая ужасная ошибка.

*"Да, взгляни на это дерьмо. Тебе понравится на него натыкаться."*

Благодарю?

*"Вот возьми и бутылочку эстрадиола, она тебе пригодится."*

Подожди, зачем?

*"Поверь мне, скоро ты начнёшь думать о моделях памяти."*

> **ГОЛОС ЗА КАДРОМ:** после этого таинственный незнакомец превратился в лису и ускользнул сквозь дыру в стене.
> Автор несколько минут смотрела в никуда, пытаясь осмыслить произошедшее.

-------

Таинственная лиса оказалась права не только по поводу моего пола: miri — действительно Отличная Вещь.

Так что же *такое* [miri](https://github.com/rust-lang/miri)?

> Экспериментальный интерпретатор промежуточного представления среднего уровня (Mid-level Intermediate Representation, MIR).
> Он может запускать бинарники и тестовые наборы из проектов cargo и выявлять целые классы неопределённого поведения, например:
> 
> * Выход за границы допустимого диапазона памяти и использование памяти после освобождения
> * Недопустимое использование неинициализированных данных
> * Нарушение внутренних предусловий (достижение `unreachable_unchecked`, вызов `copy_nonoverlapping` с перекрывающимися диапазонами, ...)
> * Доступ к памяти и ссылкам с недостаточно строгим выравниванием
> * Нарушение некоторых базовых инвариантов для типов (значение bool, отличное от 0 и 1 или неизвестная метка перечисления)
> * Экспериментально: Нарушение правил многоуровневого заимствования, регулирующих псевдонимы для ссылочных типов
> * Экспериментально: Состояние гонки данных (но без эффектов ослабленной модели памяти)
> 
> Кроме того, miri сообщит вам об утечках памяти.
> Если после завершения программы останется выделенная память, недоступная из глобальной статической переменной, miri выдаст ошибку.
> 
> ...
> 
> Однако, имейте в виду, что miri не может обнаружить все случаи неопределённого поведения в вашей программе и не может выполнить любую программу.

В двух словах: он интерпретирует вашу программу и сообщает о всех нарушениях правил *во время выполнения*, приведших к Неопределённому Поведению.
Это нужно, поскольку Неопределённое Поведение происходит *именно* во время выполнения.
Если бы проблему можно было обнаружить на этапе компиляции, компилятор просто выдал бы ошибку!

Возможно, вы знакомы с такими инструментами, как ubsan и tsan.
По сути, miri — такой же инструмент, но более полный и продвинутый.

-------

Выглядит так, что за стенами нашего домика miri шастает с ножом в руках.
Правда, нож учебный.

Чтобы miri проверил нашу работу, надо запустить его с такими параметрами:

```text
> cargo +nightly-2022-01-21 miri test
```

Давайте посмотрим, что он вырезал на нашей парте:

```text
error: Undefined Behavior: trying to reborrow for Unique at alloc84055, but parent tag <209678> does not have an appropriate item in the borrow stack

   --> \lib\rustlib\src\rust\library\core\src\option.rs:846:18
    |
846 |             Some(x) => Some(f(x)),
    |                  ^ trying to reborrow for Unique at alloc84055, 
    |                    but parent tag <209678> does not have an 
    |                    appropriate item in the borrow stack
    |

    = help: this indicates a potential bug in the program: it 
      performed an invalid operation, but the rules it 
      violated are still experimental
    
    = help: see 
      https://github.com/rust-lang/unsafe-code-guidelines/blob/master/wip/stacked-borrows.md 
      for further information
```

Ладно, я вижу, что мы допустили ошибку, но сообщение об ошибке сбивает меня с толку.
Что такое «стек заимствований».

Постараемся разобраться с этим в следующей части.
