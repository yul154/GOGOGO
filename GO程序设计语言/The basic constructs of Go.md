```
package main

import  'fmt'

func mai(){
   
   fmt.Println("Hello, world")

}
```

* Package *main* is special. It defines a standalone executable program, not a library
* the function *main* is also special—it’s where execution of the program begins.

A function declaration consists of 
* the keyword func, 
* the name of the function
* a parameter list (empty for main), 
* a result list (also empty here), 
* the body of the function—the state- ments that define what it does—enclosed in braces.



```
package main

import (
   "os"
   "fmt"
)

func main() {
   var s, sep string
   for i :=1; i < len(os.Args);i++{

      s += sep + os.Args[i]
      sep = " "

   }

   fmt.Println(s)
}

for initialization; condition; post { 
    // zero or more statements
}


```
* The`:=` symbol is part of a short variable declaration,
* In each iteration of the loop, `range` produces a pair of values: the index and the value of the element at that index.
* use the blank identifier, whose name is` _ `(that is, an underscore). 
  * The blank identifier may be used whenever syntax requires a variable name but program logic does not


a short variable declaration to declare and initialize
```
s := "" // the most compact, but it may be used only within a function, not for package-level variables.
var s string //  relies on default initialization to the zero value for strings,
var s ="" // rarely used except when declaring multiple variables
var s string = "" // explicit about the variable’s type, which is redundant when it is the same as that of the initial value but necessary in other cases where they are not of the same type.
```
A simpler and more efficient solution would be to use the `Join` function from the strings package


## Finding Duplicate Lines

