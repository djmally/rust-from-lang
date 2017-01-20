# Error Handling

When writing code, you may encounter cases in which something bad happens-- a
network connection fails, a database operation goes wrong, etc. At this point,
your program could simply fall over, crash, and decide that it cannot continue.
More likely, though, you'll employ some error handling techniques using the
primitives provided by your language. Here, we will take a look at the
differences between error handling in Go and Rust.

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
database, you could return an `error` with an `Error()` method that returns
`"Database Unavailable"` when the database could not be reached, or instead an
`error` that with an `Error()` method that returns `"Could not find data"` when
the requested query fails.

```go
// TODO rewrite this to make more sense
func (db *Database) ReadData(query string) (result string, err error) {
   // ...
   if (!db.connect()) {
      return errors.New("Database Unavailable")
   }
   // ...
   if (!db.runQuery(query)) {
      return errors.New("Could not find data")
   }
   // ...
}
```

The `error` type is an `interface` in Go, so it may be `nil`; in general, an
`error` that is `nil` indicates that there was no error at all. Consequently,
errors in Go are checked based on whether or not they are `nil`. A common
Go pattern for error handling looks like this:

```go
if bytes, err := json.Marshal(`{"go": "error", "rust": "result"}`); err != nil {
   // Handle the error
} else {
   // No error!
}
```

## The Rust Way

Rust's error-handling story is substantially longer, and has quite a few options
built into the language. We will cover two `enum` types: `Option` and `Result`;
the `try!` macro and `?` operator; error combinators; and error best practices.

Rust's standard library provides two types, `Option<T>` and `Result<T,E>`, that
are generally used to handle errors.

`Option<T>` and `Result<T,E>` are Rust `enum` types, and are defined like so:

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

An `Option<T>` _might_ contain a value of type `T`; if it does, it's wrapped in
the `Some` variant, like `Some(T)`.


A `Result<T,E>` might contain the result of some computation, in this case a
value of type `T`. If everything was OK, it will be an `Ok(T)`; otherwise, it
will be an `Err(E)`.


You may also notice the `#[must_use]` annotation above the
definition for `Result`. This is a _language attribute_ that provides additional
information to the compiler. In this case, `#[must_use]` indicates that all
`Result`s _must always be used_ in some way; if you do not inspect a `Result` in
some way, for example via pattern matching or the `try!` macro (more on that
later), the compiler will emit a warning:

```
warning: unused result which must be used, #[warn(unused_must_use)] on by default
```


### Similarities between the two

### Strengths Compared to Go

### Weaknesses Compared to Go

Generally, if you have a function in Go that returns `(someType, error)`, an
analogous function in Rust would return `Result<SomeType, Error>`.


