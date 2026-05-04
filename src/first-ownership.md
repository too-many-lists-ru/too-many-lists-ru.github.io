> # Ownership 101

# Владение 101

> Now that we can construct a list, it'd be nice to be able to *do* something with it.
> We do that with "normal" (non-static) methods.
> Methods are a special case of function in Rust because of the `self` argument, which doesn't have a declared type:

Теперь, когда мы можем сконструировать список, было бы неплохо что-нибудь с ним *делать*.
Делать будем при помощи «обычных» (не статических) методов.
Методы — это особый вид функции в Rust из-за аргумента `self`, у которого нет объявленного типа:

```rust ,ignore
fn foo(self, arg2: Type2) -> ReturnType {
    // body
    // тело
}
```

> There are 3 primary forms that self can take: `self`, `&mut self`, and `&self`.
> These 3 forms represent the three primary forms of ownership in Rust:

Есть три основные формы, которые он может принимать: `self`, `&must self` и `&self`.
Эти три формы представляют три основные формы владения в Rust:

> * `self` - Value
> * `&mut self` - mutable reference
> * `&self` - shared reference

* `self` - по значению
* `&mut self` - по мутабельной ссылке
* `&self` - по разделяемой ссылке

> A value represents *true* ownership.
> You can do whatever you want with a value: move it, destroy it, mutate it, or loan it out via a reference.
> When you pass something by value, it's *moved* to the new location.
> The new location now owns the value, and the old location can no longer access it.
> For this reason most methods don't want `self` -- it would be pretty lame if trying to work with a list made it go away!

Значение представляет собой *истинное* владение.
Мы можете делать со значением всё, что хотите: передать, уничтожить, менять или передавать по ссылке.
Если вы передаёте что-то по значению, оно *передаётся* в новое место.
Новое место теперь владеет значением, а старое место больше не имеет к нему доступа.
По этой причине большинство методов не используют `self` — было бы довольно глупо работать со списком, теряя к нему доступ!

> A mutable reference represents temporary *exclusive access* to a value that you don't own.
> You're allowed to do absolutely anything you want to a value you have a mutable reference to as long you leave it in a valid state when you're done (it would be rude to the owner otherwise!).
> This means you can actually completely overwrite the value.
> A really useful special case of this is *swapping* a value out for another, which we'll be using a lot.
> The only thing you can't do with an `&mut` is move the value out with no replacement. `&mut self` is great for methods that want to mutate `self`.

Мутабельная ссылка означает временный *эксклюзивный доступ* к значению, которым вы не владеете.
Вы можете делать абсолютно всё, что хотите со значением, на которое у вас есть мутабельная ссылка, если в конце вы оставляете его в корректном состоянии (иначе это было бы невежливо по отношению к владельцу!).
Это значит, что вы на самом деле можете полностью перезаписать значение.
Очень полезным частным случаем этого является *обмен* двух значений, к которому мы будем часто прибегать.
Едиственное, что нельзя сделать с `&mut` — избавиться от значения, не заменив его другим значением.
`&mut self` отлично подходит для методов, которых хотят изменить `self`.

> A shared reference represents temporary *shared access* to a value that you don't own.
> Because you have shared access, you're generally not allowed to mutate anything.
> Think of `&` as putting the value out on display in a museum.
> `&` is great for methods that only want to observe `self`.

Разделяемое владение означает временный *разделяемый доступ* к значению, которым вы не владеете.
Поскольку у вас разделяемый доступ, вы в общем случае вообще ничего не можете изменить.
Думайте о `&` как о способе показать значение в музее.
`&` отлично подходит для методов, которым нужно только читать `self`.

> Later we'll see that the rule about mutation can be bypassed in certain cases.
> This is why shared references aren't called *immutable* references.
> Really, mutable references could be called *unique* references, but we've found that relating ownership to mutability gives the right intuition 99% of the time.

Позже мы узнаем, что правило об изменении можно в определённых случаях обойти.
По этой причине разделяемые ссылки не называют *иммутабельными* ссылками.
На самом деле мутабельные ссылки можно называть *уникальными* ссылками, но мы выяснили, что связь между владением и мутабельностью интуитивно понятна 99% времени.