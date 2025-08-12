## Composite Types
In Go, composite types are types that are composed of other types. They allow you to create complex data structures by combining simpler ones. The main composite types in Go are arrays, slices, structs, and maps.

### Arrays
Arrays are rarely used directly in Go since they are static and have quite a few restrictions.

- All elements must be of the same type in an array.
- All elements are initialized to the zero value of the array's type.
- The size of the array is fixed and cannot be changed.
- Go considers the size of the array to be part of its type.
- Variables cannot be used to specify the size of the array. This is because array sizes must be resolved at compile time.
- Type conversion cannot be used to convert arrays of different sizes to identical types.
- Arrays can be compared using the `==` and `!=` operators.


```Go
// Array declarations
var x [3]int                              // specify the size and type of the array
var x = [3]int{10, 20, 30}                // array literal with initial values
var x = [...]int{10, 20, 30}              // array literal with inferred size
var x = [12]int{1, 5:4, 6, 10:100, 15}    // sparse array  [1, 0, 0, 0, 0, 4, 6, 0, 0, 0, 100, 15]
var x [2][3]int                           // multidimensional array based on one-dimensional arrays
                                          // this is an array of size 2 where tye type is [3]int

// Reads and writes
x[0] = 1                                  // write to the array
var y = x[2]                              // read from the array


// Array comparisons
var x = [...]int{1, 2, 3}
var y = [3]int{1, 2, 3}

x == y                                    // evaluates to true

// Built-in functions
len(x)                                    // evaluates the length of the array

```

> [!NOTE] 
> Reads or writes past the array bounds will compile, but cause a runtime panic.
> Don't use arrays unless the length is known in advance. For must purposes, use _slices_








