---
layout: post
title: Reducing Memory Allocations in Golang
tags: [go, golang, performance]
---

Go's place between C and Python in terms of abstraction and garbage collection memory management model has made it attractive to programmers looking for a fast but reasonably high level language.  However, there is no free lunch.  Go's abstractions, especially with regards to allocation, come with a cost.  This article will show ways to measure and reduce this cost.

## Measuring

On posts about performance on StackOverflow and similar forums, you will often hear some variation of the Knuth quote on optimization: "premature optimization is the root of all evil."  This simplification leaves out some important context present in the [original quote](https://web.archive.org/web/20210425190711if_/https://pic.plover.com/knuth-GOTO.pdf):

> There is no doubt that the grail of efficiency leads to abuse. Programmers waste enormous amounts of time thinking about, or worrying about, the speed of noncritical parts of their programs, and these attempts at efficiency actually have a strong negative impact when debugging and maintenance are considered. We should forget about small efficiencies, say about 97% of the time: premature optimization is the root of all evil. Yet we should not pass up our opportunities in that critical 3%. A good programmer will not be lulled into complacency by such reasoning, he will be wise to look carefully at the critical code; but only after that code has been identified.

Knuth emphasizes that instead of looking for small performance gains that harm readability so much that they aren't even worth it, we should look for the large (97%) gains that can be found by improving the critical code.  And how do we identify that critical code?  With profiling tools.  Fortunately, Go has better profiling tools than most languages.  For instance, let's say we have the following (somewhat contrived) code:

    package main

    import (
        "bytes"
        "crypto/sha256"
        "fmt"
        "math/rand"
        "strconv"
        "strings"
    )

    func foo(n int) string {
        var buf bytes.Buffer
        for i := 0; i < 100000; i++ {
            buf.WriteString(strconv.Itoa(n))
        }
        sum := sha256.Sum256(buf.Bytes())

        var b []byte
        for i := 0; i < int(sum[0]); i++ {
            x := sum[(i*7+1)%len(sum)] ^ sum[(i*5+3)%len(sum)]
            c := strings.Repeat("abcdefghijklmnopqrstuvwxyz", 10)[x]
            b = append(b, c)
        }
        return string(b)
    }

    func main() {
        // ensure function output is accurate
        if foo(12345) == "aajmtxaattdzsxnukawxwhmfotnm" {
            fmt.Println("Test PASS")
        } else {
            fmt.Println("Test FAIL")
        }

        for i := 0; i < 100; i++ {
            foo(rand.Int())
        }
    }



To add CPU profiling, we just need to change it to

    package main

    import (
        "bytes"
        "crypto/sha256"
        "fmt"
        "math/rand"
        "os"
        "runtime/pprof"
        "strconv"
        "strings"
    )

    func foo(n int) string {
        var buf bytes.Buffer
        for i := 0; i < 100000; i++ {
            buf.WriteString(strconv.Itoa(n))
        }
        sum := sha256.Sum256(buf.Bytes())

        var b []byte
        for i := 0; i < int(sum[0]); i++ {
            x := sum[(i*7+1)%len(sum)] ^ sum[(i*5+3)%len(sum)]
            c := strings.Repeat("abcdefghijklmnopqrstuvwxyz", 10)[x]
            b = append(b, c)
        }
        return string(b)
    }

    func main() {
        cpufile, err := os.Create("cpu.pprof")
        if err != nil {
            panic(err)
        }
        err = pprof.StartCPUProfile(cpufile)
        if err != nil {
            panic(err)
        }
        defer cpufile.Close()
        defer pprof.StopCPUProfile()

        // ensure function output is accurate
        if foo(12345) == "aajmtxaattdzsxnukawxwhmfotnm" {
            fmt.Println("Test PASS")
        } else {
            fmt.Println("Test FAIL")
        }

        for i := 0; i < 100; i++ {
            foo(rand.Int())
        }
    }

After compiling and running this program, the profile will be written to `./cpu.pprof`.  We can read this file using `go tool pprof`:

    $ go tool pprof cpu.pprof 

We are now in the pprof interactive tool.  We can see what our program is spending most of its time on with `top10` (`top1`, `top2`, `top99`, ..., `topn` also work). `top10` gives us:

    Showing nodes accounting for 2010ms, 86.27% of 2330ms total
    Dropped 48 nodes (cum <= 11.65ms)
    Showing top 10 nodes out of 54
          flat  flat%   sum%        cum   cum%
         930ms 39.91% 39.91%      930ms 39.91%  crypto/sha256.block
         360ms 15.45% 55.36%      720ms 30.90%  strconv.formatBits
         180ms  7.73% 63.09%      390ms 16.74%  runtime.mallocgc
         170ms  7.30% 70.39%      170ms  7.30%  runtime.memmove
         100ms  4.29% 74.68%      100ms  4.29%  runtime.memclrNoHeapPointers
          80ms  3.43% 78.11%      340ms 14.59%  bytes.(*Buffer).WriteString
          60ms  2.58% 80.69%       60ms  2.58%  runtime.nextFreeFast (inline)
          50ms  2.15% 82.83%      360ms 15.45%  runtime.slicebytetostring
          40ms  1.72% 84.55%     2070ms 88.84%  main.foo
          40ms  1.72% 86.27%      760ms 32.62%  strconv.FormatInt


\[Note: "Allocations" is used in this article to refer to [heap allocations](https://www.sciencedirect.com/topics/computer-science/heap-allocation).  Stack allocations are allocations too but they are not nearly as costly or important in the context of performance.\]

It looks like we are spending a lot of time using sha256, strconv, allocating, and garbage collecting.  Now we know what we need to improve.  Since we are not doing any sort of complex computations (except maybe sha256), it would seem likely that most of our performance problems are caused by heap allocations.  We can measure allocations precisely by replacing

    for i := 0; i < 100; i++ {
        foo(rand.Int())
    }

with

    fmt.Println("Allocs:", int(testing.AllocsPerRun(100, func() {
        foo(rand.Int())
    })))

and importing the `testing` package.  Now, when we run the program, we should see output like:

    Test PASS
    Allocs: 100158

A few months ago I was given a challenge to get this number to 0 by my [employer](https://github.com/lukechampine).  This seemed very daunting at first because 100k is a **lot** of allocations.  In this article, I'll reveal the thought process and thinking I had while I was reducing the number of allocations in this code.

## Pareto Principle

Observe that the amount of allocations is approximately 100,000.  We can recall from the code

    for i := 0; i < 100000; i++ {
        buf.WriteString(strconv.Itoa(n))
    }

that we write the integer argument `n` as a string to the buffer 100,000 times.  The benchmark also indicated that `strconv.formatBits` was taking a lot of time too.  But the value of `strconv.Itoa(n)` is not changing every iteration. We can simply calculate it once:

    x := strconv.Itoa(n)
    for i := 0; i < 100000; i++ {
        buf.WriteString(x)
    }

.  Which brings our allocations significantly down:

    Test PASS
    Allocs: 159

Finding opportunities like this should be a top priority when optimizing.  We have gotten rid of most of the allocating, but 159 allocations is still a lot for such a simple function.  Also, if we look at the CPU profiling results, we have only improved run time by about 50%.  If we could eke out the remaining allocations, we could probably get run time to < 1s (started with ~2.3s and are currently at ~1.2s, at least on my system).

## Gradual Improvement

We've made a lot of progress, but we still have some ways to go.  Let's continue to look at `foo` from top to bottom.

    sum := sha256.Sum256(buf.Bytes())

If we look at the [sha256 package](https://golang.org/pkg/crypto/sha256/), it would seem that Sum256 is some kind of shortcut function.  But if we look at the [implementation](https://golang.org/src/crypto/sha256/sha256.go?s=5634:5669#L244):

    func Sum256(data []byte) [Size]byte {
        var d digest
        d.Reset()
        d.Write(data)
        return d.checkSum()
    }

It seems we would have to do this anyways if we were to use the `New` function to create a new hash object.  If there was a way to reuse the digest object, maybe this could be useful (spoiler: there is), but for now there is probably more low hanging fruit available.

If we continue on to the loop,

    var b []byte
    for i := 0; i < int(sum[0]); i++ {
        x := sum[(i*7+1)%len(sum)] ^ sum[(i*5+3)%len(sum)]
        c := strings.Repeat("abcdefghijklmnopqrstuvwxyz", 10)[x]
        b = append(b, c)
    }

we can see that allocations are probably occuring when `b` is appended to and the buffer is grown and when `strings.Repeat` is called because that likely involves some sort of copying.  The line assigning `x` is definitely not allocating because it is merely reading individual integers from the array and performing some calculations on them.  Let's see if there's anything we can do about the `strings.Repeat`.  Well, we are repeating a constant, so we could just unroll the repetition, like:

        c := "abcdefghijklmnopqrstuvwxyzabcdefghijklmnopqrstuvwxyzabcdefghijklmnopqrstuvwxyzabcdefghijklmnopqrstuvwxyzabcdefghijklmnopqrstuvwxyzabcdefghijklmnopqrstuvwxyzabcdefghijklmnopqrstuvwxyzabcdefghijklmnopqrstuvwxyzabcdefghijklmnopqrstuvwxyzabcdefghijklmnopqrstuvwxyz"[x]

But that feels sort of hacky.  What if there was a more fundamental approach?  Let's say `x` equals 53.  If we find the 53rd character of the string, we see it's 'a'.  And the 54th is 'b'.  And the 79th, 105th, (1+26k)th are 'a' too.  Notice a pattern here?  Since the alphabet has 26 characters in it, we can just mod `x` by 26 to get the same thing

        c := "abcdefghijklmnopqrstuvwxyz"[x%26]

Now we are down to 23 allocations.

    Test PASS
    Allocs: 23

Let's look back at the byte slice `b`.  It is created empty and then appended to repeatedly.  Because the new space has to be allocated when it is needed, this creates more allocations than just allocating it once initially.  This can be done with:

    b := make([]byte, 0, int(sum[0]))
    for i := 0; i < int(sum[0]); i++ {
        x := sum[(i*7+1)%len(sum)] ^ sum[(i*5+3)%len(sum)]
        c := "abcdefghijklmnopqrstuvwxyz"[x%26]
        b = append(b, c)
    }

We're down to 19 allocations.

All that's left now is the `return string(b)` statement.  Since this copies the byte slice into a string, it allocates.  There's a trick we can use to prevent this from allocating:

    return *(*string)(unsafe.Pointer(&b))

This brings to 18 allocations.

## Looking at the Function Again
Now that we've done our initial pass through the function, let's look at it again to see if there's any other allocations we can get rid of.

    var buf bytes.Buffer
    x := strconv.Itoa(n)
    for i := 0; i < 100000; i++ {
        buf.WriteString(x)
    }

It seems like instead of creating a bytes.Buffer, we could just create a byte slice and append to it.  This way we allocate a fixed amount once instead of having to deal with the reallocating we mentioned earlier.

So we can replace this with:

    x := strconv.Itoa(n)
    buf := make([]byte, 0, 100000*len(x))
    for i := 0; i < 100000; i++ {
        buf = append(buf, x...)
    }
    sum := sha256.Sum256(buf)

Only 3 allocations now.  If I had to guess, these would be the creation of `buf`, the creation of `b`, and the creation of the digest object in `sha256.Sum256`.  So how do we get rid of these more (seemingly intractable) allocations?  Welcome `sync.Pool`.

## `sync.Pool`
\[Note: The following section borders on what Knuth would probably describe as excessive.  It is primarily meant to be a demonstration of sync.Pool.  It is, however, worth noting that while the difference between 3 allocations and 0 allocations may not seem large, in practice it can be because it means the garbage collector will not have to eventually be called (at least as a result of this function).\]

`sync.Pool` is a feature in Go that allows you to amortize allocation cost.  With our current code, when we allocate the buffers, we do so for the lifetime of the `foo` function.  But what if we were able to allocate it once, store it in a global variable, then just reuse it infinitely (resetting the contents every time while keeping the capacity)?  Doing this manually would be a little complicated and also wouldn't work concurrently.  That's what `sync.Pool` is for. `sync.Pool` allows you to `Get` an already allocated object and `Put` it back when you are done with it.  If there are no objects availabile in the pool and you request one, it will call the `New` function you define to allocate one.
Let's create a pool for the hash object and one for the hash sum (256 byte slice).  Put the following above `foo`:

    var bufPool = sync.Pool{
        New: func() interface{} {
            // length of a sha256 hash
            b := make([]byte, 256)
            return &b
        },
    }

    var hashPool = sync.Pool{
        New: func() interface{} {
            return sha256.New()
        },
    }

Now we need to update our code to use these instead of the ones we create in the function.  Add the following to the top of `foo`:

    // get buffer from pool
    bufptr := bufPool.Get().(*[]byte)
    defer bufPool.Put(bufptr)
    buf := *bufptr
    // reset buf
    buf = buf[:0]

    // get hash object from pool
    h := hashPool.Get().(hash.Hash)
    defer hashPool.Put(h)
    h.Reset()

We use `sync.Pool.Get` to retrieve an object from the pool and we (when the function returns) will put it back in the pool with `sync.Pool.Put`.  Since we now have a hasher object, we can write to that directly instead of an intermediate buffer.  Ideally, we could do something like

    x := strconv.Itoa(n)
    for i := 0; i < 100000; i++ {
        h.Write(x)
    }

Unfortunately, hash objects in Go do not have a WriteString method so we need to use `strconv.AppendInt` to get a byte slice.  Additionally, using AppendInt saves us an allocation since its writes to our buffer we got from `bufPool` instead of allocating a new string like `strconv.Itoa`

    x := strconv.AppendInt(buf, int64(n), 10)
    for i := 0; i < 100000; i++ {
        h.Write(x)
    }

Now we can get the hash and put it in `buf`:

    // reset whatever strconv.AppendInt put in the buf
    buf = buf[:0]
    sum := h.Sum(buf)

In the next for loop, we iterate from 0 to `sum[0]`, perform some calculations, and put the result in `b`.  Since `sum[0]` will never be more than 256 because is a `byte` which ranges from 0-255, we can simply say

    b := make([]byte, 0, 256)

and leave the loop contents the same.  The compiler can reason with 256 at compile time, but can't with `sum[0]`.  This saves us another allocation.

We're finally down to 1 allocation.  Replace the final return statement with the following:

    sum = sum[:0] // reset sum
    sum = append(sum, b...)
    return *(*string)(unsafe.Pointer(&sum))

And you should see

    Test PASS
    Allocs: 0

Why does this work? Well, building what we had originally with verbose escape analysis information may be helpful:

    $ go build -gcflags='-m -m' a.go

    ...
    ./a.go:55:11: make([]byte, 0, 256) escapes to heap:
    ./a.go:55:11:   flow: b = &{storage for make([]byte, 0, 256)}:
    ./a.go:55:11:     from make([]byte, 0, 256) (spill) at ./a.go:55:11
    ./a.go:55:11:     from b := make([]byte, 0, 256) (assign) at ./a.go:55:4
    ./a.go:55:11:   flow: ~r1 = b:
    ./a.go:55:11:     from &b (address-of) at ./a.go:65:35
    ./a.go:55:11:     from *(*string)(unsafe.Pointer(&b)) (indirection) at ./a.go:65:9
    ./a.go:55:11:     from return *(*string)(unsafe.Pointer(&b)) (return) at ./a.go:65:2
    ./a.go:55:11: make([]byte, 0, 256) escapes to heap
    ...

The new code's output:

    ...
    ./a.go:55:11: make([]byte, 0, 256) does not escape
    ...

If we keep the slice in the function and do not return it, it will be stack allocated.  As any C programmer knows, it is not possible to return stack allocated memory.  When the Go compiler sees the return statement, it makes the slice heap allocated instead.  But that doesn't really explain why transferring the contents to `sum` doesn't allocate.  Isn't `sum` also heap allocated?  Yes, but `sync.Pool` already performed that heap allocation for us.

## Takeaways/Tips

- `unsafe` trick to cast strings/byte slices
    - string -> byte slice: `*(*[]byte)(unsafe.Pointer(&s))`
        - the buffer can not be modified or go will panic!
    - byte slice -> string: `*(*string)(unsafe.Pointer(&buf))`
- do not run allocation heavy code in a loop if it does not depend on state updated in the loop
- use `sync.Pool` when working with large allocated objects
    - for byte slices, [bytebufferpool](https://github.com/valyala/bytebufferpool) may be simpler to implement and more performant
- reuse buffers if you can
    - reset buffer
        - `bytes.Buffer.Reset`
        - `buf = buf[:0]`
- prefer allocating fixed size arrays initially rather than leaving the size unspecified and leaving it up to the [resize factor](https://forum.golangbridge.org/t/slice-append-what-is-the-actual-resize-factor/15654)
- make sure to actually [benchmark](https://golang.org/pkg/testing/#hdr-Benchmarks) your code to see if a change improves performance / weigh against readability concerns

## Conclusion

Performance does matter, and it is not always hard to improve.  Although this code was somewhat artificial, there are several lessons that can be learned from improving its performance.
