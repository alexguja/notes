## Composite Types
In Go, composite types are types that are composed of other types. They allow you to create complex data structures by combining simpler ones. The main composite types in Go are arrays, slices, structs, and maps.

### Arrays


```Go
var x [3]int // specify the size and type of the array
var x = [3]int{10, 20, 30} // array literal
var x = [...]int{10, 20, 30} // array literal with inferred size
var x = [12]int{1, 5:4, 6, 10:100, 15} // sparse array  [1, 0, 0, 0, 0, 4, 6, 0, 0, 0, 100, 15]
var x [2][3]int // multidimensional array based on one-dimensional arrays

```

Arrays can be compared using the `==` and `!=` operators.

```Go
var x = [...]int{1, 2, 3}
var y = [3]int{1, 2, 3}

x == y // evaluates to true

```








