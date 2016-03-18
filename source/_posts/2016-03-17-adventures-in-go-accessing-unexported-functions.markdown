---
layout: post
title: "Adventures in Go: Accessing Unexported Functions"
date: 2016-03-17 19:00:00 -0700
comments: true
categories: 
---

I've been learning the Go programming language in my favorite way: by writing a
Go interpreter in Go. The source code so far is
[on GitHub](https://github.com/alangpierce/apgo), but the point of this post is
to tell the story of a particularly interesting challenge I ran into:
programmatically accessing unexported (a.k.a. private) functions in other
packages. In the process of figuring this out, I learned about lots of escape
hatches and limitations of Go. The result of the adventure is a little library I
made and put on GitHub called
[go-forceexport](https://github.com/alangpierce/go-forceexport). If you want a
TLDR, you can just read the source code, but hopefully you'll find the adventure
interesting as well!

My perspective in this post is someone who has plenty of experience with
programming both low-level and high-level languages, but is new to the Go
language and curious about its internals and how it compares to other languages.
In both the exploration process and the process of writing this post, I learned
quite a bit about the right way to think about Go. Hopefully by reading this
you'll be able to learn some of those lessons and also share the same curiosity.

What are unexported functions are and why would I want to call one?
---

In Go, capitalization matters, and determines whether a name can be accessed
from the outside world. For example:

```go
func thisFunctionIsUnexported() {
    fmt.Println("This function starts with a 't' and can only be called from this package.")
}

func ThisFunctionIsExported() {
    fmt.Println("This function starts with a 'T' and can be called from anywhere.")
}
```

Other languages use the terms "private" and "public" for this distinction, but
in Go, they're called **unexported** and **exported**.

But what about when you just want to hack and explore, and you can't easily
modify the code in question? In Python, you might see a name starting with an
underscore, like `_this_function_is_private`, meaning that it's rude to call it
from the outside world, but the runtime doesn't try to stop you. In Java, you
can generally defeat the `private` keyword using reflection and the
[setAccessible](http://docs.oracle.com/javase/6/docs/api/java/lang/reflect/AccessibleObject.html#setAccessible%28boolean%29)
method. Neither of these are good practice in professional code, but the
flexibility is nice if you're trying to figure out what's going wrong in a
library or if you want to build a proof of concept that you'll later make more
professional.

It also can be used as a substitute when other ways of exploring aren't
available. In Python, nothing is compiled, so you can add print statements to
the standard library or hack the code in other ways and it'll just work. Java
has an excellent debugging story, so you can learn a lot about library code by
stepping through it in an IDE. In Go, neither of these approaches are very
pleasant (as far as I've seen), so calling internal functions can sometimes be
the next best thing.

In my specific case, the milestone I was trying to achieve was for my interpeter
to be able to successfully run the `time.Now` function in the standard library.
Let's take a look at the relevant part of
[time.go](https://golang.org/src/time/time.go):

```go
// Provided by package runtime.
func now() (sec int64, nsec int32)

// Now returns the current local time.
func Now() Time {
    sec, nsec := now()
    return Time{sec + unixToInternal, nsec, Local}
}
```

The unexported function `now` is implemented in assembly language and gets the
time as a pair of primitive values. The exported function `Now` wraps that
result in a struct called `Time` with some convenience methods on it (not
shown).

So what does it take to get an interpreter to correctly evaluate `time.Now`?
We'll need at least these pieces:

* Parse the file into a syntax tree. Go's `parser` and `ast` packages are a big
  help here.
* Transform the struct definition for `Time` (defined elsewhere in the file) and
  its methods into some representation known to the interpreter.
* Implement multiple assignment, struct creation, and the other operations used
  in `Now`.
* While evaluating line 6, my interpreter should notice that `now` doesn't have
  a Go implementation, and prefer to just call the real `time.now`. (There are
  other possible approaches, but this one seemed reasonable.)

To prove that that last bullet point was possible, I wanted to write a quick
dummy program that just called `time.now` (even if it needed some hacky
mechanism), but this ended up being a *lot* harder than I was expecting. Most
discussions on the internet basically said "don't do that", but I decided that I
wouldn't give up so easily.

A related goal is that I wanted a way to take a string name of a function and
get that function back. It's worth noting that it's totally unclear if I should
expect this problem to be solvable in the first place. In C, there's no way to
do it, in Java it's doable, and in higher-level scripting languages it's
typically pretty easy. Go seems to be somewhere between C and Java in terms of
the reflection capabilities that I expect, so I might be attempting something
that simply can't be done.

Attempt #1: `reflect`
---

Reflection is the answer in Java, so maybe in Go it's the same way? Sure enough,
Go has a great `reflect` package that works in a lot of cases, and even lets you
read unexported struct fields by name, but it doesn't seem to have any way to
provide access to top-level functions (exported or unexported).

In a language like Python, an expression like `time.now` would take the `time`
object and pull off a field called `now`. So you might hope to do something like
this:

```go
reflect.ValueOf(time).MethodByName("now").Call([]reflect.Value{})
```

But alas, in Go, `time.now` is resolved at compile time, and `time` isn't its
own object that can be accessed like that. So it seems like `reflect` doesn't
provide an easy answer here.

Attempt #2: `runtime`
---

While I was exploring, I noticed `runtime.FuncForPC` as a way to
[programmatically get the name of any function](http://stackoverflow.com/questions/7052693/how-to-get-the-name-of-a-function-in-go):

```go
runtime.FuncForPC(reflect.ValueOf(f).Pointer()).Name()
```

I dug into the implementation, and sure enough, the Go `runtime` package keeps a
table of all functions and their names, provided by the linker. Relevant
snippets from [symtab.go](https://golang.org/src/runtime/symtab.go):

```go
var firstmoduledata moduledata  // linker symbol

func findmoduledatap(pc uintptr) *moduledata {
    for datap := &firstmoduledata; datap != nil; datap = datap.next {
        ...
    }
}

func findfunc(pc uintptr) *_func {
    datap := findmoduledatap(pc)
    ...
}

func FuncForPC(pc uintptr) *Func {
    return (*Func)(unsafe.Pointer(findfunc(pc)))
}
```

The `moduledata` struct isn't particularly friendly, but it looks like if I
could access it, then I should, theoretically, be able to loop through it to
find a pointer to a function with name `"time.now"`. With a function pointer, it
should hopefully be possible to find a way to call it.

Unfortunately, we're at the same place we started. I can't access
`firstmoduledata`, `findmoduledatap`, or `findfunc` for the same reason that I
can't access `time.now`. I looked through the package to find some place where
maybe it leaks a useful pointer, but I couldn't find anything. Drat.

If I was desperate, I might attempt to guess function pointers and call
`FuncForPC` until I find one with that right name. But that seemed like a recipe
for disaster, so I decided to look at other approaches.

Attempt #3: jump down to assembly language
---

An escape hatch that should definitely work is to just write my code in assembly
language. It should be possible to make an assembly function that calls
`time.now` then connect that function to a Go function. I cloned the Go source
code and took a look at the
[Darwin AMD64 implementation](https://golang.org/src/runtime/sys_darwin_amd64.s)
of `time.now` itself to see what it was like:

```c-objdump
// func now() (sec int64, nsec int32)
TEXT time·now(SB),NOSPLIT,$0-12
    CALL    nanotime<>(SB)

    // generated code for
    //    func f(x uint64) (uint64, uint64) { return x/1000000000, x%100000000 }
    // adapted to reduce duplication
    MOVQ    AX, CX
    MOVQ    $1360296554856532783, AX
    MULQ    CX
    ADDQ    CX, DX
    RCRQ    $1, DX
    SHRQ    $29, DX
    MOVQ    DX, sec+0(FP)
    IMULQ   $1000000000, DX
    SUBQ    DX, CX
    MOVL    CX, nsec+8(FP)
    RET
```

Ugh. Maybe I shouldn't be scared off by assembly, but learning the calling
conventions and writing a separate wrapper for every function I wanted to call
for every architecture didn't seem pleasant. I decided to defer that idea and
look at other options. Having a solution in pure Go would certainly be ideal.

Attempt #4: CGo
---

Another escape hatch that seemed promising is
[CGo](http://blog.golang.org/c-go-cgo), which is Go's mechanism for directly
calling C functions from Go code. Here's a first attempt:

```go
/*
int time·now();
int time_now_wrapper() {
    return time·now();
}
*/
import "C"

func callTimeNow() {
    C.time_now_wrapper()
}
```

And here's the error that it gives:

```plain
Undefined symbols for architecture x86_64:
  "_time·now", referenced from:
      _time_now_wrapper in foo.cgo2.o
      __cgo_b552a62968a6_Cfunc_time_now_wrapper in foo.cgo2.o
ld: symbol(s) not found for architecture x86_64
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

Hmm, it seems to be putting an underscore before every function name, which
isn't really what I want. Maybe there's a way around that, but dealing with
multiple return values in `time·now` seemed like it may be another barrier, and
from my reading, CGo calls have a lot of overhead because it's doing a lot of
translation work so that you can integrate with existing C code. In Java speak,
it seems like it's JNA, not JNI. So while CGo seems useful, it looks like it's
not really the solution to my problem.

Attempt #5: `go:linkname`
---

As I was digging through the standard library source code I saw something
interesting in the `runtime` package in
[stubs.go](https://golang.org/src/runtime/stubs.go):

```go
//go:linkname time_now time.now
func time_now() (sec int64, nsec int32)
```

Interesting! I had seen semantically-meaningful comments like this before (like
with the CGo example, and also in other places), but I hadn't seen this one. It
looks like it's saying "linker, please use `time.now` as the implementation of
`runtime.time_now`". Sure enough, the
[documentation](https://golang.org/cmd/compile/#hdr-Compiler_Directives)
suggests that this works, as long as your file imports `unsafe`. So I tried it
out:

```go
package main

import (
    "fmt"
    _ "unsafe"
)

//go:linkname time_now time.now
func time_now() (sec int64, nsec int32)

func main() {
    sec, nsec := time_now()
    fmt.Println(sec, nsec)
}
```

Let's see what happens:

```plain
# command-line-arguments
./sandbox.go:9: missing function body for "time_now"
```

Drat. Isn't the missing function body the *whole point* of the bodyless syntax
to allow for externally-implemented functions? The
[spec](https://golang.org/ref/spec#Function_declarations) certainly seems to
think that it's valid.

Just to see what would happen, I replaced the empty function body with a dummy
implementation:

```go
//go:linkname time_now time.now
func time_now() (sec int64, nsec int32) {
    return 0, 0
}
```

Then I tried again and got this error:

```plain
# command-line-arguments
2016/03/16 22:50:31 duplicate symbol time.now (types 1 and 1) in main and /usr/local/Cellar/go/1.6/libexec/pkg/darwin_amd64/runtime.a(sys_darwin_amd64)
```

When I don't implement the function, it complains that there's no
implementation, but when I do implement it, it complains that the function is
implemented twice! How frustrating!

Getting `go:linkname` to work
---

For a while, it seemed like the `go:linkname` approach wasn't going to work out,
but then I noticed something suspicious: the error formatting is different. It
looks like the "missing function body" error is from the compiler, but the
"duplicate symbol" error is from the linker. Why would the compiler care about a
function body being missing, if it's the linker's job to make sure every symbol
gets an implementation?

I decided to dig into the code for the compiler to see why it might be
generating this error. Here's what I found in
[pgen.go](https://golang.org/src/cmd/compile/internal/gc/pgen.go):

```go
func compile(fn *Node) {
    ...
    if len(fn.Nbody.Slice()) == 0 {
        if pure_go != 0 || strings.HasPrefix(fn.Func.Nname.Sym.Name, "init.") {
            Yyerror("missing function body for %q", fn.Func.Nname.Sym.Name)
            return
        }
```

Something is causing that inner `if` statement to evaluate to `true`, and my
function doesn't have to do with `init`, so it looks like `pure_go` is nonzero
when it should be zero. Searching for `pure_go` shows this compiler flag:

```go
obj.Flagcount("complete", "compiling complete package (no C or assembly)", &pure_go)
```

Makes sense: if your code doesn't have any way of defining external functions,
then it's friendlier to give an error at compile time with the location of the
problem. But it looks like `go:linkname` was overlooked somewhere in the
process. It certainly is a bit of an edge case.

After some searching, I found the culprit in
[build.go](https://golang.org/src/cmd/go/build.go):

```go
    extFiles := len(p.CgoFiles) + len(p.CFiles) + len(p.CXXFiles) + len(p.MFiles) + len(p.FFiles) + len(p.SFiles) + len(p.SysoFiles) + len(p.SwigFiles) + len(p.SwigCXXFiles)
    ...
    if extFiles == 0 {
        gcargs = append(gcargs, "-complete")
    }
```

So it's just counting the number of non-Go files of each type. Since I'm only
compiling with Go files, it assumes that every function needs a body. But on the
plus side, the code suggests a workaround: just add a file of any of those
types. I already know how to use CGo, so let's try that:

```go
package main

import (
    "C"
    "fmt"
    _ "unsafe"
)

//go:linkname time_now time.now
func time_now() (sec int64, nsec int32)

func main() {
    sec, nsec := time_now()
    fmt.Println(sec, nsec)
}
```

And here's what happens when you try to build that:

```plain
main.main: call to external function main.time_now
main.main: main.time_now: not defined
main.main: undefined: main.time_now
```

A different error! Now the linker is complaining that the function doesn't
exist. After some experimentation, I discovered that CGo seems to cause
`go:linkname` to be disabled for that file. If I remove the import of `"C"` and
move it to another file, then compile the two together, then I get this output:

```plain
1458197809 407398202
```

It worked! If your only goal is to get access to `time.now`, then this is good
enough, but I'm hoping that I can go a bit further.

Looking up functions by name
---

Now that I know that `go:linkname` works, I can use it to access the
`firstmoduledata` structure mentioned in attempt #2, which is a table containing
information on all compiled functions in the binary. My hope is that I can use
it to write a function that takes a function name as a string, like
`"time.now"`, and provides that function.

One problem is that `runtime.firstmoduledata` has type `runtime.moduledata`,
which is an unexported type, so I can't use it in my code. But as a total hack,
I *can* just copy the struct to my code (or, at least, enough of it to keep the
alignment correct) and pretend that my struct is the real thing. From there, I
can pretty much copy the code from the `runtime` package to do a full scan
through the list of functions until I find the right one:

```go
func FindFuncWithName(name string) (uintptr, error) {
    for moduleData := &Firstmoduledata; moduleData != nil; moduleData = moduleData.next {
        for _, ftab := range moduleData.ftab {
            f := (*runtime.Func)(unsafe.Pointer(&moduleData.pclntable[ftab.funcoff]))
            if f.Name() == name {
                return f.Entry(), nil
            }
        }
    }
    return 0, fmt.Errorf("Invalid function name: %s", name)
}
```

This seems to work! This code:

```go
ptr, _ := FindFuncWithName("math.Sqrt")
fmt.Printf("Found pointer 0x%x\nNormal function: %s", ptr, math.Sqrt)
```

prints this:

```plain
Found pointer 0x104250
Normal function: %!s(func(float64) float64=0x104250)
```

So the underlying code pointer is correct! Now we just need to figure out how to
use it...

Calling a function by pointer
---

You would think that having a function pointer would be the end of the story.
In C you could just cast the pointer value to the right function type, then call
it. But Go isn't quite so generous. For one, Go normally doesn't just let you
cast between types like that, but `unsafe.Pointer` can be used to circumvent
some safety checks. You might try just casting it to a function of the proper
type:

```go
ptr, _ := FindFuncWithName("time.now")
timeNow := (func() (int64, int32))(unsafe.Pointer(ptr))
sec, msec := timeNow()
```

But that type of cast doesn't compile; pointers can't be cast to functions, not
even using `unsafe.Pointer`. What if we literally cast it to a pointer to a
`func` type?

```go
ptr, _ := FindFuncWithName("time.now")
timeNow := (*func() (int64, int32))(unsafe.Pointer(ptr))
sec, msec := (*timeNow)()
```

This compiles, but crashes at runtime:

```plain
unexpected fault address 0xb01dfacedebac1e
fatal error: fault
[signal 0xb code=0x1 addr=0xb01dfacedebac1e pc=0x7e450]
```

(Look at that fault address. Apparently someone
[had a sense of humor](https://golang.org/src/runtime/signal_amd64x.go).)

This isn't a surprising outcome; functions in Go are first-class values, so
their implementation is naturally more interesting than in C. When you pass
around a `func`, you're not just passing around a code pointer, you're passing
around a *function value* of some sort, and we'll need to come up with a
function value somehow if we're to have any hope of calling our function. That
function value needs to have our pointer as its underlying code pointer.

I didn't see any obvious ways to create a function value from scratch, so I
figured I'd take a different approach: take an existing function value and hack
the code pointer to be the one I want. After spending some time reading
[how interfaces work](http://research.swtch.com/interfaces) in Go and reading
the implementation of the [reflect](https://golang.org/src/reflect/value.go)
library, an approach that seemed promising was to treat the function as an
`interface{}` (that's Go's equivalent of `Object` or `void*` or `any`: a type
that includes every other type), which internally stores it as a (type, pointer)
pair. Then I could pull the pointer off and work with it reliably. The `reflect`
source code suggests that the *code pointer* (the pointer to the actual machine
code) is the first value in a function object.

So, as a first attempt, I created a dummy function called `timeNow` then defined
some structs to make it easy to swap out its code pointer with the real
`time.now` code pointer:

```go
func timeNow() (int64, int32) {
    return 0, 0
}

type Interface struct {
    typ     unsafe.Pointer
    funcPtr *Func
}

type Func struct {
    codePtr uintptr
}

func main() {
    timeNowCodePtr, _ := FindFuncWithName("time.now")
    var timeNowInterface interface{} = timeNow
    timeNowInterfacePtr := (*Interface)(unsafe.Pointer(&timeNowInterface))
    timeNowInterfacePtr.funcPtr.codePtr = timeNowCodePtr
    sec, msec := timeNow()
    fmt.Println(sec, msec)
}
```

And, as you might guess, it crashed:

```go
unexpected fault address 0x129e80
fatal error: fault
[signal 0xa code=0x2 addr=0x129e80 pc=0x20be]
```

After some experimenting, I discovered that the crash was happening *even
without calling the function*. The crash was from the line
`timeNowInterfacePtr.funcPtr.codePtr = timeNowCodePtr`. After double-checking
that the pointers were what I expect, I realized the problem: the function
object I was modifying was probably in the code segment, in read-only memory.
Just like how the machine code isn't going to change, Go expects that the
`timeNow` function value isn't going to change at runtime. What I really needed
to do was *allocate* a function object on the heap so that I could safely change
its underlying code pointer.

So how do you dynamically allocate a function in Go? That's what lambdas are
for, right? Let's try using one! Instead of the top-level `timeNow`, we can
write our `main` function like this (the only difference is the new definition
of `timeNow`):

```go
func main() {
    timeNowCodePtr, _ := FindFuncWithName("time.now")
    timeNow := func() (int64, int32) { return 0, 0 }
    var timeNowInterface interface{} = timeNow
    timeNowInterfacePtr := (*Interface)(unsafe.Pointer(&timeNowInterface))
    timeNowInterfacePtr.funcPtr.codePtr = timeNowCodePtr
    sec, msec := timeNow()
    fmt.Println(sec, msec)
}
```

And, again, it crashes. I've seen how lambdas work in other languages, so I
suspected why: when a lambda takes in no outside variables, there's no need to
do an allocation each time, so a common optimization is to just have a single
shared instance for simple lambdas like the one I wrote, so probably I'm again
trying to write to the code segment. To work around this, we can trick the
compiler into allocating a new function object each time by making the function
a real closure and pulling in a variable from the outer scope (even a trivial
one):

```go
func main() {
    timeNowCodePtr, _ := FindFuncWithName("time.now")
    var x int64 = 0
    timeNow := func() (int64, int32) { return x, 0 }
    var timeNowInterface interface{} = timeNow
    timeNowInterfacePtr := (*Interface)(unsafe.Pointer(&timeNowInterface))
    timeNowInterfacePtr.funcPtr.codePtr = timeNowCodePtr
    sec, msec := timeNow()
    fmt.Println(sec, msec)
}
```

And it works!

```plain
1458245880 151691912
```

Turning it into a library
---

This code is almost useful, but wouldn't really work as a library yet because it
would require the function's type to be hard-coded into the library. We could
have the library caller pass in a function that will be modified, but that has
gotchas like the read-only memory problem I ran into above.

Instead, I looked around at possible API approaches, and I got some nice
inspiration from the example code for
[reflect.MakeFunc](https://golang.org/pkg/reflect/#MakeFunc).

We'll try writing a `GetFunc` function that can be used like this:

```go
var timeNow func() (int64, int32)
GetFunc(&timeNow, "time.now")
sec, msec := timeNow()
```

But how can `GetFunc` allocate a function value? Above, we used a lambda
expression, but that doesn't work if the type isn't known until runtime.

Reflection to the rescue! We can call `reflect.MakeFunc` to create a function
value with a particular type. In this case, we don't really care what the
implementation is because we're going to be modifying its code pointer anyway.
We end up with a `reflect.Value` object with a memory layout like this:

![Function pointer layout](/images/function_pointer_diagram.png)

The `ptr` field in the `reflect.Value` definition is unexported, but we can *use
reflection on the `reflect.Value`* to get it, then treat it as a pointer to a
function object, then modify that function object's code pointer to be what we
want. The full code looks like this:

```go
type Func struct {
    codePtr uintptr
}

func CreateFuncForCodePtr(outFuncPtr interface{}, codePtr uintptr) {
    // outFuncPtr is a pointer to a function, and outFuncVal acts as *outFuncPtr.
    outFuncVal := reflect.ValueOf(outFuncPtr).Elem()
    newFuncVal := reflect.MakeFunc(outFuncVal.Type(), nil)
    funcValuePtr := reflect.ValueOf(newFuncVal).FieldByName("ptr").Pointer()
    funcPtr := (*Func)(unsafe.Pointer(funcValuePtr))
    funcPtr.codePtr = codePtr
    outFuncVal.Set(newFuncVal)
}
```

And that's it! That function modifies its argument to be the function at
`codePtr`. Implementing the main `GetFunc` API is just a matter of tying
together `FindFuncWithName` and `CreateFuncForCodePtr`; details are in the
[source code](https://github.com/alangpierce/go-forceexport/blob/master/forceexport.go).

Future steps and lessons learned
---

This API still isn't ideal; the library user still needs to know the type in
advance, and if they get it wrong, there will be horrible consequences at
runtime. At the end of the day, the library isn't significantly more useful than
`go:linkname`, but it has some advantages, and is a good starting point for more
interesting tricks. It's potentially possible, but probably harder, to make a
function that takes a `string` and returns a `reflect.Value` of the function,
which would be ideal. But that's out of scope for now. Also, the
[README](https://github.com/alangpierce/go-forceexport#use-cases-and-pitfalls)
has a number of other warnings and things to consider. For example, this
approach will sometimes completely break due to function inlining.

Go is certainly in an interesting place in the space of languages. It's dynamic
enough that it's not crazy to look up a function by name, but it's much more
performance-focused than, say, Java. The reflection capabilities are good for a
systems language, but sometimes the better escape hatch is to just use pointers.

I'd be happy to hear any feedback or corrections in the comments below. Like I
mentioned, I'm still learning all of this stuff, so I probably overlooked some
things and got some terminology wrong.