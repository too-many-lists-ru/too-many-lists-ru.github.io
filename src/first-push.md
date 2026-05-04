> # Push

# Вставка

> So let's write pushing a value onto a list.
> `push` *mutates* the list, so we'll want to take `&mut self`.
> We also need to take an i32 to push:

Что ж, напишем функцию вставки значения в список.
Вставка (`push`) *изменяет* список, поэтому нам нужен параметр `&mut self`.
Кроме того, нам нужно значение `i32`, которое мы будем вставлять.

```rust ,ignore
impl List {
    pub fn push(&mut self, elem: i32) {
        // TODO
        // написать
    }
}
```

> First things first, we need to make a node to store our element in:

Прежде всего нам надо создать узел и сохранить в нём наш элемент:

```rust ,ignore
    pub fn push(&mut self, elem: i32) {
        let new_node = Node {
            elem: elem,
            next: ?????
        };
    }
```

> What goes `next`?
> Well, the entire old list!
> Can we... just do that?

Что должно быть дальше по ссылке `next`?
Да, наш старый список целиком!
Можем мы... просто так и написать?

```rust ,ignore
impl List {
    pub fn push(&mut self, elem: i32) {
        let new_node = Node {
            elem: elem,
            next: self.head,
        };
    }
}
```

```text
> cargo build
error[E0507]: cannot move out of borrowed content
  --> src/first.rs:19:19
   |
19 |             next: self.head,
   |                   ^^^^^^^^^ cannot move out of borrowed content
```

> Nooooope.
> Rust is telling us the right thing, but it's certainly not obvious what exactly it means, or what to do about it:

Неееееет.
Rust говорит нам правильную вещь, но наверное не очевидно, что это точно означает, и что мы должны с этим делать:

> > cannot move out of borrowed content

> нельзя переместить значение из заимствованного содержимого

> We're trying to move the `self.head` field out to `next`, but Rust doesn't want us doing that.
> This would leave `self` only partially initialized when we end the borrow and "give it back" to its rightful owner.
> As we said before, that's the *one* thing you can't do with an `&mut`:
> It would be super rude, and Rust is very polite (it would also be incredibly dangerous, but surely *that* isn't why it cares).

Мы пытаемся переместить поле `self.head` в `next`, но Rust не позволяет нам это сделать.
В этом случае `self` останется лишь частично инициализированным, когда мы завершим заимствование и «вернём назад» законному владельцу.
Как мы уже говорили, это *единственное*, что нельзя сделать с помощью `&mut`:
Это было бы крайне невежливо, а Rust очень вежлив (это также было бы крайне опасно, но конечно *это* не то, что его беспокоит).

> What if we put something back?
> Namely, the node that we're creating:

Что если мы поместим что-то вместо старого значения?
А именно узел, который мы создаём:


```rust ,ignore
pub fn push(&mut self, elem: i32) {
    let new_node = Box::new(Node {
        elem: elem,
        next: self.head,
    });

    self.head = Link::More(new_node);
}
```

```text
> cargo build
error[E0507]: cannot move out of borrowed content
  --> src/first.rs:19:19
   |
19 |             next: self.head,
   |                   ^^^^^^^^^ cannot move out of borrowed content
```

> No dice.
> In principle, this is something Rust could actually accept, but it won't (for various reasons -- the most serious being [exception safety][]).
> We need some way to get the head without Rust noticing that it's gone.
> For advice, we turn to infamous Rust Hacker Indiana Jones:

Ничего не вышло.
В принципе, это одна из тех штук, которые Rust мог бы принять, но он так не делает (по многим причинам, одна из которых — это [безопасность исключений][]).
Нам нужен способ забрать значение `head` так, чтобы Rust не заметил пропажи.
За советом мы обратимся к печально известному хакеру Rust Индиане Джонсу:

<!-- Речь идёт о сцене из фильма «Индиана Джонс: В поисках утраченного ковчега», в которой герой Форда быстро убирает золотого идола и кладёт на его место мешочек с песком. -->

![Indy Prepares to mem::replace](img/indy.gif)

> Ah yes, Indy suggests the `mem::replace` maneuver.
> This incredibly useful function lets us steal a value out of a borrow by *replacing* it with another value.
> Let's just pull in `std::mem` at the top of the file, so that `mem` is in local scope:

Ах, да, Инди советует прибегнуть к маневру `mem::replace`.
Эта невероятно полезная функция позволяет нам увести значение заимствованного объекта, *заменив* его другим значением.
Давайте просто добавим `std::mem` в начало файла, чтобы <!-- модуль --> `mem` оказался в локальной области видимости:

```rust ,ignore
use std::mem;
```

> and use it appropriately:

и используем его надлежащим образом:

```rust ,ignore
pub fn push(&mut self, elem: i32) {
    let new_node = Box::new(Node {
        elem: elem,
        next: mem::replace(&mut self.head, Link::Empty),
    });

    self.head = Link::More(new_node);
}
```

> Here we `replace` self.head temporarily with Link::Empty before replacing it with the new head of the list.
> I'm not gonna lie: this is a pretty unfortunate thing to have to do.
> Sadly, we must (for now).

Здесь мы временно заменяем (`replace`) `self.head` значением `Link::Empty` перед заменой его новой головой списка.
Не стану врать: это довольно неприятная штука, которую необходимо сделать.
К сожалению, мы должны (по крайней мере, сейчас).

> But hey, that's `push` all done!
> Probably.
> We should probably test it, honestly.
> Right now the easiest way to do that is probably to write `pop`, and make sure that it produces the right results.

Но, подождите, функция `push` уже готова!
Возможно.
Честно говоря, нам стоит это протестировать.
На данный момент простейший способ это сделать — возможно, написать `pop` и убедиться, что она возвращает правильные результаты.

[exception safety]: https://doc.rust-lang.org/nightly/nomicon/exception-safety.html
