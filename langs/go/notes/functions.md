## Functions
The basics of Go functions are similar to other languages. However, Go adds some twists on function features.


- A function declaration has four parts:
  - The `func` keyword
  - The function name
  - The input parameters, where the parameters are defined by name and type
  - The return type(s), written between the closing parenthesis and the opening brace of the function body
- If a function returns a value, you must supply a `return`. 
- If a function returns nothing, a `return `statement is not needed at the end of the function.
- The `return` keyword is only needed in a function that returns nothing if you are exiting from the function before the last line.

```Go
func div(numerator, denominator int) int {
    if denominator == 0 {
        return 0
    }
    return numerator / denominator
}
```

### Simulating Named and Optional Parameters
Go doesn’t have named and optional input parameters.

If you want to emulate named and optional parameters, define a struct that has fields that
match the desired parameters, and pass the struct to your function.

In practice, not having named and optional parameters isn’t a limitation. A function
shouldn’t have more than a few parameters, and named and optional parameters are
mostly useful when a function has many inputs. 


```Go
type Options struct {
    FirstName string
    LastName string
    Age int
}

func MyFunc(opts Options) error {
    // do smth here
}

func main() {
    MyFunc(Options{
        FirstName: "John",
        LastName: "Doe",
        Age: 30,
    })

    MyFunc(Options{
        FirstName: "Benjamin",
        LastName: "Dover",
    })
}

```

### Variadic Input Parameters and Slices
Go supports variadic parameters. The variadic parameter must be the last (or only) parameter in the input parameter list. You indicate it with three dots (…) before the type.


```Go
// The function below can be called in different ways:
// addTo(10)
// addTo(10, 1, 2, 3)
// addTo(10, []int{1, 2, 3}...)
func addTo(base int, vals ...int) int {
    out := make([]int, 0, len(vals))
    for _, v := range vals {
        out = append(out, base + v)
    }
    return out
}

```

- You can supply however many values you want for the variadic parameter, or no values at all.
- Since the variadic parameter is converted to a slice, you can supply a slice as the input. 
- However, you must put three dots (…) after the variable or slice literal. If you do not, it is a compile-time error.


### Multiple Return Values
Unlike other languages, Go allows for multiple return values.


```Go
func divAndRemainder(num, denom int) (int, int, error) {
    if denom == 0 {
        return 0, 0, fmt.Errorf("denominator cannot be zero")
    }
    return num / denom, num % denom, nil
}

// Example usage
func main() {
    result, remainder, err := divAndRemainder(10, 3)
    if err != nil {
        fmt.Println("Error:", err)
        os.Exit(1)
    }
    fmt.Printf("Result: %d, Remainder: %d\n", result, remainder)
}

```
- If a function returns multiple values, you must return all of them, separated by commas. 
- Don’t put parentheses around the returned values; that’s a compile-time error.
- By convention, the error is always the last (or only) value returned from a function.
- You must assign each value returned from a function. If you try to assign multiple return values to one variable, you get a compile-time error.


### Ignoring Return Values
If you want to ignore one or more return values from a function, use the blank identifier `_`.
Surprisingly, Go does let you implicitly ignore all of the return values for a function.
You can write `divAndRemainder(5,2)` and the returned values are dropped. We have
actually been doing this since our earliest examples: `fmt.Println` returns two values,
but it is idiomatic to ignore them. 


### Named Return Values
In addition to letting you return more than one value from a function, Go also allows
you to specify names for your return values.

```Go
func divAndRemainder(num, denom int) (res int, rem int, err error) {
    if denom == 0 {
        err = fmt.Errorf("denominator cannot be zero")
        return res, rem, err
    }
    res = num / denom
    rem = num % denom
    return res, rem, err
}

```

- When you supply names to your return values, what you are doing is pre-declaring variables that you use within the function to hold the return values. 
- They are written as a comma-separated list within parentheses.
- You must surround named return values in the function declaration with parentheses, even if there is only a single return value. 

- The name that’s used for a named returned value is local to the function; it doesn’t enforce any name outside of the function.
- It is perfectly legal to assign the return values to variables of different names

```Go
func main() {
    x, y, z := divAndRemainder(5, 2)
}

```

- Just like any other variable, you can shadow a named return value. Be sure that you are assigning to the return value and not to a shadow of it
- The other problem with named return values is that the named return parameters give a way to declare an intent to use variables to hold the return values, but
don’t require you to use them.

```Go
func divAndRemainder(num, denom int) (res int, rem int, err error) {
    //Notice that we assigned values to res and rema and then returned different values directly.
    res, rem = 20, 30 
    if denom== 0 {
        return 0, 0, errors.New("cannot divide by zero")
    }
    return num / denom, num % denom, nil  // The values from the return statement are returned even though they were never assigned to the named return parameters.  
} 
```

- The Go compiler inserts code that assigns whatever is returned to the return parameters 
  (i.e. `res = num / denom, rem = num % denom, err = nil`) before the function exits.


### Blank Returns. Avoid these
If you have named return values, you can just write return without specifying the values that are returned. This returns the last values assigned to the named return values.

When there’s invalid input, we return immediately. Since no values were assigned to result and
remainder, their zero values are returned. If you are returning the zero values for your named return values, be sure they make sense. 

Go developers consider blank returns a bad idea because they make it harder to understand data flow. Good software is clear and readable; it’s obvious what is happening. When you use a blank return, the reader of your code needs to scan back through the program to find the last value assigned to the
return parameters to see what is actually being returned.


```Go
func divAndRemainder(num, denom int) (res int, rem int, err error) {
    if denom == 0 {
        err = fmt.Errorf("denominator cannot be zero")
        return // blank return
    }
    res = num / denom
    rem = num % denom
    return // blank return
}

```

### Functions Are Values

Just like in many other languages, functions in Go are values. The type of a function
is built out of the keyword `func` and the types of the parameters and return values.
This combination is called the signature of the function. Any function that has the
exact same number and types of parameters and return values meets the type signature.

```Go
func add(a, b int) int { return a + b}
func sub(a, b int) int { return a - b}
func mul(a, b int) int { return a * b}
func div(a, b int) int { return a / b}


var operations = map[string]func(int, int) int {
    "add": add,
    "sub": sub,
    "mul": mul,
    "div": div,
}

func main() {
    expressions := [][]string{
        []string{"2", "+", "3"},
        []string{"2", "-", "3"},
        []string{"2", "*", "3"},
        []string{"2", "/", "3"},
        []string{"2", "%", "3"}, // invalid operator
        []string{"two", "+", "three"}, // invalid operands
        []string{"5"}
    }

    for _, expression := range expressions {
        if len(expression) != 3 {
            fmt.Println("invalid expression:", expression)
            continue
        }
        p1, err := strconv.Atoi(expression[0]) // convert a string to an int
        if err != nil {
            fmt.Println(err)
            continue
        }

        op := expression[1]
        ofFunc, ok := operations[op]
        if !ok {
            fmt.Println("unsupported operator:", op)
            continue
        }
        p2, err := strconv.Atoi(expression[2])
        if err != nil {
            fmt.Println(err)
            continue    
        }

        res := opFunc(p1, p2) // calling the function storeed in the operations map
        fmt.Println(res)
    }
}

```

>[!NOTE] 
> Error handling is what separates the professionals from the amateurs.

### Function Type Declarations
- Just like you can use the `type` keyword to define a struct, you can use it to define a
function type, too. 
- We don’t have to modify the functions at all. Any function that has two input parameters of type int and a single return value of type int automatically meets the type and
can be assigned as a value in the map.


```Go
type opFuncType func(int, int) int

var operations = map[string]opFuncType{
    // same body as before
}
```

### Anonymous Functions
- Not only can you assign functions to variables, you can also define new functions
within a function and assign them to variables.

- These inner functions are anonymous functions; they don’t have a name. You don’t
have to assign them to a variable, either. You can write them inline and call them
immediately. 

```Go
func main() {
    for i := 0; i < 5; i++ {
        // Anonymous function definition and invocation
        func(j int){
            fmt.Println("Anonymous function called with", j)
        }(i) // immediately invoke the anonymous function with i as input
    }
}

```

- You declare an anonymous function with the keyword func immediately followed by
the input parameters, the return values, and the opening brace. 
- It is a compile-time error to try to put a function name between func and the input parameters.
- There are two situations where declaring anonymous functions without assigning them to variables is useful: `defer` statements and launching goroutines.


### Closures
Functions declared inside of other functions are called _closures_ and are able to
access and modify variables declared in the outer function. 

- Closures really become interesting when they are passed to other functions or returned from a function. 
- They allow you to take the variables within your function
and use those values outside of your function.


#### Passing Functions as Parameters

If you aren’t used to treating functions like data, you might need a moment to think about
the implications of creating a closure that references local variables and then passing
that closure to another function. It’s a very useful pattern and appears several times in
the standard library.

```Go
type Person struct {
    FirstName string
    LastName string
    Age int
}

people := []Person{
    {"Pat", "Patterson", 37},
    {"Tracy", "Bobbert", 23},
    {"Fred", "Fredson", 18},
}


// Sort by LastName
sort.Slice(people, func(i, j int) bool {
    return people[i].LastName < people[j].LastName
})

// The closure that’s passed to sort.Slice has two parameters, i and j, 
// but within the closure, we can refer to people so we can sort it by the LastName field.

// Sort by Age
sort.Slice(people, func(i, j int) bool {
    return people[i].Age < people[j].Age
})

fmt.Println(people)
```
>[!NOTE]
> Passing functions as parameters to other functions is often useful for performing different operations on the same kind of data.

#### Returning Functions from Functions
Not only can you use a closure to pass some function state to another function, you
can also return a closure from a function. 

```Go
func makeMult(base int) func(int) int {
    return func(factor int) int  {
        return base * factor
    }
}

func main() {
    twoBase := makeMult(2)
    threeBase := makeMult(3)

    for i := 0; i < 3; i++ {
        fmt.Println(twoBase(i), threeBase(i))
    }
}

// output:
// 0 0
// 2 3
// 4 6
```

### defer
Programs often create temporary resources, like files or network connections, that
need to be cleaned up. This cleanup has to happen, no matter how many exit points a
function has, or whether a function completed successfully or not. In Go, the cleanup
code is attached to the function with the `defer` keyword.


```Go
// Simple version of the unix cat command
func main() {
    if len(os.Args) < 2 {
        log.Fatal("no file specified")
    }

    f, err := os.Open(os.Args[1]
    if err != nil {
        log.Fatal(err)
    }

    defer f.Close() // ensure the file is closed when main exits

    data := make([]byte, 2048)
    for {
        count, err := f.Read(data)
        os.Stdout.Write(data[:count])
        if err != nil {
            if err != io.EOF {
                log.Fatal(err)
            }
            break
        }
    }
}

```

- You can defer multiple closures in a Go function. They run in LIFO order; the last defer registered runs first.
- The code within defer closures runs after the return statement. 


```Go
func DbWrite(ctx context.Context, db *sql.DB, val1, val2 string) (err error) {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    
    defer func() {
        if err != nil {
            err = tx.Commit()
        }
        if err != nil {
            tx.Rollback()
        }
    }()
    _, err = tx.ExecContext(ctx, "INSERT INTO FOO (val) values $1", val1)
    if err != nil {
        return err
    }

    return nil
}

```

A common pattern in Go is for a function that allocates a resource to also return a
closure that cleans up the resource. 

```Go
// helper function that opens a file and returns a closure
func getFile(name string) (*os.File, func(), error) {
    file, err := os.Open(name)
    if err != nil {
        return nil, nil, err
    }
    return file, func() {
        file.Close()
    }, nil
}

f, closer, err := getFile(os.Args[1])
if err != nil {
    log.Fatal(err)
}
defer closer()
```

- Go doesn’t allow unused variables, returning the closer from the function
means that the program will not compile if the function is not called. That reminds
the user to use defer.


### Call by Value
Go is a call-by-value language. When you pass a variable to a function, what gets
passed is a copy of the variable. If the variable is a struct, the entire struct is
copied. If the variable is a pointer, the pointer is copied, but not the value pointed to.


```Go
func modMap(m map[int]string) {
    m[2] = "hello"
    m[3] = "goodbye"
    delete(m, 1)
}

func modSlice(s []int) {
    for k, v := range s {
        s[k] = v * 2
    }
    s = append(s, 10)
}


func main() {
    m := map[int]string{
        1: "first",
        2: "second",
    }

    modMap(m) // map[2: hello 3: goodbye]

    s := []int{1, 2, 3}
    modSlice(s) // s: [2, 4, 6]

```


- Any changes made to a map parameter are reflected in the variable passed into the function.
- For a slice, it’s more complicated. You can modify any element in the slice, but you can’t lengthen the slice.
- Every type in Go is a value type. It’s just that sometimes the value is a pointer.
- There are cases where you need to pass something mutable to a function. What do you do then? That’s when you need a pointer.