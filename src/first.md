> # A Bad Singly-Linked Stack

# Плохой односвязный список

> This one's gonna be *by far* the longest, as we need to introduce basically all of Rust, and are gonna build up some things "the hard way" to better understand the language.

Эта глава будет *гораздо* длиннее прочих, поскольку нам надо познакомиться практически со всем языком Rust, и мы собираемся осваивать некоторые вещи «сложным путём», чтобы лучше понять язык.

> We'll put our first list in `src/first.rs`.
> We need to tell Rust that `first.rs` is something that our lib uses.
> All that requires is that we put this at the top of `src/lib.rs` (which Cargo made for us):

Мы поместим наш первый список в файл `src/first.rs`.
Нам надо сказать компилятору Rust, что `first.rs` входит в состав нашей библиотеки.
Всё, что нам нужно — это написать в начале файла `src/lib.rus` (который создала для нас утилита Cargo):

```rust ,ignore
// in lib.rs
// в lib.rs
pub mod first;
```

