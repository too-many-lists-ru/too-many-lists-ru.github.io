# Всё обобщаем

Мы уже немного затронули тему обобщений при обсуждении Option и Box.
Однако, пока мы избегали объявления нового типа, который бы обобщал произвольные элементы.

На самом деле это очень просто.
Давайте прямо сейчас сделаем наши типы обобщёнными:

```rust ,ignore
pub struct List<T> {
    head: Link<T>,
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

Вы добавили несколько угловых скобок — и внезапно ваш код стал универсальным.
Естественно, не всё так *просто*.
Компилятор начал Очень Сильно Ругался.

```text
> cargo test

error[E0107]: wrong number of type arguments: expected 1, found 0
  --> src/second.rs:14:6
   |
14 | impl List {
   |      ^^^^ expected 1 type argument

error[E0107]: wrong number of type arguments: expected 1, found 0
  --> src/second.rs:36:15
   |
36 | impl Drop for List {
   |               ^^^^ expected 1 type argument

```

Проблема довольно прозрачна: мы везде пишем `List`, но это неправильно.
Как и в случае с Option, и Box, теперь мы должны писать `List<Something>`, то есть «Список чего-то».

Но что такое это «что-то», которое мы используем во всех реализациях?
Как и в случае с List, мы хотим, чтобы реализация работала со *всеми* T.
Поэтому, как и в случае с List, давайте добавим угловые скобки к `impl`:

```rust ,ignore
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_node = Box::new(Node {
            elem: elem,
            next: self.head.take(),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<T> {
        self.head.take().map(|node| {
            self.head = node.next;
            node.elem
        })
    }
}

impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

...и это всё!


```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 2 tests
test first::test::basics ... ok
test second::test::basics ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured

```

Весь наш код теперь полностью обобщён для произвольных T.
Блин, Rust реально *простой*.
Особо хочу отметить метод `new`, который даже не изменился.

```rust ,ignore
pub fn new() -> Self {
    List { head: None }
}
```

Купайтесь в лучах Славы, дарованной вам Self, хранителем рефакторинга и кодирования методом копи-пасты.
Кстати, интересно, что мы не пишем `List<T>` при создании экземпляра списка.
Тип выводится автоматически, на основании того факта, что мы возвращаем значение из функции, которая имеет тип `List<T>`.

Хорошо, теперь перейдём к реализации нового *поведения*!