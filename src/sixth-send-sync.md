# Send, Sync и тесты этапа компиляции

У нас есть ещё пара типажей, о которых стоит подумать, но они особенные.
Нам предстоит иметь дело со Древним Римом языка Rust: небезопасными встроенными явно-включаемыми типажами (The Unsafe Opt-In Built-In Traits, OIBITs): [Send и Sync](https://doc.rust-lang.org/nomicon/send-and-sync.html), которые на самом деле и не-встроенные и явно-отключаемые (есть 1 свойство из 3 — уже неплохо!).

Как и у Copy, у этих типажей нет никакого связанного с ними кода — они являются маркерами, показывающими, что ваш тип обладает определённым свойством.
Send говорит, что ваш тип можно безопасно передать в другой поток.
Sync говорит, что ваш тип можно безопасно разделять между потоками (&Self: Send).

Тот же аргумент, что и в случае ковариантности LinkedList, применим и здесь: в общем и целом нормальные коллекции, которые не используют причудливых трюков с внутренней изменчивостью, безопасны и для Send, и для Sync.

Но, как я сказала, они *явно отключаемые*.
Так что, на самом деле, может, у нас уже всё работает?
Как это можно проверить?

Давайте добавим немного новой магии к нашему коду: случайный приватный мусор, который не будет компилироваться, пока наши типы не обладают свойствами, которые мы ожидаем:

```rust ,ignore
#[allow(dead_code)]
fn assert_properties() {
    fn is_send<T: Send>() {}
    fn is_sync<T: Sync>() {}

    is_send::<LinkedList<i32>>();
    is_sync::<LinkedList<i32>>();

    is_send::<IntoIter<i32>>();
    is_sync::<IntoIter<i32>>();

    is_send::<Iter<i32>>();
    is_sync::<Iter<i32>>();

    is_send::<IterMut<i32>>();
    is_sync::<IterMut<i32>>();

    is_send::<Cursor<i32>>();
    is_sync::<Cursor<i32>>();

    fn linked_list_covariant<'a, T>(x: LinkedList<&'static T>) -> LinkedList<&'a T> { x }
    fn iter_covariant<'i, 'a, T>(x: Iter<'i, &'static T>) -> Iter<'i, &'a T> { x }
    fn into_iter_covariant<'a, T>(x: IntoIter<&'static T>) -> IntoIter<&'a T> { x }
}
```

```text
cargo build
   Compiling linked-list v0.0.3 
error[E0277]: `NonNull<Node<i32>>` cannot be sent between threads safely
   --> src\lib.rs:433:5
    |
433 |     is_send::<LinkedList<i32>>();
    |     ^^^^^^^^^^^^^^^^^^^^^^^^^^ `NonNull<Node<i32>>` cannot be sent between threads safely
    |
    = help: within `LinkedList<i32>`, the trait `Send` is not implemented for `NonNull<Node<i32>>`
    = note: required because it appears within the type `Option<NonNull<Node<i32>>>`
note: required because it appears within the type `LinkedList<i32>`
   --> src\lib.rs:8:12
    |
8   | pub struct LinkedList<T> {
    |            ^^^^^^^^^^
note: required by a bound in `is_send`
   --> src\lib.rs:430:19
    |
430 |     fn is_send<T: Send>() {}
    |                   ^^^^ required by this bound in `is_send`

<и ещё миллион ошибок>
```

Боже, ну что за дела?
А я ведь уже заготовила отличную шутку про Древний Рим!

Ладно, я соврала вам, когда сказала, что у сырых указателей есть только одна защита.
Есть и вторая.
У `*const` И `*mut` в целях безопасности явно отключены Send и Sync, так что нам *на самом деле* надо включить их обратно:

```rust ,ignore
unsafe impl<T: Send> Send for LinkedList<T> {}
unsafe impl<T: Sync> Sync for LinkedList<T> {}

unsafe impl<'a, T: Send> Send for Iter<'a, T> {}
unsafe impl<'a, T: Sync> Sync for Iter<'a, T> {}

unsafe impl<'a, T: Send> Send for IterMut<'a, T> {}
unsafe impl<'a, T: Sync> Sync for IterMut<'a, T> {}
```

Обратите внимание, что здесь мы должны писать *unsafe impl*: ведь это *небезопасные типажи*!
Небезопасный код (например, конкурентные библиотеки) зависят от того, насколько правильно мы их реализуем!
Но, поскольку в них нет никакого кода, мы гарантируем лишь то, что да, мы действительно можем передавать и разделять список между потоками!

Не стоит добавлять их просто так, но я, как Сертифицированный Специалист, могу сказать, да, с ними всё в порядке.
Обратите внимание, что нам не надо реализовывать Send и Sync для IntoIter: поскольку он просто содержит LinkedList, они для него определены автоматически.
Я же говорила, что Send и Sync явно отключаемые!

```text
cargo build
   Compiling linked-list v0.0.3
    Finished dev [unoptimized + debuginfo] target(s) in 0.18s
```

Что ж, прекрасно!

...Подождите, на самом деле это очень опасно, когда типы обладают свойством, которым *не должны* обладать.
Например, IterMut *определённо* не должен быть ковариантным, потому что он «как бы» `&mut T`.
Но как нам это проверить?

С помощью Магии!
Ну, на самом деле, с помощью rustdoc!
Ладно, нам не обязательно использовать для этого rustdoc, но это самый весёлый способ проверки.
Смотрите, если вы напишите документацию и включите в неё блок кода, rustdoc попытается скомпилировать и запустить его, и с помощью этого мы можем создавать новые безымянные «программы», которые не оказывают влияния на главный код:

```rust ,ignore
    /// ```
    /// use linked_list::IterMut;
    /// 
    /// fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
    /// ```
    fn iter_mut_invariant() {}
```

```text
cargo test

...

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 458) ... FAILED

failures:

---- src\lib.rs - assert_properties::iter_mut_invariant (line 458) stdout ----
error[E0308]: mismatched types
 --> src\lib.rs:461:86
  |
6 | fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
  |                                                                                      ^ lifetime mismatch
  |
  = note: expected struct `linked_list::IterMut<'_, &'a T>`
             found struct `linked_list::IterMut<'_, &'static T>`
```

Хорошо, мы доказали что наш тип инвариантный, но, хм, теперь наши тесты не проходят.
Не беспокойтесь, rustdoc позволяет написать аннотацию, которая подскажет, что это ожидаемое поведение: compile_fail!

(На самом деле мы всего лишь доказали, что тип «не ковариантный», но если вам удастся случайно сделать неправильный «контравариантный» тип, то... мои поздравления?)

```rust ,ignore
    /// ```compile_fail
    /// use linked_list::IterMut;
    /// 
    /// fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
    /// ```
    fn iter_mut_invariant() {}
```

```text
cargo test
   Compiling linked-list v0.0.3
    Finished test [unoptimized + debuginfo] target(s) in 0.49s
     Running unittests src\lib.rs

...

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 458) - compile fail ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.12s
```

Ура!
Я рекомендую в начале всегда создавать тест без compile_fail, чтобы убедиться, что он не компилируется *по правильной причине*.
Например, в этом же тесте возникнет ошибка (и, соответственно, он будет пройден), если вы забудете написать `use`, но это ведь не то, что нам нужно!
Хотя теоретически было бы правильно «требовать» у компилятора проверки на определённую ошибку, это стало бы абсолютным кошмаром, поскольку всякий раз, когда в *компиляторе улучшали бы выдачу ошибок*, это ломало бы тесты.
А мы хотим, чтобы компилятор улучшался, поэтому нет, вы этого не получите.

(Впрочем, мы можем просто указать нужный код ошибки вместе с compile_fail, **но это работает только в nightly-версиях и на это не стоит полагаться по причинам, озвученным выше. В не-nightly-версиях код ошибки молча игнорируется.**)

```rust ,ignore
    /// ```compile_fail,E0308
    /// use linked_list::IterMut;
    /// 
    /// fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
    /// ```
    fn iter_mut_invariant() {}
```

...кстати, вы заметили, когда мы на самом деле сделали IterMut инвариантным?
Пропустить было легко, поскольку я «всего-навсего» скопировала Iter и вставила его в конец.
Вот, в последней строке:

```rust ,ignore
pub struct IterMut<'a, T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<&'a mut T>,
}
```

Попробуем удалить PhantomData:

```text
 cargo build
   Compiling linked-list v0.0.3 (C:\Users\ninte\dev\contain\linked-list)
error[E0392]: parameter `'a` is never used
  --> src\lib.rs:30:20
   |
30 | pub struct IterMut<'a, T> {
   |                    ^^ unused parameter
   |
   = help: consider removing `'a`, referring to it in a field, or using a marker such as `PhantomData`
```

Ха!
Компилятор поддерживает нас и не позволяет нам просто *не* указывать время жизни.
Давайте теперь попробуем использовать *неправильный* пример:

```rust ,ignore
    _boo: PhantomData<&'a T>,
```

```text
cargo build
   Compiling linked-list v0.0.3 (C:\Users\ninte\dev\contain\linked-list)
    Finished dev [unoptimized + debuginfo] target(s) in 0.17s
```

Он собрался!
Смогут ли сейчас наши тесты выявить проблему?

```text
cargo test

...

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 458) - compile fail ... FAILED

failures:

---- src\lib.rs - assert_properties::iter_mut_invariant (line 458) stdout ----
Test compiled successfully, but it's marked `compile_fail`.

failures:
    src\lib.rs - assert_properties::iter_mut_invariant (line 458)

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.15s
```

Йееее!!!
Система работает!
Мне очень нравится, когда тесты делают свою работу, так что мне не приходиться переживать из-за возможных ошибок!
