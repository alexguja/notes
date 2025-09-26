## Blocks, Shadows, and Control Structures

### Blocks
Each place where a variable declaration can occur is called a _block_. For example, Go 
allows variable declartion outside of functions, as function parameters, and as local variables within functions.

Variables, constants, types, and functions declared outside of any function are placed in the _package_ block.

Import statements define names for other packages that are valid for the file that declares the import statement. These names are in the _file_ block.

All of the variables defined at the top level of a function, including the function parameters are in a block. Within every function, every set of `{}` defines a block.

Identifiers defined in an outer block can be accessed in an inner block. If there are identical declarations in an outer block and an inner block, this is called _shadowing_.

### Shadowing

A shadowing variable is a variable that has the same name as a variable in a containing block. For as long as the shadowing variable exists, you cannot access a shadowed variable.

```Go
// Shadowing
func main() {
    x := 10
    if x > 5 {          // start of block
        fmt.Println(x)  // x = 10
        x := 5          // Shadowing the outer block x (the shadowed variable)
        fmt.Println(x)  // x = 5 (the shadowing variable)
    }                   // end of block
    fmt.Println(x)      // x = 10, the original outer block variable x
}
```

```Go

func main() {
    x := 10
    if x > 5 {
        x, y := 5, 20
        fmt.Println(x, y) // x = 5, y = 20, the outer block x was shadowed 
    }
    fmt.Println(x) // x = 10
}

```

>[!NOTE]
>When using `:=`, make sure that you don’t have any variables from
an outer scope on the lefthand side, unless you intend to shadow them.
Note, it's also possible to accidentally shadow package names.


### Detecting shadowed variables 
Use the followign tool and command to detect shadowed variables in your projects.

```sh 
go install golang.org/x/tools/go/analysis/passes/shadow/cmd/shadow@latest
```

```Makefile
# Example makefile
vet:
    go vet ./...
    shadow ./...
.PHONY:vet    
```

>[!NOTE]
>In Go, the built-in types like `int` or `string`, constants (like `true` and `false`), 
and functions like `make` or `close` aren't included in the language keywords list.
Instead, they are considered as predefiend identifiers and are defined in the _universe_ block. This is the block that holds all other blocks. You should never redefine these identifiers.


### `if` statements
- The most visible difference between if statements in Go and other languages is that
you don’t put parenthesis around the condition.
- Any variable declared within the braces of an if or else statement exists only within that block.

```Go
n := rand.Intn(10)
if n == 0 {
    fmt.Println("That's too low")
} else if n > 5 {
    fmt.Println("That's too big:", n)
} else {
    fmt.Println("That's good number:", n)
}

```

- Go adds is the ability to declare variables that are scoped to the condition and to both the `if` and `else` blocks. Note thse variables can still shadow other variables in an outer block.

```Go
if n := rand.Intn(10); n == 0 {
    fmt.Println("That's too low")   
} else if n > 5 {
    fmt.Println("That's too big:", n)
} else {
    fmt.Println("That's a good number", n)
}

fmt.Println(n) // n is undefined beyond the if/else blocks

```

### `for`, Four Ways
The `for` keyword is the only looping keyword in Go.
There are four ways of using it:
- A complete C-style `for`
- A condition-only `for`
- An infinite `for`
- `for-range`
  

```Go
// A complete for statement
for i := 0; i < 10; i++ {
    fmt.Println(i)
}
```


- You must use `:=` to initialize the variables; `var` is not legal here.
- Variables can be shadowed here.


```Go
// A condition-only statement. A while loop equivalent
i := 1
for i < 100 {
    fmt.Println(i)
    i = i * 2
}

```

```Go
// Infinite loop, press ctrl + c to break
func main() {
    for {
        fmt.Println("Hello")
    }
}
```


```Go
// For-range looop
vals := []int{2, 4, 6, 8, 10, 12}
for i, v := range vals {
    fmt.Println(i ,v)
}

```


### `break` and `continue`
The `break` statement allows breaking out of a loop. The `continue` statement skips over the rest of the body of a for
loop and proceeds directly to the next iteration.

```Go
// do-while equivalent in Go

for {
    if !condition {
        break
    }
}
```

Example where `continue` makes the code easier to read:

```Go
// Confusing code
for i := 1; i <= 100; i++ {
    if i % 3 == 0 {
        fmt.Println("FizzBuzz")
    } else {
      fmt.Println("Fizz")
    } else if i % 5 == 0 {
        fmt.Println("Buzz")
    } else {
        fmt.Println(i)
    }
}

```

The example above is not idiomatic. Go encourages short if-statement bodies, as left-aligned as possible.
Nested code is difficult to follow. The next example uses a `continue` statement to make it simpler.


```Go
for i := 1; i <= 100; i++ {
    if i % 3 == 0 && i % 5 == 0 {
        fmt.Println("FizzBuzz")
        continue
    }

    if i % 3 == 0 {
        fmt.Println("Fizz")
        continue    
    }
    
    if i % 5 == 0 {
        fmt.Println("Buzz")
        continue
    }
    fmt.Println(i)
}

```


### The `for-range` statement
The `for-range` statement iterates over a variety of data structures, including arrays, slices, maps, strings, and channels.

- When ranging over an array or slice, two values are returned for each iteration. The first is the index of the element, and the second is a copy of the element at that index.
- The iteration happens over a copy of the compound, so modifying an element does not change the original compound.

```Go
vals := []int{2, 4, 6, 8, 10, 12}
for i, v := range vals {
    fmt.Println(i ,v)
}

// When the index is not needed, use _ to ignore it
for _, v := range vals {
    fmt.Println(v)
}

// When you want the key but not the value, you can just omit it
m := map[string]bool{"a": true, "b": false, "c": true}
for k := range m {
    fmt.Println(k) // prints a, b, c in some order
}
```

>[!NOTE]
> The idiomatic names for the two loop variables depend on
what is being looped over. When looping over an array, slice, or string, an `i` for index
is commonly used. When iterating through a map, `k` (for key) is used instead. The second variable is frequently called `v` for value, but is sometimes given a name based on the type of the values being iterated. If there are only a few statements in the body of the loop,
single letter variable names work well. For longer (or nested) loops, you’ll want to use
more descriptive names.

>[!NOTE]
> Any time you are in a situation where there’s a value returned, but you want to ignore it, use an underscore `_` to hide the value.


#### Iterating over maps
Keep in mind, iterating over maps does not guarantee any specific order. If you need a specific order, you must sort the keys first.
This is a security feature to make Hash DoS attacks more difficult to carry out.

```Go
m := map[string]int{"a": 3, "b": 1, "c": 2}
for i := 0; i < 3; i++ {
    fmt.Println("Loop", i)
    for k, v := range m {
        fmt.Println(k, v) // prints the map k,v pairs in some order
    }
}
```

#### Iterating over strings

 
```Go
samples := []string{"hello", "こんにちは", "👋🌍"}
for _, s := range samples {
    for i, r := range s {
        fmt.Println(i, r, string(r)) // r is a rune (Unicode code point)
    }
    fmt.Println()
}
```


### Labeling `for` statements

```Go
func main() {
    samples := []string{"hello", "apple"}
outer:
    for _, sample := range samples {
        for i, r := range sample {
            fmt.Println(i, r, string(r))
            if r == 'l' {
                continue outer 
            }
        }
        fmt.Println()
    }
}

// Nested loops with labels are rare
outer:
    for _, outerVal := range outerValues {
        for _, innerVal := range outerVal {
            // process innerVal
            if invalidCondition(innerVal) {
                continue outer // Skip to the next iteration of the outer loop
            }
        }
        // here the code will run only if all innerVals were processed successfully
    }
```

### Choosing the right `for` statement
>[!NOTE]
> Favor a for-range loop when iterating over all the contents of an
> instance of one of the built-in compound types. It avoids a great
> deal of boilerplate code that’s required when you use an array, slice,
> or map with one of the other for loop styles.

- The complete `for` loop is best used for situation where you aren't iterating from the first element to the last element in a compound type.
- The remaining two for statement formats are used less frequently. The condition-
only `for` loop is, like the `while` loop it replaces, useful when you are looping based on
a calculated value.
- The infinite `for` loop is used in long-running processes, such as servers, or when
waiting for events to occur, such as input from a user or messages arriving on a channel.
There should always be a break
somewhere within the body of the for loop. Real-world programs should bound iteration and fail gracefully when operations cannot be completed.

### `switch` statements
- As is the case with `if` statements, the condition in a `switch` statement does not need to be enclosed in parentheses.
- Also like an `if` statement, you can declare a variable that is scoped to all of the branches of the `switch` statement. In the example below `size` is scoped to the entire `switch`.
- All of the case clauses (and the optional default clause) are contained inside a single set of
braces.
- You can have multiple lines inside a case (or default) clause and they are all
considered to be part of the same block.
- By default, cases in switch statements in Go don’t fall through.
- An empty case is allowed, and it simply does nothing.
- Go includes the `fallthrough` keyword for completeness, but its use is discouraged.
- You can switch on any type that is comparable with `==`. This includes all of the predeclared types except slices, maps, channels, functions, and structs that contain fields of these types.
- When using a `switch` statement inside a loop, you need to label the `for` statement and include it after the `break`. If you don't include a label, Go will break out of the `switch` case.

```Go
words := []string{
    "a",
    "cow",
    "smile",
    "gopher",
    "octopus",
     "anthropologist",
}

for _, word := range words {
    switch size := len(word); size {
    case: 1, 2, 3, 4: // separate multiple cases with commas
        fmt.Printf("%s is a short word\n", word)
    case 5:
        wordLen := len(word)
        fmt.Println(word, "is exactly the right length:", wordLen)
    case 6, 7, 8, 9: // empty case, nothing happens
    default: // optional
        fmt.Printf("%s is a long word\n", word)
    }
}

```


```Go
// Incorrect break behavior
func main() {
    for i := 0; i < 10; i++ {
        switch {
        case i % 2 == 0:
            fmt.Println(i, "is even")
        }
        case i % 3 == 0:
            fmt.Println(i, "is divisible by 3, but not 2")
        }
        case i % 7 == 0:
            fmt.Println("exit the loop")
            break // only breaks out of the case
        default:
            fmt.Println(i, "is boring")
    }
}

// Correct break behavior with label
func main() {
    loop:
    for i := 0; i < 10; i++ {
        switch {
        case i % 2 == 0:
            fmt.Println(i, "is even")
        case i % 3 == 0:
            fmt.Println(i, "is divisible by 3, but not 2")
        case i % 7 == 0:
            fmt.Println("exit the loop")
            break loop // breaks out of the loop
        default:
            fmt.Println(i, "is boring")
        }
    }
}

```


### Blank `switch`
A blank `switch` is a switch statement with no condition. It is equivalent to `switch true`.
This construct is useful when you want to use a `switch` statement instead of a long
series of `if-else-if` statements.

A regular `switch` only allows you to check a value for equality. A blank `switch` allows you to use any boolean comparison for each case


```Go
words := []string{"hi", "salutations", "hello"}
for _, word := range words {
    switch wordLen := len(word); {
    case wordLen < 5: // notice the boolean expression
        fmt.Println(word, "is a short word")
    case wordLen > 10:
        fmt.Println(word, "is a long word")
    default:
        fmt.Println(word, "is exactly the right length")
    }
}
```

### Choosing between `if` and `switch`
- Use an `if` statement when you have a simple condition that results in a boolean value
- Use a `switch` statement when you have multiple conditions that result in discrete values, especially when those values are known at compile time.
- Favor blank `switch` statements over if/else chains when you have
multiple related cases. Using a switch makes the comparisons more visible and reinforces that they are a related set of concerns


```Go
// if-else statement
n := rand.Intn(10)
if n == 0 {
    fmt.Println("That's too low")
} else if n > 5 {
    fmt.Println("That's too big:", n)
} else {
    fmt.Println("That's good number:", n)
} 

// switch statement
switch n := rand.Intn(10); {
case n == 0:
    fmt.Println("That's too low")
case n > 5:
    fmt.Println("That's too big:", n)
default:
    fmt.Println("That's a good number:", n)
}
```

### `goto` statements

- The `goto` statement provides an unconditional jump from the `goto` to a labeled statement in the same function. It can be useful for breaking out of nested loops or for error handling in some cases.
- Labeled `break` and `continue` statements allow you to jump out of deeply nested loops or skip iteration, so `goto` is rarely needed in Go.
- Littering your code with boolean flags to control the logic flow is arguably the same functionality as the `goto` statement, just more verbose.

```Go
// The code below causes errors. It shows why goto is discouraged.
func main() {
    a := 10
    goto skip
    b := 20
skip:
    c := 30
    fmt.Println(a, b, c)
    if c > a {
        goto inner
    }
    if a < b {
    inner:
        fmt.Println("a is less than b")
    }
}

// Errors:
// goto skip jumps over declaration of b at ./main.go:8:4
// goto inner jumps into block starting at ./main.go:15:11


// Valid use case for goto
func main() {
    a := rand.Intn(10)
    for a < 100 {
        if a % 5 == 0 {
            goto done
        }
        a = a * 2
    }
    fmt.Println("the loop completes normally")
done:
    fmt.Println("done:", a)
}

```

