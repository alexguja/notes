## Composite Types
In Go, composite types are types that are composed of other types. They allow you to create complex data structures by combining simpler ones. The main composite types in Go are arrays, slices, structs, and maps.

### Arrays
Arrays are rarely used directly in Go since they are static and have quite a few restrictions.

- All elements must be of the same type in an array.
- All elements are initialized to the zero value of the array's type.
- The size of the array is fixed and cannot be changed.
- Go considers the size of the array to be part of its type.
- Negative indices are not allowed.
- Variables cannot be used to specify the size of the array. This is because array sizes must be resolved at compile time.
- Type conversion cannot be used to convert arrays of different sizes to identical types.
- Arrays can be compared using the `==` and `!=` operators.


```Go
// Array declarations
var x [3]int                              // Specify the size and type of the array
var x = [3]int{10, 20, 30}                // Array literal with initial values
var x = [...]int{10, 20, 30}              // Array literal with inferred size
var x = [12]int{1, 5:4, 6, 10:100, 15}    // Sparse array  [1, 0, 0, 0, 0, 4, 6, 0, 0, 0, 100, 15]
var x [2][3]int                           // Multi-dimensional array based on one-dimensional arrays
                                          // This is an array of size 2 where tye type is [3]int

// Reads and writes
x[0] = 1                                  // Write to the array
var y = x[2]                              // Read from the array


// Array comparisons
var x = [...]int{1, 2, 3}
var y = [3]int{1, 2, 3}

x == y                                    // Evaluates to true

// Built-in functions
len(x)                                    // Evaluates the length of the array

```

> [!NOTE] 
> Reads or writes past the array bounds will compile, but cause a runtime panic.
> Don't use arrays unless the length is known in advance. For must purposes, use _slices_



### Slices
Slices can be thought of as Go's dynamic arrays. The size of the slice is _not_ part of its type, which makes it more flexible than the array.

- The zero value for a slice is `nil`.
- Slices are _not_ comparable. The only thing a slice can be compared to is `nil` (`x == nil`).
- If you need slice comparison, it can be achieved using the `reflect` pacakge (See `DeepEqual`).
- Since Go 1.21 slices can also be compared using `slices.Equal` and `slices.EqualFunc`.
- Every slice has a length and a capacity.
  - The capacity of the slice represents the number of reserved consecutive memory locations.
  - The length of the slice represents the current number of elements in the slice.
  - The length of the slice `s` should not exceed the capacity, or `len(s) <= cap(s)`.
  - When the length of the slice reaches its capacity, a new slice is created, typically double in size.
  - Empty slices cannot be read into (i.e. cannot use `s[0]` if `s` is empty).
  

> [!NOTE]
> In Go, `nil` is an identifier that represents a missing value for some type. `nil` has no type, so it can be assigned or compared against values of different types.

```Go
var x []int{}                             // Zero length slice different from nil
var x = []int{10, 20, 30}                 // Slice literal.
                                          // The size does not need to be specified

var x = []int{1, 5:4, 6, 10:100, 15}      // Sparse slice  
var x = [][]int                           // Multi-dimensional slice


// Reads and writes: must be within bounds
x[0] = 1                                  // Write to the slice
var y = x[2]                              // Read from the slice


// Built-in functions
len(x)                                    // Evaluates the length of the slice
cap(x)                                    // Evaluates the capacity of the slice

var x []int
x = append(x, 1)                          // Append 1 to the slice
x = append(x, 2, 3)                       // Append multiple values to the slice
y := []int{20, 30, 40}
x = append(x, y...)                       // Append another slice to the slice

```


>[!WARNING]
> It is a compile-time error if you forget to assign the value returned from append. 

>[!NOTE]
> Go is a call by value language. Every time you pass a parameter to a function, Go makes a copy of the value that’s passed in.


```Go
// More len and cap examples
var x []int
fmt.Println(x, len(x), cap(x)) // [] 0 0

x = append(x, 10)
fmt.Println(x, len(x), cap(x)) // [10] 1 1

x = append(x, 20)
fmt.Println(x, len(x), cap(x)) // [10 20] 2 2

x = append(x, 30)
fmt.Println(x, len(x), cap(x)) // [10 20 30] 3 4

x = append(x, 40)
fmt.Println(x, len(x), cap(x)) // [10 20 30 40] 4 4

x = append(x, 50)
fmt.Println(x, len(x), cap(x)) // [10 20 30 40 50] 5 8

```

While it’s nice that slices grow automatically, it’s far more efficient to size them once.


### `make`
`make` allows us to create empty slices with a specified length and capacity (optional).



```Go
// Example using make
x := make([]int, 5)              // Creates an int slice with length 5 and capacity 5
x := make([]int, 5, 10)          // Creates an int slice with length 5 and capacity 10
x := make([]int, 0, 10)          // Creates an int slice with length 0 and capacity 10

```



### Slice declarations
When declaring slices, the goal is to minimise the need for slice doubling.
Below are some rules of thumb

- When the slice doesn't need to grow, use a `var` declaration with no assigned value (`var x []int`). 
  This is useful for functions that might not return anything.
- When you have initial values that are not likely to change, use a slice literal (`var x = []int{2, 4, 6}`).
- If you have an approximate idea of how large the slice needs to be, but don't know exactly, use `make`. There are 3 common scenarios:
  - If you're using a slice as buffer, specify a non-zero length.
  - If the size is known upfront, specify the `length`, and index into the slice to set values. This approach is often used in slice transformations
  - Otherwise, specifcy a zero length and a non-zero capacity. Items can then be appended to the slice.


