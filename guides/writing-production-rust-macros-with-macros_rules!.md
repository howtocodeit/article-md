---
title: Writing production Rust macros with macro_rules!
meta_title: Writing Production Rust Macros with macro_rules!
slug: writing-production-rust-macros-with-macro-rules
description: Learn how experts write real production macros.
meta_description: Learn how experts write real production Rust macros with macro_rules!
color: turquoise
tags: [rust, macros]
version: 1.0.4
---

This is an intervention.

I'm sorry to spring it on you like this, but we need to talk. It's ok – you're among friends.

I need you to stop describing macros as "magic".

Listen, I get it. Macro definitions spook me too. Confronted by a large macro, my eyes glaze over and elevator music fills my brain.

But each time you say "magic", it entrenches the superstition that macros are too arcane, too advanced, too scary for the likes of you and I.

It creates a false distinction between "Rust" and "Rust with macros". In reality, they're part of 
the same language, so if you're not comfortable with macros, you're not comfortable with Rust.

>If you're not comfortable with macros, you're not comfortable with Rust.

In this guide, we're going to demystify three macros that underpin hundreds of thousands of Rust applications. They're not written by wizards, but by the talented Rust engineers we aspire to be.

All three macros are "declarative" – the kind you define with `macro_rules!`. Think `vec!`, `panic!`, etc.

These make up the bulk of all Rust macros, taking tokens as input, pattern matching, and outputting source code based on the match.

Procedural macros constitute the rest of the macro family, and involve programmatic manipulation of a token stream. That's a story for another guide, though!

I've ordered today's case studies by complexity, from simplest to scariest.

It's an intermediate guide focused on real production code, not an introduction to macros. I assume that you have a basic familiarity with declarative macro syntax and are comfortable with the idea of pattern matching against tokens that aren't necessarily valid Rust.

If you're shaky on the basics, here's [the relevant section of _The Book_](https://doc.rust-lang.org/book/ch19-06-macros.html). 

[Programming Rust](https://www.oreilly.com/library/view/programming-rust-2nd/9781492052586/) tackles declarative macro syntax in substantially more depth, and is a good follow-up read.

Ready? Here's how to code it.

> Newsletter sign-up

## A standard Rust macro: `assert_eq!`

We're starting off simple but instructive. `assert_eq!` is a standard library macro that we use every day. Here's the example [from the docs](https://doc.rust-lang.org/std/macro.assert_eq.html):

```rust
let a = 3;  
let b = 1 + 2;  
assert_eq!(a, b);

assert_eq!(a, b, "we are testing addition with {} and {}", a, b);
```

My goal is to show you that even simple, readable macros written by exceptional Rust devs are packed with advice and inspiration for crafting our own.


@@@info
Light reading

Do you regularly read standard library code? I highly recommend it.

Not only does it teach you about the kind and quality of code underpinning a mainstream language, but a strong understanding of stdlib internals makes debugging almost _any_ Rust easier.
@@@

`assert_eq!` supports two argument patterns. The first matches two expressions, `a, b`, and compares their values for equality. The second also takes a format string and an arbitrary number of arguments to interpolate.

If the equality check for either matched pattern fails, `assert_eq!` panics.

Since we have two argument patterns, you'd be correct to expect two rules in the macro definition:

```rust
#[macro_export]
macro_rules! assert_eq {
	($left:expr, $right:expr $(,)?) => { ... }; ^1
	($left:expr, $right:expr, $($arg:tt)+) => { ... }; ^2
}
```

Rule `^1` takes two expressions – the left-hand and right-hand sides of the equality – and an optional trailing comma.

Rule `^2` matches the same two expressions, but also matches one or more `TokenTree`s corresponding to the format string and its arguments.



@@@info
TokenTree

Either a single Rust token, like `3` or `"hello"`, or any number of nested tokens contained by `()`, `[]` or `{}`.
@@@


Their expansions are almost identical:

```rust
#[macro_export]
macro_rules! assert_eq {  
    ($left:expr, $right:expr $(,)?) => {  
        match (&$left, &$right) {  
            (left_val, right_val) => {  ^3
                if !(*left_val == *right_val) {  ^4
                    let kind = $crate::panicking::AssertKind::Eq;  ^5
					// ...
                }  
            }  
        }  
    };  
    ($left:expr, $right:expr, $($arg:tt)+) => {  
        match (&$left, &$right) {  
            (left_val, right_val) => {  
                if !(*left_val == *right_val) {  
                    let kind = $crate::panicking::AssertKind::Eq;  
                    // ...
                }  
            }  
        }  
    };  
}
```

Both rules produce code that evaluates the left- and right-hand expressions and stores references to the outputs in `left_val` and `right_val` `^3`. This saves us from evaluating `$left` or `$right` more than once. Remember that, although the `assert_eq!` example uses primitive types, `$left` and `$right` could be expensive function calls.

Next, a simple check for equality using `PartialEq` `^4`.

Things get interesting when that check fails.

Both rules assign `$crate::panicking::AssertKind::Eq` to the variable `kind` `^5`. Rust is smart enough that this macro-local `kind` variable won't conflict with any variable named `kind` where `assert_eq!` is called – a feature known as macro hygiene.

But why use a fully qualified module path to `AssertKind::Eq`?

When you export a macro with `#[macro_export]` there are no guarantees about what will be in scope where that macro is used. Users of `assert_eq!` shouldn't be required to import this implementation detail themselves, and the fully qualified path avoids this.

> Use a fully qualified path to declarative macro dependencies, or import them in the body of the macro.

Now let's look at how the rule expansions differ:

```rust
#[macro_export]  
macro_rules! assert_eq {  
    ($left:expr, $right:expr $(,)?) => {  
        match (&$left, &$right) {  
            (left_val, right_val) => {  
                if !(*left_val == *right_val) {  
                    let kind = $crate::panicking::AssertKind::Eq;  
					$crate::panicking::assert_failed(
						kind,
						&*left_val,
						&*right_val,
						$crate::option::Option::None,
					);  
                }  
            }  
        }  
    };  
    ($left:expr, $right:expr, $($arg:tt)+) => {  
        match (&$left, &$right) {  
            (left_val, right_val) => {  
                if !(*left_val == *right_val) {  
                    let kind = $crate::panicking::AssertKind::Eq;  
					$crate::panicking::assert_failed(
						kind,
						&*left_val,
						&*right_val, 
						$crate::option::Option::Some(
							$crate::format_args!($($arg)+)
						),
					);  ^6
                }  
            }  
        }  
    };  
}
```

Aha! Both are implemented as calls to the same `core` function, `panicking::assert_failed`:

```rust
pub fn assert_failed<T, U>(  
    kind: AssertKind,  ^7
    left: &T,  
    right: &U,  
    args: Option<fmt::Arguments<'_>>,  ^8
) -> !  
where  
    T: fmt::Debug + ?Sized,  ^9
    U: fmt::Debug + ?Sized,  
{  
    assert_failed_inner(kind, &left, &right, args)  
}
```

`assert_failed` makes use of the `AssertKind` `^7` assigned in the body of `assert_eq!`. It's not relevant to our understanding of macros, so we won't follow it any further.

The `args` parameter is where the party's at `^8`. `assert_failed` takes an optional `fmt::Arguments`, which the first `assert_eq!` rule fills with `None`, and the second rules fills with `Some`.

How does it get hold of a `fmt::Arguments`? Why, with a nested macro call, of course `^6`.

`core::format_args!` takes an expression that evaluates to a format string, and zero or more arguments, then outputs a `fmt::Arguments`. It's implemented as a compiler built-in, but here's the signature for the curious:

```rust
macro_rules! format_args {  
    ($fmt:expr) => {{ /* compiler built-in */ }};  
    ($fmt:expr, $($args:tt)*) => {{ /* compiler built-in */ }};  
}
```


@@@info
`assert_eq!` args must be `Debug`

If you've ever wondered where the requirement that arguments to `assert_eq!` must be `Debug` comes from, wonder no more. It's a bound on `core::panicking::assert_failed` `^9`.
@@@


`assert_eq!` doesn't do very much, does it? It surely won't be going to Hogwarts.

It's a convenience that saves us from writing our own call to `PartialEq::eq`, and constructing optional format arguments manually.

It's a very convenient convenience. I particularly like how it hides the existence of the `Option<fmt::Arguments>` required by `assert_failed` from the caller. We're busy developers, after all. Not having to write `None` at every assertion without format args might just be the thing standing between us and total burnout. I pray we'll never find out.

## Write dynamic Rust with intermediate `macro_rules!`

Alright, let's kick it up a notch.

We've seen how to hide inconvenient code with macros but, as with any gateway drug, those entry-level macros just leave us jonesing to disguise vast amounts of ugly, generic boilerplate behind macro invocations.

[The axum crate](https://github.com/tokio-rs/axum) does this so effectively it makes Rust feel like a dynamic language.

axum is a web server in the Tokio ecosystem. Put its macros back in the 1600s, and they'd be immediately burned for being witches – that's how close to magic they feel.

We're people of science, however, and the Enlightenment is upon us 🧑‍🔬. 

Using the `IntoResponse` trait and a vast array of implementations, axum can convert most objects into HTTP responses.

`IntoResponse` is manually implemented for all the usual suspects: `&'static str`, `String`, `Box<str>`, `Result<T, E> where T: IntoResponse, E: IntoResponse`, `axum::http::StatusCode`, and many, many more.

Useful, but neither surprising nor terribly impressive. It's a simple trait. Look:

```rust
pub trait IntoResponse {  
    /// Create a response.  
    fn into_response(self) -> Response;  
}
```

Be still, my beating heart.

What _is_ impressive is that it implements `IntoResponse` for tuples of seemingly any length:

```rust
(a).into_response()
(a, b).into_response()
(a, b, c).into_response()
(a, b, c, d).into_response()
...
```

How does it do this? What satanic deal has been struck to attain such power? 👹

None! axum invokes no magic, only declarative macros.

```rust
macro_rules! impl_into_response {
	( $($ty:ident),* $(,)? ) => { ... } ^10
}
```

`impl_into_response!` has one rule `^10`. It matches zero or more comma-separated idents, with an optional trailing comma.

In the rule body, there are four different implementations of `IntoResponse`:

```rust
macro_rules! impl_into_response {
	( $($ty:ident),* $(,)? ) => {
		impl<R, $($ty,)*> IntoResponse for ($($ty),*, R)  ^11
		where  
		    $( $ty: IntoResponseParts, )*  
		    R: IntoResponse,
		{ ... }
		
		impl<R, $($ty,)*> IntoResponse for (StatusCode, $($ty),*, R)  ^12
		where  
		    $( $ty: IntoResponseParts, )*  
		    R: IntoResponse,
		{ ... }
		
		impl<R, $($ty,)*> IntoResponse for (http::response::Parts, $($ty),*, R)  ^13
		where  
		    $( $ty: IntoResponseParts, )*  
		    R: IntoResponse,  
		{ ... }
		
		impl<R, $($ty,)*> IntoResponse for (http::response::Response<()>, $($ty),*, R)  ^14
		where  
		    $( $ty: IntoResponseParts, )*  
		    R: IntoResponse,  
		{ ... }		
	}
}
```

Yeesh. Hide your kids, the frail and the elderly. This code has a face to turn junior devs to stone.

Taking it `impl` by `impl` reveals that axum doesn't actually implement `IntoResponse` for _any_ tuple, but for all tuples with a type corresponding to one of these implementations.

The first implements `IntoResponse` for tuples of the matched length, where the last item already implements `IntoResponse`, and all other items implement `IntoResponseParts` `^11`.

The second provides an implementation for tuples where the last item is `IntoResponse`, the first item is a `StatusCode`, and all other items implement `IntoResponseParts` `^12`.

The third swaps `StatusCode` for `http::response::Parts` `^13`, while the fourth uses `http::Response::Response<()>` as its first item `^14`.


@@@info
`IntoResponseParts`

This is a discussion about macros, not axum, so the definition of `IntoResponseParts` isn't important.

I know, I know – you're addicted to Rust. It's causing problems at home. You need your fix.

[Here are the docs](https://docs.rs/axum/latest/axum/response/trait.IntoResponseParts.html) on `IntoResponseParts`. Just... don't tell anyone where you got them, ok?
@@@

Remember how `assert_eq!` disguised the fact that both rules called the same underlying function – `assert_failed` – with different arguments? Each of the `impl_into_response!` implementations does something similar.

You can think of each tuple-based `IntoResponse` implementation as a reducer that takes the `IntoResponse` as its initial value and iterates over each `IntoResponseParts` in the tuple, accumulating the parts into the response.

The first implementation is the simplest:

```rust
impl<R, $($ty,)*> IntoResponse for ($($ty),*, R)  
where  
    $( $ty: IntoResponseParts, )*  
    R: IntoResponse,  
{  
    fn into_response(self) -> Response {  
        let ($($ty),*, res) = self;  ^15
  
        let res = res.into_response();  
        let parts = ResponseParts { res };  ^16
  
        $(  
            let parts = match $ty.into_response_parts(parts) {  
                Ok(parts) => parts,  
                Err(err) => {  
                    return err.into_response(); ^17 
                }  
            };  
        )*  ^18
  
        parts.res  ^19
    }  
}
```

It starts by separating the `IntoResponse` from the other tuple items `^15`. It immediately calls `into_response` on it, then sets that `Response` as the `res` field of a new `ResponseParts` object `^16`.

We then iterate over the remaining items in the tuple, calling their `into_response_parts` methods with the `parts` accumulated so far, then redefining `parts` to hold the return value, which becomes the input for the next iteration `^18`.

If any `into_response_parts` returns an error, the iteration short-circuits with an error response `^17`.

If the iteration succeeds, the final `Response` contained inside `parts` is returned `^19`.

The second and third implementations follow the same principle, but the response they build is wrapped in a new, two-item tuple whose first item is either a `StatusCode` or an `http::response::Parts`, depending on the input.

The `Response` returned is the result of calling `into_response` on this intermediate tuple:

```rust
impl<R, $($ty,)*> IntoResponse for (StatusCode, $($ty),*, R)  
where  
    $( $ty: IntoResponseParts, )*  
    R: IntoResponse,  
{  
    fn into_response(self) -> Response {  
        let (status, $($ty),*, res) = self;  
  
        let res = res.into_response();  
        let parts = ResponseParts { res };  
  
        $(  
            let parts = match $ty.into_response_parts(parts) {  
                Ok(parts) => parts,  
                Err(err) => {  
                    return err.into_response();  
                }  
            };  
        )*  
  
        (status, parts.res).into_response()  
    }  
}
```

axum provides manual implementations for these special two-item cases. The `StatusCode` tuple is trivial:

```rust
impl<R> IntoResponse for (StatusCode, R)  
where  
    R: IntoResponse,  
{  
    fn into_response(self) -> Response {  
        let mut res = self.1.into_response();  
        *res.status_mut() = self.0;  
        res  
    }  
}
```

The `http::response::Parts` implementation is more exciting, because it invokes another macro-generated implementation to output the final response.

```rust
impl<R> IntoResponse for (http::response::Parts, R)  
where  
    R: IntoResponse,  
{  
    fn into_response(self) -> Response {  
        let (parts, res) = self;  
        (parts.status, parts.headers, parts.extensions, res).into_response()  ^20
    }  
}
```

The call at `^20` is actually a call to the second, generated `IntoResponse` implementation for tuples with leading `StatusCode`s.

And what do you know? It looks like the fourth and final implementation results in a call to the third implementation, with leading `http::response::Parts`:

```rust
impl<R, $($ty,)*> IntoResponse for (http::response::Response<()>, $($ty),*, R)  
where  
    $( $ty: IntoResponseParts, )*  
    R: IntoResponse,  
{  
    fn into_response(self) -> Response {  
        let (template, $($ty),*, res) = self;  
        let (parts, ()) = template.into_parts();  
        (parts, $($ty),*, res).into_response()  
    }  
}
```

> Reducing complexity to handful of fundamental function calls is a hallmark of a well-written macro.

Wait. I've said `impl_into_response` matches zero or more idents... but where do the idents come from? Who's giving these idents to `impl_into_response` to generate the implementations?

Buckle up.

```rust axum/axum-core/src/response/into_response.rs
all_the_tuples_no_last_special_case!(impl_into_response);
```

Apply pressure to the nosebleed and tilt your head forward until it clots.

Yes, my friend, we've got macros within macros. `all_the_tuples_no_last_special_case!` takes the `impl_into_response!` identifier as an argument. And why not? Macros operate on tokens, and the name of a macro is just another token!


@@@info
What's the "last special case"?

Special it may be, but it's not part of our macro discussion. Here's the [source](https://github.com/tokio-rs/axum/blob/51bb82bb2d317f67edf9e07bd8ff5c603e92b824/axum-core/src/macros.rs#L226).
@@@


How is `all_the_tuples_no_last_special_case!` defined?

```rust
macro_rules! all_the_tuples_no_last_special_case {  
    ($name:ident) => {  
        $name!(T1);  
        $name!(T1, T2);  
        $name!(T1, T2, T3);  
        $name!(T1, T2, T3, T4);  
        $name!(T1, T2, T3, T4, T5);  
        $name!(T1, T2, T3, T4, T5, T6);  
        $name!(T1, T2, T3, T4, T5, T6, T7);  
        $name!(T1, T2, T3, T4, T5, T6, T7, T8);  
        $name!(T1, T2, T3, T4, T5, T6, T7, T8, T9);  
        $name!(T1, T2, T3, T4, T5, T6, T7, T8, T9, T10);  
        $name!(T1, T2, T3, T4, T5, T6, T7, T8, T9, T10, T11);  
        $name!(T1, T2, T3, T4, T5, T6, T7, T8, T9, T10, T11, T12);  
        $name!(T1, T2, T3, T4, T5, T6, T7, T8, T9, T10, T11, T12, T13);  
        $name!(T1, T2, T3, T4, T5, T6, T7, T8, T9, T10, T11, T12, T13, T14);  
        $name!(T1, T2, T3, T4, T5, T6, T7, T8, T9, T10, T11, T12, T13, T14, T15);  
        $name!(T1, T2, T3, T4, T5, T6, T7, T8, T9, T10, T11, T12, T13, T14, T15, T16);  
    };  
}
```

This is the Rust equivalent of learning that Santa isn't real.

There's no magic here. The Easter Bunny has been shot. Rust is not, shockingly, a dynamic language. And it still doesn't support variadic parameters.

axum _appears_ to implement `IntoResponse` for tuples of arbitrary length, provided they have the type signatures we've discussed.

In reality, it statically implements `IntoResponse` for tuples with a maximum length of 16 by passing increasing numbers of identifiers to `impl_into_response!`.

The names of these idents are arbitrary, but must be unique within each `impl_into_response!` invocation, or we'd force the tuple items with matching idents to have the same type.

You don't need to know what types the idents `T1..=T16`  represent in advance. This is an advantage of using `ident` instead of `ty` fragments in this scenario, even though these are, conceptually, types.

Each of these placeholders becomes a generic parameter of the implementations generated by `impl_into_response!`. They only become concrete when your code calls them with specific types.

## Advanced Rust `macro_rules!` for normalization, counting and concurrency

Are you ready to impress your friends at dinner parties?*


@@@warning
\*Assuming your friends are Rustaceans. If not, keep this to yourself, or you may be asked to leave.
@@@

 
Once you wrap your head around this next one, no macro will stand in your way.

We're going to downtown [Tokio](https://tokio.rs/) for a walking tour of the `join!` macro.

`join!` takes a list of async expressions and evaluates them concurrently. _Concurrently_, not in parallel – all expressions are multiplexed to the same Tokio task. It returns once all branches are done executing.

Amusingly, concurrency is by far the least intimidating part of this code. Grab a notepad; it'll help.

Let's start with my favorite rule:

```rust
#[macro_export]
macro_rules! join {
	() => { async {}.await }
}
```

If you call `join!` with no args, it creates and awaits a no-op `Future` that completes immediately 💯. **Tokio expertise intensifies**.

My second-favorite branch is the standard entrypoint for user calls to `join!`:

```rust
#[macro_export]
macro_rules! join {
	// ===== Entry point =====

	( $($e:expr),+ $(,)?) => {  ^21
		$crate::join!(@{ () (0) } $($e,)*)  ^22
	};

	() => { async {}.await }
}
```

No cause for alarm at `^21`. This matches one or more comma-separated expressions with an optional trailing comma. These are the async expressions we want `join!` to run.

`^22` is a bit... funky. It calls `join!` recursively, passing whatever the hell `@{ () (0) }` is, along with all of the input expressions.

Curious, no?

At first, this recursive call gets matched by the rule labeled "Normalize":

```rust
#[macro_export]
macro_rules! join {
	// ===== Normalize =====
	
	(@ { ( $($s:tt)* ) ( $($n:tt)* ) $($t:tt)* } $e:expr, $($r:tt)* ) => { ^23
        $crate::join!(@{ ($($s)* _) ($($n)* + 1) $($t)* ($($s)*) $e, } $($r)*) ^24
    };

	// ===== Entry point =====
	
	( $($e:expr),+ $(,)?) => {
		$crate::join!(@{ () (0) } $($e,)*)
	};

	() => { async {}.await }
}
```

This is where your head may start to spin.

The normalize arm matches three things from the peculiar `@{...}` construct `^23`:
1. `$s`, zero or more `TokenTree`s wrapped by `()`. This corresponds to `()` in the entry point rule, so we know that `$s` is initially empty.
2. `$n`, also zero or more `TokenTree`s wrapped by `()`. This corresponds to `(0)` in the entry point rule, so `$n` is initially a list containing `0`.
3. `$t` is zero or more `TokenTree`s without surrounding parentheses. We don't have any of these in the entry point.

It also matches two things from outside the `@{...}` construct `^23`:
1. `$e`, an expression. This is the first of the async expressions matched by the entry point.
2. `$r`, which represents "everything else". This is all the other expressions.

Well, what do you know? The body of this rule is another recursive call to `join!` `^24`. And it is _wild_.

`$s` gets expanded into the first set of parentheses within the `@{...}` construct, and suffixed with an underscore.

`$n` gets expanded into the second set of parentheses within the `@{...}` construct, and suffixed with `+ 1`. The sum isn't evaluated – the token literal "+1" is just tacked onto the end of whatever's inside `$n`.

`$t` is expanded into the `@{...}` construct, without wrapping parentheses.

That's followed by the matched value of `$s`, this time without a trailing underscore 🤷.

The matched async expression, `$e`, is the last item to be added to the `@{...}` construct.

The rest of the async expressions, `$r`, is fed back into the `join!` call outside of the `@{...}` construct.

It feels like we've taken the head off the list of async expressions and placed it inside `@{...}`, along with a substantial amount of nonsense. This item is then considered "normalized", and we recurse with the accumulated normalized data and the rest of the list.


@@@info
The technical term for this macro design pattern is an "incremental `tt` muncher". Delicious.
@@@


This process is the same reductive pattern used by `assert_eq!` and `impl_into_response` – on steroids.

Let's pause here to look at an example, because thinking about this is causing a weird ringing in my ears.

Consider this two-expression `join!` call from the Tokio docs:

```rust
let (first, second) = tokio::join!(  
	do_stuff_async(),
	more_async_work(),
);
```

This matches against the entry point rule of `join!`.

The entrypoint, in turn, triggers a normalize match with these args:

```rust
join!(@{ () (0) }, do_stuff_async(), more_async_work(),)
```

The normalize branch leads to another recursion, this time with the following args:

```rust
join!(@{ (_) (0+1) () do_stuff_async(), } more_async_work(),)
```

What matches this? Normalize again.

This time, `$s` is `_`, `$n` is `0+1`, `$t` is a list containing the `TokenTree`s `()` and `do_stuff_async()`, `$e` is `more_async_work()`, and `$r` is empty.

This leads to a further call to `join!`:

```rust
join!(@{ (__) (0+1+1) () do_stuff_async(), (_) more_async_work(), })
```

The normalize branch no longer matches this, since there are no async expressions outside the `@{...}` construct.

Inspecting the fourth and final rule, things come into focus. Here's the pattern:

```rust
(@ {  
    // One `_` for each branch in the `join!` macro. This is not used once  
    // normalization is complete.
	( $($count:tt)* )  ^24
  
    // The expression `0+1+1+ ... +1` equal to the number of branches.  
    ( $($total:tt)* )  ^25
  
    // Normalized join! branches  
    $( ( $($skip:tt)* ) $e:expr, )*  ^26
  
}) => {...}
```

The normalization process is designed to build this zany, macro-specific data structure from the standard Rust expressions passed into `join!` by the caller. 

It consists of:
* The delimiter for the data structure, `@ {...}`.
* `$count`, in the form of zero or more underscores in parentheses.
* `$total`, of the form `0+1+1+ ... +1`.
* A list of pairs of the form `(__..._) async_expr,`. The underscores are matched by `$skip`, and the expression is matched by `$e`.

Let's break this down.

### Private macro rules in Rust

`@ {...}` isn't some esoteric Rust syntax you haven't seen before. It's not valid Rust at all! It's merely a pattern of tokens, matched by the macro, to help structure its inputs. The authors could also have chosen `@@@ {...}`, `@pufferfish {...}` or `hobgoblins {...}`.

Prefixing like this has two benefits:
* A leading `@` effectively marks a macro rule as private, since no sensible user would structure their own inputs this way. This is preferable to having `join!` call a separate macro, because, for `join!` to depend on it, that macro would also have to be exported with `#[macro_export]`, making it an explicit part of Tokio's public API.
* The curly braces give the normalize rule a way to distinguish processed from unprocessed inputs as it recurses (they're not required to make the rule "private").


@@@info
`@` vs other symbols

Although you can use symbols other than `@` to denote an internal rule, `@` is a generally accepted community standard.

At present, no other Rust syntax uses `@` in prefix position, so it won't be treated specially by the compiler.

To learn more about internal rules, check out Daniel Keep's [_Little Book of Rust Macros_](https://danielkeep.github.io/tlborm/book/pat-internal-rules.html).
@@@


### Counting with `macro_rules!`

The leading underscores, `@{ (__) ...}` matched by `$count` are used by the normalizer to track which async expression it's currently normalizing, and prefix that expression with the correct number of underscores  `^24`.

Since the initial input `$s` is `()`, the normalizer places no underscores in front of the `do_stuff_async()` branch, resulting in the `TokenTree`s `()` and `do_stuff_async()` being added to the normalized structure.

On the second recursion, `$s` is `(_)`, which causes the next two `TokenTree`s to be `(_)` and `more_async_work()`.

Doesn't this seem like a complex way to count? After all, the matched `$total` `^25` is a numeric expression: `0+1+1`, in the case of our example.

To understand the difference between `$total` and `$count`, we need to look at the body of the rule.

I've omitted the authors' comments about how this code works except where they relate specifically to the macro expansion. Focus on the numbered lines, but please do study [the original](https://github.com/tokio-rs/tokio/blob/39cf6bba0035bbb32fe89bf8ad0189e96fa2d926/tokio/src/macros/join.rs#L58) when we're done – it's informative on the broader topics of async, safety and Tokio internals.

```rust
use $crate::macros::support::{maybe_done, poll_fn, Future, Pin};  
use $crate::macros::support::Poll::{Ready, Pending};  
  
let mut futures = ( $( maybe_done($e), )* );  ^27
let mut futures = &mut futures;  
let mut skip_next_time: u32 = 0;  
  
poll_fn(move |cx| {  ^28
    const COUNT: u32 = $($total)*;  ^29
  
    let mut is_pending = false;  
  
    let mut to_run = COUNT;  
  
    let mut skip = skip_next_time;  
  
    skip_next_time = if skip + 1 == COUNT { 0 } else { skip + 1 };  
  
	loop {  
	    $(  ^30
	        if skip == 0 {  
	            if to_run == 0 {  
	                break;  
	            }  
	            to_run -= 1;  
	  
	            // Extract the future for this branch from the tuple.  
	            let ( $($skip,)* fut, .. ) = &mut *futures;  ^31
	  
				let mut fut = unsafe { Pin::new_unchecked(fut) };  
	  
	            if fut.poll(cx).is_pending() {  
	                is_pending = true;  
	            }  
	        } else {  
	            skip -= 1;  
	        }  
	    )*  
    }  
  
    if is_pending {  
        Pending  
    } else {  
        Ready(($({  
            // Extract the future for this branch from the tuple.  
            let ( $($skip,)* fut, .. ) = &mut futures;  ^32
  
			let mut fut = unsafe { Pin::new_unchecked(fut) };  
  
            fut.take_output().expect("expected completed future")  
        },)*))  
    }  
}).await
```

Examining how `$total` gets used `^29` , we see that the sum is evaluated to a numeric constant at compile time: `const COUNT: u32 = $($total)*`. This is how the `poll_fn` closure `^28`, not the macro, knows how many async branches it's dealing with.

The closure sets up a polling loop for the tuple of `Future`s created from each of the async expressions at `^27`. Within this loop, all of the code at `^30` is repeated for each `$skip $e` pair in the normalized input.

`$skip`, as we know, is a variable number of underscores. The first expression is associated with zero underscores, the second expression with one underscore, and so on.

Why underscores?

This really is such an impressive insight from the authors. Consider me a fanboy.

An underscore is the syntax used in Rust patterns to mean "match anything".

Within the loop, for each `$skip $e` pair, the code at `^31` matches the `Future` in `futures` corresponding to the number of underscores in `$skip`.

```rust
let ( $($skip,)* fut, .. ) = &mut *futures;
```

When `$skip` is empty, the `Future` at position `0`  gets matched. This corresponds to `do_stuff_async`.

When `$skip` is `_`, the assignment is `let (_, fut, ..) = &mut *futures`, matching the `Future` associated with `more_async_work`. 

The same technique is used to generate the return value of `join!` from all the completed `Future`s `^32`.

This is all stored on the stack. No heap allocations required.

"Smart", you might think, "but couldn't we do this more readably with array indexing?"

We've already seen how to calculate a numeric constant from a list of tokens. Why not store the `Future`s in an array, and associate each async expression with an index in the form `(0) do_stuff_async(), (0+1) more_async_work`?

This would result in the correct indices, it's true. But unlike tuples, arrays must be homogeneous. This would force the `Output` type of all `Future`s to be the same.

The `join!` macro's compile-time tuple manipulation avoids this limitation entirely.


@@@warning
Recursive counting

Macros are limited to 128 recursions by default, so this approach is unsuitable for counting large collections.

Try [the exercises](#exercises) to explore alternative ways of counting with macros.
@@@


## Rust macros rule

There you have it – three, production-grade Rust macros at varying degrees of complexity but not a speck of magic dust between them.

The best way to continue dispelling the aura that surrounds `macro_rules!` is to write your own, and supplement by studying code that takes you outside your comfort zone.

This is the path to exceptional software engineering. By setting out to study macros from great Rust devs, we've broadened our knowledge of the standard library, HTTP handling and async too.

Thanks for joining me on the journey.

What are some of the best macros you've seen in the Rust ecosystem? Let me know in the comments below!

## Exercises

1. 🦀
Without looking at the stdlib definition, write your own version of `assert_ne!`.

2. 🦀🦀🦀
Implement a trait of your own definition for all tuples of up to six items. The last item in each tuple should implement `Display`.

3. 🦀🦀🦀🦀
Implement a macro that takes an arbitrary number of tokens and returns the number of tokens it was passed as a `usize`.

Can you design such a macro that takes more than 128 tokens? More than 500? [_The Little Book of Rust Macros_](https://danielkeep.github.io/tlborm/book/blk-counting.html) will help if you get stuck.
