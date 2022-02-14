# Program Structure

* Expressions are combinations of values and functions that are combined and interpreted by the compiler to create a new value,
* Declarations are constructs that introduce identifiers
* Statement is a standalone unit of execution and doesn't return anything

Expressions are used in statements whose execution order is determined by control-flow statements like if and for

Statements are grouped into functions for isolation and reuse.

Functions are gathered into source files and packages.


## Name

 a name begins with a letter (that is, anything that Unicode deems a letter) or an underscore and may have any number of additional letters, digits, and underscores
 
 *  If declared outside of a function, however, it is visible in all files of the package to which it belongs. 
 *  If the name begins with an upper-case letter, it is exported, which means that it is visible and accessible outside of its own package and may be referred to by other parts of the program,
 
 
 ## Declarations
 > A declaration names a program entity and specifies some or all of its propertie

There are four major kinds of declarations: `var, const, type, and fun`


Each `.go` file 
1.  begins with a package declaration that says what package the file is part of.
2.  The package declaration is followed by any import declarations
3.   a sequence of package-level declarations of types, variables, constants, and functions, in any order. 

```
package main
import "fmt"
const boilingF = 212.0
func main() {
```

## Variables

```
var name type = expression

var i, j, k int                 // int, int, int
var b, f, s = true, 2.3, "four" // bool, float64, string
var f, err = os.Open(name)  
```

### Short Variable Declarations

```
 name := expression
 i, j := 0, 1
```
the type of name is determined by the type of expression.

 a short variable declaration does not necessarily declare all the variables on its left-hand side.
 ```
 in, err := os.Open(infile)
out, err := os.Create(outfile)
 ```
 
 A short variable declaration must declare at least one new variabl
 ```
f, err := os.Open(infile)
// ...
f, err := os.Create(outfile) // compile error: no new variables
 ```
## Pointers

A pointer value is the address of a variable.
* Not every value has an address, but every variable does.
* If a variable is declared var `x `int, the expression `&x` (‘‘address of x’’) yields a pointer to an integer variable, that is, a value of type *int,
* The variable to which p points is written `*p`. The expression `*p` yields the value of that variable, an int, 
* The zero value for a pointer of any type is nil.  The test p != nil is true if p points to a vari- able.
* Pointers are comparable,two pointers are equal if and only if they point to the same variable or both are nil.

```
fmt.Println(f() == f()) // false
```
* A pointer contains the address of a variable, 
* Passing a pointer argument to a function makes it possible for the function to update the variable that was indirectly passed.


### `flag` package

```
package main
     import (
         "flag"
         "fmt"
         "strings"
)
     var n = flag.Bool("n", false, "omit trailing newline")
     var sep = flag.String("s", " ", "separator")

  func main() {
     flag.Parse()
     fmt.Print(strings.Join(flag.Args(), *sep))
     if !*n {
         fmt.Println()
  }

}
```
`flag.Bool` creates a new flag variable of type bool.
*  It takes three arguments
 1. the name of the flag ("n"), 
 2. the variable’s default value (false)
 3. a message that will be printed if the user provides an invalid argument, 


it must call flag.Parse before the flags are used,

##  The `new` Function
> Another way to create a variable is to use the built-in function new.

The expression `new(T)` creates an unnamed variable of type T
* initializes it to the zero value of T
* returns its address, which is a value of type `*T`.

```
p := new(int)   // p, of type *int, points to an unnamed int variable
fmt.Println(*p) // "0"
*p = 2          // sets the unnamed int to 2
fmt.Println(*p) // "2"
```

## Lifetime of Variables

The lifetime of a package-level variable is the entire execution of the program

Local variables have dynamic lifetimes: 
* a new instance is created each time the declaration statement is executed, and the variable lives on until it becomes unreachable, at which point its storage may be recycled.
* Function parameters and results are local variables too; they are created each time their enclosing function is called.


```
var global *int
func f() {
   var x int
   x = 1
   global = &x
}

```
escapes from f
* x must be heap-allocated because it is still reachable from the variable global after f has returned, despite being declared as a local variable; 

##  Assignments





