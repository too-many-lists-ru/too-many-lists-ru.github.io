> # The Double Singly-Linked List

# Двойной односвязный список

> We struggled with doubly-linked lists because they have tangled ownership semantics: no node strictly owns any other node.
> However we struggled with this because we brought in our preconceived notions of what a linked list *is*.
> Namely, we assumed that all the links go in the same direction.

У нас возникли трудности при работе с двусвязными списками, потому что у них сложная семантика владения: строго говоря, ни один узел не владеет никаким другим узлом.
Однако эти трудности вызваны, скорее, нашим ограниченным представлением о том, чем связные списки *являются*.
А именно, мы считаем, что все ссылки должны идти в одном и том же направлении.

> Instead, we can smash our list into two halves: one going to the left, and one going to the right:

Вместо этого мы можем разбить наш список на две половины: одна направлено влево, а вторая вправо:

```rust ,ignore
// lib.rs
// ...
pub mod silly1;     // НОВЫЙ!
```

```rust ,ignore
// silly1.rs
use crate::second::List as Stack;

struct List<T> {
    left: Stack<T>,
    right: Stack<T>,
}
```

> Now, rather than having a mere safe stack, we have a general purpose list.
> We can grow the list leftwards or rightwards by pushing onto either stack.
> We can also "walk" along the list by popping values off one end and onto the other.
> To avoid needless allocations, we're going to copy the source of our safe Stack to get access to its private details:

Теперь вместо обычного безопасного стека мы получили список общего назначения.
Мы можем расширять список влево или вправо, добавляя значения в любой из стеков.
Мы также можно «перемещаться» по списку, извлекая значения с одного конца и вставляя их в другой.
Чтобы избежать ненужного выделения памяти, мы скопируем исходный код нашего безопасного стека, чтобы получить доступ к его приватным полям:


```rust ,ignore
pub struct Stack<T> {
    head: Link<T>,
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<T> Stack<T> {
    pub fn new() -> Self {
        Stack { head: None }
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
            let node = *node;
            self.head = node.next;
            node.elem
        })
    }

    pub fn peek(&self) -> Option<&T> {
        self.head.as_ref().map(|node| {
            &node.elem
        })
    }

    pub fn peek_mut(&mut self) -> Option<&mut T> {
        self.head.as_mut().map(|node| {
            &mut node.elem
        })
    }
}

impl<T> Drop for Stack<T> {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

> And just rework `push` and `pop` a bit:

И немного переработаем `push` и `pop`:

```rust ,ignore
pub fn push(&mut self, elem: T) {
    let new_node = Box::new(Node {
        elem: elem,
        next: None,
    });

    self.push_node(new_node);
}

fn push_node(&mut self, mut node: Box<Node<T>>) {
    node.next = self.head.take();
    self.head = Some(node);
}

pub fn pop(&mut self) -> Option<T> {
    self.pop_node().map(|node| {
        node.elem
    })
}

fn pop_node(&mut self) -> Option<Box<Node<T>>> {
    self.head.take().map(|mut node| {
        self.head = node.next.take();
        node
    })
}
```

Теперь мы можем создать наш список:

```rust ,ignore
pub struct List<T> {
    left: Stack<T>,
    right: Stack<T>,
}

impl<T> List<T> {
    fn new() -> Self {
        List { left: Stack::new(), right: Stack::new() }
    }
}
```

> And we can do the usual stuff:

И мы можем выполнять над ним обычные операции:

```rust ,ignore
pub fn push_left(&mut self, elem: T) { self.left.push(elem) }
pub fn push_right(&mut self, elem: T) { self.right.push(elem) }
pub fn pop_left(&mut self) -> Option<T> { self.left.pop() }
pub fn pop_right(&mut self) -> Option<T> { self.right.pop() }
pub fn peek_left(&self) -> Option<&T> { self.left.peek() }
pub fn peek_right(&self) -> Option<&T> { self.right.peek() }
pub fn peek_left_mut(&mut self) -> Option<&mut T> { self.left.peek_mut() }
pub fn peek_right_mut(&mut self) -> Option<&mut T> { self.right.peek_mut() }
```

> But most interestingly, we can walk around!

Но, что гораздо интереснее, мы можем обойти его кругом!


```rust ,ignore
pub fn go_left(&mut self) -> bool {
    self.left.pop_node().map(|node| {
        self.right.push_node(node);
    }).is_some()
}

pub fn go_right(&mut self) -> bool {
    self.right.pop_node().map(|node| {
        self.left.push_node(node);
    }).is_some()
}
```

> We return booleans here as just a convenience to indicate whether we actually managed to move.
> Now let's test this baby out:

Здесь мы возвращаем булевы значения просто для удобства, чтобы показать, действительно ли нам удалось переместиться по списку.
Протестируем:

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn walk_aboot() {
        let mut list = List::new();             // [_]

        list.push_left(0);                      // [0,_]
        list.push_right(1);                     // [0, _, 1]
        assert_eq!(list.peek_left(), Some(&0));
        assert_eq!(list.peek_right(), Some(&1));

        list.push_left(2);                      // [0, 2, _, 1]
        list.push_left(3);                      // [0, 2, 3, _, 1]
        list.push_right(4);                     // [0, 2, 3, _, 4, 1]

        while list.go_left() {}                 // [_, 0, 2, 3, 4, 1]

        assert_eq!(list.pop_left(), None);
        assert_eq!(list.pop_right(), Some(0));  // [_, 2, 3, 4, 1]
        assert_eq!(list.pop_right(), Some(2));  // [_, 3, 4, 1]

        list.push_left(5);                      // [5, _, 3, 4, 1]
        assert_eq!(list.pop_right(), Some(3));  // [5, _, 4, 1]
        assert_eq!(list.pop_left(), Some(5));   // [_, 4, 1]
        assert_eq!(list.pop_right(), Some(4));  // [_, 1]
        assert_eq!(list.pop_right(), Some(1));  // [_]

        assert_eq!(list.pop_right(), None);
        assert_eq!(list.pop_left(), None);

    }
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 16 tests
test fifth::test::into_iter ... ok
test fifth::test::basics ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test fourth::test::into_iter ... ok
test fourth::test::basics ... ok
test fourth::test::peek ... ok
test first::test::basics ... ok
test second::test::into_iter ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test third::test::basics ... ok
test third::test::iter ... ok
test second::test::peek ... ok
test silly1::test::walk_aboot ... ok

test result: ok. 16 passed; 0 failed; 0 ignored; 0 measured
```

> This is an extreme example of a *finger* data structure, where we maintain some kind of finger into the structure, and as a consequence can support operations on locations in time proportional to the distance from the finger.

Это экстремальный пример структуры данных с *указателем* (*finger* data structure), в которой используется своего рода указатель, благодаря чему можно выполнять операции над элементами за время, пропорциональное расстоянию до указателя.

> We can make very fast changes to the list around our finger, but if we want to make changes far away from our finger we have to walk all the way over there.
> We can permanently walk over there by shifting the elements from one stack to the other, or we could just walk along the links with an `&mut` temporarily to do the changes.
> However the `&mut` can never go back up the list, while our finger can!

Можно очень быстро вносить изменения в список рядом с указателем, но если нам надо поправить список где-то далеко от указателя, нам придётся туда добраться.
Мы можем либо идти по списку, перекладывая элементы из одного стека в другой, либо можем двигаться с помощью временного изменяемого итератора.
Правда, итератор не позволит нам вернуться назад, в то время как указатель позволит!
