---
slug: ultimate-guide-rust-newtypes
git_tag_base: ultimate-guide-rust-newtypes
title: The ultimate guide to Rust newtypes
meta_title: The Ultimate Guide to Rust Newtypes
description: Clean up your code, clarify business logic and improve test coverage with Rust's newtype wrappers.
meta_description: Clean up your code, clarify business logic and improve test coverage with Rust's newtype wrappers.
tags: [rust, newtypes, type-driven design]
version: 1.0.15
color: peony
hero_image_url: https://res.cloudinary.com/dkh7xdo6x/image/upload/v1734678002/mechacrab_300x300_e0c164f945.webp
hero_image_alt: Illustration of a new type of railgun-mounted mecha-crab
og_image_url: https://res.cloudinary.com/dkh7xdo6x/image/upload/v1715592802/newtypes_og_4221823a11.jpg
published_at: 2025-05-13
---

# The Ultimate Guide to Rust Newtypes

::toc

::::::standalone

## What are newtype wrappers in Rust?

You've read [The Book](https://doc.rust-lang.org/book/), so I'm sure you've heard of newtypes – thin wrappers around other Rust types that allow you to extend their functionality.

But if reading The Book is your only exposure to newtypes, you might think that they're only useful for getting around the Orphan Rule. Think again.

:::aside{type=info}
::uh1[The Orphan Rule]

You can implement a trait for a type only if either the trait or the type is defined in your crate.
:::

By wrapping `Vec<String>` in a tuple struct, The Book shows us how to implement a trait defined in a crate we don't control on a struct which is also outside our control.

```rust
struct Wrapper(Vec<String>);

impl fmt::Display for Wrapper {
	fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
		write!(f, "[{}]", self.0.join(", ")) }
	}
}
```

This is an essential Rust skill, to be sure. But it pales in comparison to the true power of newtypes as the building blocks of type-safe, maintainable, well tested applications.

Would you like to prevent your codebase from becoming a hot mess? Is your codebase _already_ a hot mess? Well then, let's get into it.

## Why newtype-driven design will change your life

Newtypes are the raw ingredients of type-driven design in Rust, a practice which makes it almost impossible for invalid data to enter your system.

Right now, somewhere in your organization's codebase, is a function that looks like this – I guarantee it.

```rust
pub fn create_user(email: &str, password: &str) -> Result<User, CreateUserError> { :coderef[1]
    validate_email(email)?;
    validate_password(password)?; :coderef[2]
    let password_hash = hash_password(password)?;
    // Save user to database
    // Trigger welcome emails :coderef[3]
    // ...
    Ok(User)
}
```

Does this make you uneasy? It should.

At :codelink[1] we accept two `&str`s: `email` and `password`. The probability that someone, at some point, will screw up the order of these arguments and pass a password as `email` and an email address as `password` is 1.

I've been that person. You've been that person. Everyone on your team _is_ that person. It's a time bomb.

This is problematic when you consider that an email address is generally safe to log (depending on your data protection regime 🙃), whereas logging a plaintext password will bring great shame upon your family – and great fines upon your company when the predictable breach occurs.

Because of this liability, your business-critical function has to concern itself with checking that the `&str`s it's been given are, in fact, an email address and a password :codelink[2].

It would much rather be doing important business-logic things :codelink[3], but it's cluttered with code making sure its arguments are what they claim to be. What is this, JavaScript?

::subscribe

This uncomfortable cohabitation results in complex error types:

```rust
#[derive(Error, Clone, Debug, PartialEq)] :coderef[4]
pub enum CreateUserError {
    #[error("invalid email address: {0}")]
    InvalidEmail(String),
    #[error("invalid password: {reason}")]
    InvalidPassword {
        reason: String,
    },
    #[error("failed to hash password: {0}")]
    PasswordHashError(#[from] BcryptError),
    #[error("user with email address {email} already exists")]
    UserAlreadyExists {
        email: String,
    },
    // ...
}
```

:::aside{type=info}
::uh1[`thiserror`]

When you see `#[derive(Error)]` :codelink[4], you're usually watching the [`thiserror` crate](https://docs.rs/thiserror/latest/thiserror/) in action. `thiserror` is a powerful library for quickly creating expressive error types, and I highly recommend it.
:::

And complex error types means you need a lot of test cases to cover all practical outcomes:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_create_user_invalid_email() {
        let email = "invalid-email";
        let password = "password";
        let result = create_user(email, password);
        let expected = Err(CreateUserError::InvalidEmail(email.to_string()));
        assert_eq!(result, expected);
    }

    #[test]
    fn test_create_user_invalid_password() { unimplemented!() }

    #[test]
    fn test_create_user_password_hash_error() { unimplemented!() }

    #[test]
    fn test_create_user_user_already_exists() { unimplemented!() }

    #[test]
    fn test_create_user_success() { unimplemented!() }
}
```

If this looks reasonable and easy to follow, keep in mind that I'm a freak for naming and documentation. You should be too. But we both have to come to terms with the fact that your standard teammate won't name these functions consistently.

When asked what their test function tests, this teammate might tell you, "just read the code". This individual is dangerous, and should be treated with the same fear and suspicion you reserve for C++.

Functions with this many branching return values are not reasonable.

Imagine if the validations inside `create_user` occurred in parallel, or that the success of the function was dependent on a subset of validations succeeding, but not all of them. Suddenly you'd find yourself testing _permutations_ of failure cases – a scenario that should induce tremors and a cold sweat.

This is how many real production functions behave and, let me tell you, I don't want to be the one testing that code.

Newtyping is the practice of investing extra time upfront to design datatypes that are always valid. In the long run, you prevent human error, maintain fabulously readable code and make unit testing trivial.

> "Newtyping is the practice of investing extra time upfront to design datatypes that are always valid."

I'll show you how.

## Newtype essentials

We agree on why this matters. Excellent. Now, let's take our first step in the right direction.

```rust
#[derive(Debug, Clone, PartialEq)]
pub struct EmailAddress(String);

#[derive(Debug, Clone, PartialEq)]
pub struct Password(String); :coderef[5]

pub fn create_user(email: EmailAddress, password: Password) -> Result<User, CreateUserError> { :coderef[6]
    validate_email(&email)?;
    validate_password(&password)?;
    let password_hash = hash_password(&password)?;
    // ...
    Ok(User)
}
```

We can define tuple structs :codelink[5] as wrappers around owned `String`s that represent an email address and a password. Just like The Book showed us!

Now, our function requires distinctly typed arguments :codelink[6]. It's impossible to pass a `Password` to as an argument of type `EmailAddress`, and vice versa.

We've eliminated one source of human error, but believe me, plenty remain. Never forget, if a software engineer _can_ fuck it up, they _will_ fuck it up.

> "If a software engineer _can_ fuck it up, they _will_ fuck it up."

In the depths of a hangover, you might be tempted to unleash this particular evil into your repo:

```rust
pub struct EmailAddress(pub String)
```

Don't.

If you make the wrapped type `pub`, there's absolutely nothing to stop you from doing this after a little hair of the dog:

```rust
let email_address = EmailAddress(raw_password);
let password = Password(raw_email_address);
create_user(email_address, password)?;
```

Good work. 👏

A strong test suite will catch this mistake. Hell, a crappy test suite should catch _this_ mistake. But you're on How To Code It – you've sworn off the crappy code. So how can you guarantee that any `EmailAddress` or `Password` that `create_user` gets passed is valid?

I'm glad you asked.

### Constructors as the source of truth

Instead of running validations on data that may or may not be valid when it's already inside the core of your application, require your business logic to accept only data that has been parsed into an acceptable representation.

:::aside{type=info}
::uh1["Parse, don't validate"]

This phrase was made famous in software engineering circles by [Alexis King's blog post](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/) of the same name. It's one of the all-time great pieces of writing on type-driven design.
:::

First, let's pull our email address validation code out of the business function it was cluttering. From this point onwards, I'll be giving code only for `EmailAddress` – I've left the implementation of `Password` as an [exercise](#exercises).

```rust
#[derive(Debug, Clone, PartialEq)]
pub struct EmailAddress(String);

#[derive(Error, Debug, Clone, PartialEq)]
#[error("{0} is not a valid email address")]
pub struct EmailAddressError(String); :coderef[7]

impl EmailAddress {
    pub fn new(raw_email: &str) -> Result<Self, EmailAddressError> {
        if email_regex().is_match(raw_email) { :coderef[8]
            Ok(Self(raw_email.into()))
        } else {
            Err(EmailAddressError(raw_email))
        }
    }
}

#[derive(Error, Clone, Debug, PartialEq)]
#[error("a user with email address {0} already exists")]
pub struct UserAlreadyExistsError(EmailAddress); :coderef[9]

fn create_user(email: EmailAddress, password: Password) -> Result<User, UserAlreadyExistsError> {
    // Save user to database
    // Trigger welcome emails
    // ... :coderef[10]
    Ok(User)
}
```

This is very exciting. In order to get hold of an `EmailAddress`, a raw string _must_ pass the validation performed in the `EmailAddress::new` constructor. That means that any email address passed to `create_user` _must_ be valid, so `create_user` no longer needs to check – it's all business logic, baby! :codelink[10]

:::aside{type=warning}
::uh1[Email address validation]

You may be surprised by what counts as a valid email address. Take this Lovecraftian nightmare: `@1st.relay,@2nd.relay:user@final.domain`.

No, really. This is an honest-to-god example from an actual [RFC](https://datatracker.ietf.org/doc/html/rfc1711.html#section-7).

Never write your own email validation regex :codelink[8]. Don't copy-paste regexes you find on Stack Overflow. Use a well maintained library with good community standing, and proceed on the assumption that they've also got it wrong.
:::

Voilá. We have drastically simplified the error handling. Both `EmailAddress::new` and `create_user` now return only one type of error each :codelink[7] :codelink[9]. And notice how, at :codelink[9], even our error types contain guaranteed-valid, type-safe fields!

Now, we can write sane unit tests instead of badly disguised integration tests.

```rust
#[cfg(test)]
mod email_address_tests {
    use super::*;

    #[test]
    fn test_new_invalid_email() {
        let raw_email = "invalid-email";
        let result = EmailAddress::new(raw_email);
        let expected = Err(EmailAddressError(raw_email.to_string()));
        assert_eq!(result, expected);
    }

    #[test]
    fn test_new_success() { unimplemented!() }
}

#[cfg(test)]
mod create_user_tests {
    use super::*;

    #[test]
    fn test_user_already_exists() { unimplemented!() }

    #[test]
    fn test_success() { unimplemented!() }
}
```

Do you see how we're getting extraordinary value from a small shift in mindset? We're using Rust's remarkable type system to do a lot of heavy lifting for us. If an instance of a newtype exists, we know that it's valid.

:::aside{type=info}
If you're especially perceptive, you may have spotted something off about our definition of `create_user` and our desire to test whether a user already exists.

Checking that a user exists requires a database lookup but, for the sake of simplifying my examples, `create_user` doesn't allow for a mock database to be injected during test.

If you're hungry to learn a clean, maintainable way to achieve this, check out [Master Hexagonal Architecture in Rust](https://www.howtocodeit.com/articles/master-hexagonal-architecture-rust)!
:::

### Newtype mutability

It makes sense for some newtypes to be mutable. Just take care that every mutating method preserves the newtype's invariants:

```rust
struct NonEmptyVec<T>(Vec<T>);

impl<T> NonEmptyVec<T> {
    fn pop(&mut self) -> Option<T> {
        if self.0.len() == 1 { :coderef[11]
            None
        } else {
            self.0.pop()
        }
    }

    fn last(&self) -> &T {
        self.0.last().unwrap() :coderef[12]
    }
}
```

`NonEmptyVec` is a wrapper around a `Vec<T>` that must always have at least one element. I've omitted the constructor for brevity.

`NonEmptyVec::pop` takes `&mut self`, which means we need to check that we make only valid mutations. We can't pop the final element from a `NonEmptyVec` :codelink[11]!

The flip side of taking these precautions is that other operations become simpler. Unlike `Vec<T>::last`, `NonEmptyVec<T>::last` is infallible, so we don't need to return an `Option<&T>` :codelink[12].

## The most important newtype trait implementations

So we're agreed that newtypes are fire 🔥. Let's turn our attention to making them as easy as possible to work with. I'll start simple, and work up to more adventurous code.

### `derive` standard traits

You likely want your newtype to behave in a similar way to the underlying type. An `EmailAddress` is identical to a `String` for the purposes of equality, ordering, hashing, cloning and debugging. We can handle this simple case with `derive`:

```rust
#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct EmailAddress(String);
```

`String` also implements `Default`, but a "default email address" doesn't make much sense, so we don't derive it. And, since `String` isn't `Copy`, neither is `EmailAddress`.

:::aside{type=info}
::uh1[Build the habit]

Deriving these standard traits for _any_ type where they make sense is a good habit to develop – even if you don't immediately see a use for them.

This is especially important if you're writing a library, since your users won't be able to implement these traits themselves due to [the Orphan Rule](#the-orphan-rule). They'd have to wrap your newtypes in their own newtypes just to get this basic functionality.

Kiss those GitHub stars goodbye.
:::

But what about `Display`? There's no `derive` macro for `Display`, so let's do it manually for now.

```rust
impl Display for EmailAddress {
    fn fmt(&self, f: &mut Formatter) -> fmt::Result {
        write!(f, "{}", self.0)
    }
}
```

:::aside{type=info}
Implementing `Display` gets us a free implementation of `std::string::ToString`. 🎉
:::

### Manually implement special cases

For more complex newtypes, manual implementations of common traits may be required. Here's `Subsecond`, an `f64` wrapper that represents a fraction of a single second in the range `0.0..1.0`.

```rust
#[derive(Error, Debug, Copy, Clone, PartialEq)]
#[error("subsecond value must be in the range 0.0..1.0, but was {0}")]
pub struct SubsecondError(f64);

#[derive(Debug, Copy, Clone, PartialEq, PartialOrd, Default)]
pub struct Subsecond(f64);

impl Subsecond {
    pub fn new(raw: f64) -> Result<Self, SubsecondError> {
        if !(0.0..1.0).contains(&raw) {
            Err(SubsecondError(raw))
        } else {
            Ok(Self(raw))
        }
    }
}
```

`f64` can implement neither `Eq` nor `Ord`, since `f64::NAN` isn't equal to anything – even itself! `f64` has no "total" equality and ordering that encompasses every possible value an `f64` may be. How sad. 😔

Happily, that's not true of `Subsecond`. There _is_ a total equality and ordering of all `f64`s between `0.0` and `1.0`. This calls for manual trait implementations.

```rust
#[derive(Debug, Copy, Clone, PartialEq, Default)] :coderef[13]
pub struct Subsecond(f64);

impl Eq for Subsecond {}

impl PartialOrd for Subsecond { :coderef[14]
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

impl Ord for Subsecond {
    fn cmp(&self, other: &Self) -> Ordering {
        match self.0.partial_cmp(&other.0) {
            Some(ordering) => ordering,
            None => unreachable!(), :coderef[15]
        }
    }
}
```

Notice that we're now deriving `PartialEq` but not `PartialOrd` :codelink[13]. How come?

If an implementation of `Ord` exists, a correct implementation of `PartialOrd::partial_cmp` simply wraps the return value of `Ord::cmp` in an `Option` :codelink[14] :codelink[15]. This prevents us from accidentally implementing `Ord` and `PartialOrd` in ways that disagree with each other.

A derived implementation of `PartialOrd` _wouldn't_ call our manual `Ord` implementation – it would call `PartialOrd` on the underlying `f64`. This isn't what we want, so we need to define both `PartialOrd` and `Ord` ourselves, or [Clippy will yell at us](https://rust-lang.github.io/rust-clippy/master/index.html#derive_ord_xor_partial_ord).

The logic is reversed for `Eq` and `PartialEq`. If an implementation of `PartialEq` exists, `Eq` is simply a marker trait that indicates that the type has a total equality. By deriving `PartialEq` and manually adding an `Eq` implementation, we're telling the compiler, "chill out, I know what's good".

## Write ergonomic newtype constructors with `From` and `TryFrom`

At some point your newtypes will – sadly – have to interact with Other People's Code. These Other People didn't have your domain-specific type in mind when they wrote their "code". They either have their own set of newtypes, or they pass around `&str` and `f64` like lunatics.

We need to make it easy to convert from their types to our types, using classic Rust patterns that won't surprise other devs. That means `From` and `TryFrom`.

### Infallible conversions

Choose `From` when your conversion is infallible. The standard library gives us a blanket implementation of `Into` for every type that implements `From`. For example:

```rust
struct WrappedI32(i32);

impl From<i32> for WrappedI32 {
    fn from(raw: i32) -> Self {
        Self(raw)
    }
}

fn demonstrate() {
    let _ = WrappedI32::from(84);
    let _: WrappedI32 = 84.into(); :coderef[16]
}
```

We only implemented `From`, but we got `Into` for free :codelink[16].

### Fallible conversions

More often than not, though, newtypes _don't_ have infallible conversions from other types. We can't turn just any `f64` into a `Subsecond`!

`TryFrom` is the trait of choice in this scenario.

```rust
impl TryFrom<f64> for Subsecond {
    type Error = SubsecondError;

    fn try_from(raw: f64) -> Result<Self, Self::Error> {
        Subsecond::new(raw) :coderef[17]
    }
}

fn demonstrate() {
    let _ = Subsecond::try_from(0.5);
    let _: Result<Subsecond, _> = 0.5.try_into();
}
```

Note how `TryFrom` is implemented as a simple call to the `Subsecond` constructor :codelink[17]. A newtype's constructor serves as its source of truth – never have multiple constructors for the same use case.

For instance, it's valid to have two constructors `Subsecond::default()` and `Subsecond::new(raw: f64)`, since these serve two distinct purposes. Avoid having competing implementations for `Subsecond::new(raw: f64)` and `Subsecond::try_from(raw: f64)`, however. This doubles the code you need to maintain and test for no benefit. Define conversion traits in terms of a canonical constructor.

> "Define conversion traits in terms of a canonical constructor."

:::aside{type=info}
::uh1[What's up with `FromStr`?]

Newcomers to Rust are often confused by the existence of `FromStr`, which is identical to `TryFrom<&str>` for most practical purposes.

Well, `FromStr` came first. It predates the addition of `TryFrom` to the standard library, and is something of a relic. In certain contexts it offers a little extra functionality, though.

Whether you choose to implement `FromStr` in addition to `From` or `TryFrom` depends on three things:

- Whether you're interfacing with older code that has parameters bounded by `FromStr`, in which case you have no choice but to implement it for your newtype.
- Whether you want to take advantage of imnplementations over `FromStr` types, like [`str::parse`](https://doc.rust-lang.org/stable/std/primitive.str.html#method.parse) and [`serde_with::DeserializeFromStr`](https://docs.rs/serde_with/latest/serde_with/derive.DeserializeFromStr.html).
- Personal preference.

In the latter case, _be consistent_. Whatever your team decides, document it and use a linter to enforce your choice, because none of you can be trusted to remember.

I don't usually implement `FromStr`. It's one less thing to test.
:::

## From newtypes to primitives: `AsRef`, `Deref`, `Borrow` and more

How should you get an underlying primitive back out of a newtype? I don't know about your database client, but mine accepts `&str`, not `EmailAddress`.

This requires a little more care than you might think.

### Writing user-friendly getters

As a starting point, we should implement getters with common-sense names to return the inner value.

```rust
impl EmailAddress {
    pub fn into_string(self) -> String {
        self.0
    }

	:coderef[18]

}
```

:::aside{type=info}
::uh1[Naming conventions]

Confused about when to name your methods `to_x`, `into_x` and `as_x`? [The Rust API guidelines](https://rust-lang.github.io/api-guidelines/naming.html#ad-hoc-conversions-follow-as_-to_-into_-conventions-c-conv) have got you covered.
:::

Recall that implementing `Display` gives us a free implementation of `ToString`. Shadowing this implementation is such bad news that [Clippy considers this an error](https://rust-lang.github.io/rust-clippy/master/index.html#inherent_to_string_shadow_display). That's why I haven't defined `EmailAddress::to_string` at :codelink[18].

Consider carefully how other developers will use your code. If you're writing an application maintained by one team, which won't be a dependency of some higher-level code, you can stop implementing getters here.

If you're a library author with a specific target audience in mind, think about whether there are conversion traits in third-party crates that your users will almost certainly need.

I'm a contributor to an astrodynamics library for a space mission simulator. We perform many conversions between numeric types, and it's safe to assume that our users will too. This makes implementing `num::ToPrimitive` from [the popular `num` crate](https://github.com/rust-num/num) on our newtypes a reasonable thing to do.

### `AsRef`

Library code you interface with will expect `&str`, not `&EmailAddress`.

`AsRef` provides a convenient way to get a reference to a newtype's wrapped type:

```rust
impl AsRef<str> for EmailAddress {
    fn as_ref(&self) -> &str {
        &self.0
    }
}
```

### `Deref`

Sometimes your newtype will behave so much like its underlying type that you'd like it to dereference directly to the wrapped type.

```rust
impl Deref for EmailAddress {
    type Target = str;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}
```

Stop. Pause what you're listening to. Take a toilet break. Move your cat off your keyboard. I need your full attention.

`Deref` is a powerful trait that you should approach like you would disarming a very small bomb. It might not kill you, but it could leave you with nasty burns.

I'll show you the right wires to cut after demonstrating why it's attractive for use with newtypes:

```rust
fn demonstrate() {
    let raw = "EMAIL@ADDRESS.COM";
    let email = EmailAddress::new(raw).unwrap();
    takes_a_str(&email); :coderef[19]

    let lower = email.to_lowercase(); :coderef[20]
    assert_eq!(lower, "email@address.com");

	assert_eq!(*raw, *email); :coderef[21]
}

fn takes_a_str(_: &str) {}
```

Pretty impressive, no? `&email` isn't a `&str` :codelink[19], but `Deref` tells the compiler to treat it like one. This is an example of "[deref coercion](https://doc.rust-lang.org/std/ops/trait.Deref.html#deref-coercion)". We could also have written `takes_a_str(email.deref())`.

Deref coercion also gives us all the `&self` methods of `str` on `EmailAddress` :codelink[20]. This feels similar to inheritance in object-oriented languages, and it's what makes a `Box<T>` or `Rc<T>` almost as convenient to use as an instance of `T` itself. [The Rust Reference](https://doc.rust-lang.org/reference/expressions/method-call-expr.html) has the full details on how method lookup works in these cases.

Finally, it causes `*email` to desugar to `*Deref::deref(&email)` :codelink[21].

:::aside{type=warning}
Careful! `*email` is _not_ equivalent to `Deref::deref(&email)`, which is a source of confusion for people who expect the dereference operator `*` to return the return type of the `deref` implementation, which in our case would be `&str`.

`*` dereferences all the way down to memory bedrock – `str`.
:::

So why do we have to be cautious with `Deref`?

By adding every `&self` method of a wrapped type to a newtype, we vastly expand the newtype's public interface. This is the opposite of how we typically choose to publish methods, exposing the minimum viable set of operations and gradually extending the type as new use cases arise.

Does it make sense for your newtype to have all these methods? That's a judgement call. There's no good reason for an `EmailAddress` to have an `is_empty` method, since an `EmailAddress` can never be empty, but implementing `Deref` means that `str::is_empty` comes along for the ride.

This decision is critical if your newtype wraps a user-controlled type generically. In this situation, you don't know what methods will be available on the user's underlying type. What happens if your newtype defines a method that happens to have the same signature as a method on the user-provided type? The newtype's method takes precedence, so if the user is relying on your newtype's `Deref` implementation to call through to their underlying type, they're out of luck.

The best advice I've seen on this issue comes from [Rust for Rustaceans](https://rust-for-rustaceans.com/) (an essential read): prefer associated functions to inherent methods on generic wrapper types.

:::aside{type=info}
::uh1[Inherent methods]

An [inherent method](https://doc.rust-lang.org/reference/items/implementations.html#inherent-implementations) of type `T` is any method defined in a standard `impl T` block. Remember, methods have a `self` or `&self` receiver (or their `mut` variants). Functions don't.
:::

If a newtype has only associated functions, it has no methods that could inadvertently intercept method calls intended for the wrapped type. Behold:

```rust
/// A transparent wrapper around any type `T`.
struct SmartBox<T>(T);

impl<T> Deref for SmartBox<T> {
    type Target = T; :coderef[22]

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

impl<T> SmartBox<T> {
    fn print_best_rust_resource() { :coderef[23]
        println!("howtocodeit.com");
    }
}

struct ConfusedUnitStruct;

impl ConfusedUnitStruct {
    fn print_best_rust_resource(&self) { :coderef[24]
        println!("Scrabbling around on YouTube or something?");
    }
}

fn demonstrate() {
    let smart_box = SmartBox(ConfusedUnitStruct);
    smart_box.print_best_rust_resource();
    SmartBox::<ConfusedUnitStruct>::print_best_rust_resource(); :coderef[25]
}
```

Calling `demonstrate` outputs

```text
Scrabbling around on YouTube or something?
howtocodeit.com
```

At :codelink[22] we implement `Deref` so that `SmartBox<T>::deref` returns `&T`. Our `SmartBox`, being clever, has an associated function informing us that [howtocodeit.com](https://www.howtocodeit.com) is the best Rust resource :codelink[23].

But what's this? A bewildered developer wants to wrap their `ConfusedUnitStruct` in `SmartBox`! `ConfusedUnitStruct` has a method with some very concerning views about the path to Rust mastery :codelink[24].

Luckily, `SmartBox` believes that all views should be heard – even the ones that are wrong. Because `SmartBox` implements `print_best_rust_resource` as an associated function, it can't clash with any method implemented by the types it derefs to. Both functions can be called unambiguously :codelink[25].

> "Prefer associated functions to inherent methods on generic wrapper types."

### `Borrow`

:::aside{type=warning}
If you're looking for a way to shoot yourself in the foot with safe Rust, the `Borrow` trait is a great option.
:::

`Borrow` is deceptively simple. A type which is `Borrow<T>` can give you a `&T`.

```rust
impl Borrow<str> for EmailAddress {
    fn borrow(&self) -> &str {
        &self.0
    }
}
```

When used properly, a newtype that implements `Borrow` says, "I am identical to my underlying type for all practical purposes". It is expected that the outputs of `Eq`, `Ord` and `Hash` are the same for both the owned newtype and the borrowed, wrapped type – but this isn't statically enforced. 🫠

For example, if we manually implement `PartialEq` for `EmailAddress` to be case insensitive (as indeed email addresses are), `EmailAddress` _cannot_ implement `Borrow<str>` without unleashing this unspeakable darkness:

```rust
impl PartialEq for EmailAddress {
    fn eq(&self, other: &Self) -> bool {
        self.0.eq_ignore_ascii_case(&other.0) :coderef[26]
    }
}

impl Hash for EmailAddress {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.0.to_ascii_lowercase().hash(state); :coderef[27]
    }
}

impl Borrow<str> for EmailAddress {
    fn borrow(&self) -> &str {
        &self.0 :coderef[28]
    }
}

fn no_no_no_no_no() {
    let mut login_attempts: HashMap<EmailAddress, i32> = HashMap::new(); :coderef[29]

    let raw_uppercase = "EMAIL@TEST.COM";
    let raw_lowercase = "email@test.com";

    let email = EmailAddress::new(raw_uppercase).unwrap(); :coderef[30]

    login_attempts.insert(email, 2); :coderef[31]

    let got_attempts = login_attempts.get(raw_lowercase); :coderef[32]
    assert_eq!(got_attempts, Some(&2));
}
```

```text
assertion `left == right` failed
  left: None
  right: Some(2)
```

At :codelink[26] we make `EmailAddress` equality case insensitive, remembering that two equal objects must also have equal hashes :codelink[27]. The `Borrow` implementation at :codelink[28] heralds the end of days.

We instantiate a `HashMap` with `EmailAddress` keys :codelink[29]. `HashMap` owns its keys, but allows you to look up entries using any hashable type for which the owned key type implements `Borrow`. Since `EmailAddress` implements `Borrow<str>`, we _should_ be able to query `login_attempts` using `&str`.

We create an `EmailAddress` from the raw, uppercase `&str` :codelink[30], and insert it into `login_attempts` with a value of `2` :codelink[31].

When we attempt to get the value back out using `raw_lowercase` as the key :codelink[32]... armageddon. There is no corresponding entry in the `HashMap`. 😱

This happens because we've violated `HashMap`'s assumption about how `Borrow` should be implemented. A type which is `Borrow<T>` must hash and compare _identically_ to `T`. Since `EmailAddress` hashes and defines equality differently to `&str`, we cannot use `&str` to look up `EmailAddress`es in a `HashMap`.

Since these invariants are assumed but not enforced, I consider `Borrow` implementations "unofficially unsafe". Scrutinize any `Borrow` implementation you see in code review.

> "Scrutinize any `Borrow` implementation you see in code review."

In its current form, `EmailAddress` should not implement `Borrow`. We can fix it though. If we perform the lowercasing in the `EmailAddress` constructor, there's no need to implement `PartialEq` and `Hash` manually.

```rust
impl EmailAddress {
    pub fn new(raw_email: &str) -> Result<Self, EmailAddressError> {
        if email_regex().is_match(raw_email) {
            let lower = raw_email.to_ascii_lowercase();
            Ok(Self(lower))
        } else {
            Err(EmailAddressError(raw_email.to_string()))
        }
    }
}
```

In this arrangement, `EmailAddress` is equivalent to `&str` for the sake of hashing, equality and ordering, so it's safe to implement `Borrow`. Yet another reason to make your constructors the source of truth for what it means to be a valid instance of a type.

You have been warned.

## Bypassing newtype validations

We've come so far. We've learned how to build a type-driven utopia, populated by happy, valid instances of newtypes. Testing is easy now, and human error is a passing memory.

But wait – what's that on the horizon? Hark, tis the dark, acrid smoke of a developer who thinks they know better, seeking to bypass our constructors!

You might be expecting me to cast this evil into the abyss, but bypassing strict constructors can be a reasonable thing to do. We often know that a raw value is valid and don't want to incur the cost of revalidating it just to wrap it in a newtype. Every email address that went into the database was valid, so why would we check them again when pulling them out?

Building backdoors in your newtypes requires care, but there are two ways to limit the fallout: marker types and `unsafe`.

Marker types are a powerful feature of Rust's type system which we can use to constrain what can be done with a value depending on its source. Consider this approach a role-based permissions system within the type system itself!

As it sounds, using marker types to represent provenance is more advanced than anything we've discussed so far. In fact, it can introduce more complexity than it solves. For this reason, I'll be saving marker types for a separate article. I know, I'm a tease, but they need at least 3,000 words of their own.

There's plenty to keep us occupied with `unsafe`, however.

`unsafe` is a signal to other developers that some invariant exists that is not enforced by the compiler, and extra care must be taken to make sure that it holds. It's also a signal to yourself, when a week has passed and you have no memory of what you were doing. A better keyword would perhaps be `check_this_carefully_youre_on_your_own`.

The Rust standard library provides several examples of `unsafe` constructors that bypass costly validations. Convention is to suffix such functions with `_unchecked`. For example, on `std::string::String` we have:

```rust
pub unsafe fn from_utf8_unchecked(bytes: Vec<u8>) -> String
```

which doesn't check that the content of `bytes` is valid UTF-8.

Following this convention for `EmailAddress` gives us this:

```rust
impl EmailAddress {
    pub fn new(raw_email: &str) -> Result<Self, EmailAddressError> {
        if email_regex().is_match(raw_email) {
            let lower = raw_email.to_ascii_lowercase();
            Ok(Self(lower))
        } else {
            Err(EmailAddressError(raw_email.to_string()))
        }
    }

    pub unsafe fn new_unchecked(raw_email: &str) -> Self {
        Self(raw_email.to_string())
    }

    // ...
}
```

For raw email addresses entering your system from web requests, `EmailAddress:new` should be used to ensure validity before they're saved to the database. For email addresses coming out of the database, `EmailAddress::new_unchecked` is acceptable, because the validation and case normalization has already been done.

If you ever see `unsafe` used in code you're reviewing, make yourself a fresh coffee. You need to pay attention.

## How to eliminate newtype boilerplate

We're almost at the end of our journey. We're walking taller, writing better code and better tests than before. We stride like giants over Rust developers still writing validations into their business logic. You know which traits to implement, and when to implement them. You understand the power and responsibility of dipping into `unsafe` code. But...

...doesn't this all seem like a _lot_ of effort?

Indeed, writing robust newtypes comes with a lot of boilerplate. I have two libraries to share with you that can slash that boilerplate down to size, but let's make an agreement first: before you start using them, practice writing your own newtypes by hand.

Before introducing magic from external crates, you should understand what you're automating and why. Check out [the exercises](#exercises) at the end of this article for a head start!

### Using `derive_more` to... derive more

[`derive_more`](https://docs.rs/derive_more/latest/derive_more/) is a crate designed to ease the burden of implementing traits on newtypes. It lets you `derive` implementations for `From`, `IntoIterator`, `AsRef`, `Deref`, arithmetic operators, and more.

```rust
#[derive(Clone, Debug, Display, PartialEq, Eq, PartialOrd, Ord, AsRef, Deref)]
struct EmailAddress(String);
```

If you want to simplify the process of writing newtypes while minimizing the magic in your codebase, this is where I'd start.

### Going all-in with `nutype`

[`nutype`](https://docs.rs/nutype/latest/nutype/) is a formidable procedural macro that generates sanitization and validation code for your newtypes, including dedicated error types.

```rust
#[nutype(
    sanitize(trim, lowercase),
    validate(regex = EMAIL_REGEX),
	derive(
		Clone, Debug, Display,
		PartialEq, Eq, PartialOrd,
		Ord, AsRef, Deref,
		Borrow, TryFrom
	),
	new_unchecked,
)]
pub struct EmailAddress(String);

pub fn demonstrate() {
    let result = EmailAddress::new("not an email address");
    if let Err(e) = result {
        match e {
            EmailAddressError::RegexViolated => println!("{}", e),
        }
    }
}
```

As you can see, practically _everything_ we've been doing manually up to this point can be generated by `nutype`. It's a huge time saver.

:::aside{type=info}
`nutype`'s support for `new_unchecked` generation and `regex` validation lies behind their respective feature flags. You can add them with `cargo add nutype --features new_unchecked regex`.
:::

Beware the corners that you choose to cut, though. `nutype`'s generated error messages are quite vague, and there's no way to override them or include additional detail:

```text
EmailAddress violated the regular expression.
```

This hampers debugging and puts the onus on the caller to wrap the newtype's associated error with additional context. It's good practice to do so regardless, but this omission feels out of place in Rust – just think about how good Rust's compiler error messages are.

:::aside{type=warning}
Unlike `derive_more`, using `nutype` is a commitment. By adopting its santizers, validations and error types, you will find it quickly becomes a hard dependency that pervades your codebase and is tough to migrate away from.

This might be fine for you, but take your time and think about how `nutype` will work for your particular application and organization before adopting it.
:::

Go now. I have nothing more to teach you.
::::::

::::::exercises
:::exercise{difficulty=2}
Design a newtype, `Password`, representing a user's password.

The `Password` constructor should ensure that passwords submitted by users are at least 8 ASCIl characters long – any other password policy you'd like to enforce is up to you!

In real-life code, we want to minimise the time that user passwords are exposed as human-readable strings. A classic mistake is logging raw passwords accidentally. With this in mind, `Password` should not hold onto the user-submitted string, but should store a hash instead.
:::

:::exercise{difficulty=1}
Implement a method on `Password` allowing it to be compared with candidate password strings submitted by users during login attempts.

Your method should return a `Result`: `0k()` if `Password` matches the candidate, or an `Err` of your choice if not.
:::

:::exercise{difficulty=3}
Company password policies change over time! Redesign `Password` so that future changes to the password policy won't require changes to the `Password` type.

For example, the password policy in question 1 is that all passwords must be at least 8 ASCIl characters long. A product manager has now had the great idea that all passwords must contain at least one emoji. 🙃

This means ensuring that new passwords comply with the emoji policy, but also that existing passwords continue to be valid according to the old policy, or your users will be locked out!

The `Password` newtype is maintainable to the extent that we can implement any new password policy without changing how the `Password` type works. That is, we must decouple the representation of the password from the implementation detail of the password policy.

This will require some Rust that I haven't demonstrated in this article, so don't be afraid to think outside the box.
:::
::::::

::discussion{title="The ultimate guide to Rust newtypes"}
