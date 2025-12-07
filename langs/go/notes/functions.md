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