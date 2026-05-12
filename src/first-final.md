# Финальный код

Итак, 6000 слов спустя, мы написали такой код:

```rust
use std::mem;

pub struct List {
    head: Link,
}

enum Link {
    Empty,
    More(Box<Node>),
}

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: Link::Empty }
    }

    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: mem::replace(&mut self.head, Link::Empty),
        });

        self.head = Link::More(new_node);
    }

    pub fn pop(&mut self) -> Option<i32> {
        match mem::replace(&mut self.head, Link::Empty) {
            Link::Empty => None,
            Link::More(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = mem::replace(&mut self.head, Link::Empty);

        while let Link::More(mut boxed_node) = cur_link {
            cur_link = mem::replace(&mut boxed_node.next, Link::Empty);
        }
    }
}

#[cfg(test)]
mod test {
    use super::List;

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

Только посмотрите!
80 строк, из них половина — тесты!
Ну, я предупреждала, что первая глава потребует времени!