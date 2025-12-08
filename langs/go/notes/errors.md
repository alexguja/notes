## Errors

Go handles errors by returning a value of type error as the last return value for a function. This is entirely by convention, but it is such a strong convention that it should never be breached. When a function executes as expected, nil is returned for the error parameter. If something goes wrong, an error value is returned instead. 

```Go
func calcRemainderAndMod(num, denom int) (int, int, error) {
    if denom == 0 {
        return 0, 0, errors.New("denominator cannot be zero")
    }
    return num / denom, num % denom, nil
}

// Error handling
rem, mod, err := calcRemainderAndMod(20, 3)
if err != nil {
    fmt.Println(err)
}
```

- In most cases, you should set the other return values to their zero values when a non-nil error is returned.
-  The reason why we return `nil` from a function to indicate that no error occurred is that nil is the zero value for any interface type.
- The Go compiler requires that all variables must be read. Making errors returned values forces developers to either check and handle error conditions or make it explicit that they are ignoring errors by using an underscore `_` for the returned error value.
- The error handling is indented inside an if statement. The business logic is not. This gives a quick visual clue to which code is along the “golden path” and which code is the exceptional condition.s
- `error` is a built-in interface that defines a single method.
  
```Go
type error interface {
    Error() string
}
```

### Using strings for simple errors

- Go’s standard library provides two ways to create an error from a string. The first is the errors.New function.
- The first is the `zerrors.New` function. It takes in a string and returns an error. This string is returned when you call the Error method on the returned error instance. 
- The second way is to use the `fmt.Errorf` function. This function allows you to use all of the formatting verbs for `fmt.Printf` to create an error. Like `errors.New`, this string is returned when you call the Error method on the returned error instance


```Go
// Using errors.New
func doubleEven(i int) (int, error) {
    if i % 2 != 0 {
        return 0, errors.New("only even numbers are processed")
    }
    return i * 2, nil
}

// Using fmt.Errorf
func doubleEven(i int) (int, error) {
    if i % 2 != 0 {
        return 0, fmt.Errorf("%d is not an even number", i)
    }
    return i * 2, nil
}
```

### Sentinel Errors

Sentinel errors are one of the few variables that are declared at the package level. By convention, their names start with Err (with the notable exception of io.EOF). They
should be treated as read-only; there’s no way for the Go compiler to enforce this, but it is a programming error to change their value. Sentinel errors are usually used to indicate that you cannot start or continue processing.


```Go
package consterr

type Sentinel string

func(s Sentinel) Error() string {
    return string(s)
}

package mypkg

const (
    // This looks like a function call, but it’s actually casting a string literal 
    // to a type that implements the error interface. 
    ErrFoo = consterr.Sentinel("foo error")
    ErrBar = consterr.Sentinel("bar error")
)
```

At first glance, the above example looks like a good solution. However, this practice isn’t considered idiomatic. If you used the same type to create constant errors across packages, two errors would be equal if their error strings are equal. They’d also be equal to a string literal with the same value.

Meanwhile, an error created with errors.New is only equal to itself or to variables explicitly assigned its
value. You almost certainly do not want to make errors in different packages equal to each other

The sentinel error pattern is another example of the Go design philosophy. Sentinel
errors should be rare, so they can be handled by convention instead of language rules.
Yes, they are public package-level variables. This makes them mutable, but it’s highly
unlikely someone would accidentally reassign a public variable in a package. In short,
it’s a corner case that is handled by other features and patterns. The Go philosophy is
that it’s better to keep the language simple and trust the developers and tooling than it
is to add additional features.

