## Predeclared Types

The built-in Go types are called predeclared types. They are the basic building blocks of Go programs and include booleans, integers, floats, strings, etc.

> [!NOTE]  
> Go assigns a default zero value to declared but unassigned variables. For example, an `int` variable will have a default value of `0`, a `bool` will be `false`, and a `string` will be an empty string `""`, etc. This eliminates a lot of bugs found in C/C++ programs.

### Literals

Literals the raw, direct representation of values in your code (e.g., `123`, `'a'`, `"hello"`).

Types of Literals:

- **Integer Literals**: Whole numbers. Can be base 10 (normal), binary (`0b`), octal (`0o` or confusingly `0`), or hexadecimal (`0x`). You can use underscores (e.g., `1_000_000`) for readability, but not at the start/end or next to each other.

  ```Go
  // Examples of integer literals
  123         // Decimal literal
  0b1010      // Binary literal for 10
  0o12        // Octal literal for 10
  0xA         // Hexadecimal literal for 10
  1_000_000   // Readable integer with underscores
  ```

- **Floating-Point Literals**: Numbers with decimal points

  ```Go
  // Examples of floating-point literals
  3.14                  // Decimal literal
  6.03e23               // Scientific notation
  0x1.999999999999ap+3  // Hexadecimal floating-point literal
  ```

- **Rune Literals**: Single characters, enclosed in single quotes (e.g., `'a'`). They represent Unicode characters and can be written in various hexadecimal or octal escape forms (e.g., `'\u0061'`). Common escapes include `\n` (newline), `\t` (tab), `\'`, `\"`, `\\`.

  ```Go
  // Examples of rune literals
  'a'         // Character 'a'
  '\n'        // Newline character
  '\u0061'    // Unicode for 'a'
  '\x61'      // Hexadecimal for 'a'
  '\u{1F600}' // Unicode emoji (grinning face)
  ```

- **String Literals**: Sequences of characters.
  Can be enclosed in double quotes (interpreted strings) or backquotes (raw strings).

  ```Go
  // Examples of string literals
  "Hello, World!"          // Interpreted string
  "Line1\nLine2"          // Newline in interpreted string
  `This is a raw string`   // Raw string, no escapes
  `Multi-line
  raw string`              // Multi-line raw string
  ```

- **Interpreted Strings**: Enclosed in double quotes (e.g., `"Hello\nWorld"`). Backslashes are used for escapes.

- **Raw Strings**: Enclosed in backquotes (e.g., `` `Hello\nWorld` ``). They interpret everything literally, including newlines and backslashes, except for backquotes themselves. Useful for multi-line strings or strings with many backslashes.

> [!Note]
> Practical Advice: Generally use base-10 for numbers and double quotes for strings. Avoid complex rune escapes unless they make the code much clearer. Hexadecimal/binary are sometimes used for bitwise operations or networking.

### Untyped Nature of Literals

A key concept in Go is that literals are untyped. This means a number like `100` doesn't have a specific type (like `int32` or `int64`) until it's assigned to a variable or used in an expression that forces a type.

This allows flexibility: an integer literal `100` can be used directly in a floating-point expression or assigned to a floating-point variable. However, type compatibility still applies: you can't assign a number literal to a string variable, or a float literal to an integer variable.

There are size limitations: a literal cannot overflow the variable it's assigned to (e.g., `1000` cannot be assigned to a byte variable, which maxes out at `255`). If a literal's type isn't clear from the context, Go assigns it a default type (e.g., `int` for integers, `float64` for floats).

### Booleans

The `bool` type represents boolean values, which can be either `true` or `false`.

```Go
// Example of boolean literals
var flag bool // Default unassigned value is false
var isTrue bool = true
var isFalse bool = false
```

### Numeric Types

Go has 12 different numeric types which can be viewed into 3 categories: integers, floating-point numbers, and complex numbers.

```Go
// Example of numeric types
var i int = 42
var f float64 = 3.14
var c complex128 = 1 + 2i
```

#### Integer Types

Go has several integer types, both signed and unsigned, with varying sizes. The size of an integer type determines its range of values.

| Type              | Value Range                                             |
| ----------------- | ------------------------------------------------------- |
| `int8` or `byte`  | -128 to 127                                             |
| `int16`           | -32,768 to 32,767                                       |
| `int32` or `rune` | -2,147,483,648 to 2,147,483,647                         |
| `int64`           | -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807 |
| `uint8`           | 0 to 255                                                |
| `uint16`          | 0 to 65,535                                             |
| `uint32`          | 0 to 4,294,967,295                                      |
| `uint64`          | 0 to 18,446,744,073,709,551,615                         |
| `int`             | Platform-dependent (32 or 64 bits)                      |
| `uint`            | Platform-dependent (32 or 64 bits)                      |
| `uintptr`         | Unsigned integer large enough to hold a pointer         |

#### Floating-Point Types

Go uses the IEEE 754 specification, giving a large range and limited precision.

| Type      | Largest absolute value  | Smallest absolute value |
| --------- | ----------------------- | ----------------------- |
| `float32` | 3.402823466e+38         | 1.401298464324817e-45   |
| `float64` | 1.7976931348623157e+308 | 5.0000000000000000e-324 |

> [!NOTE]
> While Go lets you use `==` and `!=` to compare floats, don’t do it. Due to the inexact nature of floats, two floating point values might not be equal when you think they should be. Instead, define a maximum allowed variance and see if the difference
> between two floats is less than than that variance. For example:

```Go
const epsilon = 0.00001
if math.Abs(a - b) < epsilon {
    // a and b are considered equal
}
```

#### Complex Types

Go has first-class support for complex numbers, with two types: `complex64` and `complex128`.

```Go
// Example of complex types
var c1 complex64 = 1 + 2i
var c2 complex128 = 3 + 4i
```

### Strings and Runes

The `string` type represents a sequence of Unicode characters, while the `rune` type represents a single Unicode character.
The zero value of a string is an empty string `""`, and the zero value of a rune is `0`, which represents the Unicode character U+0000 (the null character).

```Go
// Example of strings and runes
var s string = "Hello, 世界"
var r rune = '世'
```

Like integers and floats, strings are compared for equality using `==`, difference
with `!=`, or ordering with `>`, `>=`, `<`, or `<=`. They are concatenated by using the `+`
operator.


### Type Conversion
To prevent bugs associated with automatic type conversion, Go requires explicit type conversion between different types. This is done using the syntax `Type(value)`. Since all type conversions are explicit, Go doesn’t allow truthiness. In fact, no
other type can be converted to a bool, implicitly or explicitly.

```Go
var x int = 8
var y float64 = 30.2
var z = float64(x) + y
var a int = x + int(y)
```

>[!NOTE]
> Idiomatic Go values comprehensibility over conciseness.

