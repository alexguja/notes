## Types, Methods, and Interfaces

In Go, you can define your own types and associate methods with them. This allows you to create complex data structures and behaviors. Additionally, Go has a powerful interface system that enables you to define behavior without specifying the underlying type.


### Types in Go

You can define a new type using the `type` keyword. For example, let's define a `Person` struct:
```go
type Person struct {
    FirstName string
    LastName  string
    Age int

}

```

You can also use the primitive types to create new types. For example:
```go   
type Score int
type Converter func(string)Score
type TeamScore map[string]Score


````

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

Method declarations look just like function declarations, with one addition: the receiver specification.The receiver appears between the keyword func and the name of the method. Just like all other variable declarations, the receiver name appears before the type. By convention, the receiver name is a short abbreviation of the type’s name, usually its first letter. It is nonidiomatic to use this or self.

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

Go, `p.Method()` is just sugar for `Method(p)`.

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
> It's clever that Go allows calling methods on `nil` instances, however most of the time this is not very useful. Pointer receivers work jus like pointer function parameters. It is a copy of the pointer that gets passed to functions. 
Just like nil parameters passed to funcions, if you change the copy of the pointer, you haven't changed the original. This means you cannot write a pointer receiver method that handles `nil` and makes the original pointer non-nil.

### Methods are functions too
Methods in Go are a lot like functions. You can use a method as a replacement for a function any time here's a variable of parameter of a function type

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
>Package-level state should be immutable. Any time your logic depends on values that are configured at startup or chagned while your program is running those values should be stored in a struct and that logic should be implemented as a method. If your logic only depends on the input parameters, then it should be a function.


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
Types can be viewed as documenation. They make code clearer by providing a name for a concept and describing the kind of data that is expected. They make code clearer by providing a name
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
common pattern is to assign the first iota value in the constant block to or to a constant that indicates the value is invalid.


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

out := Outer{
    Inner: Inner{X: 1},
    X: 2,
}

o.X       // 2
o.Inner.X // 1

```