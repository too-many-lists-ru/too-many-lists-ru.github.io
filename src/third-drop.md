> # Drop

# Уничтожение

> Like the mutable lists, we have a recursive destructor problem.
> Admittedly, this isn't as bad of a problem for the immutable list: if we ever hit another node that's the head of another list *somewhere*, we won't recursively drop it.
> However it's still a thing we should care about, and how to deal with isn't as clear.
> Here's how we solved it before:

Как и у мутабельных списков, у нас есть проблема рекурсивного уничтожения.
Следует признать, что для иммутабельного списка это не такая серьёзная проблема: даже если мы наткнёмся на другой узел, который является головой другого списка *где-то*, мы не должны рекурсивно его уничтожать.
Однако, это всё ещё та вещь, о которой мы должны беспокоиться, поскольку не совсем понятно, как с ней быть.
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

> The problem is the body of the loop:

Проблема заключается в теле цикла:

```rust ,ignore
cur_link = boxed_node.next.take();
```

> This is mutating the Node inside the Box, but we can't do that with Rc; it only gives us shared access, because any number of other Rc's could be pointing at it.

Здесь изменяется узел внутри Box, но мы не может сделать то же самое с Rc; он продоставляет нам только разделяемый доступ, потому что на него может указывать любое количество Rc-указателей.

> But if we know that we're the last list that knows about this node, it *would* actually be fine to move the Node out of the Rc.
> Then we could also know when to stop: whenever we *can't* hoist out the Node.

Но если мы точно знаем, что имеем дело с последним списком, который знает о данном 
узле, на самом деле было бы *правильно* забрать узел у Rc.
Тогда мы могли бы знвть, когда остановиться, всякий раз, когда мы *не можем* забрать узел.

> And look at that, Rc has a method that does exactly this: `try_unwrap`:

И смотрите: в Rc есть метод, который делает именно это: `try_unwrap`:


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

> Great!
> Nice.

Великолепно!
Здорово.