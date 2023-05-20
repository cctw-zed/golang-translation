# Errors are values

https://go.dev/blog/errors-are-values

A common point of discussion among Go programmers, especially those new to the language, is how to handle errors. The conversation often turns into a lament at the number of times the sequence.

在Go语言程序员中，特别是初学者，错误处理是很常见的一个讨论话题。然而，讨论最终经常会以一声声抱怨收尾。

```
if err != nil {
    return err
}
```

shows up. We recently scanned all the open source projects we could find and discovered that this snippet occurs only once per page or two, less often than some would have you believe. Still, if the perception persists that one must type

近期我们搜寻了所能找到的几乎所有Go源码，发现这段代码每一页才仅仅出现一两次，并不像一些人想象的那么频繁。

```
if err != nil
```

all the time, something must be wrong, and the obvious target is Go itself.

一定有什么地方出了问题，而显而易见的目标就是Go本身。（Go源码写的不对哈哈哈）

This is unfortunate, misleading, and easily corrected. Perhaps what is happening is that programmers new to Go ask, “How does one handle errors?”, learn this pattern, and stop there. In other languages, one might use a try-catch block or other such mechanism to handle errors. Therefore, the programmer thinks, when I would have used a try-catch in my old language, I will just type `if` `err` `!=` `nil` in Go. Over time the Go code collects many such snippets, and the result feels clumsy.

这种想法是不幸的、存在误导的并且很容易被证伪的。获取很多Go语言初学者会问：”如何处理errors？“，然后停滞不前。在其他语言中（例如Java），大家可能会使用try-catch块或其他类似的机制来处理错误。因此，他们可能会想，当可以使用旧语言时可以使用try-catch的位置，使用Go时就可以使用`if err != nil`来替代。随着时间的推移，Go代码收集了许多这样的片段，结果这样其实很笨拙。

Regardless of whether this explanation fits, it is clear that these Go programmers miss a fundamental point about errors: *Errors are values.*

不管这种解释是否合理，很明显这些Go程序员误解了Errors的本质：Errors are values.

Values can be programmed, and since errors are values, errors can be programmed.

值可以被编程，因为错误也是值，所以错误也可以被编程。

Of course a common statement involving an error value is to test whether it is nil, but there are countless other things one can do with an error value, and application of some of those other things can make your program better, eliminating much of the boilerplate that arises if every error is checked with a rote if statement.

当然，关于error的一个常见语句是检查其是否为空，但是也可以对error做一些其他的操作，在一些应用场景下，这些操作可以使你的程序更加优雅，消除大量包含`if err != nil`的样板代码。

Here’s a simple example from the `bufio` package’s [`Scanner`](https://go.dev/pkg/bufio/#Scanner) type. Its [`Scan`](https://go.dev/pkg/bufio/#Scanner.Scan) method performs the underlying I/O, which can of course lead to an error. Yet the `Scan` method does not expose an error at all. Instead, it returns a boolean, and a separate method, to be run at the end of the scan, reports whether an error occurred. Client code looks like this:

`bufio`包的`Scanner`类型就是一个简单的例子。它的`Scan`方法执行底层的I/O操作，这当然可能会导致error产生。然而，`Scan`方法并没有暴露`error`给上层。相反，它返回一个布尔值，并在扫描结束时运行一个单独的方法，报告是否发生了错误。客户端代码看起来像这样:

```
scanner := bufio.NewScanner(input)
for scanner.Scan() {
    token := scanner.Text()
    // process token
}
if err := scanner.Err(); err != nil {
    // process the error
}
```

Sure, there is a nil check for an error, but it appears and executes only once. The `Scan` method could instead have been defined as

当然，这里有一个对于error的nil判断，但是只执行了一次。Scan方法可以修改为下面的定义：

```
func (s *Scanner) Scan() (token []byte, error)
```

and then the example user code might be (depending on how the token is retrieved),

这样的话，对应的客户端代码可能就变成了下面这样：

```
scanner := bufio.NewScanner(input)
for {
    token, err := scanner.Scan()
    if err != nil {
        return err // or maybe break
    }
    // process token
}
```

This isn’t very different, but there is one important distinction. In this code, the client must check for an error on every iteration, but in the real `Scanner` API, the error handling is abstracted away from the key API element, which is iterating over tokens. With the real API, the client’s code therefore feels more natural: loop until done, then worry about errors. Error handling does not obscure the flow of control.

这并没有太大的不同，但有一个重要的区别。在这段代码中，客户端必须在每次迭代中检查错误，但在实际的API中，错误处理是从key API元素中抽象出来的，该元素遍历令牌。使用真正的API，客户端的代码更自然:循环到最后，然后担心产生了错误。错误处理不会模糊控制流。

Under the covers what’s happening, of course, is that as soon as `Scan` encounters an I/O error, it records it and returns `false`. A separate method, [`Err`](https://go.dev/pkg/bufio/#Scanner.Err), reports the error value when the client asks. Trivial though this is, it’s not the same as putting

当然，在幕后发生的事情是，一旦遇到I/O错误，它就会记录并返回。一个单独的方法在客户端请求时报告错误值。虽然这很琐碎，但和判断err是否为nil不一样

```
if err != nil
```

everywhere or asking the client to check for an error after every token. It’s programming with error values. Simple programming, yes, but programming nonetheless.

任何时候或要求客户端在每个令牌之后检查错误。这是的确是对error的编程，但是是很简单而无意义的编程。

It’s worth stressing that whatever the design, it’s critical that the program check the errors however they are exposed. The discussion here is not about how to avoid checking errors, it’s about using the language to handle errors with grace.

值得强调的是，无论设计是什么，程序检查错误是至关重要的。这里讨论的不是如何避免检查错误，而是如何使用语言优雅地处理错误。

The topic of repetitive error-checking code arose when I attended the autumn 2014 GoCon in Tokyo. An enthusiastic gopher, who goes by [`@jxck_`](https://twitter.com/jxck_) on Twitter, echoed the familiar lament about error checking. He had some code that looked schematically like this:

当我参加2014年秋季东京GoCon时，出现了重复错误检查代码的话题。一位热心的gopher也表达了对错误检查的抱怨。他的一些代码大致是这样的:

```
_, err = fd.Write(p0[a:b])
if err != nil {
    return err
}
_, err = fd.Write(p1[c:d])
if err != nil {
    return err
}
_, err = fd.Write(p2[e:f])
if err != nil {
    return err
}
// and so on
```

It is very repetitive. In the real code, which was longer, there is more going on so it’s not easy to just refactor this using a helper function, but in this idealized form, a function literal closing over the error variable would help:

这是重复性非常高的。在实际的代码中，它更长，有更多的事情要做，所以使用辅助函数来重构它并不容易，但在这种理想形式中，函数字面上关闭错误变量会有所帮助:

```
var err error
write := func(buf []byte) {
    if err != nil {
        return
    }
    _, err = w.Write(buf)
}
write(p0[a:b])
write(p1[c:d])
write(p2[e:f])
// and so on
if err != nil {
    return err
}
```

This pattern works well, but requires a closure in each function doing the writes; a separate helper function is clumsier to use because the `err` variable needs to be maintained across calls (try it).

这种模式工作得很好，但需要在每个函数中使用闭包进行写操作;单独的辅助函数使用起来更笨拙，因为变量需要在多次调用之间进行维护(尝试一下)。

We can make this cleaner, more general, and reusable by borrowing the idea from the `Scan` method above. I mentioned this technique in our discussion but `@jxck_` didn’t see how to apply it. After a long exchange, hampered somewhat by a language barrier, I asked if I could just borrow his laptop and show him by typing some code.

我们可以借鉴上述方法的思想，使其更清晰、更通用和可重用。我在讨论中提到了这个技巧，但没有看到如何应用它。经过长时间的交流，有点受语言障碍的阻碍，我问他是否可以借他的笔记本电脑，然后输入一些代码给他看。

I defined an object called an `errWriter`, something like this:

我定义了一个叫做`errWriter`的对象，就像这样:

```
type errWriter struct {
    w   io.Writer
    err error
}
```

and gave it one method, `write.` It doesn’t need to have the standard `Write` signature, and it’s lower-cased in part to highlight the distinction. The `write` method calls the `Write` method of the underlying `Writer` and records the first error for future reference:

并且给了它一种方法，它不需要有标准的签名，它的小写部分是为了突出区别。该方法调用底层的方法并记录第一个错误以备将来参考:

```
func (ew *errWriter) write(buf []byte) {
    if ew.err != nil {
        return
    }
    _, ew.err = ew.w.Write(buf)
}
```

As soon as an error occurs, the `write` method becomes a no-op but the error value is saved.

一旦出现错误，该方法将不会做任何操作，但会保存错误值。

Given the `errWriter` type and its `write` method, the code above can be refactored:

通过`errWriter`类型及其`write`方法，上面的代码可以重构为:

```
ew := &errWriter{w: fd}
ew.write(p0[a:b])
ew.write(p1[c:d])
ew.write(p2[e:f])
// and so on
if ew.err != nil {
    return ew.err
}
```

This is cleaner, even compared to the use of a closure, and also makes the actual sequence of writes being done easier to see on the page. There is no clutter anymore. Programming with error values (and interfaces) has made the code nicer.

即使与使用闭包相比，这也更简洁，而且还使正在执行的实际写操作顺序更容易在页面上看到。再也没有杂乱了。使用错误值(和接口)编程使代码变得更好了。

It’s likely that some other piece of code in the same package can build on this idea, or even use `errWriter` directly.

很可能同一包中的其他代码段可以构建在这个思想之上，甚至可以直接使用。

Also, once `errWriter` exists, there’s more it could do to help, especially in less artificial examples. It could accumulate the byte count. It could coalesce writes into a single buffer that can then be transmitted atomically. And much more.

此外，一旦`errWrite`存在，它可以做更多的事情，特别是在较少人为的例子中。它可以累加字节计数。它可以将写入合并到一个缓冲区中，然后可以自动传输。还有更多。

In fact, this pattern appears often in the standard library. The [`archive/zip`](https://go.dev/pkg/archive/zip/) and [`net/http`](https://go.dev/pkg/net/http/) packages use it. More salient to this discussion, the [`bufio` package’s `Writer`](https://go.dev/pkg/bufio/) is actually an implementation of the `errWriter` idea. Although `bufio.Writer.Write` returns an error, that is mostly about honoring the [`io.Writer`](https://go.dev/pkg/io/#Writer) interface. The `Write` method of `bufio.Writer` behaves just like our `errWriter.write` method above, with `Flush` reporting the error, so our example could be written like this:

事实上，这种模式经常出现在官方库中。`archive/zip`和`net/http`包都使用了它。在这个讨论中更突出的是，`bufio`的`Writer`实际上是这个想法的实现。虽然q其返回了一个错误，但这主要是需要遵守`io.Writer`接口。

```
b := bufio.NewWriter(fd)
b.Write(p0[a:b])
b.Write(p1[c:d])
b.Write(p2[e:f])
// and so on
if b.Flush() != nil {
    return b.Flush()
}
```

There is one significant drawback to this approach, at least for some applications: there is no way to know how much of the processing completed before the error occurred. If that information is important, a more fine-grained approach is necessary. Often, though, an all-or-nothing check at the end is sufficient.

这种方法有一个明显的缺点，至少对于某些应用程序来说是这样:无法知道在错误发生之前完成了多少次处理。如果该信息很重要，则需要更细粒度的方法。不过，通常情况下，在最后做一个“全有或全无”的检查就足够了。

We’ve looked at just one technique for avoiding repetitive error handling code. Keep in mind that the use of `errWriter` or `bufio.Writer` isn’t the only way to simplify error handling, and this approach is not suitable for all situations. The key lesson, however, is that errors are values and the full power of the Go programming language is available for processing them.

我们只讨论了一种避免重复错误处理代码的技术。请记住，使用或并不是简化错误处理的唯一方法，而且这种方法并不适用于所有情况。然而，关键的教训是，错误是值，Go编程语言的全部功能可用于处理它们。

Use the language to simplify your error handling.

使用该语言简化错误处理。

But remember: Whatever you do, always check your errors!

但请记住:无论你做什么，总是检查你的错误!