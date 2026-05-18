# Метод `drop`

Как и у изменяемых списков, снова возникает проблема рекурсивного уничтожения.
Следует признать, что для неизменяемого списка это не такая большая проблема
Наткнувшись на узел, который является головой *какого-то другого* списка, мы останавливаем рекурсию, потому что этот узел должен остаться в живых.
Но худших случаев никто не отменял, так что мы должны предоставить нерекурсивную реализацию.
Вот как мы решали эту проблему раньше:

```rust ,ignore
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

Но теперь проблема возникает в теле цикла:

```rust ,ignore
cur_link = boxed_node.next.take();
```

Здесь меняется узел внутри Box, но мы не может сделать то же самое с Rc; он предоставляет нам только разделяемый доступ, так как на него может указывать любое количество Rc-указателей.

Но если мы точно знаем, что имеем дело с последним списком, знающим об этом
узле, на самом деле было бы *правильно* забрать узел у Rc.
Тогда мы точно знали бы, когда остановить рекурсию: всякий раз, когда мы *не можем* забрать узел.

Оказывается, в Rc есть метод, который делает именно то, что нам нужно — `try_unwrap`:

```rust ,ignore
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut head = self.head.take();
        while let Some(node) = head {
            if let Ok(mut node) = Rc::try_unwrap(node) {
                head = node.next.take();
            } else {
                break;
            }
        }
    }
}
```

```text
cargo test
   Compiling lists v0.1.0 (/Users/ADesires/dev/too-many-lists/lists)
    Finished dev [unoptimized + debuginfo] target(s) in 1.10s
     Running /Users/ADesires/dev/too-many-lists/lists/target/debug/deps/lists-86544f1d97438f1f

running 8 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 8 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

Великолепно!
Здорово.