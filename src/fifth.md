> # An Ok Unsafe Singly-Linked Queue

# Хорошая безопасная односвязная очередь

> Ok that reference-counted interior mutability stuff got a little out of control.
> Surely Rust doesn't really expect you to do that sort of thing in general?
> Well, yes and no.
> Rc and Refcell can be great for handling simple cases, but they can get unwieldy.
> Especially if you want to hide that it's happening.
> There's gotta be a better way!

Ну, ладно, штука с внутренней изменчивостью счётчика ссылок немного вышла из под контроля.
Конечно, вряд ли Rust действительно ожидает, что вы будете делать вещи такого рода в целом?
Ну, и да, и нет.
Rc и RefCell могут прекрасно подходить для обработки простых случаев, но они могут стать неуклюжими <!-- или громоздкими -->.
Особенно, если вы пытаетесь скрыть наличие проблемы <!-- скрыть эту неуклюжесть -->.
Должен быть лучший способ!

> In this chapter we're going to roll back to singly-linked lists and implement a singly-linked queue to dip our toes into *raw pointers* and *Unsafe Rust*.

В этой главые мы собираемся вернуться к односвязным спискам и реализовать односвязную очередь, попробовав себя в *сырых указателях* и *небезопасном Rust*.

<!-- to dip our toes into... -- опустить пальцы ног в..., означает испытать-испробовать что-то новое в какое-то области  -->

> > **NARRATOR:** And I will point out the mistakes.

> **ГОЛОС ЗА КАДРОМ:** А я *укажу* на ошибки.

> And we won't make *any* mistakes.

И мы не станем делать *любые* ошибки. <!-- не собираемся совершать -->

> Let's add a new file called `fifth.rs`:

Давайте добавим новый файл с именем `fifth.rs`:

```rust ,ignore
// in lib.rs
// в lib.rs

pub mod first;
pub mod second;
pub mod third;
pub mod fourth;
pub mod fifth;
```

> Our code is largely going to be derived from second.rs, since a queue is mostly an augmentation of a stack in the world of linked lists.
> Still, we're going to go from scratch because there's some fundamental issues we want to address with layout and what-not.

Наш код в большой степени будет основан на second.rs, так как очереди в каком-то смысле являются расширенным стеком в мире связных списков.
Кроме того, мы мы собираемся начать с чистого листа поскольку есть несколько фундаментальных задач, которые мы хотим адресовать представлению и другим подобным вопросам.

<!-- what-not == всякая всячина, и тому подобное, и прочее -->