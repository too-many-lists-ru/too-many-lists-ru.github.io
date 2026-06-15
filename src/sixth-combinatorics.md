# Скучная комбинаторика

Ладно, возвращаемся к нашим ежедневным связным спискам!

Для начала разберёмся с `Drop`, который тривиально реализуется с помощью pop:

```rust ,ignore
impl<T> Drop for LinkedList<T> {
    fn drop(&mut self) {
        // Вызываем pop, пока есть элементы
        while let Some(_) = self.pop_front() { }
    }
}
```

Нам предстоит написать множество действительно скучных комбинаторных реализаций, таких как front, front_mut, back, back_mut, iter, iter_mut, into_iter, ...

Можно было бы воспользоваться макросами или чем то подобным, но, честно говоря, это хуже, чем обычная копи-паста.
Нам предстоит много копи-пасты.
Я *очень тщательно* подошла к реализации предыдущих push/pop.
Нам осталось *буквально* поменять местами front и back, и код будет работать!
Да здравствует болезненный опыт!
(Заманчиво говорить о «предыдущем и следующем» узлах, но я обнаружила, что термины «передний» и «задний» ведут к большей согласованности и позволяют избегать ошибок.)

Хорошо, начнём с `front`:

```rust ,ignore
pub fn front(&self) -> Option<&T> {
    unsafe {
        self.front.map(|node| &(*node.as_ptr()).elem)
    }
}
```

На самом деле эта книга очень старая, так что я решила добавить в код несколько приятных нововведений, скажем, оператор `?`, который приводит к раннему возврату в случае `Option::None`.
Получится улучшить код?

```rust ,ignore
pub fn front(&self) -> Option<&T> {
    unsafe {
        Some(&(*self.front?.as_ptr()).elem)
    }
}
```

Нормально?
В таком простом код практически ничего не улучшилось, а, кроме того, в предыдущем разделе мы говорили о том, что ранние возвраты для нас опасны.
Возможно, нам надо писать здесь чуть более явный код (мне всё ещё нравится старая реализация через `map`).
Теперь front_mut:

```rust ,ignore
pub fn front_mut(&mut self) -> Option<&mut T> {
    unsafe {
        self.front.map(|node| &mut (*node.as_ptr()).elem)
    }
}
```

Чуть позже я добавлю все версии `back`.

Далее, итераторы.
В отличие от наших прошлых списков, мы *наконец* можем реализовать [DoubleEndedIterator](https://doc.rust-lang.org/std/iter/trait.DoubleEndedIterator.html, и, поскольку мы вышли на продуктовый уровень — и [ExactSizeIterator](https://doc.rust-lang.org/std/iter/trait.ExactSizeIterator.html).

Таким образом, помимо `next` и `size_hint`, мы реализуем `next_back` и `len`.

Самые внимательные читатели могут заметить, что IterMut в случае итерации с двумя концами выглядит гораздо подозрительнее, но но самом деле он всё ещё работает!

...боже, будет тонна рутинного кода.
Возможно, мне следует написать макрос... нет, нет, это всё ещё плохая идея.

```rust ,ignore
pub struct Iter<'a, T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<&'a T>,
}

impl<T> LinkedList<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { 
            front: self.front, 
            back: self.back,
            len: self.len,
            _boo: PhantomData,
        }
    }
}

impl<'a, T> IntoIterator for &'a LinkedList<T> {
    type IntoIter = Iter<'a, T>;
    type Item = &'a T;

    fn into_iter(self) -> Self::IntoIter {
        self.iter()
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;
    
    fn next(&mut self) -> Option<Self::Item> {
        // Хотя условие self.front == self.back кажется очевидным,
        // оно не подходит для возврата последнего элемента! Подобный код
        // работает только с массивами из-за указателя «на элемент, следующий
        // за последним»
        if self.len > 0 {
            // Мы могли бы извлечь значение из front, но так быстрее и безопаснее
            self.front.map(|node| unsafe {
                self.len -= 1;
                self.front = (*node.as_ptr()).back;
                &(*node.as_ptr()).elem
            })
        } else {
            None
        }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.len, Some(self.len))
    }
}

impl<'a, T> DoubleEndedIterator for Iter<'a, T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        if self.len > 0 {
            self.back.map(|node| unsafe {
                self.len -= 1;
                self.back = (*node.as_ptr()).front;
                &(*node.as_ptr()).elem
            })
        } else {
            None
        }
    }
}

impl<'a, T> ExactSizeIterator for Iter<'a, T> {
    fn len(&self) -> usize {
        self.len
    }
}
```

...это всего лишь `.iter()`...

Позже мы добавим IterMut.
Будет буквально тот же код, но с большим количеством `mut`.
Для начала разберёмся с `into_iter`.
К счастью, мы всё ещё можем положиться на проверенное решение: завернуть нашу коллекцию и использовать pop для next:

```rust ,ignore
pub struct IntoIter<T> {
    list: LinkedList<T>,
}

impl<T> LinkedList<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter { 
            list: self
        }
    }
}


impl<T> IntoIterator for LinkedList<T> {
    type IntoIter = IntoIter<T>;
    type Item = T;

    fn into_iter(self) -> Self::IntoIter {
        self.into_iter()
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;

    fn next(&mut self) -> Option<Self::Item> {
        self.list.pop_front()
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.list.len, Some(self.list.len))
    }
}

impl<T> DoubleEndedIterator for IntoIter<T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        self.list.pop_back()
    }
}

impl<T> ExactSizeIterator for IntoIter<T> {
    fn len(&self) -> usize {
        self.list.len
    }
}
```

Всё ещё много рутинного кода, но, по крайней мере, это *удовлетворительный* рутинный код.

Хорошо, вот весь наш код со всеми комбинаторными вариантами:

```rust
use std::ptr::NonNull;
use std::marker::PhantomData;

pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<T>,
}

type Link<T> = Option<NonNull<Node<T>>>;

struct Node<T> {
    front: Link<T>,
    back: Link<T>,
    elem: T, 
}

pub struct Iter<'a, T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<&'a T>,
}

pub struct IterMut<'a, T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<&'a mut T>,
}

pub struct IntoIter<T> {
    list: LinkedList<T>,
}

impl<T> LinkedList<T> {
    pub fn new() -> Self {
        Self {
            front: None,
            back: None,
            len: 0,
            _boo: PhantomData,
        }
    }

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
                (*old.as_ptr()).front = Some(new);
                (*new.as_ptr()).back = Some(old);
            } else {
                // Если переднего узла нет, у нас пустой список и нам надо
                // установить и значение заднего узла.
                self.back = Some(new);
            }
            // Этот код выполняется всегда!
            self.front = Some(new);
            self.len += 1;
        }
    }

    pub fn push_back(&mut self, elem: T) {
        // БЕЗОПАСНОСТЬ: это связный список, что вы делаете?
        unsafe {
            let new = NonNull::new_unchecked(Box::into_raw(Box::new(Node {
                back: None,
                front: None,
                elem,
            })));
            if let Some(old) = self.back {
                // Вставляем новый задний узел перед старым
                (*old.as_ptr()).back = Some(new);
                (*new.as_ptr()).front = Some(old);
            } else {
                // Если заднего узла нет, у нас пустой список и нам надо
                // установить и значение переднего узла.
                self.front = Some(new);
            }
            // Этот код выполняется всегда!
            self.back = Some(new);
            self.len += 1;
        }
    }

    pub fn pop_front(&mut self) -> Option<T> {
        unsafe {
            // Будем что-то делать, только если у списка есть передний узел.
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
                    self.back = None;
                }

                self.len -= 1;
                result
                // Здесь Box неявным образом освобождается, потому что больше нет T.
            })
        }
    }

    pub fn pop_back(&mut self) -> Option<T> {
        unsafe {
            // Будем что-то делать, только если у списка есть задний узел.
            self.back.map(|node| {
                // Возвращаем к жизни Box, так что можно извлечь и уничтожить его
                // значение (Box магически делает это за нас).
                let boxed_node = Box::from_raw(node.as_ptr());
                let result = boxed_node.elem;

                // Делаем следующий узел новым задним узлом.
                self.back = boxed_node.front;
                if let Some(new) = self.back {
                    // Убираем ссылку заднего узла на удалённый узел.
                    (*new.as_ptr()).back = None;
                } else {
                    // Если задний узел равен null, значит, список пустой!
                    self.front = None;
                }

                self.len -= 1;
                result
                // Здесь Box неявным образом освобождается, потому что больше нет T.
            })
        }
    }

    pub fn front(&self) -> Option<&T> {
        unsafe {
            self.front.map(|node| &(*node.as_ptr()).elem)
        }
    }

    pub fn front_mut(&mut self) -> Option<&mut T> {
        unsafe {
            self.front.map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn back(&self) -> Option<&T> {
        unsafe {
            self.back.map(|node| &(*node.as_ptr()).elem)
        }
    }

    pub fn back_mut(&mut self) -> Option<&mut T> {
        unsafe {
            self.back.map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn len(&self) -> usize {
        self.len
    }

    pub fn iter(&self) -> Iter<T> {
        Iter { 
            front: self.front, 
            back: self.back,
            len: self.len,
            _boo: PhantomData,
        }
    }

    pub fn iter_mut(&mut self) -> IterMut<T> {
        IterMut { 
            front: self.front, 
            back: self.back,
            len: self.len,
            _boo: PhantomData,
        }
    }

    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter { 
            list: self
        }
    }
}

impl<T> Drop for LinkedList<T> {
    fn drop(&mut self) {
        // Вызываем pop, пока есть элементы
        while let Some(_) = self.pop_front() { }
    }
}

impl<'a, T> IntoIterator for &'a LinkedList<T> {
    type IntoIter = Iter<'a, T>;
    type Item = &'a T;

    fn into_iter(self) -> Self::IntoIter {
        self.iter()
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        // Хотя условие self.front == self.back кажется очевидным,
        // оно не подходит для возврата последнего элемента! Подобный код
        // работает только с массивами из-за указателя «на элемент, следующий
        // за последним»
        if self.len > 0 {
            // Мы могли бы извлечь значение из front, но так быстрее и безопаснее
            self.front.map(|node| unsafe {
                self.len -= 1;
                self.front = (*node.as_ptr()).back;
                &(*node.as_ptr()).elem
            })
        } else {
            None
        }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.len, Some(self.len))
    }
}

impl<'a, T> DoubleEndedIterator for Iter<'a, T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        if self.len > 0 {
            self.back.map(|node| unsafe {
                self.len -= 1;
                self.back = (*node.as_ptr()).front;
                &(*node.as_ptr()).elem
            })
        } else {
            None
        }
    }
}

impl<'a, T> ExactSizeIterator for Iter<'a, T> {
    fn len(&self) -> usize {
        self.len
    }
}

impl<'a, T> IntoIterator for &'a mut LinkedList<T> {
    type IntoIter = IterMut<'a, T>;
    type Item = &'a mut T;

    fn into_iter(self) -> Self::IntoIter {
        self.iter_mut()
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        // Хотя условие self.front == self.back кажется очевидным,
        // оно не подходит для возврата последнего элемента! Подобный код
        // работает только с массивами из-за указателя «на элемент, следующий
        // за последним»
        if self.len > 0 {
            // Мы могли бы извлечь значение из front, но так быстрее и безопаснее
            self.front.map(|node| unsafe {
                self.len -= 1;
                self.front = (*node.as_ptr()).back;
                &mut (*node.as_ptr()).elem
            })
        } else {
            None
        }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.len, Some(self.len))
    }
}

impl<'a, T> DoubleEndedIterator for IterMut<'a, T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        if self.len > 0 {
            self.back.map(|node| unsafe {
                self.len -= 1;
                self.back = (*node.as_ptr()).front;
                &mut (*node.as_ptr()).elem
            })
        } else {
            None
        }
    }
}

impl<'a, T> ExactSizeIterator for IterMut<'a, T> {
    fn len(&self) -> usize {
        self.len
    }
}

impl<T> IntoIterator for LinkedList<T> {
    type IntoIter = IntoIter<T>;
    type Item = T;

    fn into_iter(self) -> Self::IntoIter {
        self.into_iter()
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;

    fn next(&mut self) -> Option<Self::Item> {
        self.list.pop_front()
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.list.len, Some(self.list.len))
    }
}

impl<T> DoubleEndedIterator for IntoIter<T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        self.list.pop_back()
    }
}

impl<T> ExactSizeIterator for IntoIter<T> {
    fn len(&self) -> usize {
        self.list.len
    }
}


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
