# Метод Peek

В прошлой версии списка мы не реализовали один полезный метод, а именно — заглядывание.
Настало время реализации.
Всё, что нам надо — вернуть ссылку на элемент в голове списка, если он существует.
Звучит легко, попробуем сделать:


```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    self.head.map(|node| {
        &node.elem
    })
}
```


```text
> cargo build

error[E0515]: cannot return reference to local data `node.elem`
  --> src/second.rs:37:13
   |
37 |             &node.elem
   |             ^^^^^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:36:9
   |
36 |         self.head.map(|node| {
   |         ^^^^^^^^^ cannot move out of borrowed content


```

*Вздох*.
Что теперь, Rust?

Функция `map` получает `self` по значению, что удаляет Option из `self.head`.
Раньше это было нормально, потому что мы *забирали* узел себе, но сейчас мы хотим оставить его на месте.
*Корректный* способ обработать такую ситуацию — вызвать у Option метод `as_ref`.
Он имеет определение:

```rust ,ignore
impl<T> Option<T> {
    pub fn as_ref(&self) -> Option<&T>;
}
```

Метод сужает `Option<T>` до опциональной ссылки на внутреннее содержимое.
То же самое можно сделать с помощью оператора `match`, но в этом нет смысла, поскольку в стандартной библиотеке есть готовый метод.
Теоретически, нам надо выполнить дополнительное разыменование, чтобы убрать один уровень косвенности, но, к счастью оператор `.` делает это за нас.


```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        &node.elem
    })
}
```

```text
cargo build

    Finished dev [unoptimized + debuginfo] target(s) in 0.32s
```

Справились.

Мы также можем сделать *изменяемую* версию этого метода, используя `as_mut`:

```rust ,ignore
pub fn peek_mut(&mut self) -> Option<&mut T> {
    self.head.as_mut().map(|node| {
        &mut node.elem
    })
}
```

```bash
cargo build
```

Лег! Ко!

Не забудем протестировать:

```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert_eq!(list.peek(), None);
    assert_eq!(list.peek_mut(), None);
    list.push(1); list.push(2); list.push(3);

    assert_eq!(list.peek(), Some(&3));
    assert_eq!(list.peek_mut(), Some(&mut 3));
}
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 3 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::peek ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured

```

Всё это здорово, но мы так и не проверили, можно ли изменить значение, которое возвращает `peek_mut`, ведь так?
Если ссылка изменяемая, но никто её не изменил, действительно ли мы протестировали изменяемость?
Попробуем вызвать `map` на экземпляре `Option<&mut T>`, чтобы изменить содержимое:

```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert_eq!(list.peek(), None);
    assert_eq!(list.peek_mut(), None);
    list.push(1); list.push(2); list.push(3);

    assert_eq!(list.peek(), Some(&3));
    assert_eq!(list.peek_mut(), Some(&mut 3));
    list.peek_mut().map(|&mut value| {
        value = 42
    });

    assert_eq!(list.peek(), Some(&42));
    assert_eq!(list.pop(), Some(42));
}
```

```text
> cargo test

error[E0384]: cannot assign twice to immutable variable `value`
   --> src/second.rs:100:13
    |
99  |         list.peek_mut().map(|&mut value| {
    |                                   -----
    |                                   |
    |                                   first assignment to `value`
    |                                   help: make this binding mutable: `mut value`
100 |             value = 42
    |             ^^^^^^^^^^ cannot assign twice to immutable variable          ^~~~~
```

Компилятор жалуется, что переменная `value` неизменяемая, но мы довольно ясно написали `&mut value`.
Так в чём же дело?
Оказывается, эта запись не означает, что `value` является неизменяемой ссылкой.
Она означает образец, который сопоставляется с аргументом замыкания; `|&mut value|` — это «аргумент является изменяемой ссылкой, но ты просто скопируй значение, на которое она ссылается, в переменную `value`, пожалуйста».
А вот если мы напишем просто `|value|`, тип переменной `value` превратится в `&mut i32` и мы действительно сможем изменить голову:


```rust ,ignore
    #[test]
    fn peek() {
        let mut list = List::new();
        assert_eq!(list.peek(), None);
        assert_eq!(list.peek_mut(), None);
        list.push(1); list.push(2); list.push(3);

        assert_eq!(list.peek(), Some(&3));
        assert_eq!(list.peek_mut(), Some(&mut 3));

        list.peek_mut().map(|value| {
            *value = 42
        });

        assert_eq!(list.peek(), Some(&42));
        assert_eq!(list.pop(), Some(42));
    }
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 3 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::peek ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured

```

Гораздо лучше!
