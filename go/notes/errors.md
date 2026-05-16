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
- The reason why we return `nil` from a function to indicate that no error occurred is that nil is the zero value for any interface type.
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
Sentinel errors are one of the few variables that are declared at the package level. By convention, their names start with Err (with the notable exception of io.EOF). They should be treated as read-only; there’s no way for the Go compiler to enforce this, but it is a programming error to change their value. Sentinel errors are usually used to indicate that you cannot start or continue processing.


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

Meanwhile, an error created with errors.New is only equal to itself or to variables explicitly assigned its value. You almost certainly do not want to make errors in different packages equal to each other

The sentinel error pattern is another example of the Go design philosophy. Sentinel errors should be rare, so they can be handled by convention instead of language rules. They are public package-level variables. This makes them mutable, but it’s highly unlikely someone would accidentally reassign a public variable in a package. In short, it’s a corner case that is handled by other features and patterns.


### Errors Are Values
Since error is an interface, you can define your own errors that include additional information for logging or error handling. For example, you might want to include a status code as part of the error to indicate the kind of error that should be reported back to the user.


```Go
type Status int

const (
    InvalidLogin Status = iota + 1
    NotFound
)

type StatusErr struct {
    Status Status
    Message string
}

func (se *StatusErr) Error() string {
    return se.Message
}


// use StatusErr to provide more details about what went wrong:
func LoginAndGetData(uid, pwd, file string) ([]byte, error) {
    err := login(uid, pwd)
    if err != nil {
        return nil, StatusErr{
            Status: InvalidLogin,
            Message: fmt.Sprintf("invalid credentials for user %s", uid),
        }
    }

    data, err := getData(file)
    if err != nil {
        return nil, StatusErr{
            Status: NotFound,
            Message: fmt.Sprintf("file %s not found", file),
        }
    }
    return data, nil
}
```

- Even when you define your own custom error types, always use error as the return type for the error result. 
- If you are using your own error type, be sure you don’t return an uninitialized instance. This means that you shouldn’t declare a variable to be the type of your custom error and then return that variable.

```Go
func generateError(falg bool) error {
    var genErr StatusErr // Avoid doing this
    if flag {
        genErr = StatusErr{
            Status: NotFound
        }
    }
    return genErr // don’t return an uninitialized instance. 
}


func main() {
    err := generateError(true)
    fmt.Println(err != nil) // true
    err = generateError(false)
    fmt.Println(err != nil) // true, err is non-nil is that error is an interface
}


// Fix
func generateError(flag bool) error {
    if flag {
        return &StatusErr{
            Status: NotFound,
        }
    }
    return nil // explicitly return nil
}

// Another fix
func generateError(flag bool) error {
    var genErr error // Use the error type
    if flag {
        genErr = &StatusErr{
            Status: NotFound,
        }
    }
    return genErr // genErr is nil if flag is false
}
```

### Wrapping Errors
When an error is passed back through your code, you often want to add additional context to it. This context can be the name of the function that received the error or the operation it was trying to perform. When you preserve an error while adding additional information, it is called wrapping the error. When you have a series of wrapped errors, it is called an error chain.

There’s a function in the Go standard library that wraps errors, and we’ve already seen
it. The `fmt.Errorf` function has a special verb, `%w`. Use this to create an error whose
formatted string includes the formatted string of another error and which contains the original error as well. The convention is to write `: %w` at the end of the error format string and make the error to be wrapped the last parameter passed to `fmt.Errorf`.

The standard library also provides a function for unwrapping errors, the `Unwrap` function in the errors package. You pass it an error and it returns the wrapped error,
if there is one. If there isn’t, it returns `nil`. 

>[!NOTE]
> You don’t usually call errors.Unwrap directly. Instead, you use
`errors.Is` and `errors.As` to find a specific wrapped error

```Go
func fileChecker(name string) error {
    f, err := os.Open(name)
    if err != nil {
        return fmt.Errorf("in fileChecker: %w", err) // wrap the error
    }
    f.Close()
    return nil
}

func main() {
    err := fileChecker("not_here.txt")
    if err != nil {
        fmt.Println(err
        if wrappedErr := errors.Unwrap(err); wrappedErr != nil {
            fmt.Println("wrapped error:", wrappedErr)
        }
    }
}

```

- If you want to wrap an error with your custom error type, your error type needs to
implement the method `Unwrap`.

```Go
type StatusErr struct {
    Status Status
    Message string
    Err error
}

func (se StatusErr) Error() string {
    return se.Message
}

func (se StatusErr) Unwrap() error {
    return se.err
}

// use StatusErr to wrap underlying errors
func LoginAndGetData(uid, pwd, file string) ([]byte, error) {
    err := login(uid, pwd)
    if err != nil {
        return nil, StatusErr{
            Status: InvalidLogin,
            Message: fmt.Sprintf("invalid credentials for user %s", uid),
            Err: err,
        }
    }

    data, err := getData(file)
    if err != nil {
        return nil, StatusErr{
            Status: NotFound,
            Message: fmt.Sprintf("file %s not found", file),
            Err: err,
        }
    }
    return data, nil
}

```

- Not all errors need to be wrapped. A library can return an error that means processing cannot continue, but the error message contains implementation details that aren’t needed in other parts of your program. In this situation it is perfectly acceptable to return a new error without wrapping the original error.

- If you want to create a new error that contains the message from another error, but don’t want to wrap it, use `fmt.Errorf` to create an error, but use the %v verb instead of `%w`

```Go
err := internalFunc()
if err != nil {
    return fmt.Errorf("internal failure: %v", err)
}
```

### Is and As
Wrapping errors is a useful way to get additional information about an error, but it introduces problems. If a sentinel error is wrapped, you cannot use `==` to check for it, nor can you use a type assertion or type switch to match a wrapped custom error. Go solves this problem with two functions in the `errors` package, `Is` and `As`.

```Go
func fileChecker(name string) error {
    f, err := os.Open(name)
    if err != nil {
        return fmt.Errorf("in fileChecker: %w", err) // wrap the error
    }
    f.Close()
    return nil
}

func main() {
    err := fileChecker("not_here.txt")
    if err != nil {
        if errors.Is(err, os.ErrNotExist) {
            fmt.Println("the file does not exist")
        }
    }
}

```


- By default, `errors.Is` uses `==` to compare each wrapped error with the specified error. 
- If this does not work for an error type that you define, implement the `Is` method for your error type.


```Go
type MyErr struct {
    Codes []int
}

func (me MyErr) Error() string {
    return fmt.Sprintf("codes: %v", me.Codes)
}

// Implement Is method to customize comparison
func (me MyErr) Is(target error) bool {
    if me2, ok := target.(MyErr); ok {
        return reflect.DeepEqual(me, me2)
    }
    return false
}

```

- Another use for defining your own Is method is to allow comparisons against errors that aren’t identical instances.  You might want to pattern match your errors, specifying a filter instance that matches errors that have some of the same fields. 

```Go
type ResourceErr struct {
    Resource string
    Code int
}

func (re ResourceErr) Error() string {
    return fmt.Sprintf("%s: %d", re.Resource, re.Code)
}

// If we want two ResourceErr instances to match when either field is set, we can do so
// by writing a custom Is method
func (re ResourceErr) Is(target error) bool {
    if other, ok := target.(ResourceErr); ok {
        ignoreResource := other.Resource == ""
        ignoreCode := other.Code == 0
        
        matchResource := other.Resource == re.Resource
        matchCode := other.Code == re.Code

        return matchResouce && matchCode ||
            matchResouce && ignoreCode || 
            ignoreResource && matchCode
    }

    return false
}

// Find all errors that refer to the database, regardless of the code
if errors.Is(err, ResourceErr{Resource: "Database"}) {
    fmt.Println("The database is broken:", err)
    // process the codes
} 
```

- The `errors.As` function allows you to check if a returned error (or any error it
wraps) matches a specific type. It takes in two parameters. The first is the error being
examined and the second is a pointer to a variable of the type that you are looking for.
If the function returns `true`, an error in the error chain was found that matched, and
that matching error is assigned to the second parameter. If the function returns `false`, no match was found in the error chain.

```Go
err := someFunc()
var myErr MyErr
if errors.As(err, &myErr) {
    fmt.Println(myErr.Code)
}
```
- Note that you use var to declare a variable of a specific type set to the zero value. You
then pass a pointer to this variable into `errors.As`.
- You don’t have to pass a pointer to a variable of an error type as the second parameter
to errors.As. You can pass a pointer to an interface to find an error that meets the
interface
- If the second parameter to `errors.As` is anything other than a pointer to an error or a pointer to an interface, the method panics.

```Go
err := someFunc()
var coder interface {
    Code() int
}

if errors.As(err, &coder) {
    fmt.Println(coder.Code())
}
```

>[!NOTE]
>Use ``errors.Is` when you are looking for a specific instance or specific values. Use `errors.As` when you are looking for a specific type


### Wrapping Errors with `defer`

```Go
// Wrapping multiple errors with the same message
func doSmth(val1 int, val2 string) (string, error) {
    val3, err := doThing1(val1)
    if err != nil {
        return "", fmt.Errorf("in doSmth: %w", err)
    }
    val4, err := doThing2(val2)
    if err != nil {
        return "", fmt.Errorf("in doSmth: %w", err)
    }
    res, err := doThing3(val3, val4)
    if err != nil {
        return "", fmt.Errorf("in doSmth: %w", err)
    }
    return res, nil
}

// Simplify the above using defer
func doSmth(val1 int, val2 string) (_ string, err error) {
    defer func() {
        // In the defer closure, we check if an error was returned. If so, we reassign the error to
        // a new error that wraps the original error with a message that indicates which function
        // detected the error.
        if err != nil {
            err = fmt.Errorf("in doSmth: %w", err)
        }
    }()

    val3, err := doThing1(val1)
    if err != nil {
        return "", err
    }
    val4, err := doThing2(val2)
    if err != nil {
        return "", err
    }

    return doThing3(val3, val4)
}
```


### `panic` and `recover`
Go generates a panic whenever there is a situation where the Go runtime is unable to figure out what should happen next. This could be due to a programming error (like an attempt to read past the end of a slice) or environmental problem (like running out of memory). As soon as a panic happens, the current function exits immediately and any defers attached to the current function start running. When those defers complete, the defers attached to the calling function run, and so on, until main is reached. The program then exits with a message and a stack trace.

If there are situations in your programs that are unrecoverable, you can create your own panics. The built-in function panic takes one parameter, which can be of any type. Usually, it is a string. 

```Go
func doPanic(msg string) {
    panic(msg)
}

func main() {
    doPanic(os.Args[0])
}

```

Go provides a way to capture a panic to provide a more graceful shutdown or to prevent shutdown at all. 
The built-in recover function is called from within a defer to check if a panic happened. If there was a panic, the value assigned to the panic is returned. Once a recover happens, execution continues normally. 

```Go
func div60(i int) {
    defer func() {
        if v := recover(); v != nil {
            fmt.Println(v)
        }
    }()
    fmt.Println(60 / i)
}

func main() {
    for _, v := range []int{1, 2, 0, 6}{
        div60(v)
    }
}

```

>[!NOTE]
>There’s a specific pattern for using `recover`. We register a function with defer to handle a potential panic. We call recover within an if statement and check to see if a non-nil value was found. You must call recover from within a defer because once a panic happens, only deferred functions are run.


While panic and recover look a lot like exception handling in other languages, they are not intended to be used that way. Reserve panics for fatal situations and use recover as a way to gracefully handle these situations. 

If your program panics, be very careful about trying to continue executing after the panic. It’s very rare that you want to keep your program running after a panic occurs. If the panic was triggered
because the computer is out of a resource like memory or disk space, the safest thing to do is use recover to log the situation to monitoring software and shut down with `os.Exit(1)`.

The reason we don’t rely on panic and recover is that recover doesn’t make clear what could fail. It just ensures that if something fails, we can print out a message and continue. Idiomatic Go favors code that explicitly outlines the possible failure conditions over shorter code that handles anything while saying nothing.

There is one situation where recover is recommended. If you are creating a library for third parties, do not let panics escape the boundaries of your public API. If a panic is possible, a public function should use a recover to convert the panic into an error, return it, and let the calling code decide what to do with them.


### Stack Traces
You can use error wrapping to build a call stack by hand, but there are third-party libraries with error types that generate those stacks automatically. The best known [third-party library](https://pkg.go.dev/github.com/pkg/errors) provides functions for wrapping errors with stack traces. By default, the stack trace is not printed out. If you want to see the stack trace, use `fmt.Printf` and the verbose output verb `(%+v)`. 

>[!NOTE]
> When you have a stack trace in your error, the output includes the full path to the file on the computer where the program was compiled. If you don’t want to expose the path, use the -trimpath flag
when building your code. This replaces the full path with the package.


