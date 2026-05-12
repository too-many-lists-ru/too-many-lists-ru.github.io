# Использование Option

Внимательные читатели могли заметить, что мы, фактически, переизобрели не самую удачную версию Option:

```rust ,ignore
enum Link {
    Empty,
    More(Box<Node>),
}
```

Link — это просто `Option<Box<Node>>`.
Приятно, что нам не надо было везде писать `Option<Box<Node>>` и, в отличие от `pop`, не надо было делать реализацию, доступную всему внешнему миру, что, кажется, неплохо.
Однако, в Option есть несколько *поистине приятных* методов, которые нам пришлось написать самим.
Давайте *не* будем больше так делать и заменим всё на вызовы методов Option.
Для начала давайте без изысков просто переименуем все элементы в `Some` и `None`:

```rust ,ignore
use std::mem;

pub struct List {
    head: Link,
}

// да, псевдонимы типов!
type Link = Option<Box<Node>>;

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: mem::replace(&mut self.head, None),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<i32> {
        match mem::replace(&mut self.head, None) {
            None => None,
            Some(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = mem::replace(&mut self.head, None);
        while let Some(mut boxed_node) = cur_link {
            cur_link = mem::replace(&mut boxed_node.next, None);
        }
    }
}
```

Код не стал кардинально лучше, но основную выгоду нам принесут методы Option.

Во-первых, `mem::replace(&mut option, None)` — настолько распространённая идиома, что Option просто взял и превратил её в метод `take`.

```rust ,ignore
pub struct List {
    head: Link,
}

type Link = Option<Box<Node>>;

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: self.head.take(),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<i32> {
        match self.head.take() {
            None => None,
            Some(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

Во-вторых, `match option { None => None, Some(x) => Some(y) }` — настолько распространённая идиома, что её назвали `map` (отображение).
Метод `map` принимает функцию, которая берёт `x`, отдаёт `y`, а затем с её помощью превращает `Some(x)` в `Some(y)`.
Мы могли бы написать подходящую функции и передать её в `map`, но вместо этого мы *встроим* её в место вызова.

Для этого воспользуемся *замыканиями*.
Замыкания — это анонимные функции (то есть функции без имени) с дополнительной супер-способностью: им доступны локальные переменные *вне* замыкания!
Благодара этому они супер-удобны для условной логики любого рода.
В нашем коде `match` встречается в единственном месте — методе `pop`, так что давайте просто его перепишем:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    self.head.take().map(|node| {
        self.head = node.next;
        node.elem
    })
}
```

Так намного лучше.
Давайте убедимся, что мы ничего не сломали:

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 2 tests
test first::test::basics ... ok
test second::test::basics ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured

```

Великолепно!
Теперь от внешнего вида перейдём к улучшению *поведения*.