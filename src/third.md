> # A Persistent Singly-Linked Stack

# Устойчивый односвязный стек

> Alright, we've mastered the art of mutable singly-linked stacks.

Прекрасно, мы овладели искусством мутебельных односвязных стеков.

> Let's move from *single* ownership to *shared* ownership by writing a *persistent* immutable singly-linked list.
> This will be exactly the list that functional programmers have come to know and love.
> You can get the head *or* the tail and put someone's head on someone else's tail... and... that's basically it.
> Immutability is a hell of a drug.

Давайте перейдём с *единоличного* владения к *разделямому* владению путём написания *устойчивого* иммутабельного односвязного списка.
Это именно такой список, который знают и любят функциональные программисты. <!-- узнали и полюбили, но, кажется, наст. время лучше звучит. -->
Вы можете получить голову *или* хвост и поместить чью-то голову на чей-то ещё хвост... и... в принципе, это всё.
Иммутабельность — охренительный наркотик.

> In the process we'll largely just become familiar with Rc and Arc, but this will set us up for the next list which will *change the game*.

В процессе мы, в основном, будем знакомиться с Rc и Arc, но это подготовит нас к следующему списоку, который *изменит игру*. <!-- изменит правила игры -->

> Let's add a new file called `third.rs`:

Создадим новый файл с именем `third.rs`:

```rust ,ignore
// in lib.rs
// в lib.rs

pub mod first;
pub mod second;
pub mod third;
```

> No copy-pasta this time.
> This is a clean room operation.

Пока никакой копи-пасты.
Начинаем с чистого листа.