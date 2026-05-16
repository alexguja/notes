## Declarations

### `var` vs `:=`

In Go, `var` is used to declare a variable with a specific type, while `:=` is a shorthand for declaring and initializing a variable without explicitly specifying its type. The `:=` syntax can only be used within functions.

```Go
// Using var to declare a variable with a specific type
var x int = 10  
var y = 10 // Type is inferred as int
// Using := to declare and initialize a variable without specifying its type
z := 20
```

Here are some more examples using `var` declarations:

```Go
var x, y = 10, 20

var x, y int

var x, y = 10, "hi"

var (
    x int
    y = 20
    z int = 30
    d, e = 40, "hello"
    f,g string
)
```

Here are some examples using `:=` declarations:

```Go
x := 10 // same as var x = 10
y, z := 20, "hello" // same as var y, z = 20, "hello"
```

>[!NOTE]
> The `:=` operator can do one trick that you cannot do with `var`: it allows you to assign
values to existing variables, too. As long as there is one new variable on the lefthand
side of the `:=`, then any of the other variables can already exist



Good practices to follow
- When initializing a variable to its zero value, use `var x int`. This makes it clear
that the zero value is intended.
- When assigning an untyped constant or a literal to a variable and the default type
for the constant or literal isn’t the type you want for the variable, use the long `var`
form with the type specified. While it is legal to use a type conversion to specify
the type of the value and use `:=`to write `x := byte(20)`, it is idiomatic to write
`var x byte = 20`.
- While var and `:=` allow you to declare multiple variables on the same line, only use
this style when assigning multiple values returned from a function or the comma ok
idiom
- As a general rule, you should only declare variables in the package block that are effectively immutable.


### Using `const`
In Go, `const` is used to declare constants, which are immutable values that cannot be changed after they are defined. Constants can be of any type, including basic types like `int`, `float64`, and `string`. Constants in Go are a way to give names to literals. They can only hold values that the compiler can figure out at compile time.

```Go
const x int64 = 10

const (
    idKey = "id"
    nameKey = "name"
)

const z = 20 * 10

```


[Back](../README.md)