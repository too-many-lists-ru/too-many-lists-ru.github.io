> # Drop

# Освобождение

> We can make a stack, push on to, pop off it, and we've even tested that it all works right!

Мы можем создать стек, вставить в него элемент, извлечь его обратно, и мы даже протестировали, что всё это правильно работает!

> Do we need to worry about cleaning up our list?
> Technically, no, not at all!
> Like C++, Rust uses destructors to automatically clean up resources when they're done with.
> A type has a destructor if it implements a *trait* called Drop.
> Traits are Rust's fancy term for interfaces.
> The Drop trait has the following interface:

Надо ли нам беспокоиться об освобождении памяти?
Технически, нет, вообще не надо!
Как и C++, Rust использует деструкторы, чтобы автоматически освобождать ресурсы, после их использования.
Тип имеет деструктор, если он реализует *типаж*, который называется Drop.
Типажи — это научное название интерфейсов в языке Rust.
Типаж Drop имеет следующий вид:

```rust ,ignore
pub trait Drop {
    fn drop(&mut self);
}
```

> Basically, "when you go out of scope, I'll give you a second to clean up your affairs".

В целом: «когда вы выйдете из области видимости, я дам вам секунду, чтобы прибрать за собой».

> You don't actually need to implement Drop if you contain types that implement Drop, and all you'd want to do is call *their* destructors.
> In the case of List, all it would want to do is drop its head, which in turn would *maybe* try to drop a `Box<Node>`.
> All that's handled for us automatically... with one hitch.

Вам не надо реализовывать Drop, если вы содержите типы, реализующие Drop и всё, что вы хотите, это вызвать *их* деструкторы.
В случае List, всё что ему может потребоваться, это освободить голову, что, *возможно*, приведёт к освобождению `Box<Node>`.
Всё это будет сделано за нас автоматически... с одним «но».

> The automatic handling is going to be bad.

Автоматическая обработка может пойти не по плану.

> Let's consider a simple list:

Посмотрим на простой список:

```text
list -> A -> B -> C
```

> When `list` gets dropped, it will try to drop A, which will try to drop B, which will try to drop C.
> Some of you might rightly be getting nervous.
> This is recursive code, and recursive code can blow the stack!

Когда удаляется `list`, он попытается удалить A, который попытается удалить B, который попытается удалить C.
Возможно, кое-кто из вас занервничал.
Это рекурсивный код, а рекурсивный код может привести к переполнению стека!

> Some of you might be thinking "this is clearly tail recursive, and any decent language would ensure that such code wouldn't blow the stack".
> This is, in fact, incorrect!
> To see why, let's try to write what the compiler has to do, by manually implementing Drop for our List as the compiler would:

Некоторые из вас, возможно, подумают: «это определённо хвостовая рекурсия, а любой приличный язык программирования гарантирует, что подобный код не приведёт к переполнению стека».
Но на самом деле это неверно!
Чтобы понять, почему, давайте попытаемся написать, что должен сделать компилятор, вручную реализовав Drop для нашего списка, как это сделал бы компилятор:

```rust ,ignore
impl Drop for List {
    fn drop(&mut self) {
        // ПРИМЕЧАНИЕ: в реальном коде нельзя вызвать `drop` явно;
        // так что мы притворимся компилятором!
        // NOTE: you can't actually explicitly call `drop` in real Rust code;
        // we're pretending to be the compiler!
        self.head.drop(); // tail recursive - good! хвостовая рекурсия - хорошо!
    }
}

impl Drop for Link {
    fn drop(&mut self) {
        match *self {
            Link::Empty => {} // Done! Готово!
            Link::More(ref mut boxed_node) => {
                boxed_node.drop(); // tail recursive - good! хвостовая рекурсия - хорошо!
            }
        }
    }
}

impl Drop for Box<Node> {
    fn drop(&mut self) {
        self.ptr.drop(); // uh oh, not tail recursive! ой, не хвостовая рекурсия!
        deallocate(self.ptr);
    }
}

impl Drop for Node {
    fn drop(&mut self) {
        self.next.drop();
    }
}
```

> We *can't* drop the contents of the Box *after* deallocating, so there's no way to drop in a tail-recursive manner!
> Instead we're going to have to manually write an iterative drop for `List` that hoists nodes out of their boxes.

Мы *не можем* освободить содержимое Box *после* деаллокации, то что нет способа освободить в манере хвостовой рекурсии!
Вместо этого мы собираемся вручную написать итерактивное освобождение для `List`, которое выведет узлы из их боксов.

```rust ,ignore
impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = mem::replace(&mut self.head, Link::Empty);
        // `while let` == "do this thing until this pattern doesn't match"
        // `while let` значит "делай это, пока образец совпадает"
        while let Link::More(mut boxed_node) = cur_link {
            cur_link = mem::replace(&mut boxed_node.next, Link::Empty);
            // boxed_node goes out of scope and gets dropped here;
            // but its Node's `next` field has been set to Link::Empty
            // so no unbounded recursion occurs.
        }
    }
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

```

> Great!

Великолепно!

> ----------------------

----------------------

> <span style="float:left">![Bonus](img/profbee.gif)</span>

> ## Bonus Section For Premature Optimization!

> ## Бонусный раздел для преждевременной оптимизации!

> Our implementation of drop is actually *very* similar to `while let Some(_) = self.pop() { }`, which is certainly simpler.
> How is it different, and what performance issues could result from it once we start generalizing our list to store things other than integers?

Наша реализация освобождения *очень* напоминает код `while let Some(_) = self.pop() { }`, который, безусловно, проще.
Чем они отличаются, и какие проблемы с производительностью могут возникнуть, если мы обобщим наш список для хранения объектов любого типа, а не только целых чисел?

> <details>
> <summary>Click to expand for answer</summary>

> Pop returns `Option<i32>`, while our implementation only manipulates Links (`Box<Node>`).
> So our implementation only moves around pointers to nodes, while the pop-based one will move around the values we stored in nodes.
> This could be very expensive if we generalize our list and someone uses it to store instances of VeryBigThingWithADropImpl (VBTWADI).
> Box is able to run the drop implementation of its contents in-place, so it doesn't suffer from this issue.
> Since VBTWADI is *exactly* the kind of thing that actually makes using a linked-list desirable over an array, behaving poorly on this case would be a bit of a disappointment.

> If you wish to have the best of both implementations, you could add a new method, `fn pop_node(&mut self) -> Link`, from-which `pop` and `drop` can both be cleanly derived.

> </details>

<details>
<summary>Раскрыть ответ</summary>

`pop` возвращает `Option<i32>` в то время, как наша реализация манипулирует только ссылками (`Box<Node>`).
Так что наша реализация всего лишь переносит указатели на узлы, в то время как `pop` переносит значия, которые хранятся в узлах.
Может быть очень дорого обобщать наш список, ведь кто-то может использовать его для экземляров типа VBTWADI (VeryBigThingWithADropImpl — очень большая штука с реализацией Drop).
Боксы способены запускать реализацию drop для своего содержимого непосрердственно, так что у них нет такой проблемы.
Ну, а поскольку VBTWADI — *именно* та штука, для которой связные списки гораздо предпочтительнее массивов, такое поведение кажется немного разочаровывающим.

> If you wish to have the best of both implementations, you could add a new method, `fn pop_node(&mut self) -> Link`, from-which `pop` and `drop` can both be cleanly derived.

Если вы хотите взять лучшее от обеих реализаций, вы можете добавить новый метод `fn pop_node(&mut sefl) -> Link` который можно корректно вызывать, как из `pop`, так и из `drop`.

</details>
