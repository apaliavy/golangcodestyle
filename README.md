# Before you start

Please take a look to the following posts:

* Real world advice for writing maintainable Go programs: https://dave.cheney.net/practical-go/presentations/qcon-china.html
This article was originally a workshop at QCon Shanghai 2018. Author gives us his advice for best practices writing Go code.

* Ho to structure your go project: https://github.com/golang-standards/project-layout

* Packages that you *should not* create: https://dave.cheney.net/2019/01/08/avoid-package-names-like-base-util-or-common

* Think you understand the Single Responsibility Principle? https://hackernoon.com/you-dont-understand-the-single-responsibility-principle-abfdd005b137

* Layered Architecture (and how it shows up in Microservice Architectures): https://medium.com/code-smells/layered-architecture-f11bc04c5d6c

All of them are quite important for us as for backend developers.

# Golang Coding Guidelines

Based on following articles:

* Effective Go - https://golang.org/doc/effective_go.html#interface-names

* Code review comments - https://github.com/golang/go/wiki/CodeReviewComments

## Avoid global variables

Global variables make testing and readability hard and every method has access to them (even those, that don't need it).
Use structs to encapsulate the variables and make them available only to those functions that actually need them by making them methods implemented for that struct.

Alternatively higher-order functions can be used to inject dependencies via closures.

Bad:

```Go
var db *sql.DB

func main() {
	db = // ...
	http.HandleFunc("/drop", DropHandler)
	// ...
}

func DropHandler(w http.ResponseWriter, r *http.Request) {
	db.Exec("DROP DATABASE prod")
}
```

Good:

```Go
func main() {
	db := // ...
	handlers := Handlers{DB: db}
	http.HandleFunc("/drop", handlers.DropHandler)
	// ...
}

type Handlers struct {
	DB *sql.DB
}

func (h *Handlers) DropHandler(w http.ResponseWriter, r *http.Request) {
	h.DB.Exec("DROP DATABASE prod")
}
```

If you really need global variables or constants, e.g., for defining errors or string constants, put them at the top of your file:

```Go
import "xyz"

const route = "/some-route"

var NotFoundErr = errors.New("not found")

func someFunc() {
	//...
}

func someOtherFunc() {
	// usage of route
}

func yetAnotherFunc() {
	// usage of NotFoundErr
}
```

## Avoid side-effects

Bad:

```Go
func init() {
	someStruct.Load()
}
```

Side effects are only okay in special cases (e.g. parsing flags in a cmd). If you find no other way, rethink and refactor.

## Interfaces

Go interfaces generally belong in the package that uses values of the interface type, not the package that implements those values. The implementing package should return concrete (usually pointer or struct) types: that way, new methods can be added to implementations without requiring extensive refactoring.

Do not define interfaces on the implementor side of an API "for mocking"; instead, design the API so that it can be tested using the public API of the real implementation.

Do not define interfaces before they are used: without a realistic example of usage, it is too difficult to see whether an interface is even necessary, let alone what methods it ought to contain

Bad:

```Go
package producer

type Thinger interface { Thing() bool }

type defaultThinger struct{ … }
func (t defaultThinger) Thing() bool { … }

func NewThinger() Thinger { return defaultThinger{ … } }
```

Good:

```Go
package producer

type Thinger struct{ … }
func (t Thinger) Thing() bool { … }

func NewThinger() Thinger { return Thinger{ … } }
```

and then:

```Go
package consumer  // consumer.go

type Thinger interface { Thing() bool }

func Foo(t Thinger) string { … }
```

```Go
package consumer // consumer_test.go

type fakeThinger struct{ … }
func (t fakeThinger) Thing() bool { … }
…
if Foo(fakeThinger{…}) == "x" { … }
```

## Interface names

By convention, one-method interfaces are named by the method name plus an -er suffix or similar modification to construct an agent noun: `Reader`, `Writer`, `Formatter`, `CloseNotifier` etc.

There are a number of such names and it's productive to honor them and the function names they capture. `Read`, `Write`, `Close`, `Flush`, `String` and so on have canonical signatures and meanings.
To avoid confusion, don't give your method one of those names unless it has the same signature and meaning. Conversely, if your type implements a method with the same meaning as a method on a well-known type, give it the same name and signature; call your string-converter method `String` not `ToString`.


## Don't over-interface

Bad:

```Go
type Server interface {
	Serve() error
	Some() int
	Fields() float64
	That() string
	Are([]byte) error
	Not() []string
	Necessary() error
}
```

Good:

```Go
type Server interface {
	Serve() error
}

func run(srv Server) {
	srv.Serve()
}
```

### Mocks

Use counterfeiter (https://github.com/maxbrunsfeld/counterfeiter) to generate the mocks.
Using counterfeiter conflicts with the guidelines presented above :(. 
One way around it (I would appreciate better ideas!) is to create an interface inside the package that have declared the mocked struct, and then create mock based on this interface. 
Then you can still use another interface declared in the package that uses the mocked struct to use the mock in the tests. 
It means two inteface declarations, but little code duplication is better than the alternatiives in my opinion.

## Imports

Avoid renaming imports except to avoid a name collision; good package names should not require renaming. In the event of collision, prefer to rename the most local or project-specific import.

Imports are organized in groups, with blank lines between them. The standard library packages are always in the first group (goimports will do this for you).

```Go
package main

import (
	"fmt"
	"hash/adler32"
	"os"

	"appengine/foo"
	"appengine/user"

	"rsc.io/goversion/version"
)
```

Divide imports into four groups sorted from internal to external for readability:
* Standard library
* Project internal packages
* Company internal packages
* External packages

## Goimports

https://godoc.org/golang.org/x/tools/cmd/goimports

Command `goimports` updates your Go import lines, adding missing ones and removing unreferenced ones. In addition to fixing imports, goimports also formats your code in the same style as gofmt so it can be used as a replacement for your editor's gofmt-on-save hook.

Goimports will organize imports in groups, with blank lines between them. The standard library packages are always in the first group. Detailed order/formatting:

* Standard library
* Project internal packages
* Company internal packages
* External packages

For an example please see "Imports" section of this document.

## Comment Sentences
Comments documenting declarations should be full sentences, even if that seems a little redundant. Comments should begin with the name of the thing being described and end in a period:

```Go
// Request represents a request to run a command.
type Request struct { ...

// Encode writes the JSON encoding of req to w.
func Encode(w io.Writer, req *Request) { ...
```

## Doc Comments

All top-level, exported names should have doc comments, as should non-trivial unexported type or function declarations.
See https://golang.org/doc/effective_go.html#commentary for more information about commentary conventions.


## Declaring Empty Slices

When declaring an empty slice, prefer

```Go
var t []string
```

over

```Go
t := []string{}
```

The former declares a nil slice value, while the latter is non-nil but zero-length. They are functionally equivalent—their len and cap are both zero—but the nil slice is the preferred style.

Note that there are limited circumstances where a non-nil but zero-length slice is preferred, such as when encoding JSON objects (a nil slice encodes to null, while []string{} encodes to the JSON array []).

## Don't Panic

Don't use panic for normal error handling. Use error and multiple return values.
See https://golang.org/doc/effective_go.html#errors for more information.

## Exports

Do not export functions and variables unless necessary.

Keep the exported variables and error in the same package that is using/returning them.
For example i/o error like `io.EOF` from the go std lib is in the io package, where the function returning this error are also defined.

## Errors

* If your function return a specific error, and the caller might act differently if it receives this specific error, then document this behaviour.

* Do not return an unique error (an package level error variable that is exported) unless you are sure that the caller might actually need to discern between such error and any other error.

* Error strings should not be capitalized (unless beginning with proper nouns or acronyms) or end with punctuation, since they are usually printed following other context.
That is, use `fmt.Errorf("something bad")` not `fmt.Errorf("Something bad")`, so that `log.Printf("Reading %s: %v", filename, err)` formats without a spurious capital letter mid-message.
This does not apply to logging, which is implicitly line-oriented and not combined inside other messages.

Also consider:

* https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully
* https://dave.cheney.net/2016/04/07/constant-errors
* https://dave.cheney.net/2014/12/24/inspecting-errors

## Handle Errors

Do not discard errors using _ variables. If a function returns an error, check it to make sure the function succeeded. Handle the error, return it, or, in truly exceptional situations, panic.
See https://golang.org/doc/effective_go.html#errors for more information.


## In-Band Errors

In C and similar languages, it's common for functions to return values like -1 or null to signal errors or missing results:

```Go:
// Lookup returns the value for key or "" if there is no mapping for key.
func Lookup(key string) string

// Failing to check a for an in-band error value can lead to bugs:
Parse(Lookup(key))  // returns "parse failure for value" instead of "no value for key"
```

Go's support for multiple return values provides a better solution. Instead of requiring clients to check for an in-band error value, a function should return an additional value to indicate whether its other return values are valid. This return value may be an error, or a boolean when no explanation is needed. It should be the final return value.

```Go:
// Lookup returns the value for key or ok=false if there is no mapping for key.
func Lookup(key string) (value string, ok bool)
```

This prevents the caller from using the result incorrectly:

```Go:
Parse(Lookup(key))  // compile-time error
```

And encourages more robust and readable code:

```Go:
value, ok := Lookup(key)
if !ok  {
    return fmt.Errorf("no value for %q", key)
}
return Parse(value)
```

This rule applies to exported functions but is also useful for unexported functions.

Return values like nil, "", 0, and -1 are fine when they are valid results for a function, that is, when the caller need not handle them differently from other values.

Some standard library functions, like those in package "strings", return in-band error values. This greatly simplifies string-manipulation code at the cost of requiring more diligence from the programmer. In general, Go code should return additional values for errors.

## Getters and Setters

Go doesn't provide automatic support for getters and setters. There's nothing wrong with providing getters and setters yourself, and it's often appropriate to do so, but it's neither idiomatic nor necessary to put `Get` into the getter's name.
If you have a field called owner (lower case, unexported), the getter method should be called `Owner` (upper case, exported), not `GetOwner`. The use of upper-case names for export provides the hook to discriminate the field from the method.
A setter function, if needed, will likely be called `SetOwner`. Both names read well in practice:

```Go
owner := obj.Owner()
if owner != user {
    obj.SetOwner(user)
}
```

## Happy-path program flow

Try to keep the normal code path at a minimal indentation, and indent the error handling, dealing with it first. This improves the readability of the code by permitting visually scanning the normal path quickly.

Bad:

```Go
if err != nil {
	// error handling
} else {
	// normal code
}
```

Good:

```Go
if err != nil {
	// error handling
	return // or continue, etc.
}
// normal code
```

If the if statement has an initialization statement, such as:

```Go
if x, err := f(); err != nil {
	// error handling
	return
} else {
	// use x
}
```

then this may require moving the short variable declaration to its own line:

```Go
x, err := f()
if err != nil {
	// error handling
	return
}
// use x
```


## Initialisms

Words in names that are initialisms or acronyms (e.g. "URL" or "NATO") have a consistent case. For example, "URL" should appear as "URL" or "url" (as in "urlPony", or "URLPony"), never as "Url". As an example: ServeHTTP not ServeHttp. For identifiers with multiple initialized "words", use for example "xmlHTTPRequest" or "XMLHTTPRequest".

This rule also applies to "ID" when it is short for "identifier" (which is pretty much all cases when it's not the "id" as in "ego", "superego"), so write "appID" instead of "appId".

Code generated by the protocol buffer compiler is exempt from this rule. Human-written code is held to a higher standard than machine-written code.


## Line Length

There is no rigid line length limit in Go code, but avoid uncomfortably long lines. Similarly, don't add line breaks to keep lines short when they are more readable long--for example, if they are repetitive.

Most of the time when people wrap lines "unnaturally" (in the middle of function calls or function declarations, more or less, say, though some exceptions are around), the wrapping would be unnecessary if they had a reasonable number of parameters and reasonably short variable names. Long lines seem to go with long names, and getting rid of the long names helps a lot.

In other words, break lines because of the semantics of what you're writing (as a general rule) and not because of the length of the line. If you find that this produces lines that are too long, then change the names or the semantics and you'll probably get a good result.

## Logging

Some words about logging in Golang: https://dave.cheney.net/2015/11/05/lets-talk-about-logging

By default we're using logrus package: https://github.com/sirupsen/logrus

Logrus encourages careful, structured logging through logging fields instead of long, unparseable error messages. For example, instead of: `log.Fatalf("Failed to send event %s to topic %s with key %d")`, you should log the much more discoverable:
```
log.WithFields(log.Fields{
  "event": event,
  "topic": topic,
  "key": key,
}).Fatal("Failed to send event")
```

Logrus has seven logging levels: Trace, Debug, Info, Warning, Error, Fatal and Panic.

```
log.Trace("Something very low level.")
log.Debug("Useful debugging information.")
log.Info("Something noteworthy happened!")
log.Warn("You should probably take a look at this.")
log.Error("Something failed but I'm not quitting.")
// Calls os.Exit(1) after logging
log.Fatal("Bye.")
// Calls panic() after logging
log.Panic("I'm bailing.")
```

If the function needs to add more tags to the log, it can use `WithFields` function. The function will return a copy of the logger, so the parent logger will not be modified when the function returns.
In our services we use an additional field: `correlation_id`. It allows us to trace requests call stack.

We should be logging the correlation ID in our logs wherever possible (that is when the flow is initiated by an user action (http/grpc call).
To do this we need to use function `LoggerFromContext` from https://github.com/karhoo/lib-common/log package.

After that we should pass logger to the child functions as a function argument.

Besides the fields added with WithField or WithFields some fields are automatically added to all logging events:

* `time`. The timestamp when the entry was created.
* `msg`. The logging message passed to {Info,Warn,Error,Fatal,Panic} after the AddFields call. E.g. Failed to send event.
* `level`. The logging level. E.g. info.

## Mixed Caps

The convention in Go is to use MixedCaps or mixedCaps rather than underscores to write multiword names even when it breaks conventions in other languages.
For example an unexported constant is `maxLength` not `MaxLength` or `MAX_LENGTH`.

## Named Result Parameters

Consider what it will look like in godoc. Named result parameters like:

```Go
func (n *Node) Parent1() (node *Node)
func (n *Node) Parent2() (node *Node, err error)
```

will stutter in godoc; better to use:

```Go
func (n *Node) Parent1() *Node
func (n *Node) Parent2() (*Node, error)
```

On the other hand, if a function returns two or three parameters of the same type, or if the meaning of a result isn't clear from context, adding names may be useful in some contexts. Don't name result parameters just to avoid declaring a var inside the function; that trades off a minor implementation brevity at the cost of unnecessary API verbosity.

```Go
func (f *Foo) Location() (float64, float64, error)
```

is less clear than:

```Go
// Location returns f's latitude and longitude.
// Negative values mean south and west, respectively.
func (f *Foo) Location() (lat, long float64, err error)
```

Naked returns are okay if the function is a handful of lines. Once it's a medium sized function, be explicit with your return values. Corollary: it's not worth it to name result parameters just because it enables you to use naked returns. Clarity of docs is always more important than saving a line or two in your function.

Finally, in some cases you need to name a result parameter in order to change it in a deferred closure. That is always OK.

## Optimization

Premature optimization is forbidden. Easy, clean and readable code >>>>> optimised code.

If we need to optimise the code we need to first figure out the biggest problems, which usually means CPU profile on live data and then good benchmarks to compare the before and after results. Without its all useless.

## Package Names

All references to names in your package will be done using the package name, so you can omit that name from the identifiers.
For example, if you are in package `chubby`, you don't need type `ChubbyFile`, which clients will write as `chubby.ChubbyFile`. Instead, name the type `File`, which clients will write as `chubby.File`. Avoid meaningless package names like util, common, misc, api, types, and interfaces.

See http://golang.org/doc/effective_go.html#package-names and http://blog.golang.org/package-names for more.

## Variable Names

Variable names in Go should be short rather than long. This is especially true for local variables with limited scope. Prefer `c` to `lineCount`. Prefer `i` to `sliceIndex`.

The basic rule: the further from its declaration that a name is used, the more descriptive the name must be. For a method receiver, one or two letters is sufficient. Common variables such as loop indices and readers can be a single letter (`i`, `r`). More unusual things and global variables need more descriptive names.
## Pass Values

Don't pass pointers as function arguments just to save a few bytes. If a function refers to its argument `x` only as `*x` throughout, then the argument shouldn't be a pointer. Common instances of this include passing a pointer to a string (`*string`) or a pointer to an interface value (`*io.Reader`). In both cases the value itself is a fixed size and can be passed directly. This advice does not apply to large structs, or even small structs that might grow.

## Receiver Names

The name of a method's receiver should be a reflection of its identity; often a one or two letter abbreviation of its type suffices (such as "c" or "cl" for "Client").
Don't use generic names such as "me", "this" or "self", identifiers typical of object-oriented languages that gives the method a special meaning.
In Go, the receiver of a method is just another parameter and therefore, should be named accordingly. The name need not be as descriptive as that of a method argument, as its role is obvious and serves no documentary purpose.
It can be very short as it will appear on almost every line of every method of the type; familiarity admits brevity. Be consistent, too: if you call the receiver "c" in one method, don't call it "cl" in another.

## Receiver Type

Choosing whether to use a value or pointer receiver on methods can be difficult, especially to new Go programmers.
If in doubt, use a pointer, but there are times when a value receiver makes sense, usually for reasons of efficiency, such as for small unchanging structs or values of basic type. Some useful guidelines:

* If the receiver is a map, func or chan, don't use a pointer to them. If the receiver is a slice and the method doesn't reslice or reallocate the slice, don't use a pointer to it.
* If the method needs to mutate the receiver, the receiver must be a pointer.
* If the receiver is a struct that contains a sync.Mutex or similar synchronizing field, the receiver must be a pointer to avoid copying.
* If the receiver is a large struct or array, a pointer receiver is more efficient. How large is large? Assume it's equivalent to passing all its elements as arguments to the method. If that feels too large, it's also too large for the receiver.
* Can function or methods, either concurrently or when called from this method, be mutating the receiver? A value type creates a copy of the receiver when the method is invoked, so outside updates will not be applied to this receiver. If changes must be visible in the original receiver, the receiver must be a pointer.
* If the receiver is a struct, array or slice and any of its elements is a pointer to something that might be mutating, prefer a pointer receiver, as it will make the intention more clear to the reader.
* If the receiver is a small array or struct that is naturally a value type (for instance, something like the time.Time type), with no mutable fields and no pointers, or is just a simple basic type such as int or string, a value receiver makes sense. A value receiver can reduce the amount of garbage that can be generated; if a value is passed to a value method, an on-stack copy can be used instead of allocating on the heap. (The compiler tries to be smart about avoiding this allocation, but it can't always succeed.) Don't choose a value receiver type for this reason without profiling first.
* Finally, when in doubt, use a pointer receiver.

## Structures initialization

Do not initialise structs with empty values. It's not necessary and does not improve readability.

Bad:

```Go
type MyStruct {
    ValueA int
    ValueB string
    ValueC []float
}

var empty = MyStruct {
    ValueA = 0,
    ValueB = "",
    ValueC= nil
}
```

Good:

```Go
type MyStruct {
    ValueA int
    ValueB string
    ValueC []float
}

var empty = MyStruct{}
```

The only exceptions is when initialising maps in a new struct:

```Go
type MyStruct {
    Value map[string]string
}

var mapValue = MyStruct {
    Value: map[string]string{},
}
```

Even in such cases, consider writing constructor (NewMyStruct() *MyStruct)

## Synchronous Functions

Prefer synchronous functions - functions which return their results directly or finish any callbacks or channel ops before returning - over asynchronous ones.

Synchronous functions keep goroutines localized within a call, making it easier to reason about their lifetimes and avoid leaks and data races. They're also easier to test: the caller can pass an input and check the output without the need for polling or synchronization.

If callers need more concurrency, they can add it easily by calling the function from a separate goroutine. But it is quite difficult - sometimes impossible - to remove unnecessary concurrency at the caller side.

## Testing

### Main

No `TestMain` unless absolutely necessary. The reasoning behind it should be put in the function comment.

### Mocks

Use counterfeiter (https://github.com/maxbrunsfeld/counterfeiter) to generate the mocks.

### Parallelism

Try to use `t.Parallel()` for the top level tests and the table sub-tests when possible.

### Table-driven tests

Writing good tests is not trivial, but in many situations a lot of ground can be covered with table-driven tests: Each table entry is a complete test case with inputs and expected results, and sometimes with additional information such as a test name to make the test output easily readable.

We also propose a single table driven test for success and failure cases.

See https://github.com/golang/go/wiki/TableDrivenTests for more.

# Conclusion

* If you are unsure about how to design something or how to handle errors in a specific case, have a look at the stdlib code - most of the time the go standard library is well written and designed.
* If some kid of “solution”/approach is done in the stdlib it’s usually the right solution/approach.
