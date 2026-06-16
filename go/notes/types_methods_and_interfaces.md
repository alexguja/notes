## Types, Methods, and Interfaces

In Go, you can define your own types and associate methods with them. This allows you to create complex data structures and behaviors. Additionally, Go has a powerful interface system that enables you to define behavior without specifying the underlying type.


### Types in Go

You can define a new type using the `type` keyword. For example, let's define a `Person` struct:
```go
type Person struct {
    FirstName string
    LastName  string
    Age       int
}

```

You can also use the primitive types to create new types. For example:
```go   
type Score int
type Converter func(string)Score
type TeamScore map[string]Score
```

>[!Note]
> An abstract type is a type that specifies _what_ a type should do, but not _how_ it does it. In Go, interfaces are a way to define abstract types. A concrete type specifies what and how a type does. In Go, structs are a way to define concrete types.

### Methods 

Go supports methods on user-defined types.

```go
type Person struct {
    FirstName string
    LastName string
    Age int
}


func (p Person) String() string {
    return fmt.Sprintf("%s %s, age %d", p.FirstName, p.LastName, p.Age)
}
```

Method declarations look just like function declarations, with one addition: the receiver specification. The receiver appears between the keyword func and the name of the method. Just like all other variable declarations, the receiver name appears before the type. By convention, the receiver name is a short abbreviation of the type’s name, usually its first letter. It is nonidiomatic to use this or self.

>[!Note]
> Just like functions, method names cannot be overloaded. You can use the same method names for different types, but you can’t use the same method name for two different methods on the same type.


While you can define a method in a different file within the same package as the type declaration, it is best to keep your type definition and its associated methods together so that it’s easy to follow the implementation.

### Pointer and value receivers

- If your method modifies the receiver, you must use a pointer receiver.
- If your method needs to handle nil instances, then it must use a pointer receiver.
- If your method doesn’t modify the receiver, you can use a value receiver.
- When a type has any pointer receiver methods, a common practice is to be consistent and use pointer receivers for all methods, even the ones that don’t modify the receiver.
- When you use a pointer receiver with a local variable that’s a value type, Go automatically converts it to a pointer type. For example `c.Increment()` is converted to `(&c).Increment()`.
- Go considers both pointer and value receiver methods to be in the method set for a pointer instance.
- Do not write getter and setter methods for Go structs, unless you need them to meet an interface.


```go
type Counter struct {
    total int
    lastUpdated time.Time
}

func (c *Counter) Increment() {
    c.total++
    c.lastUpdated = time.Now()
}

func(c Counter) String() string {
    return fmt.Sprintf("total %d, last updated %v", c.total, c.lastUpdated)
}


// using the code above 
var c Counter
fmt.Println(c.String()) // total: 0, last updated: 0001-01-01 00:00:00 +0000 UTC
c.Increment()
fmt.Println(c.String()) // total: 1, last updated: 2024-06-01 12:00:00 +0000 UTC
```



```go
func doUpdateWrong(c Counter) {
    c.Increment()
    fmt.Println("in doUpdateWrong:", c.String())
}

func doUpdateRight(c *Counter) {
    c.Increment()
    fmt.Println("in doUpdateRight:", c.String())
}

func main() {
    var c Counter
    doUpdateWrong(c)
    fmt.Println("in main:", c.String())
    doUpdateRight(&c)
    fmt.Println("in main:", c.String())
}


```

Output:
```sh
in doUpdateWrong: total: 1, last updated: 2009-11-10 23:00:00 +0000 UTC
m=+0.000000001
in main: total: 0, last updated: 0001-01-01 00:00:00 +0000 UTC
in doUpdateRight: total: 1, last updated: 2009-11-10 23:00:00 +0000 UTC
m=+0.000000001
in main: total: 1, last updated: 2009-11-10 23:00:00 +0000 UTC m=+0.000000001
```

### `nil` instances
When calling a method on a `nil` instance, Go will try to invoke the method. If it's a method with a value receiver, you'll get a panic, since there is no value being pointed to by the pointer. If it's a method with a pointer receiver, the call can work if the method is written to handle the possibility of a `nil` instance.

In Go, `p.Method()` is just sugar for `Method(p)`.

```go
type IntTree struct {
    val int
    left, right *IntTree
}

func (it *IntTree) Insert(val int) *IntTree {
    if it == nil {
        return &IntTree{val: val}
    }
    if val < it.val {
        it.left = it.left.Insert(val)
    } else {
        it.right = it.right.Insert(val)
    }
    return it
}
```

```Go
func (it *IntTree) Contains(val int) bool {
    switch {
        case it == nil:
            return false    
        case val < it.val:
            return it.left.Contains(val)
        case val > it.val:
            return it.right.Contains(val)
        default:
            return true
    }
}
```

>[!Note] 
> It's clever that Go allows calling methods on `nil` instances, however most of the time this is not very useful. Pointer receivers work just like pointer function parameters. It is a copy of the pointer that gets passed to functions. 
Just like nil parameters passed to functions, if you change the copy of the pointer, you haven't changed the original. This means you cannot write a pointer receiver method that handles `nil` and makes the original pointer non-nil.

### Methods are functions too
Methods in Go are a lot like functions. You can use a method as a replacement for a function any time there's a variable or parameter of a function type.

```Go
type Adder struct {
    start int
}

func (a Adder) AddTo(val int) int {
    return a.start + val
}
```



```Go
myAdder := Adder{start: 10}
myAdder.AddTo(5) // 15

f1 := myAdder.AddTo
f1(10) // 20 
```

A method value is a bit like a closure, since it can access the values in the fields of the instance from which it was created. 

### Method expressions

You can also create a function from the type itself. This is called a _method expression_.

```Go
f2 := Adder.AddTo
f2(myAdder, 15) // 25
```
In the case of a method expression, the first parameter is the receiver for the method;
our function signature is `func(Adder, int) int`.

> [!Note]
>Package-level state should be immutable. Any time your logic depends on values that are configured at startup or changed while your program is running those values should be stored in a struct and that logic should be implemented as a method. If your logic only depends on the input parameters, then it should be a function.


### Type declarations aren't inheritance

In Object-oriented programming (OOP), _inheritance_ means the state and methods of a parent type are declared to
be available on a child type and values of the child type can be substituted for the parent type


Declaring a type based on another type looks a bit like inheritance, but it isn’t. The two types have the same underlying type, but that’s all. There is no hierarchy
between these types.

```Go
type HighScore Score
type Employee Person
```

In languages with inheritance, a child instance can be used anywhere the parent instance is used. The child instance also has all the methods and data structures of the parent instance. That’s not the case in Go.


```Go
// assigning untyped constants is valid
var i int = 300
var s Score = 100
var hs HighScore = 200
hs = s // compilation error!
s = i // compilation error!
s = Score(i) // ok
hs = HighScore(s) // ok
```
For user-defined types whose underlying types are built-in types, a user-declared type can be used with the operators for those types.

>[!Note]
>A type conversion between types that share an underlying type keeps the same underlying storage but associates different methods.

### Types are executable documentation
Types can be viewed as documentation. They make code clearer by providing a name
for a concept and describing the kind of data that is expected. It’s clearer for someone
reading your code when a method has a parameter of type `Percentage` than of type
`int`, and it’s harder for it to be invoked with an invalid value. The same logic applies when declaring one user-defined type based on another user-defined type. 


### `iota` for enumerations
Go does not have a built-in enumeration type, but it has `iota`, which lets you assign an increasing value to a set of constants.

> [!Note]
> The concept of `iota`s comes from the programming language APL.


```Go
type MailCategory int

const (
    Uncategorized MailCategory = iota
    Personal
    Work
    Spam
    Social
    Ads
)
```
When the Go compiler sees this, it repeats the type and the assignment to all of the subsequent constants in the block, and increments the value of iota on each line. This means that it
assigns 0 to the first constant (Uncategorized), 1 to the second constant (Personal),
and so on. When a new const block is created, iota is set back to 0.

`iota` is best suited for internal use, where constants are referred to by name rather than by value.
That way new values can be added later without the risk of breaking existing code.


Be aware that iota starts numbering from 0. If you are using your set of constants to
represent different configuration states, the zero value might be useful. We saw this
earlier in our `MailCategory` type. When mail first arrives, it is uncategorized, so the
zero value makes sense. If there isn’t a sensical default value for your constants, a
common pattern is to assign the first iota value in the constant block to a constant that indicates the value is invalid.


### Embedding for composition

```Go
type Employee struct {
    Name string
    ID   string
}


func (e Employee) Description() string {
    return fmt.Sprintf("%s (%s)", e.Name, e.ID)
}

type Manager struct {
    Employee //  👈 Embedded field
    Reports []Employee
}

func (m Manager) FindNewEmployees() []Employee {
    // business logic
}

```

> [!Note]
> Any fields or methods declared on an embedded field are promoted to the containing struct and can be invoked directly on it. You can embed any type within a struct, not just another struct.
This promotes the methods on the embedded type to the containing struct.


```Go
m := Manager{
    // 👇 Embedded field 
    Employee: Employee{ 
        Name: "Bober",
        ID: "1234",
    },
    Reports: []Employee{},
}

```

If the containing struct has fields or methods with the same name as an embedded
field, you need to use the embedded field’s type to refer to the obscured fields or
methods.

```Go
type Inner struct {
    X int
}

type Outer struct {
    Inner
    X int
}

o := Outer{
    Inner: Inner{X: 1},
    X: 2,
}

o.X       // 2
o.Inner.X // 1

```

>[!Note]
> Embedding is not the same as inheritance. For example, you cannot assign a `Manager` value to an `Employee` variable, even though `Manager` has all the fields and methods of `Employee`. Embedding is a way to reuse code, but it does not create a subtype relationship between the containing struct and the embedded struct.

There is also no _dynamic dispatch_ for concrete types in Go. The methods on the embedded field have no idea they are embedded. If you have a method on an embedded field that calls another method on the embedded field, and the containing struct has a method with the same name, the method on the embedded field will not invoke the method on the containing struct.

```Go
type Inner struct {
    A int
}


func (i Inner) IntPrinter(val int) string {
    return fmt.Sprintf("Inner: %d", val)
}


func (i Inner) Double() string {
    return i.IntPrinter(i.A * 2)
}


type Outer struct {
    Inner
    S string
}

func (o Outer) IntPrinter(val int) string {
    return fmt.Sprintf("Outer: %d", val)
}

func main() {
    o := Outer{
        Inner: Inner{A:10},
        S: "hello",
    }
    fmt.Println(o.Double()) // Inner: 20
}

```

> [!Note]
> While embedding one concrete type inside another won’t allow you to treat the outer
> type as the inner type, the methods on an embedded field do count toward the
> method set of the containing struct.


### Interfaces

Despite Go's concurrency model getting all the publicity, the real star of Go's design is its implicit interfaces. Interfaces are the only abstract type in Go.

```Go
type Stringer interface {
    String() string
}

```

- An interface lists the methods that must be implemented by a concrete type to meet the interface. 
- The methods defined by an interface are called the method set of the interface.
- Like other types, an interface can be declared in any block.
- Typically, interface names end with the suffix "er", such as `Reader`, `Writer` etc.

> [!Note]
> Interfaces are type-safe duck typing. If the method set for a concrete type contains all of the methods in the method set of an interface, the concrete type implements the interface. This means the concrete type can be assigned to a variable or field declared to be of the type of the interface.

This implicit behavior makes interfaces the most interesting thing about types in Go, because they enable both type-safety and decoupling, bridging the functionality in both static and dynamic languages.

Programming wisdom suggests “Program to an interface, not an implementation.” Doing so allows you to depend on behavior, not on implementation, allowing you to swap implementations as needed.


```Go
type LogicProvider struct {}

func (lp LogicProvider) Process(data string) string {
    // business logic
}

type Logic interface {
    Process(data string) string
}

type Client struct {
    L Logic // 👈 Client depends on an interface, not on an implementation
}

func (c Client) Program() {
    // get data from somewhere
    c.L.Process(data)
}

func main() {
    c  := Client{
        L: LogicProvider{},
    }
    c.Program()
}
```

In the code above, there is an interface, but only the caller (`Client`) knows about it. There is nothing declared on `LogicProvider` to indicate that it meets the interface. This is sufficient to both allow a new logic provider in the future and provide executable documentation to ensure that any type passed to the client will match the client's need.

> [!Note]
> Interfaces specify what callers need. The client code defines the
interface to specify what functionality it requires


It is common in Go to write factory functions that take in an instance of an interface and return another type that implements the same interface. For example

```Go
func process(r io.Reader) error

// Processing data from a file
r, err := os.Open(fileName)
if err != nil {
    return err
}

defer r.Close()
return process(r)

// Processing data from a compressed file
r, err := os.Open(fileName)
if err != nil {
    return err
}

defer r.Close()
gz, err := gzip.NewReader(r)
if err != nil {
    return err
}
defer gz.Close()
return process(gz)
```


It’s perfectly fine for a type that meets an interface to specify additional methods that aren’t part of the interface. One set of client code may not care about those methods, but others do. For example, the `io.File` type also meets the `io.Writer` interface. If your code only cares about reading from a file, use the `io.Reader` interface to refer to the file instance and ignore the other methods.


### Embedding and interfaces

Just like you can embed a type in a struct, you can also embed an interface in an interface.

```Go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Closer interface {
    Close() error
}

type ReadCloser interface {
    Reader
    Closer
}

```


### Accept interfaces, return structs

The business logic invoked by your functions should be invoked via interfaces, but the output of your functions should be a concrete type.


- If you create an API that returns interfaces, you are losing one of the main advantages of implicit interfaces: decoupling. You want to limit the third-party interfaces thatyour client code depends on because your code is now permanently dependent on the module that contains those interfaces, as well as any dependencies of that module, and so on.

- Another reason to avoid returning interfaces is versioning. If a concrete type is returned, new methods and fields can be added without breaking existing code.

> [!Warning]
> Adding a new method to an interface means that you need to update all existing implementations of the interface, or your code breaks. If you make a backward-breaking change to an API, you should increment your
major version number.


### Interfaces and `nil`

`nil` is also used to represent the zero value for an interface. In order for an interface to be considered `nil`, _both_ the type and the value must be `nil`.


```Go
var s *string
s == nil // true
var i interface{}
i == nil // true
i = s
i == nil // false, because the type of i is *string, which is not nil

```

In the Go runtime, interfaces are implemented as a pair of pointers:
- One to the underlying type
- One to the underlying value

As long as the type is non-nil, the interface is non-nil. (Since you cannot have a variable without a type, if the value pointer is non-nil, the type pointer is always non-nil.)

What `nil` indicates for an interface is whether or not you can invoke methods on it.

### The empty interface says nothing

An empty interface type simply states that the variable can store any value whose type implements zero or more methods. This happens to match every type in Go. A common use case for the empty interface is a placeholder for data of uncertain schema.

```Go
var i interface{}
i = 20
i = "hello"
i = struct{
    FirstName string
    LastName string
}{
    FirstName: "Bob",
    LastName: "Smith",
}
```

```Go
// Using the empty interface as a placeholder for data
data := map[string]interface{}{}
contents, err := ioutil.ReadFile("test.json")
if err != nil {
    return err
}

err = json.Unmarshal(contents, &data)
if err != nil {
    return err
}

```

### Type assertions and type switches

Go provides two ways to check if a variable of an interface type is a specific concrete type: type assertions and type switches.


```Go
type MyInt int

func main() {
    var i interface{}
    var mine MyInt = 20
    i = mine
    i2 = i.(MyInt) // 👈 type assertion, i2 is of type MyInt

}

```

```Go
i2 := i.(string) // 👈 panic: interface conversion: interface {} is main.MyInt, not string
```

>[!Note]
> Even if two types share an underlying type, a type assertion must match the type of the underlying value.

To avoid crashing, use the following pattern


```Go
// comma ok idiom
i2, ok := i.(string) // 👈 type assertion with check
if !ok {
    return fmt.Errorf("unexpected type for %v", i)
}
```

> [!Note]
> A type assertion is very different from a type conversion. Type conversions can be applied to both concrete types and interfaces and are checked at compilation time. Type assertions can only be applied to interface types and are checked at runtime. Because they are checked at runtime, they can fail. Conversions change, assertions reveal.

Even if you are absolutely certain that your type assertion is valid, use the comma ok
idiom version. You don’t know how other people (or you in six months) will reuse
your code. Sooner or later, your unvalidated type assertions will fail at runtime.

When an interface could be one of multiple possible types, use a type switch instead

```Go
// type switches
func do(i interface{} {
    switch j := i.(type) {
    case nil:
        // i is nil, type of j is interface{}
    case int:
        // j is of type int
    case MyInt:
        // j is of type MyInt
    case io.Reader:
        // j is of type io.Reader
    case string:
        // j is of type string
    case bool, rune:
        // i is of type bool or rune, so j is of type interface{}
    default:
        // no idea what i is, so j is of type interface{}
    }
})
```

> [!Note]
> Since the purpose of a type switch is to derive a new variable from an existing one, it is idiomatic to assign the variable being switched on to a variable of the same name (`i := i.(type)`), making this one of the few places where shadowing is a good idea.


Type assertions and type switches should be used sparingly. For the most part, treat a parameter or return value as the type that was supplied and not what else it could be. Otherwise, your function’s API isn’t accurately declaring what types it needs
to perform its task. If you needed a different type, then it should be specified.

That said, there are use cases where type assertions and type switches are useful. One common use of a type assertion is to see if the concrete type behind the interface also implements another interface. This is called the optional interface technique.


```Go
// copyBuffer is the actual implementation of Copy and CopyBuffer.
// if buf is nil, one is allocated.
func copyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error) {
    // If the reader has a WriteTo method, use it to dos the copy.
    if wt, ok := src.(WriteTo); ok {
        return wt.WriteTo(dst)
    }

    // Similarly, if the writer has a ReadFrom method, use it to do the copy.
    if rf, ok := dst.(ReadFrom); ok {
        return rf.ReadFrom(src)
    }
    ...
}

```
The optional interface technique has one drawback: it doesn't work well with the decorator pattern. When an implementation wraps another to layer behaviour, a type assertion or type switch can't see optional interfaces implemented by the wrapped value. For example, bufio.NewReader wraps any io.Reader in a *bufio.Reader. If the original reader implemented io.ReaderFrom, wrapping it hides that interface and prevents the optimisation.

We see this with errors too. They implement the error interface and can wrap other errors to add information, but a type switch or assertion can't detect or match a wrapped error. To branch on the concrete type of a wrapped error, use ``errors.Is` and `errors.As` to test for and access it.


Type switches let you handle multiple implementations of an interface that each need different processing. They work best when only a limited set of valid types is expected. Always include a default case to handle types you didn't anticipate, protecting you if you forget to update the switch when adding new implementations:

```Go
func walkTree(t *treeNode) (int, error) {
    switch val := t.val.(type) {
    case nil:
        return 0, errors.New("invalid expression")
    case number:
        // we know that t is of type number, so return the int value
        return int(val), nil
    case operator:
        // we know that t.val is of type operator, so find
        // the values of the left and right children, then
        // call the process() method on operator to return
        // the result of processing their values.
        left, err := walkTree(t.lchild)
        if err != nil {
            return 0, err
        }
        right, err := walkTree(t.rchild)
        if err != nil {
            return 0, err
        }   
        return val.process(left, right), nil
    default:
        // if a new treeVal type is defined, but walkTree wasn't updated
        // to process it, this block detects it
        return 0, errors.New("unknown node type")
    }
}
```

You can further protect yourself from unexpected interface implementations by making the interface unexported and at least one method unexported. If the interface is exported, then it can be embedded in a struct in another package, making the struct implement the interface. 



### Function types are bridges to interfaces
Go allows methods on any user-defined type, including user-defined function types.


```Go
type Handler interface {
    ServeHTTP(http.ResponseWriter, *http.Request)
}

type HandlerFunc func(http.ResponseWriter, *http.Request)

func (f HandlerFunc) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    f(w, r)
}

```

This lets you implement HTTP handlers as functions, methods, or closures, all sharing the same code path as any other type that satisfies `http.Handler`.

Since functions are first-class in Go, they're often passed as parameters, and a single-method interface could just as easily replace a function-type parameter. So when should you use each? 

> [!Note] 
> If the function likely depends on other functions or state not captured in its parameters, use an interface and define a function type to bridge to it, as the `http` package does with `Handler` (the entry point for a chain of calls that needs configuring). For a simple, self-contained function like the one in `sort.Slice`, a function-type parameter is the better choice.


### Implicit interfaces make dependency injection easier

Applications inevitably change over time, and one technique for easing decoupling is dependency injection: the idea that your code should explicitly specify the functionality it needs to do its job. It's older than you might think, dating to Robert Martin's 1996 article "The Dependency Inversion Principle".

Go's implicit interfaces make dependency injection an excellent way to decouple code. Where other languages often rely on large frameworks to inject dependencies, Go needs no additional libraries. Let's work through a simple web application to see how implicit interfaces compose applications via dependency injection. (Go's built-in HTTP server support is covered in "The Server" on page 249; consider this a preview.) We'll start with a small logger utility and a data store:


```Go
// LogOutput is a plain function, not tied to any interface yet.
func LogOutput(msg string) {
    fmt.Println(msg)
}

// SimpleDataStore is a concrete data store backed by an in-memory map.
type SimpleDataStore struct {
    userData map[string]string
}

func (sds SimpleDataStore) UserNameForID(userID string) (string, bool) {
    name, ok := sds.userData[userID]
    return name, ok
}

// factory function to create a SimpleDataStore instance
func NewSimpleDataStore() SimpleDataStore {
    return SimpleDataStore{
        userData: map[string]string{
            "1": "Alice",
            "2": "Bob",
        },
    }
}
```

Next, the business logic needs data and wants to log when it's invoked. We don't want to bind it to `LogOutput` or `SimpleDataStore` directly, because we might swap in a different logger or store later. So the business logic depends on *interfaces* describing what it needs, not on concrete types:

```Go
// Interfaces describe what the business logic depends on,
// not the concrete implementations it gets at runtime.
type DataStore interface {
    UserNameForID(userID string) (string, bool)
}

type Logger interface {
    Log(msg string)
}
```

`LogOutput` doesn't yet meet the `Logger` interface, so we bridge it with a function type that has a `Log` method on it (the function-type-to-interface trick from the previous section):

```Go
// LoggerAdapter bridges the LogOutput function to the Logger interface.
type LoggerAdapter func(msg string)

func (lg LoggerAdapter) Log(msg string) {
    lg(msg)
}
```

By a stunning coincidence, `LoggerAdapter` and `SimpleDataStore` happen to meet the interfaces the business logic needs, but neither type knows anything about those interfaces. Now the business logic itself:

```Go
// SimpleLogic depends only on the interfaces, never the concrete types,
// so it carries no dependency on any particular logger or data store.
type SimpleLogic struct {
    l  Logger
    ds DataStore
}

func (sl SimpleLogic) SayHello(userID string) (string, error) {
    sl.l.Log("in SayHello for " + userID)
    name, ok := sl.ds.UserNameForID(userID)
    if !ok {
        return "", errors.New("unknown user")
    }
    return "Hello, " + name, nil
}

func (sl SimpleLogic) SayGoodbye(userID string) (string, error) {
    sl.l.Log("in SayGoodbye for " + userID)
    name, ok := sl.ds.UserNameForID(userID)
    if !ok {
        return "", errors.New("unknown user")
    }
    return "Goodbye, " + name, nil
}

// factory: accept interfaces, return a struct
func NewSimpleLogic(l Logger, ds DataStore) SimpleLogic {
    return SimpleLogic{
        l:  l,
        ds: ds,
    }
}
```

> [!Note]
> There's nothing in `SimpleLogic` that mentions the concrete types, so swapping in implementations from an entirely different provider is painless. This is unlike explicit interfaces in languages like Java, where the interface binds client and provider together, making a dependency far harder to replace. The unexported fields (`l`, `ds`) can only be set by code in the same package, which makes accidental modification less likely.

The API has a single `/hello` endpoint. The controller only needs business logic that says hello, so it defines a narrow interface owned by the client, customised to its needs (note `SayGoodbye` is deliberately left out):

```Go
// Logic is owned by the controller (the client) and contains only
// the methods it actually uses, even though SimpleLogic offers more.
type Logic interface {
    SayHello(userID string) (string, error)
}

type Controller struct {
    l     Logger
    logic Logic
}

func (c Controller) SayHello(w http.ResponseWriter, r *http.Request) {
    c.l.Log("In Controller.SayHello")
    userID := r.URL.Query().Get("userID")
    msg, err := c.logic.SayHello(userID)
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        w.Write([]byte(err.Error()))
        return
    }
    w.Write([]byte(msg))
}

// again: accept interfaces, return a struct
func NewController(l Logger, logic Logic) Controller {
    return Controller{
        l:     l,
        logic: logic,
    }
}
```

Finally, `main` is the only place that knows the concrete types and wires everything together:

```Go
func main() {
    // main is the only part of the code aware of the concrete types,
    // so swapping implementations only ever changes this function.
    l := LoggerAdapter(LogOutput)
    ds := NewSimpleDataStore()
    logic := NewSimpleLogic(l, ds)
    c := NewController(l, logic)

    // SayHello is treated as a function value here; HandleFunc converts
    // it to http.HandlerFunc, which satisfies the http.Handler interface.
    http.HandleFunc("/hello", c.SayHello)
    http.ListenAndServe(":8080", nil)
}
```

Externalising dependencies via dependency injection limits the changes needed to evolve the code over time. It also makes testing easier: a unit test can inject a type that captures log output and meets the `Logger` interface to validate logging behaviour.

> [!Note]
> Writing dependency injection wiring by hand can get tedious. [Wire](https://github.com/google/wire), a DI helper from Google, uses code generation to produce the concrete type declarations we wrote by hand in `main`.


### Go isn't particularly object-oriented (and that's great)

Go resists easy categorisation. It isn't strictly procedural, yet its lack of method overriding, inheritance, and objects means it isn't really object-oriented either. It has function types and closures but isn't a functional language. Force Go into any one of these categories and you end up with nonidiomatic code.

If Go's style needs a label, the best word is **practical**. It borrows concepts from many places with the overriding goal of being simple, readable, and maintainable by large teams over many years.


### Wrapping up

This chapter covered types, methods, interfaces, and their best practices, including how implicit interfaces enable dependency injection. Next up: how to properly use one of Go's most controversial features, errors.















[Back](../README.md)