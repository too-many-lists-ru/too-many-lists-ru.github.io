> # Peek

# Метод Peek

> One thing we didn't even bother to implement last time was peeking.
> Let's go ahead and do that.
> All we need to do is return a reference to the element in the head of the list, if it exists.
> Sounds easy, let's try:

Одна вещь, которую мы даже не удосужились реализовать в прошлый раз было заглядывание.
Давайте двинемся дальше и реализуем её.
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

> *Sigh*.
> What now, Rust?

*Вздох*.
Что теперь, Rust?

> Map takes `self` by value, which would move the Option out of the thing it's in.
> Previously this was fine because we had just `take`n it out, but now we actually want to leave it where it was.
> The *correct* way to handle this is with the `as_ref` method on Option, which has the following definition:

Функция `map` получает `self` по значению, что удаляет Option из `self.head`.
Раньше это было нормально, потому что мы *забирали* его себе, но сейчас на самом деле мы хотим оставить его так, где оно было.
*Корректный* способ обработать такую ситуацию — вызвать метод `as_ref` у Option, который имеет следующее определение:

```rust ,ignore
impl<T> Option<T> {
    pub fn as_ref(&self) -> Option<&T>;
}
```

> It demotes the `Option<T>` to an Option to a reference to its internals.
> We could do this ourselves with an explicit match but *ugh no*.
> It does mean that we need to do an extra dereference to cut through the extra indirection, but thankfully the `.` operator handles that for us.

Она понижает `Option<T>` до опциональной ссылки на внутренее содержимое.
Мы могли бы сделать это сами, с явным оператором `match`, но *пожалуйта, не надо*.
Это значит, что нам придётся выполнить дополнительное разыменование, чтобы убрать один уровень косвенности, на к счастью оператор `.` делает это за нас.


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

> Nailed it.

Справились.

> We can also make a *mutable* version of this method using `as_mut`:

Мы также можем сделать *мутабельную* версию этого метода, используя `as_mut`:

```rust ,ignore
pub fn peek_mut(&mut self) -> Option<&mut T> {
    self.head.as_mut().map(|node| {
        &mut node.elem
    })
}
```

```text
> cargo build

```

> EZ

Легко! <!-- EZ == ee zee == easy -->

> Don't forget to test it:

Не забудьте протестировать:

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

> That's nice, but we didn't really test to see if we could mutate that `peek_mut` return value, did we?
> If a reference is mutable but nobody mutates it, have we really tested the mutability?
> Let's try using `map` on this `Option<&mut T>` to put a profound value in:

Всё это здорово, но мы так и не проверили, можно ли изменить значение, которое возвращает `peek_mut`, верно?
Если ссылка мутабельная, но никто её не поменял, действительно ли мы протестировали мутабельность?
Попробуем вызвать `map` на этом экземпляре `Option<&mut T>`, чтобы изменить содержимое:

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

> The compiler is complaining that `value` is immutable, but we pretty clearly wrote `&mut value`; what gives?
> It turns out that writing the argument of the closure that way doesn't specify that `value` is a mutable reference.
> Instead, it creates a pattern that will be matched against the argument to the closure; `|&mut value|` means "the argument is a mutable reference, but just copy the value it points to into `value`, please."
If we just use `|value|`, the type of `value` will be `&mut i32` and we can actually mutate the head:

Компилятор жалуется, что переменная `value` иммутабельная, но мы довольно ясно написали `&mut value`; так в чём дело?
Оказывается, эта запись не означает, что `value` является мутабельной ссылкой.
Она означает образец, который сопоставляется с аргументом замыкания; `|&mut value|` значит «аргумент является мутабельной ссылкой, но ты просто скопируй значение, на которое он ссылается в переменную `value`, пожалуйста».
Если мы просто напишем `|value|`, тип переменной `value` будет `&mut i32` и мы сможем действительно изменить голову:


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

> Much better!

Гораздо лучше!
