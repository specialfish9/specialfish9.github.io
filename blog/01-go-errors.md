---
layout: none
---

<style>
body {
  margin: 0;
  padding: 3rem 1.5rem;
  background: #000;
  color: #fff;
  font-family: system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI",
               Roboto, Helvetica, Arial, sans-serif;
  line-height: 1.6;
}

main {
  max-width: 720px;
  margin: 0 auto;
}

a {
  color: #6cf;
  text-decoration: none;
}

a:hover {
  text-decoration: underline;
}

pre {
  background: #0e0e0e;
  padding: 1rem;
  overflow-x: auto;
  border-radius: 4px;
}

code {
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace;
  font-size: 0.95rem;
}

p code {
  color: #ffcc66;
}
</style>

<main>

# Go and Errors

If you have experience with other programming languages, when you first
deal with the way Go handles errors, chances are that you will ask yourself
"why?!". In this blog post I will _go_ (no pun intended)
through some rules of thumb that I find useful when programming in Go.

__Important note__: I will illustrate three types of rules:
- Conventions _(C)_: these are rules defined by the Go team. You should
  always follow them.
- Best practices _(B)_: these are widely accepted rules in the Go community.
  You should follow them unless you have a good reason not to.
- Personal opinions _(P)_: these are my personal preferences, and
  you may disagree with them. 

## The basics
Let's start simple: in Go errors are simple values, much like all the other
variables you may define. They are not "thrown" and "caught" like exceptions
in other programming languages, they are simply returned to the function's
caller like any normal return value:
```go
// doStuff1 behaves like a 'void' function, but it could
// fail.
func doStuff1() error

// doStuff2 instead computes some `int` value. If it fails,
// an error is also returned. 
func doStuff2() (int, error)
```

### Rule No.1 (C)
If a function has multiple return values, the `error` should always be
    the last one.

Let's move on to the caller side now:
```go
a, err := doStuff2()
if err != nil {
    // Handle the error
}
// Happy path
// ...
```

The caller should always check for an error, and the usual way
to do that is using the famous `if err != nil` patterns. As you 
can imagine, this single line of code is super popular in every Go 
project, causing many people's complaints and desires for some sugar
syntax like Rust's `?`. There was (and still is) a humongous number 
of online threads arguing whether the language should do something
about it. However, as always with Go's philosophy, explicit is better
than implicit and, by consequence, `if err != nil` won.

### Rule No.2 (B)
Always prefer `if err != nil` over `if err == nil` statements. 

### Rule No.3 (B)
When calling a function that returns an error, you should always
check it. If, for some reason, you don't want to, be sure to highlight
it using the blank identifier `_`:
```go
// We're using '_' to signal that we don't want to handle the error.
a, _ := doStuff2()  
```

## Creating errors
To create an error you may want to use two handy functions provided by
the standard library: `errors.New` and `fmt.Errorf`. The former, defined
in the `errors` package (more on that later) takes a single string 
argument representing an error message:
```go
var err = errors.New("something bad happened")
```

This is nice and clean, but if you want to include a more structured 
message, you better use `fmt.Errorf`, which works similarly to the
other format functions defined in the `fmt` package:
```go
var err = fmt.Errorf("file '%s' not found", fileName)
```

Errors created with these functions, when kept as package-level variables,
are often referred to as _sentinel errors_.
Later on, we will come back to `fmt.Errorf`, as it hides more power
than it seems at first glance, but for now let's move on.

If you look under the hood of the `error` type implementation, you will
see that it's actually a super simple interface:
```go
type error interface {
    Error() string
}
```

This means that you can create your custom _error types_, holding all the
information you want. A typical example would be:
```go
type APIError struct {
    StatusCode  int     `json:"statusCode`
    Message     string  `json:"message`
}

func (ae *APIError) Error() string {
    return ae.Message
}

func MyWonderfulAPI() error {
    // ...
    return &APIError {
        StatusCode: 404,
        Message:    "Resource not found",
    }
}
```

### Rule No.4 (P)
When writing a signature for a public function, prefer returning
`error` over your custom error type:
```go
// Do
func MyWonderfulAPI() error 
// Don't
func MyWonderfulAPI() APIError
```

We will go back to error types later on.

## Handling errors
In the previous section we saw many ways of creating new errors,
so now we will see that there are also multiple ways of handling 
them. Starting from the first: the easiest way to handle an error is
...to delegate this honor to the caller, i.e., not handling it.
In fact, there are some cases in which you don't want to handle an
error in any way, but you just want to let it slip to the caller of
the function you are writing. This is very common and, as you have
all probably guessed, you can achieve it with a simple:
```go
err := doStuff1()
if err != nil {
    // Delegate the error handling to the caller
    return err
}
```

Easy-peasy. This is 100% fine, but I advise you: before writing 
this, think about the context that your caller receives. Let's go
back to our `fmt.Errorf` example. Using that function, we created
a new error with a message `"file 'insertFileName.here' not found"`.
When we created it, it sounded like a reasonable explanation for
an error message. However, in the big scheme of things that is 
your current shiny new Go project, that information alone might not
be enough. For example, you may want to know which component of 
your application tried to open that file and why.
Fortunately, Go has us covered with the concept of _error wrapping_.
The idea is similar to a matrioska: while your error goes down the
call tree, whenever needed, it gets wrapped again and again with
additional context, enabling for a better post-mortem inspection. Sounds
kinda complex, doesn't it? But I can assure you it's not. To do so
in practice we just have to go back again to the `fmt.Errorf` function,
and use the appropriate placeholder:
```go
err := doStuff1()
if err != nil {
    // Wrap the error with additional context.
    // This becomes: "can't initialize cache: file 'insertFileName.here' not found"
    return fmt.Errorf("can't initialize cache: %w", err)
}
```
And you can go on and on and on...
```
app initialization: cron service: unable to start "someRandomJob" job: cache
service: can't initialize cache: file 'insertFileName.here' not found
```

Writing a good error message is kind of hard, and it comes with experience 
I guess. Here are some rules that I personally find useful.

### Rule No.6 (C)
Error messages should always start with a lowercase letter and should always be
chained using a colon (`:`).

### Rule No.7 (B)
Add context to errors only when it is needed. Do not abuse it, otherwise the error message
will become tedious to read.

### Rule No.8 (P)
Try to avoid writing the word "error" in the message as it will cause the message's
length to increase, and it's probably already implicit:
```
error initializing app: cron service: error starting "someRandomJob" job: cache
service: error while initializing cache: error: file 'insertFileName.here' not found
```

### Rule No.9 (P)
Sometimes it is useful to add a label with the name of the component handling the error
(e.g.: `cron service:`), just to highlight that the error passed through that component.
However, this should only be done in public functions, to avoid the risk of having it
repeated many times in the same error message:
```
user service: fetching admin: user service: can't fetch user %s: user service: reading
from db: user not found
```

Last but not least, Go offers another way of handling errors: panicking. It's okay
to panic, but it should be done very, very carefully. For those who live under a
rock, the Go programming language offers a `panic` function, which causes the program
to crash. If you return an error, you give the option to recover what is recoverable
and optionally panic. Instead, when you panic, you make the decision on behalf of the
calling code. This must be used only as your code's last resort. I usually follow the next
rule:

### Rule No.10 (B)
Don't `panic` if it's not strictly necessary.

With that said, sometimes there is really no other thing to do: it might be that you can't
proceed with the flow, and you can't return a normal error at the same time. In that case,
panicking is indeed okay, but 

### Rule No.11 (B)
Advise the developer who's going to call your panicking function with a godoc comment.

Additionally, sometimes it's useful for a package to offer two versions of the same utility
function. Taking the `regexp` package from the standard library as example:
```go

// Compile parses a regular expression and returns, if successful,
// a [Regexp] object that can be used to match against text.
// [...]
func Compile(expr string) (*Regexp, error)

// MustCompile is like [Compile] but panics if the expression cannot be parsed.
// [...]
func MustCompile(str string) *Regexp
```

### Rule No.12 (B)
Use the prefix "Must" in your function's name if it panics instead of returning 
an error.

While we are here, there's another bonus rule related to handling errors

### Rule No.13 (P)
Never log AND return the error, as it could lead to the same error being logged 
multiple times.

## Combined errors
Many times you might need to perform multiple operations each one returning potentially
an error. Something like:
```go
for _, input := range inputs {
    err = check(input)
    // what should we do with err???
} 
```
Often, you can simply fail at the first occurrence of an error, and call it a day. 
However, sometimes it's better to run _all_ the operations first, and then return 
a _combined error_ later. Meet `errors.Join`: 
```go
var err error
for _, input := range inputs {
    err = errors.Join(err, check(input))
} 
if err != nil {
    return fmt.Errorf("checking inputs: %w", err)
}
```

`errors.Join` _joins_ all the errors given as argument together returning a new,
combined, error. If one of the arguments is nil, it will be skipped, hence
if all the arguments are nil, `errors.Join` will return nil too.
Note that the `for` loop will terminate only when all inputs are consumed,
i.e., `check` will be executed once for each element of the `inputs` slice,
even if some previous iteration returned an error.
Additionally, note also that this code was intentionally kept short for
simplicity's sake, but in a real life scenario you would probably want to
add some context to the errors returned by `check`, for example which input
failed.

### Rule No.14 (B)
Use `errors.Join` when dealing with combined errors.

Now that I introduced you to combined errors, it is time to abandon our "matrioska"
metaphor. In fact, the structure we build by wrapping errors is actually a tree.
Sorry for having lied to you. Later we will come back to this tree concept and see
what we can do with it, but for now let's stick with combined errors.

A similar, yet different pattern that often occurs is dealing with multiple
asynchronous operations. In that case, you may want to use the `errgroup` package
from the `golang.org/x/sync/errgroup` package:
```go
import (
    "golang.org/x/sync/errgroup"
)

// ...
var g errgroup.Group

for _, input := range inputs {
    g.Go(func() error {
        return check(input)
    })
}
// Wait blocks until all function calls from the Go method have returned, 
// then returns the first non-nil error (if any) from them. 
err := g.Wait()
if err != nil {
    return fmt.Errorf("checking inputs: %w", err)
}
```

Notice that `g.Wait` returns only the first non-nil error. If you want to
collect all the errors instead, you may want to use a channel and `errors.Join`
together with a `sync.WaitGroup`.

## Exploring the error tree
Before going through how can we explore the error tree I would like to spend
a few more words on how to add nodes to this tree using error wrapping.

Until now, the only way to wrap errors I've shown you is through `fmt.Errorf`,
but it's not the only way. You can implement your own wrapping logic by making
your errors by adding either a `Unwrap() error` or `Unwrap() []error` (for
composite errors) method.

```go
type APIError struct {
    StatusCode  int     `json:"statusCode"`
    Message     string  `json:"message"`
    innerErr    error
}

func NewAPIErr(status int, err error) error {
    return &APIError {
        StatusCode: status,
        innerErr: err,
        Message: err.Error(),
    }
}

func (ae *APIError) Error() string { return ae.Message }

func (ae *APIError) Unwrap() error { 
    return ae.innerErr
}
```


### Rule No.15 (P)
If your custom error contains another error inside, always implement the `Unwrap`
method, as (__Spoiler alert!__) the `Unwrap` method will be used by some cool
functions to navigate the tree.

One last thing on wrapping: what if you want to add some context to an error
using `fmt.Errorf` without actually wrapping it? Easy: use the `%v` placeholder


```go

// wrapping
fmt.Errorf("can't initialize cache: %w", err)

// not wrapping
fmt.Errorf("can't initialize cache: %v", err) 
// ^^^ This is equivalent to:
errors.New(fmt.Sprintf("can't initialize cache: %v", err))
```

If you are still reading, you may wonder: _"okay, this is nice. But
what's the actual advantage of wrapping errors?". You are smart, aren't you?
Let me introduce you our last functions: `errors.Is` and `errors.As`, both defined
in the `errors` package. With these two functions, you can navigate through an
error tree checking for a target (respectively a sentinel error for `errors.Is`
and an error type for `errors.As`). This is useful if you want to handle
different errors in different ways. For example:

```go
// auth.go
package auth

var ErrUnauthorized error = errors.New("unauthorized")

func Login(username string, password string) error { /* ... */ }

// httpserver.go

// ...

err := auth.Login(username, password)
if errors.Is(err, auth.ErrUnauthorized) {
    return 401, err
}

// APIError from a previous example
var apiErr APIError
if errors.As(err, &apiErr) {
    return apiErr.StatusCode, apiErr
}

return 200, nil
```

In this example, `errors.Is` navigate the error tree of `err`, returning `true`
if it finds an instance of `auth.ErrUnauthorized`.
On the other hand, `errors.As` navigate the same tree looking for _some_ error
of the `APIError` type, and assigns it to the given pointer to `apiErr`.
I'm sure that some of you think that `errors.As` is a little clunky, and
personally I agree. Fortunately Go 1.26 will introduce (or introduced, if you 
are reading this in the future) a new:

```go
func AsType[E error](err error) (E, bool)
```

that will transform our code snippet into
```go
// APIError from a previous example
if apiErr, ok := error.AsType[APIError](err); ok {
    return apiErr.StatusCode, apiErr
}
```

Needless to say, this is MUCH cleaner and, as a bonus, it's also reflection 
free. 

### Rule No.16 (C)
Don't compare errors against each other, as the comparison with `==` doesn't
go throughout the tree. Use `errors.Is`, and `errors.As` or `errors.AsType` instead.

### Rule No.17 (P)
Prefer creating public sentinel errors and types over private ones: you'll never know who
is going to need to `errors.Is/As/AsType`-them. But be aware that it's a trade-off, as
they become part of the API commitment you share with the other developers using your
code.

### Rule No.18 (P)
Prefer `errors.AsType` over `errors.As`: it is more efficient and has a cleaner signature!

---

Mattia Girolimetto - A.K.A. Specialfish9

22/01/2026

</main>
