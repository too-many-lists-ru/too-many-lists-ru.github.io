> # Testing Cursors

# Тестирование Курсоров

> Time to find out how many horribly embarassing mistakes I made in the previous section!

Время узнать, сколько ужасных неловких ошибок я сделала в предыдущем разделе!

> Oh god we made our API unlike both std and the old impl.
> Alright well I'm just gonna hastily cobble together something from both of them.
> Yeah let's "borrow" these tests from std:

Господи, наш API отличается, как от стандартной библиотеки, так и от моей старой реализации.
Ладно, я просто наспех соберу что-то работающее из них обеих.
Да, давайте «позаимствуем» эти тесты из стандартной реализации:

```rust ,ignore
    #[test]
    fn test_cursor_move_peek() {
        let mut m: LinkedList<u32> = LinkedList::new();
        m.extend([1, 2, 3, 4, 5, 6]);
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        assert_eq!(cursor.current(), Some(&mut 1));
        assert_eq!(cursor.peek_next(), Some(&mut 2));
        assert_eq!(cursor.peek_prev(), None);
        assert_eq!(cursor.index(), Some(0));
        cursor.move_prev();
        assert_eq!(cursor.current(), None);
        assert_eq!(cursor.peek_next(), Some(&mut 1));
        assert_eq!(cursor.peek_prev(), Some(&mut 6));
        assert_eq!(cursor.index(), None);
        cursor.move_next();
        cursor.move_next();
        assert_eq!(cursor.current(), Some(&mut 2));
        assert_eq!(cursor.peek_next(), Some(&mut 3));
        assert_eq!(cursor.peek_prev(), Some(&mut 1));
        assert_eq!(cursor.index(), Some(1));

        let mut cursor = m.cursor_mut();
        cursor.move_prev();
        assert_eq!(cursor.current(), Some(&mut 6));
        assert_eq!(cursor.peek_next(), None);
        assert_eq!(cursor.peek_prev(), Some(&mut 5));
        assert_eq!(cursor.index(), Some(5));
        cursor.move_next();
        assert_eq!(cursor.current(), None);
        assert_eq!(cursor.peek_next(), Some(&mut 1));
        assert_eq!(cursor.peek_prev(), Some(&mut 6));
        assert_eq!(cursor.index(), None);
        cursor.move_prev();
        cursor.move_prev();
        assert_eq!(cursor.current(), Some(&mut 5));
        assert_eq!(cursor.peek_next(), Some(&mut 6));
        assert_eq!(cursor.peek_prev(), Some(&mut 4));
        assert_eq!(cursor.index(), Some(4));
    }

    #[test]
    fn test_cursor_mut_insert() {
        let mut m: LinkedList<u32> = LinkedList::new();
        m.extend([1, 2, 3, 4, 5, 6]);
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.splice_before(Some(7).into_iter().collect());
        cursor.splice_after(Some(8).into_iter().collect());
        // check_links(&m);
        assert_eq!(m.iter().cloned().collect::<Vec<_>>(), &[7, 1, 8, 2, 3, 4, 5, 6]);
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.move_prev();
        cursor.splice_before(Some(9).into_iter().collect());
        cursor.splice_after(Some(10).into_iter().collect());
        check_links(&m);
        assert_eq!(m.iter().cloned().collect::<Vec<_>>(), &[10, 7, 1, 8, 2, 3, 4, 5, 6, 9]);
        
        /* remove_current not impl'd
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.move_prev();
        assert_eq!(cursor.remove_current(), None);
        cursor.move_next();
        cursor.move_next();
        assert_eq!(cursor.remove_current(), Some(7));
        cursor.move_prev();
        cursor.move_prev();
        cursor.move_prev();
        assert_eq!(cursor.remove_current(), Some(9));
        cursor.move_next();
        assert_eq!(cursor.remove_current(), Some(10));
        check_links(&m);
        assert_eq!(m.iter().cloned().collect::<Vec<_>>(), &[1, 8, 2, 3, 4, 5, 6]);
        */

        let mut cursor = m.cursor_mut();
        cursor.move_next();
        let mut p: LinkedList<u32> = LinkedList::new();
        p.extend([100, 101, 102, 103]);
        let mut q: LinkedList<u32> = LinkedList::new();
        q.extend([200, 201, 202, 203]);
        cursor.splice_after(p);
        cursor.splice_before(q);
        check_links(&m);
        assert_eq!(
            m.iter().cloned().collect::<Vec<_>>(),
            &[200, 201, 202, 203, 1, 100, 101, 102, 103, 8, 2, 3, 4, 5, 6]
        );
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.move_prev();
        let tmp = cursor.split_before();
        assert_eq!(m.into_iter().collect::<Vec<_>>(), &[]);
        m = tmp;
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        let tmp = cursor.split_after();
        assert_eq!(tmp.into_iter().collect::<Vec<_>>(), &[102, 103, 8, 2, 3, 4, 5, 6]);
        check_links(&m);
        assert_eq!(m.iter().cloned().collect::<Vec<_>>(), &[200, 201, 202, 203, 1, 100, 101]);
    }

    fn check_links<T>(_list: &LinkedList<T>) {
        // would be good to do this!
        // было бы неплохо написать!
    }
```

> Moment of truth!

Момент истины!

```text
cargo test

   Compiling linked-list v0.0.3
    Finished test [unoptimized + debuginfo] target(s) in 1.03s
     Running unittests src\lib.rs

running 14 tests
test test::test_basic_front ... ok
test test::test_basic ... ok
test test::test_debug ... ok
test test::test_iterator_mut_double_end ... ok
test test::test_ord ... ok
test test::test_cursor_move_peek ... FAILED
test test::test_cursor_mut_insert ... FAILED
test test::test_iterator ... ok
test test::test_mut_iter ... ok
test test::test_eq ... ok
test test::test_rev_iter ... ok
test test::test_iterator_double_end ... ok
test test::test_hashmap ... ok
test test::test_ord_nan ... ok

failures:

---- test::test_cursor_move_peek stdout ----
thread 'test::test_cursor_move_peek' panicked at 'assertion failed: `(left == right)`
  left: `None`,
 right: `Some(1)`', src\lib.rs:1079:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace

---- test::test_cursor_mut_insert stdout ----
thread 'test::test_cursor_mut_insert' panicked at 'assertion failed: `(left == right)`
  left: `[200, 201, 202, 203, 10, 100, 101, 102, 103, 7, 1, 8, 2, 3, 4, 5, 6, 9]`,
 right: `[200, 201, 202, 203, 1, 100, 101, 102, 103, 8, 2, 3, 4, 5, 6]`', src\lib.rs:1153:9


failures:
    test::test_cursor_move_peek
    test::test_cursor_mut_insert

test result: FAILED. 12 passed; 2 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> I'll admit, I had some hubris here and was hoping I nailed it.
> This is why we write tests (but maybe I just did a bad job of porting the tests..?).

Признаюсь, я была немного самонадеянна, и надеялась, что всё сделала правильно.
Вот почему мы пишем тесты (но, может быть, я просто плохо перенесла тесты?..).

> What's the first failure?

Какова первая ошибка?

```rust ,ignore
let mut m: LinkedList<u32> = LinkedList::new();
m.extend([1, 2, 3, 4, 5, 6]);
let mut cursor = m.cursor_mut();

cursor.move_next();
assert_eq!(cursor.current(), Some(&mut 1));
assert_eq!(cursor.peek_next(), Some(&mut 2));
assert_eq!(cursor.peek_prev(), None);
assert_eq!(cursor.index(), Some(0));

cursor.move_prev();
assert_eq!(cursor.current(), None);
assert_eq!(cursor.peek_next(), Some(&mut 1)); // DIES HERE
```

> Geez I really messed up some basic functionality.
> Wait,

Блин, я и правда накосячила с базовой функциональностью.
Подождите,

> > Head empty, Option methods and (omitted) compiler errors do all thinking now.

> Даже думать не пришлось, всю работу сделали методы Option и (опущенные) ошибки компилятора.

> Well I am nothing if not honest.

Что ж, я человек честный.

```rust ,ignore
pub fn peek_next(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur
            .and_then(|node| (*node.as_ptr()).back)
            .map(|node| &mut (*node.as_ptr()).elem)
    }
}
```

> ...Yeah this is just wrong.
> If `self.cur` is None, we aren't just supposed to give up, we need to check `self.list.front` too, because we're on the ghost!
> So we just need to add an or_else to the chain:

Да, это просто неправильно.
Если `self.cur` равен `None`, мы не должны завершать проверку, мы должны также проверить `self.list.front`, потому что находимся над псевдоэлементом!
Так что нам надо добавить в цепочку вызов or_else:

```rust ,ignore
pub fn peek_next(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur
            .and_then(|node| (*node.as_ptr()).back)
            .or_else(|| self.list.front)
            .map(|node| &mut (*node.as_ptr()).elem)
    }
}

pub fn peek_prev(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur
            .and_then(|node| (*node.as_ptr()).front)
            .or_else(|| self.list.back)
            .map(|node| &mut (*node.as_ptr()).elem)
    }
}
```

> Did that fix it?

Помогло?

```text
---- test::test_cursor_move_peek stdout ----
thread 'test::test_cursor_move_peek' panicked at 'assertion failed: `(left == right)`
  left: `Some(6)`,
 right: `None`', src\lib.rs:1078:9
```

> Wait now it's wrong *further back*.
> Ok I need to stop head-emptying peek because apparently it's a lot harder than I was willing to give it credit for.
> Just trying to blindly chain these cases is a disaster, let's have a proper if for the cases of ghost vs not:

Подождите, теперь ошибка *сдвинулась на элемент назад*.
Ладно, мне надо остановить своё «даже думать не пришлось» в отношении peek, потому что, оказывается, всё намного сложнее, чем я представляла.
Просто пытаться увязать в цепочку все эти варианты — это катастрофа, так что давайте добавим явное условие if для псевдоэлемента и обычного элемента:

```rust ,ignore
pub fn peek_next(&mut self) -> Option<&mut T> {
    unsafe {
        let next = if let Some(cur) = self.cur {
            // Normal case, try to follow the cur node's back pointer
            // Обычный вариант, пытаемся следовать указателю назад текущего узла
            (*cur.as_ptr()).back
        } else {
            // Ghost case, try to use the list's front pointer
            // Псевдоэлемент, пытаемся использовать передний указатель списка
            self.list.front
        };

        // Yield the element if the next node exists
        // Возвращаем элемент, если следующий узел существует
        next.map(|node| &mut (*node.as_ptr()).elem)
    }
}

pub fn peek_prev(&mut self) -> Option<&mut T> {
    unsafe {
        let prev = if let Some(cur) = self.cur {
            // Normal case, try to follow the cur node's front pointer
            // Обычный вариант, пытаемся следовать указателю вперёд текущего узла
            (*cur.as_ptr()).front
        } else {
            // Ghost case, try to use the list's back pointer
            // Псевдоэлемент, пытаемся использовать задний указатель списка
            self.list.back
        };

        // Yield the element if the prev node exists
        // Возвращаем элемент, если предыдущий узел существует
        prev.map(|node| &mut (*node.as_ptr()).elem)
    }
}
```

> Feelin' confident about this one!

Ну, в этом коде я уверена!

```text
failures:

---- test::test_cursor_mut_insert stdout ----
thread 'test::test_cursor_mut_insert' panicked at 'assertion failed: `(left == right)`
  left: `[200, 201, 202, 203, 10, 100, 101, 102, 103, 7, 1, 8, 2, 3, 4, 5, 6, 9]`,
 right: `[200, 201, 202, 203, 1, 100, 101, 102, 103, 8, 2, 3, 4, 5, 6]`', src\lib.rs:1168:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    test::test_cursor_mut_insert

test result: FAILED. 13 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> Yesss.
> Ok one more failure to go... oh.

Даааа.
Осталась последняя ошибка... уф.

> Did you notice the part where I commented out some code for testing remove_current?
> Yeah I wasn't paying attention to the fact that this test is stateful.
> Let's just create a new list with the state the remove_current part would have left us in:

Вы заметили, что я закомментировала часть кода, предназначенного для тестирования remove_current?
Да, я не обратила внимания на то, что этот тест меняет состояние.
Давайте просто создадим новый список в том виде, в каком бы его оставил вызов remove_current:

```rust ,ignore
let mut m: LinkedList<u32> = LinkedList::new();
m.extend([1, 8, 2, 3, 4, 5, 6]);
```

```text
 cargo test
   Compiling linked-list v0.0.3
    Finished test [unoptimized + debuginfo] target(s) in 0.70s
     Running unittests src\lib.rs

running 14 tests
test test::test_basic_front ... ok
test test::test_basic ... ok
test test::test_cursor_move_peek ... ok
test test::test_eq ... ok
test test::test_cursor_mut_insert ... ok
test test::test_iterator ... ok
test test::test_iterator_double_end ... ok
test test::test_ord_nan ... ok
test test::test_mut_iter ... ok
test test::test_hashmap ... ok
test test::test_debug ... ok
test test::test_ord ... ok
test test::test_iterator_mut_double_end ... ok
test test::test_rev_iter ... ok

test result: ok. 14 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 803) - compile fail ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.12s
```

> Heyyyy look at thaaat... ok now I'm getting paranoid.
> Let's properly fill in check_links and test it under miri:

Эээй, только взгляните на... ладно, теперь я начинаю параноить.
Давайте аккуратно заполним check_links и протестируем его в miri:

```rust ,ignore
fn check_links<T: Eq + std::fmt::Debug>(list: &LinkedList<T>) {
    let from_front: Vec<_> = list.iter().collect();
    let from_back: Vec<_> = list.iter().rev().collect();
    let re_reved: Vec<_> = from_back.into_iter().rev().collect();

    assert_eq!(from_front, re_reved);
}
```

> Is this the best way to do this?
> No.
> Is it fine?
> Yes.

Это лучшим способ проверки?
Нет.
Нормальный?
Да.

```text
$env:MIRIFLAGS="-Zmiri-tag-raw-pointers"
cargo miri test
   Compiling linked-list v0.0.3
    Finished test [unoptimized + debuginfo] target(s) in 0.25s
     Running unittests src\lib.rs

running 14 tests
test test::test_basic ... ok
test test::test_basic_front ... ok
test test::test_cursor_move_peek ... ok
test test::test_cursor_mut_insert ... ok
test test::test_debug ... ok
test test::test_eq ... ok
test test::test_hashmap ... ok
test test::test_iterator ... ok
test test::test_iterator_double_end ... ok
test test::test_iterator_mut_double_end ... ok
test test::test_mut_iter ... ok
test test::test_ord ... ok
test test::test_ord_nan ... ok
test test::test_rev_iter ... ok

test result: ok. 14 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 803) - compile fail ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.10s
```

> DONE.

ГОТОВО.

> Done.

Готово.

> We did it.
> We made a god damn production-quality LinkedList, with basically all the same functionality as the one in std.
> Are we missing little convenience methods here and there?
> Absolutely.
> Will I add them into the final published version of the crate?
> Probably!

У нас получилось.
Мы сделали связный список офигенно продуктового уровня, практически со всей функциональность аналоги из стандартной библиотеки.
Нам не хватает пары удобных методов там и тут?
Никаких сомнений.
Добавлю ли я их чуть позже?
Возможно!

> But, I am, So Very Tired.

Но Сейчас Я Очень Устала.

> So.
> We win.

В любом случае.
Мы победили.

> Wait fuck.
> We're being production quality.
> Ok one last final boss: clippy.

Блин, подождите.
Нам же нужно продуктовое качество.
Ладно, последний финальный босс: clippy.

```text
cargo clippy

cargo clippy
    Checking linked-list v0.0.3 (C:\Users\ninte\dev\contain\linked-list)
warning: redundant pattern matching, consider using `is_some()`
   --> src\lib.rs:189:19
    |
189 |         while let Some(_) = self.pop_front() { }
    |         ----------^^^^^^^------------------- help: try this: `while self.pop_front().is_some()`
    |
    = note: `#[warn(clippy::redundant_pattern_matching)]` on by default
    = note: this will change drop order of the result, as well as all temporaries
    = note: add `#[allow(clippy::redundant_pattern_matching)]` if this is important
    = help: for further information visit https://rust-lang.github.io/rust-clippy/master/index.html#redundant_pattern_matching

warning: method `into_iter` can be confused for the standard trait method `std::iter::IntoIterator::into_iter`
   --> src\lib.rs:210:5
    |
210 | /     pub fn into_iter(self) -> IntoIter<T> {
211 | |         IntoIter {
212 | |             list: self
213 | |         }
214 | |     }
    | |_____^
    |
    = note: `#[warn(clippy::should_implement_trait)]` on by default
    = help: consider implementing the trait `std::iter::IntoIterator` or choosing a less ambiguous method name
    = help: for further information visit https://rust-lang.github.io/rust-clippy/master/index.html#should_implement_trait

warning: redundant pattern matching, consider using `is_some()`
   --> src\lib.rs:228:19
    |
228 |         while let Some(_) = self.pop_front() { }
    |         ----------^^^^^^^------------------- help: try this: `while self.pop_front().is_some()`
    |
    = note: this will change drop order of the result, as well as all temporaries
    = note: add `#[allow(clippy::redundant_pattern_matching)]` if this is important
    = help: for further information visit https://rust-lang.github.io/rust-clippy/master/index.html#redundant_pattern_matching

warning: re-implementing `PartialEq::ne` is unnecessary
   --> src\lib.rs:275:5
    |
275 | /     fn ne(&self, other: &Self) -> bool {
276 | |         self.len() != other.len() || self.iter().ne(other)
277 | |     }
    | |_____^
    |
    = note: `#[warn(clippy::partialeq_ne_impl)]` on by default
    = help: for further information visit https://rust-lang.github.io/rust-clippy/master/index.html#partialeq_ne_impl

warning: `linked-list` (lib) generated 4 warnings
    Finished dev [unoptimized + debuginfo] target(s) in 0.29s
```

> Alright clippy, let's do this.

Ладно, clippy, добьём.

> Complaint 1 (and 3): we use `while let Some(_) = ` instead of `while .is_some()`.
> The loop is empty so this truly doesn't matter but ok fine, clippy, I'll do things your way.

Предупреждение 1 (и 3): мы используем `while let Some(_) = ` вместо `while .is_some()`.
Тело цикла пустое, так что писать можно как угодно, но, хорошо, clippy, я сделаю по твоему.

> Complaint 2: We have an actual inherent into_iter method.
> Wait, what *checks std* ok, point to clippy.
> IntoIterator is in the prelude (and basically a lang item) so, we don't need an inherent version too.

Предупреждение 2: У нас уже есть унаследованный метод into_iter.
Подожди, что... *проверяет стандартную библиотеку*... ладно, очко уходит clippy.
IntoIterator уже есть в прелюдии (и по сути является языковым примитивом), так что нам не нужна своя версия into_iter.

> Complaint 4: we copied a weird cargocult from std.
*shrug* fine I'll remove it.

Предупреждение 4: мы скопировали кусок странного карго-кода из стандартной библиотеки.
*пожимает плечами* ладно, просто удалю его.

```text
cargo clippy
    Finished dev [unoptimized + debuginfo] target(s) in 0.00s
```

> Nice.
> Just one last thing to do before calling it production quality: fmt.

Прекрасно.
Ещё одна последняя вещь, прежде чем мы сможем считать наш код кодом продуктового уровня: fmt.

```text
cargo fmt
```

> ...yeah it added some newlines and removed some trailing whitespace.
> Nothing interesting.

...так, он добавил несколько новый строк и удалил несколько пробелов в конце строки.
Ничего интересного.

> **WE ARE NOW TRULY FINALLY DONE!!!!!!!!!!!!!!!!!!!!!**

**НАКОНЕЦ, МЫ И ПРАВДА ЗАКОНЧИЛИ!!!**
