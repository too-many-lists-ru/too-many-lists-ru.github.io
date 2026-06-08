> # Send, Sync, and Compile Tests

# Send, Sync и тесты этапа компиляции

> Ok actually we do have one more pair of traits to think about, but they're special.
> We have to deal with Rust's Holy Roman Empire: The Unsafe Opt-In Built-In Traits (OIBITs): [Send and Sync](https://doc.rust-lang.org/nomicon/send-and-sync.html), which are in fact opt-out and built-out (1 out of 3 is pretty good!).

Ладно, у нас есть ещё пара типажей, о которых нам надо подумать, но они особенные.
Нам предстоит иметь дело со Древним Римом языка Rust: небезопасными встроенными типажами с явным включением (The Unsafe Opt-In Built-In Traits, OIBITs): [Send и Sync](https://doc.rust-lang.org/nomicon/send-and-sync.html), которые на самом деле и не-встроенные и явно-отключаемые (есть 1 свойство из 3 — уже неплохо!).

> Like Copy, these traits have absolutely no code associated with them, and are just markers that your type has a particular property.
> Send says that your type is safe to send to another thread.
> Sync says your type is safe to share between threads (&Self: Send).

Как и у Copy, у этих типажей нет абсолютно никакого связанного с ними кода, и они являются всего лишь маркерами, что ваш тип обладает определённым свойством.
Send говорит, что ваш тип можно безопасно передать в другой поток.
Sync говорит, что ваш тип можно безопасно разделять между потоками (&Self: Send).

> The same argument for LinkedList being covariant applies here: generally normal collections which don't use fancy interior mutability tricks are safe to make Send and Sync.

Тот же самый аргумент, что и для ковариантности для LinkedList применим и здесь: в общем и целом нормальные коллекции, которые не используют причудливых трюков с внутренней изменчивостью, безопасны и для Send, и для Sync.

> But I said they're *opt out*.
> So actually, are we already?
> How would we know?

Но, как я сказала, они *явно отключаемые*.
Так что, на самом деле, может, у нас уже всё работает?
Как это можно проверить?

> Let's add some new magic to our code: random private garbage that won't compile unless our types have the properties we expect:  

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

<a million more errors>
<и ещё миллион ошибок>
```

> Oh geez, what gives!
> I had that great Holy Roman Empire joke!

Боже, ну что за дела?
А я ведь уже приготовила отличную шутку про Древний Рим!

> Well, I lied to you when I said raw pointers have only one safety guard: this is the other.
> `*const` AND `*mut` explicitly opt out of Send and Sync to be safe, so we do *actually* have to opt back in:

Ладно, я соврала вам, когда сказала, что у сырых указателей есть только одна защита: есть и вторая.
У `*const` И `*mut` явно отключены Send и Sync в целях безопасности, так что нам *на самом деле* надо включить их обратно:

```rust ,ignore
unsafe impl<T: Send> Send for LinkedList<T> {}
unsafe impl<T: Sync> Sync for LinkedList<T> {}

unsafe impl<'a, T: Send> Send for Iter<'a, T> {}
unsafe impl<'a, T: Sync> Sync for Iter<'a, T> {}

unsafe impl<'a, T: Send> Send for IterMut<'a, T> {}
unsafe impl<'a, T: Sync> Sync for IterMut<'a, T> {}
```

> Note that we have to write *unsafe impl* here: these are *unsafe traits*!
> Unsafe code (like concurrency libraries) gets to rely on us only implementing these traits correctly!
> Since there's no actual code, the guarantee we're making is just that, yes, we are indeed safe to Send or Share between threads!

Обратите внимание, что здесь мы должны писать *unsafe impl*: ведь это *небезопасные типажи*!
Небезопасный код (например, конкурентные библиотеки) зависят от того, насколько правильно мы реализуем эти типажи!
Но, поскольку в них нет никакого кода, мы гарантируем лишь то, что да, мы действительно можем передавать и разделять список между потоками!

> Don't just slap these on lightly, but I am a Certified Professional here to say: yep there's are totally fine.
> Note how we don't need to implement Send and Sync for IntoIter: it just contains LinkedList, so it auto-derives Send and Sync &mdash; I told you they were actually opt out!
> (You opt out with the hillarious syntax of `impl !Send for MyType {}`.)

Не стоит добавлять их просто так, но я, как Сертифицированный Специалист, могу сказать, да, они с ними всё в порядке.
Обратите внимание, что нам не надо реализовывать Send и Sync для IntoIter: поскольку он просто содержит LinkedList, для него автоматически определяются Send и Sync — я же говорила, что они явно отключаемые!

```text
cargo build
   Compiling linked-list v0.0.3
    Finished dev [unoptimized + debuginfo] target(s) in 0.18s
```

> Ok nice!

Что ж, прекрасно!

> ...Wait, actually it would be really dangerous if stuff that *shouldn't* be these things wasn't.
> In particular, IterMut *definitely* shouldn't be covariant, because it's "like" `&mut T`.
> But how can we check that?

...Подождите, на самом деле это очень опасно, когда типы обладают свойством, которым *не должны* обладать. <!-- похоже, опечатка у автора -->
Например, IterMut *определённо* не должен быть ковариантным, потому что он «как бы» `&mut T`.
Но как нам это проверить!

> With Magic!
> Well, actually, with rustdoc!
> Ok well we don't have to use rustdoc for this, but it's the funniest way to do it.
> See, if you write a doccomment and include a code block, then rustdoc will try to compile and run it, so we can use that to make fresh anonymous "programs" that don't affect the main one:


С помощью Магии!
Ну, на самом деле, с помощью rustdoc!
Ладно, нам не обязательно использовать для этого rustdoc, но это самый весёлый способ проверки.
Смотрите, если вы напишите документацию и включите в неё блок кода, rustdoc попытается скомпилировать и запустить его, и с помощью этого мы можем делать новые безымянные «программы», которые не оказывают влияния на главный код:

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

> Ok cool, we've proved it's invariant, but uh, now our tests fail.
> No worries, rustdoc lets you say that's expected by annotating the fence with compile_fail!

Хорошо, мы доказали что наш тип инвариантный, но, хм, теперь наши тесты не проходят.
Не беспокойтесь, rustdoc позволяет написать аннотацию, которая подскажет, что это ожидаемое поведение: compile_fail!

> (Actually we only proved it's "not covariant" but honestly if you manage to make a type "accidentaly and incorrectly contravariant" then, congrats?)

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

> Yay!
> I recommend always making the test without compile_fail so that you can confirm that it fails to compile *for the right reason*.
> For instance, that test will also fail (and therefore pass) if you forget the `use`, which, is not what we want!
> While it's conceptually appealing to be able to "require" a specific error from the compiler, this would be an absolute nightmare that would effectively make it a breaking change *for the compiler to produce better errors*.
> We want the compiler to get better, so, no you don't get to have that.

Ура!
Я рекомендую в начале всегда создавать тест без compile_fail, чтобы убедиться, что он не компилируется *по правильной причине*.
Например, в этом же тесте возникнет ошибка (и, соответственно, он будет пройден), если вы забудете написать `use`, но это ведь не то, что нам нужно!
Хотя теоретически было бы правильно «требовать» у компилятора проверки на определённую ошибку, это стало бы абсолютным кошмаром, поскольку всякий раз, когда в *компиляторе улучшали бы выдачу ошибок*, это ломало бы тесты.
А мы хотим, чтобы компилятор улучшался, поэтому нет, вы этого не получите.

> (Oh wait, we can actually just specify the error code we want next to the compile_fail **but this only works on nightly and is a bad idea to rely on for the reasons state above. It will be silently ignored on not-nightly.**)

(Ой, подождите, мы можем просто указать нужный код ошибки вместе с compile_fail, **но это работает только в nightly-версиях и на это не стоит полагаться по причинам, озвученным выше. В не-nightly-версиях код ошибки молча игнорируется.**)

```rust ,ignore
    /// ```compile_fail,E0308
    /// use linked_list::IterMut;
    /// 
    /// fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
    /// ```
    fn iter_mut_invariant() {}
```

> ...also, did you notice the part where we actually made IterMut invariant?
> It was easy to miss, since I "just" copy-pasted Iter and dumped it at the end.
> It's the last line here:

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

> Let's try removing that PhantomData:

Давайте попробуем удалить PhantomData:

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

> Ha!
> The compiler has our back and won't just let us *not* use the lifetime.
> Let's try using the *wrong* example instead:

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

> It builds!
> Do our tests catch a problem now?

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

> Eyyy!!!
> The system works!
> I love having tests that actually do their job, so that I don't have to be quite so horrified of looming mistakes!

Йееее!!!
Система работает!
Мне очень нравится, когда тесты действительно делают свою работу, так что мне не приходиться переживать из-за возможных ошибок!
