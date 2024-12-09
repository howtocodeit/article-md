---
title: The definitive guide to error handling in Rust
meta_title: The Definitive Guide to Error Handling in Rust
slug: the-definitive-guide-to-rust-error-handling
description: Learn to model and handle any error using idiomatic Rust.
meta_description: Learn to model and handle any error using idomatic Rust.
color: sky
tags: [rust, architecture, error handling]
version: 1.0.4
---

Are you overwhelmed by the amount of choice Rust gives us for handling errors? Confused about when to return a structured error type or a `Box<dyn Error>`? Intimidated by `Box<dyn Error + Send + Sync + 'static>`'s beefy type signature?

Whether you're building an application or library, this guide will help you make the right decision.

I _love_ error handling. I'm obsessed. I work in the finance and space industries, and things go wrong _a lot_.

Failure cases vastly outnumber success cases. Knowing how to communicate what went wrong, to the right audience, in an appropriate amount of detail is a skill that sets you apart from other developers.

Think about how great the Rust compiler's error messages are compared to other programming languages. We want users of our code to have that same reaction, whether they're on our team or using our library. We want them to be impressed when things go wrong!

> Impress your users, even when things go wrong.

Before we dazzle anyone with our error handling skills, though, let's nail the fundamentals.

> Email sign-up

## Rust error handling basics

### What is an error in Rust?

In Rust, an error is any type that implements the `std::error::Error` trait. Here's the definition:

```rust src/core/error.rs
pub trait Error: Debug + Display {
    // Provided methods
    fn source(&self) -> Option<&(dyn Error + 'static)> { ... } ^1
    fn description(&self) -> &str { ... }
    fn cause(&self) -> Option<&dyn Error> { ... }
    fn provide<'a>(&'a self, request: &mut Request<'a>) { ... }
}
```

This is a moderately threatening trait definition, but all four of these methods have default implementations provided for us.

Any type that implements both `Debug` and `Display` can implement `Error`. There's very little manual work required.

In fact, `Error::cause` and `description` are deprecated in favor of `Error::source` and the `Display` implementation, respectively. You should never have to worry about them, except when working with older code.

`Error::provide` is part of [an experimental nightly build](https://github.com/rust-lang/rust/issues/99301), so I won't discuss it here. You won't have to worry about it unless you're working with cutting-edge, unstable code.


@@@warning
`Error::source`

Watch out for the default implementation of `Error::source`. It returns `None`.

If you want a custom error type to return the original error that caused it, you need to provide your own implementation.
@@@


The return type of `Error::source` warrants closer examination `^1`, because we'll see similar types throughout this guide.

You know what `Option` is already. `&(dyn Error + 'static)` simply means "a reference to some error that may live for the whole duration of the program".


@@@info
Umm, technically... 🙋‍♂️

I frequently refer to types with the `'static` lifetime as being "static". This is convenient shorthand, but subtly incorrect.

They're not static in the sense of a `static` variable. They have the `'static` lifetime, which means that no reference will outlive them, and they would exist until the end of the program if required to.

If the compiler determines that `'static` objects don't _need_ to live as long as the program, it's free to drop them sooner.

Please justify my sloppiness by making sure you're clear on the distinction.
@@@


The `'static` lifetime is important for error handling, because errors are often handled long after the code that causes them returns, sometimes on a different thread. 

Good luck handling an error that's been dropped unexpectedly! Rust protects us from this scenario.

You'll often see `'static` alongside `Send` and `Sync` bounds. `dyn Error + Send + Sync + 'static` describes "some error that can live as long as the program, be sent between threads by value or shared across threads by immutable reference".

Error::source's return type, `&(dyn Error + 'static)`, doesn't make any promises about thread safety.

In general, standard library code places more relaxed bounds on dynamic errors than you'll see in the broader ecosystem and use in your own projects.

This allows the widest variety of things to behave as errors, with stricter requirements left to the user's discretion.


@@@warning
`Error::source` returns a *non*-static reference to a static `Error`.
@@@


How do we make an `Error` type static? Simple – use only owned fields, or fields which specify the `'static` lifetime for references and trait objects.

The following type is only `'static` if the reference assigned to `field` happens to be `'static` itself:

```rust
pub struct QuestionablyStatic<'a> {
	field: &'a str,
}
```

These are always `'static`:

```rust
pub struct StaticByOwnership {
	field: String
}

pub struct ExplicitlyStatic {
	field: &'static str
}
```

### Errors in the context of `Result`

Surprisingly, the type wrapped by `std::result::Result::Err` doesn't need an `Error` bound:

```rust
pub enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

You can use whatever type you want to represent an error inside `Result`.

The same is true for associated types in many trait definitions, such as `std::str::FromStr`:

```rust
pub trait FromStr: Sized {
    type Err; ^2

    fn from_str(s: &str) -> Result<Self, Self::Err>;
}
```

`Err` isn't bounded by `Error` ^2!

Although you _can_ use any types in these contexts, I strongly encourage you to only use `Error` implementations.

Other Rust developers will expect these things to behave like `Error`s, and we should strive to be as unsurprising as possible. That doesn't stop you from implementing additional functionality on your `Error`s, though.

> Be unsurprising. Use `Error`-bounded types in most error contexts, even when not strictly required.

There are exceptions to this rule, often within the standard library itself. Look out for the discussion of `Error::downcast` and `Box<dyn Error>` in the next section.

Okay, we've nailed the essentials. Let's get into the choice that confuses most new Rust developers: should we use dynamic or statically typed errors?

## Dynamic error handling in Rust

### When to use `Box<dyn Error>` and friends

`Box<dyn Error>` is Rust's vaguest error type. It's just some object that implements `Error` 🤷. `Box<dyn Error + Send + Sync + 'static>` is its thread-safe counterpart.

The `Error` is boxed because, as a dynamic trait object, we don't know its size at compile time. We have to allocate it on the heap.

`Box<dyn Error>` simply says "something went wrong, check my message or my optional cause to know more".

This has two key properties:
* It's excellent for quickly communicating that something went wrong.
* It's god-awful at providing structured data for an error handler to act on.

If you would like consumers of your error – whether they're error handlers in your own application or users of your library – to be able to dynamically change their program's behavior based on the internal details of an error, don't use `Box<dyn Error>`.

Parsing error details from messages is fragile and hard to maintain. If you expect people to rely on your error messages to drive program behavior, you've also inadvertently made those error messages part of your public API. If that error message changes, code that parses it may break.

If you know that there's nothing useful a receiving program can do with the error, but that the message is helpful for a human debugger, then `Box<dyn Error>` and related trait objects are very convenient.

> Use `Box<dyn Error>` to quickly communicate that something went wrong to a human debugger or end user.

I work on an astrodynamics library for a space mission simulator funded by the European Space Agency. If someone inputs garbage data, like the time `23:59:60` on a year without leap seconds, there's really no way to recover. In this scenario, it would be perfectly reasonable to return `Box<dyn Error>` with a message that explains how silly they are.

Now, we don't actually do this – that's a story for Part III on structured errors – but it is a valid Rust error handling strategy.


@@@warning
Boxed errors aren't errors

Remember when I told you to be unsurprising and put only types that implement `Error` into `Result::Err`?

Well, surprise! `Box<dyn Error>` doesn't implement `Error` 🙄.

You need to wrap `Box<dyn Error>` in a newtype that _does_ implement `Error` to get this functionality. My [*Ultimate Guide to Rust Newtypes*](https://www.howtocodeit.com/articles/master-hexagonal-architecture-rust) has got you covered.

Alternatively, you can use a library like [anyhow](https://docs.rs/anyhow/latest/anyhow/) which provides such a type for you. We'll discuss this shortly.
@@@


#### Handling dynamic errors from other people's code

What if library code you call returns a dynamic error?

Hopefully, you just want to log it for a future debugging session. Surely the thoughtfully crafted error message will give you everything you need to solve the problem 🤡.

But say it doesn't, and you need to find out what's inside the `dyn Error`?

I don't envy you this situation. It's often an indicator of bad library design.

> The Laws of Thermodynamics state that the worst code is written by Other People.

Moaning about it won't help you in the moment, though. You need to downcast.

### Downcasting errors in Rust

Did you know that you can get a concrete error type back out of a boxed `dyn Error`?

I'm not going to get into _how_ the `std::error` crate does this, because it involves some scary `unsafe` code that has nothing to do with handling errors. That won't stop us from using it.


@@@info
If you'd like me to walk through the `std::error` internals, leave a comment and I'll write it!
@@@


`dyn Error` trait objects have three methods for attempting a transformation into some concrete type `T`:

```rust
pub fn downcast<T: Error + 'static>(self: Box<Self>) -> Result<Box<T>, Box<Self>>
pub fn downcast_mut<T: Error + 'static>(&mut self) -> Option<&mut T>
pub fn downcast_ref<T: Error + 'static>(&self) -> Option<&T>
```


@@@warning
Do you see it? The underlying error type must be `'static`, or you can't downcast to it. This is one more reason why it's good practice to design only `'static` error types.

Note also that the `Box<Self>` inside the `Result::Err` returned by `downcast` doesn't implement `Error`, but `Self` does. This is one of the cases where returning a non-`Error` inside `Result::Err` makes sense.
@@@

If the `dyn Error` is of type `T`, you'll get a `T` for closer inspection. Whether that `T` is owned or borrowed depends on which method you call.

All of this is useless if the underlying type is private to the crate the `dyn Error` came from. In this scenario, politely explain your predicament to the maintainers, then scream into a pillow.

#### Avoid forcing callers to downcast

I don't encourage designing your errors to require downcasting to figure out what's gone wrong.

If you choose to return a dynamic error, you are communicating that the internal structure of the error shouldn't matter to callers.

> Dynamic errors communicate that their internal structure shouldn't matter to callers.

Forcing them to dig into your crate's error types, identify the possible culprits, downcast, and react dynamically screams "leaky implementation details".

This is Rust, not Go.

#### So what's the point of downcasting?

If downcasting isn't an ideal way to handle errors, what is it good for? Let's use [Actix Web](https://actix.rs/) 4.7.0 as an example.

The primary Actix error struct, `Error`, has a single field, `cause`, that holds a `Box<dyn ResponseError>`.  

```rust actix-web src/error/error.rs
pub struct Error {  
    cause: Box<dyn ResponseError>,  
}
```

`ResponseError` is a trait with identical bounds to `std::error::Error`, but specifies methods to return a status code and an HTTP response body:

```rust actix-web src/error/response_error.rs
pub trait ResponseError: fmt::Debug + fmt::Display {
	fn status_code(&self) -> StatusCode
	fn error_response(&self) -> HttpResponse<BoxBody>
}
```

It has default implementations for both of these methods, but they're not important here.

What is important is the large number of concrete error types that Actix provides `ResponseError` implementations for: `Box<dyn std::error::Error + 'static>`, `Infallible`, `serde_json::Error`, `std::io::Error`, and many more.

Naturally, Actix users can implement `ResponseError` for their own types too, so `actix_web::error::Error` chooses a dynamic error type to wrap a theoretically infinite variety of `ResponseError`s.

Actix itself doesn't care about the internal structure of any particular `ResponseError`. It just needs a way to get a status code and response body when something goes wrong. This is a scenario where dynamic errors shine.

But you know who might care? *The team whose code produced the error*.

If an Actix user converts an error into Actix's opaque error format, they should reasonably expect to be able to get it out again. That's why `actix_web::error::Error` provides the `as_error` method, which downcasts to the user's original error type.

```rust actix-web src/error/error.rs

impl Error {
	pub fn as_error<T: ResponseError + 'static>(&self) -> Option<&T> {  
	    <dyn ResponseError>::downcast_ref(self.cause.as_ref())  
	}
}
```


@@@info
The implementation of `ResponseError::downcast_ref` is specific to Actix. It's not the same as `<dyn std::error::Error>::downcast_ref` – these are methods of distinct trait objects. However, the concept is the same.

(If you're confused by the `<dyn Trait>::method` syntax, it means that the method is defined on the dynamic trait object type, and not as part of the trait itself.)
@@@info


There are no leaky abstractions here, because the caller of `as_error` also owns the code that created the error in the first place.

Actix never calls `downcast_ref` itself. It doesn't use `downcast_ref` to handle errors. Rather, it provides `as_error` as a means for external parties using Actix's wrapper type to inspect their own implementation details.


@@@info
`downcast_ref` in tests

Ok, Actix _does_ call `downcast_ref`, but only in tests.

Tests are one of the few scenarios where you *should* care that some dynamic error you're returning has a specific underlying type.
@@@


### Handling Rust errors with anyhow

What discussion of dynamic error handling in Rust would be complete without talking about [anyhow](https://docs.rs/anyhow/latest/anyhow/index.html)?

anyhow is Rust's most-loved crate for handling errors in the laziest way possible.

`anyhow::Error` is effectively a `Box<dyn Error + Send + Sync + 'static>` with bells on. It always gives you a backtrace, and, unlike `Box`, it takes up only one machine word, not two (a "narrow pointer").

anyhow comes with a selection of macros, methods and blanket implementations to make wrapping and adding context to any `Display + Send + Sync + 'static` type a breeze.

Just like `actix_web::error::Error`, `anyhow::Error` is a wrapper for user-provided types. Seeing as those users might want their types back, it provides `downcast` methods in your three favorite flavors: owned, `&` and `&mut`.

I use anyhow often, and I find it's a better fit for applications than libraries.

If you return a concrete `anyhow::Error` across a crate boundary, you force the caller to depend directly on anyhow, and not everyone will want to.


@@@warning
Also, if you make anyhow part of your public interface, you can't upgrade to new major versions of anyhow without bumping the major version of your own crate.
@@@


As a general rule, return only your own or standard library error types across crate boundaries to minimize leakage of your implementation details into other people's code.

> Return only your own or standard library error types across crate boundaries.

### Who is your audience and what will they do with your error?

I hope it's becoming clear that how you choose to handle your errors depends on two key things:

* Who the audience for the error is.
* What they should be able to do with an error you give them.

Dynamic errors are great for consolidating a wide range of error types and returning them in a format where the only reasonable thing to do is write to output, whether that's a logger or an HTTP connection.

In Part III, we'll look at structured, statically typed errors as carriers of data that we can handle programmatically. More than that though, we'll see how they serve as invaluable, innate documentation for other developers.

When we understand both of these error handling styles, we'll bring them together, equipping ourselves with the knowledge to handle any kind of error that might arise, and avoid some nasty footguns.

## Structured error handling in Rust

_Part III is coming soon._