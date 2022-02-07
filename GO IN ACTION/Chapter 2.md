
* Reviewing a comprehensive Go program
* Declaring types, variables, functions, and methods
*  Launching and synchronizing goroutines
*  Writing generic code using interfaces
*  Handling errors as normal program logic


## 2.1 Architecture

Work flow

![截屏2022-02-07 上午11 52 13](https://user-images.githubusercontent.com/27160394/152721865-e8a9f976-29f4-48e3-8891-4c25818490f0.png)

Project architecture

![截屏2022-02-07 上午11 54 33](https://user-images.githubusercontent.com/27160394/152722084-bc352e47-6920-4207-827b-73e5016fcd50.png)


### `main.go`

```
package main

import {

  "log"
  "os"
}

function init(){ // init is called prior to main.

    log.SetOutPut(os.Studout)
}


function main(){ // main is the entry point for the program.

    search.Run("president") // Perform the search for the specified term
}
```

* `main` function must exist in package main
*  All code files in a folder must use the same package name
*  Every code file in Go belongs to a package
*  package like a namespace, define a unit of code and name identifiers 



## Search package
>  A matcher in our program is a value that contains specific intelligence for processing a feed type. 

### `search.go`

```
package search

import {

  "log"
  "sync"
}

var matchers = make(map[string]Matchers) // Declared as a map of `Matcher` type values with a key of type `string`
```
* This variable is located outside the scope of any function and so is considered a package-level variable
* The name of the variable matchers starts with a lowercase letter
* Exported identifiers start with a capital letter
* Unexported identifiers start with a lowercase letter

`make` and `new`
*  reference type that you’re required to `make` in Go
*  `New` does not initialize the memory, it only zeros it. It returns a pointer to a newly allocated zero value. 
*  `Make` creates slices, maps, and channels only, and it returns them initialized.


 In Go, all variables are initialized to their zero value
 
 ```
 func Run(searchTerm String){
 
    feeds, err := RetrieveFeeds()
    if err != nil{
    
      log.Fatal(err)
    }
    
    results := make(chan *Result)
    
    // Setup a wait group so we can process all the feeds.
    var waitGroup sync.WaitGroup 
    
    // Set the number of goroutines we need to wait for while
    // they process the individual feeds.
    waitGroup.Add(len(feeds))
    
    for _,feed := range feeds{
        matcher, exists := matchers[feed.type]
        if !exists {
            matcher = matchers["default"]
        }
        
        go func(matcher Matcher, feed *Feed){
            Match(matcher, feed, searchTerm, results)
            waitGroup.Done()
        }(matcher, feed)// parameter value passed into anonymous function
        
    }
    
    go func(){
    
        waitGroup.Wait()
        close(results)
    }
    
    
    
 }
 ```
 * Functions can have multiple return values.
 * declaration operator(`:=`) : used to both declare and initialize variables at the same time
 * single underscore `(_)` operator. It is used to ignore the values returned by functions or import for side-effects.
 * Declaring variables 
    * to use the keyword `var` when declaring variables that will be initialized to their zero value
    * to use the short variable declaration operator when you’re providing extra initialization or making a function call. 

Channels are also a reference type in Go
* channels implement a queue of typed values that are used to communicate data between goroutines.
* Channels provide inherent synchronization mechanisms to make communication safe


waitGroup
*  A great way to track when a goroutine is finished performing its work.
*  A WaitGroup is a counting semaphore, and we’ll use it to count off goroutines as they finish their work.
*  `waitGroup.Done()` decrement the WaitGroup count.

goroutines
* Every concurrently executing activity in Go language is known as a Goroutines
* Using `go` keyword to launch a function as a goroutine

### `feed,go`

```
package search

import {

    "encoding/json"
    "os"

}

const dataFile = *data/data.json* // specifies the relative path to the data file on disk

type Feed struct{

     Name string `json:"site"`
     URI string `json:"link"'
     Type string `json:"type"`

}

func RetrieveFeeds() ([]*Feed, error){
  
    file,err := os.Open(dataFile)
    if err != nil(
       return nil, err
    )
    
    defer file.Close()
    
    var Feeds []*Feed
    err = json.NewDecoder(file).Decode(&feeds)
    
    return feeds, err

}
```
* use a lowercase letter for the name of the constant
  * which means this constant is unexported and can only be directly accessed by code within the search package

* `defer` used to schedule a function call to be executed right after a function returns

### `match.go/default.go`

> The `match.go` code file contains the support for creating different types of matchers that can be used by the search Run function.

```
package search

import {

    "log"
}

type Result struct{

    Field string
    Content string
}

type Matcher interface{

    Search(feed *Feed， searchTerm string) ([] *Result, error)
}
```

interface type
* Interfaces declare behavior that’s required to be implemented by struct or named types to satisfy the interface
* If the interface type contains only one method, the name of the interface ends with the er suffix.


```
package search

type defaultMatcher struct{}

func init(){

    var matcher defaultMatcher
    REgister("default", matcher)

}

func (m defaultMatcher) Search(feed *Feed, searchTern string) ([]*Result, error){ //  declared with a value receiver of type defaultMatcher
    return nil,nil
}
```
* It’s best practice to declare methods using pointer receivers, since many of the methods you implement need to manipulate the state of the value being used to make
the method call. 

## RSS matcher
