# Constant in go

- In go, a constant is a variable whose value cannot be changed once it has been assigned. Constants are declared using the `const` keyword, followed by the name of the constant, its type (optional), and its value.
- Value of a constant must be a compile-time constant, which means it must be known at the time of compilation. Constants can be of any basic data type, such as integers, floats, strings, and booleans.
- Type of a constant can be explicitly specified, or it can be inferred from the value assigned to it. If the type is not specified, the constant will take on the default type for its value, but in this case, it is untyped and can be used in any context where a value of that type is expected.  
  Example:

```go
const x = 10 // untyped int constant
fmt.Printf("%T\n", x) // Output: int
var y float64 = x // y is a float64 variable, and because 10 is valid for float64, this assignment is allowed
fmt.Printf("%T\n", y) // Output: float64
```

- The answer is untyped constants have a default type associated with them and they supply it if and only if a line of code demands it.
  Example:

```go
const x = 10 // untyped int constant
y := x // y is inferred to be of type int because x is an untyped constant and the division operation requires a specific type
fmt.Printf("%T\n", y) // Output: int
```

# Package in go

<https://www.alexedwards.net/blog/an-introduction-to-packages-imports-and-modules>

A package in Go is essentially a named collection of one or more related .go files. In Go, the primary purpose of packages is to help you isolate and reuse code.

## Main package

Main package is a special package in Go that serves as the entry point for the program. It is where the `main` function resides, which is executed when the program is run. The `main` package is unique in that it cannot be imported by other packages, and it must contain a `main` function to be executable.

# Go build

- go build main.go : This command only compiles the main.go file and produces an executable binary in the current directory. The name of the binary will be the same as the name of the directory containing the main.go file.
- go build -o myprogram main.go : This command compiles the main.go file and produces an executable binary named myprogram in the current directory. The -o flag allows you to specify the name of the output binary.
- go build . : Build current package like go build
- go build: build current package and its dependencies. It will produce an executable binary in the current directory if the package is a main package, or it will produce a library file if the package is not a main package.

=> go build will build current package or can pass a specific package path to build that package. It will also build any dependencies of the package being built. The resulting binary will be placed in the current directory, and its name will be the same as the package name.
if pass file name to go build, it will only build that file and its dependencies, and produce an executable binary in the current directory. The name of the binary will be the same as the name of the file being built.
