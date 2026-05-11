> # A Bad but Safe Doubly-Linked Deque

# Плохой, но безопасный двусвязный дек

> Now that we've seen Rc and heard about interior mutability, this gives an interesting thought... maybe we *can* mutate through an Rc.
> And if *that's* the case, maybe we can implement a *doubly-linked* list totally safely!

Теперь, когда мы познакомились с Rc и с внутренней изменчивостью, появляется интересная мысль... может быть мы *можем* изменять объекты с помощью Rc.
И если *это так*, может быть мы можем сделать *двусвязный* список совершенно безопасным!

> In the process we'll become familiar with *interior mutability*, and probably learn the hard way that safe doesn't mean *correct*.
> Doubly-linked lists are hard, and I always make a mistake somewhere.

В процессе мы познакомимся с *внутренней изменчивостью* и, возможно, на горьком опыте убедимся, что безопасность не означает *корректность*.

> Let's add a new file called `fourth.rs`:

Создадим новый файл `fourth.rs`:

```rust ,ignore
// in lib.rs
// и lib.rs

pub mod first;
pub mod second;
pub mod third;
pub mod fourth;
```

> This will be another clean-room operation, though as usual we'll probably find some logic that applies verbatim again.

В этот раз нам снова предстоит начать работу с чистым файлом, хотя, как обычно мы, возможно найдём какую-то логику, которую сможем перенести без изменений.
<!-- verbatim = слово в слово -->

> Disclaimer: this chapter is basically a demonstration that this is a very bad idea.

Предупреждение: эта глава, по сути, демонстрирует, что у нас возникла очень плохая идея.