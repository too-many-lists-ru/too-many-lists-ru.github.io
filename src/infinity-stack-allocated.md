# Связный список, размещённый на стеке

Эта книга посвящена связным спискам, *размещаемым в куче*, поскольку они и распространены, и практичны.
Но не *обязательно* выделять память в куче, хотя это удобно, поскольку позволяет выделять память динамически.
В этом смысле выделение памяти в стеке менее дружелюбно: такие вещи, как `alloca` из C достаточно Ужасны и Проблемны.

Будем выделять память в стеке простым способом: вызвав функцию и получая новый фрейм стека для нового узла!
Это очень простое решение нашей проблемы, в то же время практичное и полезное.
Многие так делают, даже не подозревая, что реализуют связный список!

Всякий раз, используя рекурсию, вы можете передать указатель на состояние текущего шага в следующий шаг.
Если этот указатель сам является *частью* состояния, вы получаете связный список, выделенный на стеке!

Теперь, когда мы перешли к *нелепой* части книги, мы будем программировать нелепым способом: связный список будет главной звездой а весь пользовательский код окажется среди функций обратного вызова (callback)!
Все любят вложенные функции обратного вызова!

Наш тип списка будет обычным узлом со ссылкой на другой узел.

```rust
pub struct List<'a, T> {
    pub data: T,
    pub prev: Option<&'a List<'a, T>>,
}
```

И у нас будет всего одна операция, `push`, принимающая в параметрах старый список, состояние текущего узла и функцию обратного вызова.
Новый список будет передан в функцию обратного вызова.
Функция обратного вызова может вернуть любое значение, которое затем вернёт и метод `push`:

```rust ,ignore
impl<'a, T> List<'a, T> {
    pub fn push<U>(
        prev: Option<&'a List<'a, T>>, 
        data: T, 
        callback: impl FnOnce(&List<'a, T>) -> U,
    ) -> U {
        let list = List { data, prev };
        callback(&list)
    }
}
```

Вот и всё!
Использовать этот код можно так:

```rust ,ignore
List::push(None, 3, |list| {
    println!("{}", list.data);
    List::push(Some(list), 5, |list| {
        println!("{}", list.data);
        List::push(Some(list), 13, |list| {
            println!("{}", list.data);
        })
    })
})
```

Красиво. 😿

Пользователь может перемещаться по списку, используя конструкцию while-let и извлекая значения `prev`, но, веселья ради, давайте реализуем наш обычный итератор:

```rust ,ignore
impl<'a, T> List<'a, T> {
    pub fn iter(&'a self) -> Iter<'a, T> {
        Iter { next: Some(self) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.prev;
            &node.data
        })
    }
}
```

Протестируем:

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn elegance() {
        List::push(None, 3, |list| {
            assert_eq!(list.iter().copied().sum::<i32>(), 3);
            List::push(Some(list), 5, |list| {
                assert_eq!(list.iter().copied().sum::<i32>(), 5 + 3);
                List::push(Some(list), 13, |list| {
                    assert_eq!(list.iter().copied().sum::<i32>(), 13 + 5 + 3);
                })
            })
        })
    }
}
```

```text
> cargo test

running 18 tests
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test fifth::test::basics ... ok
test fifth::test::miri_food ... ok
test first::test::basics ... ok
test second::test::into_iter ... ok
test fourth::test::peek ... ok
test fourth::test::into_iter ... ok
test second::test::iter_mut ... ok
test fourth::test::basics ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test third::test::basics ... ok
test silly1::test::walk_aboot ... ok
test silly2::test::elegance ... ok
test second::test::peek ... ok
test third::test::iter ... ok

test result: ok. 18 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out;
```

Сейчас, вероятно, вы задаётесь вопросом, можно ли менять данные внутри узла?
Давайте проверим.
Будем хранить в списке изменяемые ссылки вместо разделяемых:


```rust
pub struct List<'a, T> {
    pub data: T,
    pub prev: Option<&'a mut List<'a, T>>,
}

pub struct Iter<'a, T> {
    next: Option<&'a List<'a, T>>,
}

impl<'a, T> List<'a, T> {
    pub fn push<U>(
        prev: Option<&'a mut List<'a, T>>, 
        data: T, 
        callback: impl FnOnce(&mut List<'a, T>) -> U,
    ) -> U {
        let mut list = List { data, prev };
        callback(&mut list)
    }

    pub fn iter(&'a self) -> Iter<'a, T> {
        Iter { next: Some(self) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.prev.as_ref().map(|prev| &**prev);
            &node.data
        })
    }
}

```


```text
> cargo test

error[E0521]: borrowed data escapes outside of closure
  --> src\silly2.rs:47:32
   |
46 |  List::push(Some(list), 13, |list| {
   |                              ----
   |                              |
   |              `list` declared here, outside of the closure body
   |              `list` is a reference that is only valid in the closure body
47 |      assert_eq!(list.iter().copied().sum::<i32>(), 13 + 5 + 3);
   |                 ^^^^^^^^^^^ `list` escapes the closure body here

error[E0521]: borrowed data escapes outside of closure
  --> src\silly2.rs:45:28
   |
44 |  List::push(Some(list), 5, |list| {
   |                             ----
   |                             |
   |              `list` declared here, outside of the closure body
   |              `list` is a reference that is only valid in the closure body
45 |      assert_eq!(list.iter().copied().sum::<i32>(), 5 + 3);
   |                 ^^^^^^^^^^^ `list` escapes the closure body here


<до бесконечности>
```

Что ж.
Кажется, компилятору не понравился наш итератор.
Возможно, мы что-то упустили?
Попробуем упростить наш тест:


```rust ,ignore
#[test]
fn elegance() {
    List::push(None, 3, |list| {
        assert_eq!(list.data, 3);
        List::push(Some(list), 5, |list| {
            assert_eq!(list.data, 5);
            List::push(Some(list), 13, |list| {
                assert_eq!(list.data, 13);
            })
        })
    })
}
```

```text
> cargo test

error[E0521]: borrowed data escapes outside of closure
  --> src\silly2.rs:46:17
   |
44 |   List::push(Some(list), 5, |list| {
   |                              ----
   |                              |
   |              `list` declared here, outside of the closure body
   |              `list` is a reference that is only valid in the closure body
45 |       assert_eq!(list.data, 5);
46 | /     List::push(Some(list), 13, |list| {
47 | |         assert_eq!(list.data, 13);
48 | |     })
   | |______^ `list` escapes the closure body here

error[E0521]: borrowed data escapes outside of closure
  --> src\silly2.rs:44:13
   |
42 |   List::push(None, 3, |list| {
   |                        ----
   |                        |
   |              `list` declared here, outside of the closure body
   |              `list` is a reference that is only valid in the closure body
43 |       assert_eq!(list.data, 3);
44 | /     List::push(Some(list), 5, |list| {
45 | |         assert_eq!(list.data, 5);
46 | |         List::push(Some(list), 13, |list| {
47 | |             assert_eq!(list.data, 13);
48 | |         })
49 | |     })
   | |______________^ `list` escapes the closure body here
```

Хмм, нет, это всё ещё ерунда.

Проблема в том, что наш список случайно (😉) полагается на *вариантность*.
[Вариантность — сложная тема](https://doc.rust-lang.org/nomicon/subtyping.html), но давайте рассмотрим её в упрощённом виде:

Каждый список содержит ссылку на список с *тем же самым типом, что и у него*.
С точки зрения самого внутреннего списка, у всех списков то же самое время жизни, что и у него, но это *объективно* неверно: каждый предыдущий узел в списке живёт строго дольше следующего, поскольку их области видимости буквально вложены!

Так... почему всё компилировалось, когда мы использовали разделяемые ссылки?
Потому что в большинстве случаев компилятор знает, что «долгожители» безопасны!
Когда мы помещаем ссылку на список в следующий список, компилятор втихую «урезает» время жизни, чтобы оно совпадало с ожиданиями нового списка.
Это урезание времени жизни и создаёт *вариантность*.

Тот же самый трюк, как в языках с наследованием, где вы можете передать Cat туда, где ожидается Animal (базовый тип Cat).
Интуитивно мы понимаем, что передавать Cat, когда ожидается Animal — это нормально, потому что Animal включает Cat и *что-то ещё*.
*Нормально* временно забыть про *что-то ещё*, так ведь?

Точно также, меньшее время жизни включает большее время жизни и *что-то ещё*.
Нормально и здесь забыть про *что-то ещё*!

Теперь вы наверняка удивляетесь, почему не работает версия с изменяемой ссылкой?

Ну, потому что вариантность *не всегда* безопасна.
Если бы наш код *компилировался*, мы могли бы реализовать использование-после-освобождения (use-after-free), например, вот так:

```rust ,ignore
List::push(None, 3, |list| {
    List::push(Some(list), 5, |list| {
        List::push(Some(list), 13, |list| {
            // ХАХАХА все временя жизни одинаковы, так что компилятор
            // позволит мне поменять моего родителя, чтобы оставить
            // мутабельную ссылку на меня!
            // Я создам объект, который можно использовать-после-освобождения!!!
            *list.prev.as_mut().unwrap().prev = Some(list);
        })
    })
})
```

Главная беда забытых мелочей в том, что *где-то помнят про них и рассчитывают на них*.
*Изменяемые данные* остаются очень большой проблемой.
Если вы не проявите осторожность, код, который не помнит про *что-то ещё*, может заменить *что-то ещё* и сломать те части программы, которые «помнят» и *ожидают*, что *что-то ещё* будет на месте.

С точки зрения наследования, такой код должен быть недопустимым:

```rust ,ignore
let mut my_kitty = Cat;                  // Создаём Кота (длинное время жизни)
let animal: &mut Animal = &mut my_kitty; // Забываем, что это Кот (кратчайшее время жизни)
*animal = Dog;                           // Меняем на Собаку (короткое время жизни)
my_kitty.meow();                         // Мяукающая Собака! (использование-после-освобождения)
```

Так что, хотя вы *можете* сокращать время жизни изменяемых ссылок, как только вы начинаете *вкладывать* их друг в друга, они становятся «инвариантными» и больше не позволяют сокращать время жизни.

В частности, `&mut &'big mut T` нельзя преобразовывать в `&mut &'small mut T`, где `'big` живёт дольше, чем `'small`.
Или, более формально `&'a mut T` ковариантен относительно `'a`, но инвариантен относительно `T`.

Занимательный факт: Java в принципе *позволяет* проворачивать такие трюки, но в ней есть [проверка времени выполнения, которая предотвращает мяуканье собак](https://docs.oracle.com/javase/7/docs/api/java/lang/ArrayStoreException.html).

----

Так как же нам менять данные?
Используя внутреннюю изменчивость!
Так мы можем сообщить компилятору, что собираемся менять только *данные*, не трогая ссылок.

Вернём предыдущую версию нашего кода с разделяемыми ссылками и используем `Cell` в новом тесте:

```rust ,ignore
#[test]
fn cell() {
    use std::cell::Cell;

    List::push(None, Cell::new(3), |list| {
        List::push(Some(list), Cell::new(5), |list| {
            List::push(Some(list), Cell::new(13), |list| {
                // Умножаем каждое значение в списке на 10
                for val in list.iter() {
                    val.set(val.get() * 10)
                }

                let mut vals = list.iter();
                assert_eq!(vals.next().unwrap().get(), 130);
                assert_eq!(vals.next().unwrap().get(), 50);
                assert_eq!(vals.next().unwrap().get(), 30);
                assert_eq!(vals.next(), None);
                assert_eq!(vals.next(), None);
            })
        })
    })
}
```

```text
> cargo test

running 19 tests
test fifth::test::into_iter ... ok
test fifth::test::basics ... ok
test fifth::test::iter_mut ... ok
test fifth::test::iter ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test second::test::into_iter ... ok
test first::test::basics ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test fifth::test::miri_food ... ok
test silly2::test::cell ... ok
test third::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test silly1::test::walk_aboot ... ok
test silly2::test::elegance ... ok
test third::test::basics ... ok
test second::test::iter ... ok

test result: ok. 19 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out;
```

Проще рекурсивной репы! ✨
