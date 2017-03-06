# Error Handling
When writing code, you may encounter cases in which something bad happens-- a
network connection fails, a database operation goes wrong, etc. At this point,
your program could simply fall over, crash, and decide that it cannot continue.
More likely, though, you'll employ some error handling techniques using the
primitives provided by your language. Here, we will take a look at the
differences in error handling between Go and Rust.

## The Go Way

Go's built-in mechanism for error handling lies within the `error` type provided
by the language. `error` is a type of Go `interface`, defined like so:

```go
type error interface {
   Error() string
}
```

The method required by the interface, `Error()`, is designed to provide a
description of the error that may be used for diagnostics or more specific
handling of various kinds of errors. For example, when reading data from a
database, you might use the `Error()` method to return an error with the
message `"Database Unavailable"` when the database could not be reached, or
use the `Error()` method to return a `"Could not find data"` error when the
requested query fails.

```go
// TODO rewrite this to make more sense
func (db *Database) ReadData(query string) (result string, err error) {
   // ...
   if (!db.connect()) {
      return ("", errors.New("Database Unavailable"))
   }
   // ...
   if (!db.runQuery(query)) {
      return ("", errors.New("Could not find data"))
   }
   // ...
}
```

The `error` type is an `interface` in Go, so it may be `nil`; in general, an
`error` that is `nil` indicates that there was no error at all. Consequently,
errors in Go are checked based on whether or not they are `nil`. A common
Go pattern for error handling looks like this:

```go
if bytes, err := json.Marshal(`{"Go": "Google", "Rust": "Mozilla"}`); err != nil {
   // Handle the error
} else {
   // No error!
}
```

A basic `error` can be created from a `string` by using the `errors.New()` function.

## The Rust Way

Rust's method of error-handling story is more sophisticated, and has quite a
few options built into the language. We will cover two `enum` types: `Option`
and `Result`. We will also cover the `try!` macro, the `?` operator, error
combinators, and error best practices.

#### `Option` and `Result`

Rust's standard library provides two types that are well-suited to error
handling: `Option<T>` and `Result<T,E>`. These are Rust `enum` types,
and are defined like so:

```rust
pub enum Option<T> {
   None,
   Some(T),
}

#[must_use]
pub enum Result<T, E> {
   Ok(T),
   Err(E),
}
```

An `Option<T>` _might_ contain a value of type `T`; if it does, its type is
wrapped in the `Some` variant. Its value will be `Some(t)`, where the value
`t` is of type `T`. Otherwise, the value of an `Option<T>` is simply `None`.


A `Result<T,E>` might contain the result of some computation, in this case a
value of type `T`. If everything in the computation was OK, the `Result` will
contain an `Ok(T)`; otherwise, it will contain an `Err(E)`.


You may also notice the `#[must_use]` annotation above the
definition for `Result`. This is a _language attribute_ that provides additional
information to the compiler. In this case, `#[must_use]` indicates that all
`Result`s _must always be used_ in some way; if you do not inspect the value of
a `Result`, for example via pattern matching or the `try!` macro
(more on that later), the compiler will emit a warning:

```
warning: unused result which must be used, #[warn(unused_must_use)] on by default
```

This is not a hard error, but nevertheless it should not be ignored!

#### `try!` and `?`

To make `Result`s a bit more ergonomic to use, Rust has two built-in mechanisms
that can both be used to handle `Result`s: the `try!` macro and the `?`
operator. `try!` and `?` are functionally identical, and both only work on
`Result`s (for now). For example:

```rust
use std::net::TcpStream;
fn bind_socket() -> Result<(), std::io::Error> {
   let socket1: TcpStream = try!(TcpStream::connect("127.0.0.1:8000"));
   // Note the question mark here ------------------------------v
   let socket2: TcpStream = TcpStream::connect("127.0.0.1:8000")?;

   let maybe_socket3: Result<TcpStream, std::io::Error> = TcpStream::connect("127.0.0.1:8000");
   let socket3: TcpStream = match maybe_socket3 {
      Ok(socket) => socket,
      Err(err) => return Err(err)
   };
   Ok(())
}
```

`try!` and `?` are equivalent to each other, and _roughly equivalent_ to the
`match` block in the code directly above this paragraph that declares
`socket3`. Generally `?` looks a bit cleaner, but it doesn't matter which one
you use. If the value of the `Result` is `Ok(val)`, `try!`/`?` yields the inner
value; if it is `Err(err)`, `try!`/`?` _returns the `Result` from the function
that is invoking `try!`/`?`_. Note that this is a bit of an oddity-- `try!` and
`?` can cause your function to _return early_, which may be surprising.
However, this makes chaining `Result` operations much easier; if you have a
number of error conditions to check, you can simply write something like this:

```rust
let val1 = err_fn4(err_fn3(err_fn2(err_fn1()?)?)?)?;
let val2 = try!(err_fn4(try!(err_fn3(try!(err_fn2(try!(err_fn1())))))));
```

Consider the much more repetetive alternative, using chained `match` blocks:

```rust
let val1 = match err_fn1() {
   Ok(t) => match err_fn2(t) {
      Ok(t) => match err_fn3(t) {
         Ok(t) => match err_fn4(t) {
            Ok(t) => t,
            Err(e) => return Err(e),
         },
         Err(e) => return Err(e),
      },
      Err(e) => return Err(e),
   },
   Err(e) => return Err(e),
};
```

#### Error Combinators

`Option` and `Result` have a number of built-in methods that make working with
these types a lot more smooth. For example, if you had a number of functions
that you wanted to chain, and they all return `Option`s, it would be quite
cumbersome to write something like this:

```rust
let x1: Option<i32> = maybe_give_me_an_int();
let x2: Option<String> = match x1 {
   Some(i) => Some(convert_int_to_string(i)),
   None => None
}
// ...etc
```

Rather than dealing with all of this boilerplate, the Rust standard library
provides a number of _error combinators_ for working with and converting between
`Option`s and `Result`s. For example, a cleaner way to write the above code:

```rust
let x1: Option<i32> = maybe_give_me_an_int();
let x2: Option<String> = x1.map(|i| convert_int_to_string(i));
```

We can do even more complex transformations as well. For example, the code below
converts an `Option` to a `Result` using a closure, maps the inner value in the
`Result` to a byte slice, then tries to move the value out of the `Result`,
returning a default value if it is an `Err`.

```rust
Some("body").ok_or_else(|| "once" + "told" + "me")
            .map(|s| s.as_bytes()).unwrap_or("L");
```

The full list of combinators may be found [here for `Option`](https://doc.rust-lang.org/std/option/enum.Option.html)
and [here for `Result`](https://doc.rust-lang.org/std/result/enum.Result.html).

#### Error Handling Best Practices

##### `unwrap` and `expect`

`Option` and `Result` provide two methods that are a sort of "nuclear option"
for error handling : `unwrap` and `expect`. Both are effectively equivalent to
this:

```rust
match some_option {
   Some(v) => v,
   None => panic!(),
}
```

`expect` is strictly more expressive than `unwrap`, as it allows you to provide
a message to `panic!`, but both are generally frowned upon _because they panic_.
In general, it is a bad practice to use these methods in real scenarios--
unless you truly cannot handle the error in any other way than panicking, these
methods should not be used. However, they are useful for proof-of-concept code,
or quick-and-dirty applications. Just make sure to remove them later unless you
_absolutely need them_.

##### Custom Error Types and the `Error` Trait

Many Rust libraries, for example `std::io`, define a type alias over `Result`
that wraps a custom error type defined in the library. `std::io::Result<T>` is
really a wrapper over `Result<T, std::io::Error>`, where `std::io::Error` is an
enum of possible IO-related errors. `std::io::Error` also implements the
`std::error::Error` trait, which is a trait used to combine error types more
effectively.


`std::error::Error` requires you to implement two traits first: `Debug` and
`Display`. `std::error::Error` itself then requires two methods: `description`
and `cause`, which simply provide a description of the error and an optional
underlying reason for the error, respectively.


While it is not strictly necessary to define your own error types and go through
all the effort of implementing the `Error` trait, it can make your error
handling process far more expressive, and can be highly beneficial to clients of
your library. Not implementing the `Error` trait also makes it much more
difficult for clients of your library to use your error types, as they will not
compose well with errors defined by clients.

### Similarities between the two

Go and Rust have very few similarities in their error handling; Rust's
error-handling mechanisms are far more complex than Go's. Generally, if you have
a function in Go that returns `(someType, error)`, an analogous function in Rust
would return `Result<SomeType, Error>` or `Option<SomeType`, if you don't care
about the kind of error being returned.

### Strengths Compared to Go

Rust's error handling story is much more expressive than Go's, and there is a
lot more built into the standard library. Rust has mechanisms for composing,
mapping/folding, and easily pattern-matching various kinds of errors, whereas Go
has a simple interface for handling errors that provides less to the programmer
at the benefit of less mental overhead. Most of Rust's error-handling
abstractions are zero-cost, whereas Go's `error` type is an interface that
requires heap allocation and dynamic dispatch. If a Go function has a return
type of `(someType, error)`, the function still has to return a value of
`someType` alongside the `error`, which can lead to scenarios where the "dummy"
value returned is accidentally used, even though there was a real error.

### Weaknesses Compared to Go

Rust's error handling's complexity comes at a fairly high mental cost along with
increased boilerplate to handle simple operations. The `Error` trait requires
two other traits in order to be implemented; `Result` and `Option` have very
many combinators that are sometimes obtuse or unintuitive; the `?` can cause
early returns, which may be unexpected; and the simplest way of handling errors
(`unwrap`/`expect`) causes panics.
