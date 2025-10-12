## Testing 

Over the past two decades, the widespread adoption of automated testing has probably done more to improve code quality 
than any other software engineering technique. As a language and ecosystem focused on improving software quality, it’s not
surprising that Go includes testing support as part of its standard library. 


### Basics of testing
Go’s testing support has two parts: libraries and tooling. The `testing` package in the
standard library provides the types and functions to write tests, while the `go test`
tool that’s bundled with Go runs your tests and generates reports.

>[!NOTE]
> Go tests are placed in the same directory and the same package as the production code. Since tests are located in the same package, they are able to access and test unexported functions and variables.


```Go
// Basic example
func addNumbers(x, y int) int {
    return x + x // deliberately bugged: it should say x + y
}

func Test_addNumbers(t *testing.T) {
    result := addNumbers(2,3)
    if result != 5 {
        t.Error("incorrect result: expected 5, got", result)
    }
}

```

- Every test is written in a file whose name ends with `_test.go` next to the original file. For example `foo.go` would have a test `foo_test.go` in the same directory.
- Test functions start with the word Test and take in a single parameter `t` of type `*testing.T`.
- Test functions do not return any values.
- The name of the test (apart from starting with the word “Test”) is meant to document what you are testing, so pick something that explains what you are testing
- When writing unit tests for individual functions, the convention is to name the
unit test Test followed by the name of the function.
- When testing unexported functions, some people use an underscore between the word Test and the name of the
function.
- The `go test` command allows you to specify which packages to test. Using `./…` for the package name specifies that you want to run tests in the current directory and all of the subdirectories of the current directory. Include a `-v` flag to get verbose testing output.



```sh
# Running tests
$ go test
--- FAIL: Test_addNumbers (0.00s)
    adder_test.go:8: incorrect result: expected 5, got 4
FAIL
exit status 1
FAIL test_examples/adder 0.006s
```

Fixing the code 

```Go
// Basic example
func addNumbers(x, y int) int {
    return x + y
}
```


And running the tests again

```sh
# Running fixed code
$ go test
PASS
ok test_examples/adder 0.006s
```

### Reporting Test Failures
We've already seen the `t.Error` reporting.

```Go
t.Errorf("incorrect result:expected %d, got %s", 5, result) // Printf-style formatting
```

- While `Error` and `Errorf` mark a test as failed, the test function continues running.
- If you think a test function should stop processing as soon as a failure is found, use the
`Fatal` and `Fatalf` methods.
- Note that this doesn’t exit all tests; any remaining test functions will execute after the current test function exits.
- If the failure of a check in a test means that further checks in the same test function will always fail or cause the test to panic, use `Fatal` or `Fatalf`. If you are testing several independent items (such as validating fields in a struct), then use `Error` or `Errorf` so you can report as many problems at once.



### Setting up and Tearing Down

Sometimes you have some common state that you want to set up before any tests run
and remove when testing is complete. Use a `TestMain` function to manage this state
and run your tests

```Go
var testTime time.Time

func TestMain(m *testing.M) {
    fmt.Println("Set up struff for tests here")
    testTime = time.Now()
    exitVal := m.Run()
    fmt.Println("Cleanup stuff for tests here")
    os.Exit(exitVal)
}

func TestFirst(t *testing.T) {
    fmt.Println("testFirst uses stuff set up in TestMain", testTime)
}

func TestSecond(t *testing.T) {
    fmt.Println("testSecond also uses stuff set up in TestMain", testTime)
}

```

- Both `TestFirst` and `TestSecond` refer to the package-level variable `testTime`. 
- We declare a function called `TestMain` with a parameter of type `*testing.M`. 
- Running `go test` on a package with a `TestMain` function calls the function instead of invoking the
tests directly. 
- Once the state is configured, call the `Run` method on `*testing.M` to run
the test functions. 
- The Run method returns the exit code; 0 indicates that all tests
passed. 
- Finally, you must call `os.Exit` with the exit code returned from `Run`.
- Be aware that `TestMain` is invoked once, not before and after each individual test.
- Also be aware that you can have only one `TestMain` per package.


```sh
# Running the tests above
$ go test
Set up stuff for tests here
TestFirst uses stuff set up in TestMain 2020-09-01 21:42:36.231508 -0400 EDT m=+0.000244286
TestSecond also uses stuff set up in TestMain 2020-09-01 21:42:36.231508 -0400 EDT m=+0.000244286
PASS
Clean up stuff after tests here
ok test_examples/testmain 0.006s
```

There are two common situations where `TestMain` is useful:
- When you need to set up data in an external repository, such as a database
- When the code being tested depends on package-level variables that need to be initialized

- The `Cleanup` method on `*testing.T`. is used to clean up temporary resources created for a single test.
- This method has a single parameter, a callback function with no input parameters or return values.
- The function runs when the test completes.
- For simple tests, you can achieve the same result by using a `defer` statement, but `Cleanup` is useful when tests rely on helper functions to set up sample data.


```Go
// createFile is a helper function called from multiple tests
func createFile(t *testing.T) (string, error) {
    f, err := os.Create("tempFile")
    if err != nil {
        return "", err
    }

    t.Cleanup(func() {
        os.Remove(f.Name())
    })
    return f.Name(), nil
}

func TestFileProcessing(t *testing.T) {
    fName, err := createFile(t)
    if err != nil {
        t.Fatal(err)
    }
    // testing code, without worrying about cleanup
}

```


As `go test` walks your source code tree, it uses the current package directory as the
current working directory. If you want to use sample data to test functions in a package,
create a subdirectory named `testdata` to hold your files.

- Go reserves this directory name as a place to hold test files. 
- When reading from `testdata`, always use a relative file reference.
- Since `go test` changes the current working directory to the current
package, each package accesses its own `testdata` via a relative file path.


### Caching Test Results

Go caches compiled packages if they haven’t changed, Go also caches test results when running tests across multiple packages if
they have passed and their code hasn’t changed. The tests are recompiled and rerun if you change any file in the package or in the testdata directory. You can also force tests to always run if you pass the flag `-count=1` to go test.

You can also use `go clean -testcache` to clean the test cache manually.


### Testing a Public API
If you want to test just the public API of your package, Go has a convention for specifying this. 
You still keep your test source code in the same directory as the production source code, but you use `packagename_test` for the package name.

- The advantage of using the `_test` package suffix is that it lets you treat your package as a black box.
- It's possible to have both black box and white box tests in the same directory.

```Go
// adder.go
package adder

// 👇 methods needs to be exported 
func AddNumbers(x, y int) int {
    return x + y
}


// adder_test.go
package adder_test

import (
    "testing" // the tested package needs to imported explicitly
    "adder"
)

func TestAddNumbers(t *testing.T) {
    result := adder.AddNumbers(2, 3)
    if result != 5 {
        t.Error("incorrect result: expect 5, got", result)
    }
}
```


### Use `go-cmp` to Compare Test Results
Use Google's `go-cmp` package to compare test results effectively instead of relying on `reflect.DeepEqual` to compare structs, slices, and maps.

```Go
type Person struct {
    Name string
    Age int
    DateAdded time.Time
}

func CreatePerson(name string, age int) Person {
    return Person{
        Name: name,
        Age: age,
        DateAdded  time.Now(),
    }
}

func TestCreatePerson(t *testing.T) {
    result := CreatePerson("Dennis", 37)
    expected := Person{
        Name: "Dennis",
        Age: 37,
        DateAdded: time.Now(),
    }
    if diff := cmp.Diff(result, expected); diff != "" {
        t.Error(diff)
    }
}
```


```sh
# Running the tests above
$ go test
--- FAIL: TestCreatePerson (0.00s)
cmp_test.go:16: ch13_cmp.Person{
    Name: "Dennis",
    Age: 37,
  - DateAdded: s"0001-01-01 00:00:00 +0000 UTC",
  + DateAdded: s"2020-03-01 22:53:58.087229 -0500 EST m=+0.001242842",
}
FAIL
FAIL cmp 0.006s
```

The lines with a - and + indicate the fields whose values differ. The test failed because
our dates didn’t match. We can’t control what date is assigned by the `CreatePerson` function. We have to ignore the `DateAdded` field. 
You can do that by specifying a comparator function. Declare the function as a local variable
in your test:

```Go
comparer := cmt.Comparer(func(x, y Person) bool {
    return x.Name == y.Name && x.Age == y.Age
})

if diff := cmp.Diff(result, expected, comparer); diff != "" {
    t.Error(diff)
}
```

- Pass a function to the `cmp.Comparer` function to create a customer comparator. 
- The function that’s passed in must have two parameters of the same type and return a `bool`.
- It must also be symmetric (the order of the parameters doesn’t matter), deterministic (the same input always returns the same output) and pure (the function doesn’t have side effects).

```sh
# Running the tests above
$ go test
PASS
ok cmp 0.006s
```

### Table Tests
Most of the time, it takes more than a single test case to validate that a function is
working correctly. You could write multiple test functions to validate your function or
multiple tests within the same function, but you’ll find that a great deal of the testing
logic is repetitive. You set up supporting data and functions, specify inputs, check the
outputs, and compare to see if they match your expectations. Rather than writing this
over and over, you can take advantage of a pattern called table tests.

```Go
func DoMath(num1, num2 int, op string) (int, error){
    switch op {
    case "+":
        return num1 + num2, nil
    case "-":
        return num1 - num2, nil
    case "*":
        return num1 + num2, nil // 👈 deliberate bug, this could lead to 100% code coverage and yet the code is not correct
    case "/":
        if num2 == 0 {
            return 0, errors.New("division by zero")
        }
        return num1 / num2, nil
    default:
        return 0, fmt.Errorf("invalid operator: %s", op)
    }
}

// Table test example

// 👇 First, we declare a slice of anonymous structs
// The struct contains fields for the name of the test, the input parameters, and
// the return values. Each entry in the slice represents another test.
data := []struct {
    name string
    num1 int
    num2 int
    op string
    expected int
    errMsg string
}{
    {"add", 2, 2, "+", 4, ""},
    {"subtract", 2, 2, "-", 0, ""},
    {"multiply", 2, 2, "*", 4, ""},
    {"divide", 2, 2, "/", 1, ""},
    {"bad_division", 2, 0, "?", 0, "invalid operator: ?"},
}

// 👇 Next, we loop over each test case in data, invoking the Run method each time.
for _, d := range data {
    t.Run(d.name, func(t, *testing.T){  // 👈 the line that does the magic
        result, err := DoMath(d.num1, d.num2, d.op)
        if result != d.expected {
            t.Errorf("expected %d, got %d", d.expected, result)
        }
        var errMsg string
        if err != nil {
            errMsg = err.Error()
        }
        if errMsg != d.errMsg {
            t.Errorf("expected error %s, got %s", d.errMsg, errMsg)
        }
    })
}

```

### Checking code coverage
Code coverage is a very useful tool for knowing if you’ve missed any obvious cases.
However, reaching 100% code coverage doesn’t guarantee that there aren’t bugs in
your code for some inputs.

- Adding the `-cover` flag to the go test command calculates coverage information
and includes a summary in the test output.
- Adding the `-coverprofile=c.out` flag to the go test command saves the coverage information to a file.
  

Use `go test -v -cover -coverprofile=c.out` to check the code coverage of your tests. This will show the coverage percentage.
Use `go tool cover -html=c.out` to open the coverage report in your browser to see what lines were not covered by tests.



>[!WARNING] 
> Code coverage is necessary, but it’s not sufficient. You can have 100% code coverage and still have bugs in your code!