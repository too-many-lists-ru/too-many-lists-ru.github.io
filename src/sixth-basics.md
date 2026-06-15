# Основы

Ладно, самая ужасная часть книги.
Вот почему мне потребовалось 7 лет, чтобы написать эту главу!
Настало время ещё раз написать множеству скучнейших функций, которые мы писали уже 5 раз, но теперь они стали ещё многословнее и длиннее, потому теперь у нас два указателя формата `Option<NonNull<Node<T>>>`!

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

PhantomData — тип без полей и его можно создать, просто написав его имя.
*пожимает плечами*

```rust ,ignore
pub fn push_front(&mut self, elem: T) {
    // БЕЗОПАСНОСТЬ: это связный список, что вы делаете?
    unsafe {
        let new = NonNull::new_unchecked(Box::into_raw(Box::new(Node {
            front: None,
            back: None,
            elem,
        })));
        if let Some(old) = self.front {
            // Вставляем новый передний узел перед старым
            (*old).front = Some(new);
            (*new).back = Some(old);
        } else {
            // Если переднего узла нет, у нас пустой список и нам надо
            // установить и значение заднего узла. Кроме того, мы добавили
            // несколько проверок, на случай, если что-то напутали.
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


Ах, да, именно поэтому я ненавижу своих детей, напичканных указателями.
Нам нужно явно извлечь сырой указатель из NotNull с помощью `as_ptr`, потому что DerefMut определён через `&mut`, а мы не хотим вводить безопасные ссылки в наш небезопасный код!

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

Прекрасно, теперь pop (и len):

```rust ,ignore
pub fn pop_front(&mut self) -> Option<T> {
    unsafe {
        // Будем что-то делать, только если у списка есть передний узел.
        // Обратите внимание, что мы больше не должны беспокоиться
        // о `take`, потому что копирование сырых указателей не приводит к
        // вызову деструкторов, если мы что-то напутаем... правда? :)
        // Праааавда? :)))
        self.front.map(|node| {
            // Возвращаем к жизни Box, так что можно извлечь и уничтожить его
            // значение (Box магически делает это за нас).
            let boxed_node = Box::from_raw(node.as_ptr());
            let result = boxed_node.elem;

            // Делаем следующий узел новым передним узлом.
            self.front = boxed_node.back;
            if let Some(new) = self.front {
                // Убираем ссылку переднего узла на удалённый узел.
                (*new.as_ptr()).front = None;
            } else {
                // Если передний узел равен null, значит, список пустой!
                debug_assert!(self.len == 1);
                self.back = None;
            }

            self.len -= 1;
            result
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

Мне кажется, всё в порядке, пора писать тест!

```rust ,ignore
#[cfg(test)]
mod test {
    use super::LinkedList;

    #[test]
    fn test_basic_front() {
        let mut list = LinkedList::new();

        // Пытаемся сломать пустой список
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);

        // Пытаемся сломать список из одного элемента
        list.push_front(10);
        assert_eq!(list.len(), 1);
        assert_eq!(list.pop_front(), Some(10));
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);

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

Ура, у нас всё идеально!

...Правда?
