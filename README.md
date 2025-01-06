# go-notes

Free form notes on go learning.

## Developer Environment

### Private Repo Setup

```
git config --global \
  url."https://${user}:${personal_access_token}@privategitlab.com".insteadOf \
  "https://privategitlab.com"
```  
More can be found here:
https://medium.com/cloud-native-the-gathering/go-modules-with-private-git-repositories-dfe795068db4

### Local Development

```
go mod edit -replace [module_name]=[local_path]
```


---

## Arrays vs. Slices

- Arrays are fixed sice where the length of the array (number of items in the array) and the capacity (max size) is always the same
- A slice on the other hand is dynamic in that the capacity grows.
- A slice under the hood implements an array which means each time the capacity is reached, it is resized (doubles the original capacity) to accomodate the new items. This operation is costly since it essentially creates a new array and copies the data over. Additionally, gargbage collection is necessary to clean up the old array. For this reason it is always a good idea to pre-allocate the slice with `make` if the size is known.

### References
- https://stackoverflow.com/questions/41668053/cap-vs-len-of-slice-in-golang
- https://oilbeater.com/en/2024/03/04/golang-slice-performance/

## Reflect

### References

https://stackoverflow.com/questions/8103617/call-a-struct-and-its-method-by-name-in-go


## Interface

> An interface can be thought of as being represented internally by a tuple (type, value). `type` is the underlying concrete type of the interface and `value` holds the value of the concrete type.

### Empty Interface

> An interface that has zero methods is called an empty interface. It is represented as interface{}. Since the empty interface has zero methods, all types implement the empty interface.

Example:
https://play.golang.org/p/Fm5KescoJb

### Type Assertion

```
// Extract value of interface 'i' of type 'T'
v, ok := i.(T)
```

`ok` will be `true` if `T` is the type of `i`. If it is not, `ok` will be false and it will not panic.

See more details at https://tour.golang.org/methods/15.

### Type Switch

Get the interface's underlying concrete type within a switc

See details here https://tour.golang.org/methods/16.


### Zero Value 

> The zero value of a interface is nil. A nil interface has both its underlying value and as well as concrete type as nil.


## Pointers

### Pointer Receiver

> For the statement v.Scale(5), even though v is a value and not a pointer, the method with the pointer receiver is called automatically. That is, as a convenience, Go interprets the statement v.Scale(5) as (&v).Scale(5) since the Scale method has a pointer receiver.

See full details at https://tour.golang.org/methods/6

## To Read

### Go Error Handling
https://medium.com/@hussachai/error-handling-in-go-a-quick-opinionated-guide-9199dd7c7f76

### Gotchas

http://devs.cloudimmunity.com/gotchas-and-common-mistakes-in-go-golang

### Tips

- https://github.com/golang/go/wiki/CodeReviewComments
- https://github.com/smallnest/go-best-practices
- https://github.com/Pungyeon/clean-go-article
- http://peter.bourgon.org/go-in-production/


## Channels

- Share data between goroutines
- Safe for concurrency
- FIFO
- It can block and unblock goroutines

CSP (Communicating Sequential Processes) paradigm:

- Each process creates a sequential execution i.e: goroutine
- Process communicate through _channels_, not by sharing memory/state i.e: channels
- Scaling by spawning new instances of the process i.e: goroutines





### Basics

```
// Initializing a channel
var ch chan int = make(chan int)


ch <- val     // Sending on a channel
val = <-ch    // Receiving on a channel and assigning it to val
<-ch          // Receiving on a channel and discarding the result
```

### Blocking

- Sender will block until the reciever recieves from the channel. 
- Reciever will block until something is sent on the channel.

> Think of this loosely as a postman delivering a package (Sender) and you (Reciever) can't leave the house until you collect the package. The postman is blocked if you don't take the package. You are blocked until the postman delivers the package.


- This behaviour provides a way to synchronize two goroutines. Another way of guranteeing that something that was sent has been received since the sender can't keep sending until the receiver takes the thing off the channel.


```
package main

import (
	"fmt"
)

func main() {
	ch := make(chan string)

	go func() {
		greet := <-ch // Blocking receive; assigns to greet
		fmt.Println(greet)	// "I have a package"
		ch <- "thanks"

	}()

	ch <- "I have a package"
	fmt.Println(<-ch) // "thanks"
}
```



The main goroutine and the anonymous goroutine can run concurrently. However, because the sender is blocked until the receiver takes the thing from the channel it will always deterministically print:

```
I have a package
Thanks
```

> This is an unbuffered channel where the capacity of the buffer is 0.




### Buffered channel


- Sender will block only if the buffer is full.
- Receiver will block only if the buffer is empty.


### Close 

- Sending on a closed channel will planic. For this reason it is almost always the sender's job to close a channel.
- Receiving from a closed channel after it is closed, will still receive any values in the channel. The channel is drained.
- Any attempt to receive after a channel has been closed, it will get the default zero value of the channel element type. 
- You needn't close every channel when you've finished with it. It's only necessary to close a channel when it is important to tell the receiving goroutines that all data have been sent. A channel that the garbage collector determinies to be unreachable will have its resources reclaimed whether or not it is closed.

See a full list here: https://github.com/imsamuel/channel-cheatsheet


#### Check if a channel is closed

- Receivers can test if the channel is empty or closed using the second `bool` parameter:

```
ch := make(chan string)

ch <- "foo"

close(ch)                          

msg, ok := <-ch
fmt.Printf("%q, %v\n", msg, ok)    // "foo", true

msg, ok := <-ch
fmt.Printf("%q, %v\n", msg, ok)    // "", false
```


### Ranging over a channel

- The `range` keyword can be used to loop over a channel that is either open or contain buffered values.
- `range` will block until a value is available or the channel is closed.
- If the channel is not closed, `range` will block indefinitely until it is closed.


```
ch := make(chan string, 3)

ch <- "foo"                 // Send three (buffered) values to the channel
ch <- "bar"
ch <- "bin"

close(ch)                   // Close the channel

for s := range ch {         // Range will continue until closed
    fmt.Println(s)
}
```


### Multi channel operations


- `select` like a switch statement can be used with case statements to send/recive on multiple channels.
- Unlike switch blocks, case statements in a `select` block aren’t tested sequentially, and execution won’t automatically fall through if none of the criterias are met.
- `select` without a `default` case will block until one of the cases are ready.
- If none of the channels in a `select` statement are ready, the statement blocks.

#### What happens when multiple channels have a value to read?

Chooses a channel by random.

#### What if there are never any channels that become ready?

Blocks unless a `default` case is used.



### Patterns

1. Trivial way to unblock goroutines on when a condition has occured

```
package main

import (
	"fmt"
	"sync"
)

func main() {
	signal := make(chan struct{})
	var wg sync.WaitGroup
	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func(i int) {
			defer wg.Done()
			// Do some pre work
			<-signal
			// Do remainder of work
			fmt.Printf("%v has begun\n", i)
		}(i)
	}
	// Do work
	fmt.Println("Unblock workers to continue")
	close(signal)
	wg.Wait()
}

```
See https://goplay.space/#s8-1OaniGIL


2. Pattern to keep a narrow scope of channel ownerhip

See https://goplay.space/#GFtP1GyU0tX


3. Fan out pattern 

This is a somewhat contrived example but it provides the framework:

https://goplay.space/#blZxv28NCDM

### Goroutine leaks

>  If you start a Goroutine that you expect to eventually terminate but it never does then it has leaked. It lives for the lifetime of the application and any memory allocated for the Goroutine can’t be released.

https://www.ardanlabs.com/blog/2018/11/goroutine-leaks-the-forgotten-sender.html
 

> Every time you use the go keyword in your program to launch a goroutine, you must know how, and when, that goroutine will exit. If you don’t know the answer, that’s a potential memory leak.

https://dave.cheney.net/2016/12/22/never-start-a-goroutine-without-knowing-how-it-will-stop


### HTTP

- Add tracing to your requests:

https://stackoverflow.com/questions/39527847/is-there-middleware-for-go-http-client

## Generics

### Basics

Read [this](https://bitfieldconsulting.com/golang/type-parameters) and [this](https://bitfieldconsulting.com/golang/generics)


## JSON

### JSON tags

Examples:

```
type Article struct {
	Id   string  `json:"id"`
	Name *string `json:"name,omitempty"`
	Desc string  `json:"desc,omitempty"`
}
````

Unmarshaling a JSON payload with no `name` will set the `Name` to `nil`


```
type Article struct {
	Id   string  `json:"id"`
	Name *string `json:"-"`
	Desc string  `json:"desc,omitempty"`
}
````

- Ignore this fied when unmarshaling and marshaling
- Unmarshaling a JSON payload will always set `Name` to `nil` even if it has a non empty value.
- Marshaling to JSON will not have `name` in its payload



## Errors

### Before Go 1.13

Note:
1. 2 kinds of errors
2. In sentinel errors you are comparing the **value**
3. In custom errors you are comparing the **type**
4. If you wrap errors with `fmt.Errorf` with the `%v` verb, it discards everything in an erorr i.: `err` and only 
   uses the string representation by calling `err.Error()`
5. To access a custom error field, you would have to type convert and access field value.
 


There are atleast two kinds of errors; **sentinel** and **custom** errors.

```
# Sentinel errors

var ErrNotFound = errors.New("not found")

# You are comparing the error "value" here 
if err == ErrNotFound {
    // something wasn't found
}
```

```
type NotFoundError struct {
    Name string
}

func (e *NotFoundError) Error() string { return e.Name + ": not found" }

// You are comparing the error "type" here
if e, ok := err.(*NotFoundError); ok {
    // e.Name wasn't found
}
```

If you wanted to share context of an error from another api call before returning the following is done:

```
if err != nil {
    return fmt.Errorf("decompress %v: %v", name, err)
}
```
Using `fmt.Errorf` we create a new error and use the error text from the `err` value. So everything esle is discarded like the type. 

The `%v` format verb used in the `fmt.Errorf` function call will be replaced with the string representation of the `err` variable. This is equivalent to calling `err.Error()` and passing the resulting string as an argument to `fmt.Errorf`.


If you had the custom error type which consists of additional information:

```
type QueryError struct {
    Query string
    Err   error
}
```

Programs would need to perform a type conversion to be able to inspect the specific fields:

```
if e, ok := err.(*QueryError); ok && e.Err == ErrPermission {
    // query failed because of a permission problem
}
```

### Errors in Go 1.13

#### Summary

1. With a custom error type that can store an error, you can use the `Unwrap` method to get the wrapped error
2. Compare an error **value** with `errors.Is` and compare a custom error **type** with `errors.As`
3. In the simplest case, the `errors.Is` function behaves like a comparison to a sentinel error, and the `errors.As` function behaves like a type assertion
4.  When operating on wrapped errors, however, these functions consider all the errors in a chain
5. The error returned from `fmt.Errorf` which wraps an error will also provide an `Unwrap` method for the return value which is an error.


#### Unwrap

It is generally a good practice to implement the `Unwrap` method when creating custom errors that wrap other errors, as it provides a way for callers to inspect the underlying error for debugging or error handling purposes.


```
type myError struct {
    message string
    cause error
}

func (e *myError) Error() string {
    return fmt.Sprintf("%s: %v", e.message, e.cause)
}

func (e *myError) Unwrap() error {
    return e.cause
}

func doSomething() error {
    _, err := someOperation()
    if err != nil {
        return &myError{"failed to do something", err}
    }
    return nil
}

func main() {
    err := doSomething()
    if errors.Is(err, io.EOF) {
        // handle EOF error
    } else if errors.Is(err, someOtherError) {
        // handle some other error
    } else if err != nil {
        fmt.Printf("error: %v\n", err)
        if unwrappedErr := errors.Unwrap(err); unwrappedErr != nil {
            fmt.Printf("cause: %v\n", unwrappedErr)
        }
    }
}
```

In this example, the myError type wraps another error, and the `Unwrap()` method is implemented to return the wrapped error. In the `main()` function, the `errors.Unwrap(err)` or `err.Unwrap()` function is used to extract the underlying error if present.

However, if the custom error type does not wrap another error or if it is not necessary to expose the wrapped error to callers, then implementing Unwrap() may not be necessary.



#### errors.Is

The `errors.Is` function compares an error to a **value**.

```
// Similar to:
//   if err == ErrNotFound { … }
if errors.Is(err, ErrNotFound) {
    // something wasn't found
}
```



#### errors.As

The `errors.As` function compares an error to a **type**.

```
// Similar to:
//   if e, ok := err.(*QueryError); ok { … }
var e *QueryError
// Note: *QueryError is the type of the error.
if errors.As(err, &e) {
    // err is a *QueryError, and e is set to the error's value
}
```

### Error chain

```
# Before to inspect a field that has error you need to peform a type conversion and then check the value.

if e, ok := err.(*QueryError); ok && e.Err == ErrPermission {
    // query failed because of a permission problem
}
```

Now with `errors.Is` you can simply do:

```
if errors.Is(err, ErrPermission) {
    // err, or some error that it wraps, is a permission problem
}
```

The `errors` package also includes a new `Unwrap` function which returns the result of calling an error’s `Unwrap` method, or `nil` when the error has no Unwrap method. 

**It is usually better to use errors.Is or errors.As, since these functions will examine the entire chain in a single call.**

### fmt.Errorf

In Go 1.13, the `fmt.Errorf` function supports a new %w verb. When this verb is present, the error returned by `fmt.Errorf` will have an `Unwrap` method returning the argument of %w, which must be an error. In all other ways, %w is identical to %v.

```
if err != nil {
    // Return an error which unwraps to err.
    return fmt.Errorf("decompress %v: %w", name, err)
}
```

Wrapping an error with %w makes it available to `errors.Is` and `errors.As`:

```
err := fmt.Errorf("access denied: %w", ErrPermission)
...
if errors.Is(err, ErrPermission) ...
```

### Whether to wrap

- When implementing a custom error type or using `fmt.Errorf` you need to decide if you want to implement an `Unwrap()` or use %w verb to expose the error to the caller. 
- If you choose to wrap an error it becomes part of your API. You must support it despite the underlying implementation changes since callers may rely on it. 
- The choice to wrap is about whether to give programs additional information so they can make more informed decisions, or to withhold that information to preserve an abstraction layer.

### Reference

From https://go.dev/blog/go1.13-errors

## Sorting

### Sorting a custom struct

```
package main

import (
	"fmt"
	"sort"
)

type MyStruct struct {
	Name  string
	Count int
}

type ByCount []MyStruct

func (s ByCount) Len() int           { return len(s) }
func (s ByCount) Less(i, j int) bool { return s[i].Count < s[j].Count }
func (s ByCount) Swap(i, j int)      { s[i], s[j] = s[j], s[i] }

func main() {
	data := []MyStruct{
		{Name: "A", Count: 10},
		{Name: "B", Count: 5},
		{Name: "C", Count: 7},
	}

	sort.Sort(ByCount(data))

	for _, item := range data {
		fmt.Println(item.Name, item.Count)
	}
}
```