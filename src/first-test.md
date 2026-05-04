> # Testing

# Тестирование

> Alright, so we've got `push` and `pop` written, now we can actually test out our stack!
> Rust and cargo support testing as a first-class feature, so this will be super easy.
> All we have to do is write a function, and annotate it with `#[test]`.

Прекрасно, теперь у нас есть `push` и `pop`, теперь мы можем действительно протестировать наш стек!
Для Rust и cargo тестирование — операция первого класса, так что это должно быть очень просто.
Всё, что нам надо, это написать функцию и пометить её аннотацией `#[test]`.

> Generally, we try to keep our tests next to the code that it's testing in the Rust community.
> However we usually make a new namespace for the tests, to avoid conflicting with the "real" code.
> Just as we used `mod` to specify that `first.rs` should be included in `lib.rs`, we can use `mod` to basically create a whole new file *inline*:

В целом, мы пытаемся размещать наши тесты рядом с тестируемым кодом, как это принято в сообществе Rust.
Тем не менее, обычно мы создаём новое пространство имён для тестов, чтобы избежать конфликтов с «реальным» кодом.

Точно также, как мы используем `mod`, чтобы указать, что `first.rs` должен включаться в `lib.rs`, мы можем использовать `mod`, чтобы создать *встроенный* модуль:


```rust ,ignore
// in first.rs
// а first.rs

mod test {
    #[test]
    fn basics() {
        // TODO
        // написать
    }
}
```

> And we invoke it with `cargo test`.

Мы запускаем тесты командой `cargo test`.

```text
> cargo test
   Compiling lists v0.1.0 (/Users/ADesires/dev/temp/lists)
    Finished dev [unoptimized + debuginfo] target(s) in 1.00s
     Running /Users/ADesires/dev/lists/target/debug/deps/lists-86544f1d97438f1f

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
; 0 filtered out
```

> Yay our do-nothing test passed!
> Let's make it not-do-nothing.
> We'll do that with the `assert_eq!` macro.
> This isn't some special testing magic.
> All it does is compare the two things you give it, and panic the program if they don't match.
> Yep, you indicate failure to the test harness by freaking out!

Ура, наши ничего не делающие тесты прошли!
Давайте сделаем их делающими что-нибудь.
Сделаем это с помощью макроса `assert_eq!`.
Это не какая-то особая магия тестирования.
Всё, что он делает — сравнивает две штуки, которые вы ему передали, и паникует, если они не совпадают.
Да, вы сигнализируете о проблеме с помощью паники!

```rust ,ignore
mod test {
    #[test]
    fn basics() {
        let mut list = List::new();

        // Check empty list behaves right
        // Проверяем, что пустой список ведёт себя правильно
        assert_eq!(list.pop(), None);

        // Populate list
        // Заполняем список
        list.push(1);
        list.push(2);
        list.push(3);

        // Check normal removal
        // Проверяем обычное удаление
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(2));

        // Push some more just to make sure nothing's corrupted
        // Вставляем новые значения, просто чтобы проверить, что ничего не сломается
        list.push(4);
        list.push(5);

        // Check normal removal
        // Проверяем обычное удаление
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), Some(4));

        // Check exhaustion
        // Проверяем граничный случай
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), None);
    }
}
```

```text
> cargo test

error[E0433]: failed to resolve: use of undeclared type or module `List`
  --> src/first.rs:43:24
   |
43 |         let mut list = List::new();
   |                        ^^^^ use of undeclared type or module `List`


```

> Oops!
> Because we made a new module, we need to pull in List explicitly to use it.

Ой!
Поскольку мы создали новый модуль, нам надо явным образом импортировать в него `List`.

```rust ,ignore
mod test {
    use super::List;
    // everything else the same
    // всё остальное точно также
}
```

```text
> cargo test

warning: unused import: `super::List`
  --> src/first.rs:45:9
   |
45 |     use super::List;
   |         ^^^^^^^^^^^
   |
   = note: #[warn(unused_imports)] on by default

    Finished dev [unoptimized + debuginfo] target(s) in 0.43s
     Running /Users/ADesires/dev/lists/target/debug/deps/lists-86544f1d97438f1f

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
; 0 filtered out
```

> Yay!

Да!

> What's up with that warning though...?
> We clearly use List in our test!

Хотя, что это за предупреждение?..
В нашем тесте мы используем List явным образом!

> ...but only when testing!
> To appease the compiler (and to be friendly to our consumers), we should indicate that the whole `test` module should only be compiled if we're running tests.

...но только при запуске тестов!
Чтобы угодить компилятору (и быть дружелюбным к другим программистам) мы должны явно указать, что весь модуль `test` должен быть скомпилирован только когда мы запускаем тесты.

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;
    // everything else the same
    // всё остальное точно также
}
```

> And that's everything for testing!

На этом тестирование завершено!