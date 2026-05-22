> # Unsafe Rust

# Небезопасный Rust

> This is a serious, big, complicated, and dangerous topic.
> It's so serious that I wrote [an entire other book][nom] on it.

Это серьёзная, большая, сложная и опасная тема.
Настолько серьёзная, что я написала про неё [отдельную книгу][nom].

<!-- речь про Растономикон -->

> The long and the short of it is that *every* language is actually unsafe as soon as you allow calling into other languages, because you can just have C do arbitrarily bad things.
> Yes: Java, Python, Ruby, Haskell... everyone is wildly unsafe in the face of Foreign Function Interfaces (FFI).

Кратко говоря, *любой* язык на самом деле небезопасный, как скоро из него можно вызывать функции на другом языке, потому что в C можно делать произвольно плохие вещи.
Да: Java, Python, Ruby, Haskell... все они крайне небезопасны при наличии интерфейса внешних функций (Foreign Function Interfaces, FFI).

> Rust embraces this truth by splitting itself into two languages: Safe Rust, and Unsafe Rust.
> So far we've only worked with Safe Rust.
> It's completely 100% safe... except that it can FFI into Unsafe Rust.

Rust принимает эту истину, разделяя себя на два языка: Безопасный Rust и Небезопасный Rust.
Пока что мы работали только с Безопасным Rust.
Это на 100% безопасно... пока FFI не приводит нас к Небезопасному Rust.

> Unsafe Rust is a *superset* of Safe Rust.
> It's completely the same as Safe Rust in all its semantics and rules, you're just allowed to do a few *extra* things that are wildly unsafe and can cause the dreaded Undefined Behaviour that haunts C.

Небезопасный Rust — это *надмножество* Безопасного Rust.
Он полностью такой же, как и Безопасный Rust со всеми его семантикой и правилами, но только вы можете делать несколько *дополнительных* вещей, которые дико небезопасный и могут вызвать ужасное Неопределённое Поведение (Undefined Behaviour), часто возникающее в языке C.

> Again, this is a really huge topic that has a lot of interesting corner cases.
> I *really* don't want to go really deep into it (well, I do. I did. [Read that book][nom]).
> That's ok, because with linked lists we can actually ignore almost all of it.

Опять же, это действительно огромная тема, в которой много интересных частных случаев.
Я *на самом деле* не хочу в неё погружаться (ну, ладно, хочу; уже погрузилась; [читайте книгу][nom]).
Это нормально, потому что в случае связных списков мы на самом деле можем игнорировать большинство из них.

> > **NARRATOR:** This was a lie, but it did seem true in 2015.

> **ГОЛОС ЗА КАДРОМ:** Это была ложь, но в 2015 она казалась правдой.

> The main Unsafe tool we'll be using are *raw pointers*.
> Raw pointers are basically C's pointers.
> They have no inherent aliasing rules.
> They have no lifetimes.
> They can be null.
> They can be misaligned.
> They can be dangling.
> They can point to uninitialized memory.
> They can be cast to and from integers.
> They can be cast to point to a different type.
> Mutability?
> Cast it.
> Pretty much everything goes, and that means pretty much anything can go wrong.

Главный инструмент Небезопасного Rust, который мы будем использовать — это *сырые указатели*.
Сырые указатели — по сути, те же указатели из C.
У них нет собственных правил псевдонимизации.
У них нет времени жизни.
Они могут быть нулевыми.
Они могут быть невыровненными.
Они могут быть висячими.
Они могут указывать на неинициализированную память.
Их можно приводить к целым числам и обратно.
Их можно приводить к указателям на другой тип.
Изменяемость?
Приводим!
Можно практически всё, а это значит, что практически всё может пойти не так.

> > **NARRATOR:** no inherent aliasing rules, eh?
> > Ah, the innocence of youth.

> **ГОЛОС ЗА КАДРОМ:** нет собственных правил псевдонимизации, да?
> Ах, наивность юности.

> This is some bad stuff and honestly you'll live a happier life never having to touch these.
> Unfortunately, we want to write linked lists, and linked lists are awful.
> That means we're going to have to use unsafe pointers.

Это довольно неприятная штука и, честно говоря, вы проживёте более счастливую жизнь, никогда с ней не сталкиваясь.
К сожалению, мы хотим писать связные списки, а связные списки ужасны.
Это значит, что нам придётся использовать небезопасные указатели.

> There are two kinds of raw pointer: `*const T` and `*mut T`.
> These are meant to be `const T*` and `T*` from C, but we really don't care about what C thinks they mean that much.
> You can only dereference a `*const T` to an `&T`, but much like the mutability of a variable, this is just a lint against incorrect usage.
> At most it just means you have to cast the `*const` to a `*mut` first.
> Although if you don't actually have permission to mutate the referent of the pointer, you're gonna have a bad time.

Есть два вида сырых указателей: `*const T` и `*mut T`.
Они соответствуют `const T*` и `T*` из языка C, но нам не надо особенно заботиться о том, что C про них думает.
Разыменовать `*const T` можно только в `&T`, но как и в случае с изменяемостью переменных, это всего лишь способ защиты от некорректного использования.
В лучшем случае это значит, что сначала надо привести `*const` к `*mut`.
Хотя если у вас нет прав изменять объект, на который ссылается указатель, у вас возникнут проблемы.

> Anyway, we'll get a better feel for this as we write some code.
> For now, `*mut T == &unchecked mut T`!

В любом случае, мы лучше разберёмся в теме, написав немного кода.
А пока `*mut T` означает `&unchecked mut T`!

[nom]: https://doc.rust-lang.org/nightly/nomicon/
