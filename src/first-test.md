# Тестирование

Прекрасно, теперь, когда у нас есть `push` и `pop`, мы можем протестировать наш стек!
Для Rust и cargo тестирование — это гражданин первого класса, так что всё должно быть очень просто.
Достаточно написать функцию и пометить её аннотацией `#[test]`.

В целом, мы пытаемся размещать наши тесты рядом с тестируемым кодом, как это принято в сообществе Rust.
Тем не менее, обычно мы пишем тесты в новом пространстве имён, чтобы избежать конфликтов с «реальным» кодом.

Ключевое слово `mod` можно использовать не только для объявления внешнего модуля (как мы сделали это с `first.rs` и `lib.rs`), но и для создания *встроенного* модуля:

```rust ,ignore
// в first.rs

mod test {
    #[test]
    fn basics() {
        // написать
    }
}
```

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

Ура, наш тест, который ничего не проверяет, успешно пройден!
Теперь давайте напишем какую-нибудь проверку.
В этом нам поможет макрос `assert_eq!`.
В нём нет никакой особой магии тестирования.
Он всего навсего сравнивает две штуки, которые ему передали, и паникует, если они не совпадают.
Да, мы сигнализируем о проблеме при помощи паники!

```rust ,ignore
mod test {
    #[test]
    fn basics() {
        let mut list = List::new();

        // Проверяем, что пустой список ведёт себя правильно
        assert_eq!(list.pop(), None);

        // Заполняем список
        list.push(1);
        list.push(2);
        list.push(3);

        // Проверяем обычное удаление
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(2));

        // Вставляем новые значения, просто чтобы проверить, что ничего не сломается
        list.push(4);
        list.push(5);

        // Проверяем обычное удаление
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), Some(4));

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

Ой!
Поскольку мы создали новый модуль, в него надо явным образом импортировать `List`.

```rust ,ignore
mod test {
    use super::List;
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

Да!

Хотя, что это за предупреждение?..
В нашем тесте мы используем List явным образом!

...но только при запуске тестов!
Чтобы угодить компилятору (и быть дружелюбным к другим программистам) мы должны явно указать, что весь модуль `test` должен быть скомпилирован только когда мы запускаем тесты.

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;
    // всё остальное точно также
}
```

На этом тестирование завершено!