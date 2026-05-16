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

```
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


