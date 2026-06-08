> # Basics

# Основы

> Alright, this is the part of the book that sucks, and why it took me 7 years to write this chapter!
> Time to just burn through a whole lot of really boring stuff we've done 5 times already, but extra verbose and long because we have to do everything twice and with `Option<NonNull<Node<T>>>`!

Ладно, вот самая ужасная часть книги, вот почему мне потребовалось 7 лет, чтобы написать эту главу!
Настало время ещё раз сделать уйму поистине скучных вещей, которые мы делали уже 5 раз, но теперь они стали ещё многословнее и длиннее, потому что нам предстоит делать всё дважды со всеми этими `Option<NonNull<Node<T>>>`!

```rust ,ignore
impl<T> LinkedList<T> {
    pub fn new() -> Self {
        Self {
            front: None,
            back: None,
            len: 0,
            _boo: PhantomData,
        }
    }
}
```

> PhantomData is a weird type with no fields so you just make one by, saying its type name.
> *shrug*

PhantomData — это странный тип без полей, так что его можно создать, просто указав его имя.
*пожимает плечами*

```rust ,ignore
pub fn push_front(&mut self, elem: T) {
    // SAFETY: it's a linked-list, what do you want?
    // БЕЗОПАСНОСТЬ: это связный список, что вы делаете?
    unsafe {
        let new = NonNull::new_unchecked(Box::into_raw(Box::new(Node {
            front: None,
            back: None,
            elem,
        })));
        if let Some(old) = self.front {
            // Put the new front before the old one
            // Вставляем новую голову перед старой
            (*old).front = Some(new);
            (*new).back = Some(old);
        } else {
            // If there's no front, then we're the empty list and need 
            // to set the back too. Also here's some integrity checks
            // for testing, in case we mess up.
            // Если головы нет, тогда у нас пустой список и нам надо
            // установить также и значение хвоста. Также здесь у нас
            // есть несколько проверок для тестирования, на случай,
            // если мы что-то напутали.
            debug_assert!(self.back.is_none());
            debug_assert!(self.front.is_none());
            debug_assert!(self.len == 0);
            self.back = Some(new);
        }
        self.front = Some(new);
        self.len += 1;
    }
}
```

```text
error[E0614]: type `NonNull<Node<T>>` cannot be dereferenced
  --> src\lib.rs:39:17
   |
39 |                 (*old).front = Some(new);
   |                 ^^^^^^
```


> Ah yes, I truly hate my pointer-y children.
> We need to explicitly get the raw pointer out of NonNull with `as_ptr`, because DerefMut is defined in terms of `&mut` and we don't want to randomly introduce safe references into our unsafe code!

О, да, именно поэтому я ненавижу своих детишек, напичканных указателями.
Нам нужно явно извлечь сырой указатель из NotNull с помощью `as_ptr`, потому что DerefMut определён через `&mut`, а мы не хотим произвольным образом вводить безопасные ссылки в наш небезопасный код!

```rust ,ignore
            (*old.as_ptr()).front = Some(new);
            (*new.as_ptr()).back = Some(old);
```

```text
   Compiling linked-list v0.0.3
warning: field is never read: `elem`
  --> src\lib.rs:16:5
   |
16 |     elem: T,
   |     ^^^^^^^
   |
   = note: `#[warn(dead_code)]` on by default

warning: `linked-list` (lib) generated 1 warning (1 duplicate)
warning: `linked-list` (lib test) generated 1 warning
    Finished test [unoptimized + debuginfo] target(s) in 0.33s
```

> Nice, now for pop (and len):

Прекрасно, теперь pop (и len):

```rust ,ignore
pub fn pop_front(&mut self) -> Option<T> {
    unsafe {
        // Only have to do stuff if there is a front node to pop.
        // Note that we don't need to mess around with `take` anymore
        // because everything is Copy and there are no dtors that will
        // run if we mess up... right? :) Riiiight? :)))
        // Должны что-то делать, только если у списка есть голова.
        // Обратите внимание, что мы больше не должны беспокоиться
        // о `take`, потому что всё теперь Copy и нет деструкторов,
        // которые будут запущены, если мы что-то напутаем... правда? :)
        // Праааавда? :)))
        self.front.map(|node| {
            // Bring the Box back to life so we can move out its value and
            // Drop it (Box continues to magically understand this for us).
            // Возвращаем к жизни Box, так что мы можем извлечь его значение
            // и уничтожить его (Box магическим образом делает это за нас).
            let boxed_node = Box::from_raw(node.as_ptr());
            let result = boxed_node.elem;

            // Make the next node into the new front.
            // Делаем следующий узел новой головой.
            self.front = boxed_node.back;
            if let Some(new) = self.front {
                // Cleanup its reference to the removed node
                // Убираем её ссылку на удалённый узел.
                (*new.as_ptr()).front = None;
            } else {
                // If the front is now null, then this list is now empty!
                // Если голова теперь равна null, значит, список теперь пустой!
                debug_assert!(self.len == 1);
                self.back = None;
            }

            self.len -= 1;
            result
            // Box gets implicitly freed here, knows there is no T.
            // Здесь Box неявным образом освобождается, потому что больше нет T.
        })
    }
}

pub fn len(&self) -> usize {
    self.len
}
```

```text
   Compiling linked-list v0.0.3
    Finished dev [unoptimized + debuginfo] target(s) in 0.37s
```

> Seems legit to me, time to write a test!

Мне кажется, всё в порядке, пора писать тест!

```rust ,ignore
#[cfg(test)]
mod test {
    use super::LinkedList;

    #[test]
    fn test_basic_front() {
        let mut list = LinkedList::new();

        // Try to break an empty list
        // Пытаемся сломать пустой список
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);

        // Try to break a one item list
        // Пытаемся сломать список из одного элемента
        list.push_front(10);
        assert_eq!(list.len(), 1);
        assert_eq!(list.pop_front(), Some(10));
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);

        // Mess around
        // Всё перемешиваем
        list.push_front(10);
        assert_eq!(list.len(), 1);
        list.push_front(20);
        assert_eq!(list.len(), 2);
        list.push_front(30);
        assert_eq!(list.len(), 3);
        assert_eq!(list.pop_front(), Some(30));
        assert_eq!(list.len(), 2);
        list.push_front(40);
        assert_eq!(list.len(), 3);
        assert_eq!(list.pop_front(), Some(40));
        assert_eq!(list.len(), 2);
        assert_eq!(list.pop_front(), Some(20));
        assert_eq!(list.len(), 1);
        assert_eq!(list.pop_front(), Some(10));
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);
    }
}
```


```text
   Compiling linked-list v0.0.3
    Finished test [unoptimized + debuginfo] target(s) in 0.40s
     Running unittests src\lib.rs

running 1 test
test test::test_basic_front ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> Hooray, we're perfect!

Ура, у нас всё идеально!

> ...Right?

...Правда?