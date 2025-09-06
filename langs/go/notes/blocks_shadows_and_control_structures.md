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

