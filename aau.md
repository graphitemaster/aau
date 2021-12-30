# Almost Always Unsigned

Written by Dale Weiler

* [Twitter](https://twitter.com/actualGraphite)
* [GitHub](https://github.com/graphitemaster)

The need for signed integer arithmetic is often misplaced as most integers
never represent negative values within a program. The indexing of arrays and
iteration count of a loop reflects this concept as well. There should be a
propensity to use unsigned integers more often than signed, yet despite this,
most code incorrectly choses to use signed integers.

The code presented in here will be mostly C/C++, but it also applies to other
languages as well.

- [The arguments against unsigned](#the-arguments-against-unsigned)
  * [The safety argument](#the-safety-argument)
  * [The loop in reverse argument](#the-loop-in-reverse-argument)
  * [The difference of two numbers can become negative](#the-difference-of-two-numbers-can-become-negative)
  * [Computing indices with signed arithmetic is safer](#computing-indices-with-signed-arithmetic-is-safer)
  * [Unsigned multiplication can overflow](#unsigned-multiplication-can-overflow)
  * [Sentinel values](#sentinel-values)
  * [It's the default](#it-s-the-default)
- [What if signed was defined to wrap?](#what-if-signed-was-defined-to-wrap-)
- [Your counter arguments are about pathological inputs](#your-counter-arguments-are-about-pathological-inputs)
- [The arguments for unsigned](#the-arguments-for-unsigned)
  * [Most integers in a program never represent negative values](#most-integers-in-a-program-never-represent-negative-values)
  * [Compiler diagnostics are better for unsigned but that's worse overall](#compiler-diagnostics-are-better-for-unsigned-but-that-s-worse-overall)
  * [Checking for overflow and underflow is easier and safer](#checking-for-overflow-and-underflow-is-easier-and-safer)
  * [Your code will be simpler and faster](#your-code-will-be-simpler-and-faster)
    + [But signed has optimizations that unsigned does not?](#but-signed-has-optimizations-that-unsigned-does-not-)

## The arguments against unsigned
There are a lot of arguments against the use of unsigned integers. Lets discuss
how they're mostly incorrect.

### The safety argument
The most typical argument against the use of unsigned integers is that it's
more error prone since it's far easier for an expression to underflow than it
is to overflow. This advice is so common that the official [Google C++ Style
Guide](https://google.github.io/styleguide/cppguide.html#Integer_Types) outright discourages the use of unsigned types.

We'll see in the following arguments where these safety issues come from and how
to easily avoid them with trivial idioms that are easier to understand than
using signed everywhere. We'll also see that these arguments are incorrect most
of the time as they encourage continuing to use invalid code.

### The loop in reverse argument
When the counter of a for loop needs to count in reverse and the body of
the loop needs to execute when the counter is also zero, most programmers will
find unsigned difficult to use because `i >= 0` will always evaluate `true`.

The temptation is to cast the unsigned value to a signed one, e.g:
```cpp
for (int64_t i = int64_t(size) - 1; i >= 0; i--) { ... }
```
Of course this is dangerous as it's a narrowing conversion, with a cast which
silences a legitimate compiler diagnostic. It invokes undefined behavior when
given specific large values and most certainly is exploitable. Most applications
would just crash on inputs `>= 0x7ffffffffffffffff`. The typical argument is
that such a value would be "pathological". Not only is this argument incorrect,
it's even more dangerous which we will see later.

This danger is one of the supporting arguments behind always using signed
integer arithmetic. The argument is incorrect though, because `int64_t` would
never permit a value `>= 0x7ffffffffffffffff` anyways. It's only avoiding
the issue in so much as the specific problematic numeric range is no longer
allowed. Tough luck if you needed a value that large and if you followed the
sage advice of Google to always used signed and had that large value anyways,
well now you have an even worse danger. Mainly, for languages like C and C++,
you just unconditionally invoked undefined behavior since signed integer
overflow is undefined. So we'll call this a "tie".

The correct approach here is that unsigned underflow is well-defined in C and
C++, and we should be teaching the behavior of wrapping arithmetic as it's
genuinely useful in general, but it also makes reverse iteration as easy as
forward.
```cpp
for (size_t i = size - 1; i < size; i--) {
    // ...
}
```
The loop counter begins from `size - 1` and counts down on each iteration. When
the counter reaches zero, the decrement causes the counter to underflow and wrap
around to the max possible value of `size_t`. This value is far larger than
`size`, so the condition `i < size` evaluates false and the loop stops.

With this approach, no casts were needed, no silent bugs were introduced, and
the "pathological" input still works correctly. In fact, this form permits every
possible value from `[0, 0xffffffffffffffff)`, covering the entire range.

### The difference of two numbers can become negative
When you want to compute the difference (or delta) between two numbers, it's
often the case to express that as:
```cpp
delta = x - y;
```
Although most of the time the sign isn't needed so you tend to see:
```cpp
delta = abs(x - y);
```
The argument is that unsigned is dangerous here because if `y > x` then you get
underflow. The problem with this argument is it's not valid because the code
itself is simply incorrect regardless of the signedness of `x` and `y`. There
are values for both `x` and `y` which will lead to signed integer underflow.
So like before, in languages like C and C++, you just unconditionally invoked
undefined behavior since signed integer underflow is undefined. So we'll also
call this a "tie".

The **only** correct way to write this is as:
```cpp
delta = max(x, y) - min(x, y);
```
This works regardless of the sigedeness of `x` and `y` and will always give you
the absolute difference safely. It may just be me, but it also reads better too,
you don't even need to name the variable `delta` anymore, the context is self
documenting.

### Computing indices with signed arithmetic is safer
An extension to the above argument is that if you have a more complicated
expression to compute an index it's just safer to express that with signed.
I think this argument primarily comes from an invalid intuition of underflow
and overflow, yet it manifests for signed in significantly worse ways.

Lets take the most trivial "slightly more complicated" expression to compute an
index as an example, the middle in an interval.
```cpp
int mid = (low + high) / 2;
```

This is how most people would write it. The average of low and high, truncated
to the nearest integer. When the sum of `low` and `high` exceeds `2^31-1` the
sum overflows to a negative value and the negative stays negative when divided.
Using a larger signed integer type here does not save you either because it's
easy for the sum to exceed `2^64-1` too.

Like the previous example, this code is just wrong.

The safe signed way of writing this isn't simpler by any means either,
since you wouldn't expect to have to write it as:
```cpp
int mid = low + (high - low) / 2;
```

However, had you just stuck with unsigned integers, you could write it the
obvious way you originally intended to, precisely because overflow is
well-defined and does the right thing here.
```cpp
size_t mid = (low + high) / 2;
```

> I picked this as real-world, non-contrived example. You can expect to find
this in binary searches, merge sort, and pretty much any other divide-and-conquer
algorithm.

### Unsigned multiplication can overflow
This is a very common complaint for memory safety specifically because unsigned
multiplication is most often seen when allocating memory. It's tempting to write
the following in C.
```c
malloc(sizeof(Object) * n)
```
If such an expression were to overflow then `malloc` will allocate and return
a pointer for memory not actually sufficiently large for all `n` `Object`. Again,
signed here does not save you, since overflow is undefined in C and C++. In
practice this will silently avoid the memory safety issue by over-allocating
around 4 GiB of memory had you used `int` since negatives casted to `size_t`.
Random resource exhaustion is not exactly a better situation.

There are a couple better ways to write this. The first obvious one is just
use `calloc`. That does have a zeroing cost though and doesn't help if you need
to `realloc`. In C++ you should just be using array `new[]` since you don't
need to multiply anything.

Those don't generally apply though. It's very trivial to check if `x * y` would
overflow though and you should just learn the extremely simple and obvious test
```cpp
if (y && x > (T)-1/y)
```
> Where `(T)-1` here is your unsigned type and casting `-1` just gives you the max
value, i.e every bit set.

### Sentinel values
One extremely common use of signed integers is using the negative range to encode
an error code or some [sentinel value](https://en.wikipedia.org/wiki/Sentinel_value).
This is a terrible programming practice and a literal category error. It's pretty
unavoidable when working with existing or legacy code that is designed around it,
but it's not a strong argument for signed integers to continuing this practice.
You can still have sentinels with unsigned too, not that you should.

We should not be encouraging this and using it as an argument in favor of signed.
```cpp
int result = connect();
if (result >= 0) // ...
```
Where anything but a positive value is an error. This is typical of early C.

There are multiple better and safer ways to express this, for instance.
* We could have an out param for the result instead.
    ```cpp
    if (uint result; connect(&result)) {
        // ...
    }
    ```
* We can return a tuple or pair
    ```cpp
    tuple<bool, uint> result = connect();
    pair<bool, uint> result = connect();
    if (result.first) {
        // ...
    }
    ```
* Could use an option type.
    ```cpp
    option<uint> result = connect();
    maybe<uint> result = connect();
    if (result) {
        // ...
    }
    ```

* However, if the numeric range is well-defined to begin with, just define
the entire domain and avoid using full-range integers to represent a subset
of possible values.

    ```cpp
    enum class ConnectionStatus : unsigned {
        Connected,
        NoRouteToHost,
        Disconnected,
        TimedOut,
    };

    ConnectionStatus status = connect();
    if (status != ConnectionStatus::Connected) {
        error();
    }
    ```

### It's the default
This is a weak argument, but it's at least the only one that is hard to refute.
C and C++ default `int` to signed and that's a historical thing that has
persisted in many languages that are descendants of C.

## What if signed was defined to wrap?
Some languages like Go and [Odin](https://odin-lang.org/) claim to avoid these
problems because their signed integer is well-defined to wrap on underflow or
overflow. The safety arguments there are incorrect as well. In all the previous
arguments, if the signed integers wrapped, they would almost always produce
negative values which would either introduce silent a logic bug, or worse,
memory safety issues if used as indices into arrays as an example.

Unfortunately, for languages based on LLVM, no amount of hand-waving and wanting
signed wrapping to be defined will work no matter how hard you try, so such
statements about safety are factually incorrect. Putting aside integer divide
by zero being a trap representation on all hardware, the less known:
`INT_MIN / -1` and conversely, `INT_MIN % -1` are [undefined in LLVM](https://blog.regehr.org/archives/887) in all possible configurations and thus invoke undefined behavior
even in Odin.

> The only way to ever make safe use of signed integers in this manner is to
bounds check all array accesses, which has a appreciable, non-negligible code.
It's also equally as error prone if not automated as it's easy to forget to check.

## Your counter arguments are about pathological inputs
It's been my experience that our intuition of what is and isn't a pathological
or malicious input is about as accurate as time estimates. Requirements change
and any and all attack vectors will be found and exploited. The mental burden
of remembering the assumptions made to correctly check for pathological or
malicious inputs in all cases and keep them updated during refactoring is far
too enormous to successfully maintain.

One of the most famous cases of this stubborn attitude is the infamous [qmail
64-bit remote code execution](https://www.guninski.com/where_do_you_want_billg_to_go_today_4.html) exploit which Daniel J. Bernstein denied a bounty
for. Which can still be exploited as of [2020](https://www.qualys.com/2020/05/19/cve-2005-1513/)

These are very classical **signed** integer overflow, pointer with **signed** index and **signedness** problems writes the very exploit author.

The use of unsigned integer arithmetic not only prevents these bugs, it forces
you to think about pathological and malicious inputs more directly because it
becomes more evident.

## The arguments for unsigned

### Most integers in a program never represent negative values
I cannot find a research paper I once read from Intel which claimed from their
observations that **only 3% of the integers** in an entire operational
Windows system on x86_64 **ever represented negative values**. Regardless, if
that 3% figure is correct, then i'd expect to see 97% of integer types in a
codebase to be unsigned.

### Compiler diagnostics are better for unsigned but that's worse overall
If safety is one of the primary motivations behind the use of signed integer
arithmetic, yet signed integers don't seem to actually avoid the very bugs
it is claimed they do, you might be wondering where the idea came from in the
first place.

If you've ever _mixed_ the use of signed and unsigned in a codebase you'll
likely be familiar with the amount of warning diagnostics they emit. These
diagnostics are unfortunately provided with the intent of being helpful, but
in practice are actively malicious because they encourage silencing in the form
of unsafe type casting.

The reality is that the use of signed and unsigned paints all your integers red or blue, respectively. [What color is Your Function](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/). The more of one you use, the more likely it is
everything will also share the same signedness regardless of if it's appropriate.
Since most integers never represent negative values though, I think it's more
appropriate to paint everything blue in this case.

### Checking for overflow and underflow is easier and safer
Since C and C++ make signed integer overflow and underflow undefined, it's
almost impossible to write safe, correct, and obvious code to check for it.
So a simple, obvious tests like:
```cpp
if (x + y < x) // addition overflows
if (x - y > x) // subtraction underflows
```
Will either get compiled away, or miscompiled as it invokes undefined behavior.
Yet this is perfectly fine and safe for unsigned integers, always.

This is the only correct way (I know of) to detect for signed integer overflow
and underflow in C or C++, good luck remembering this.
```cpp
if ((b > 0 && a > INT_MAX - b) || (b < 0 && a < INT_MIN - b)) // addition overflows
if ((b > 0 && a < INT_MIN + b) || (b < 0 && a > INT_MAX + b)) // subtraction underflows
```

### Your code will be simpler and faster
In addition to all the examples I've already shown where unsigned just does the
right thing, almost all code that uses signed integers to represent values that
will never be negative, tends to have a cacophony of range assertions and other
tests which are just as error-prone as bounds checking to remember to write, but
also maintain when refactoring. It's truly underappreciated how much unnecessary
so much of those tests can just go if your integer can never actually become
negative due to the type system itself. It's extremely similar to not having
raw pointers, in that you never have to check for null pointers. In many ways
signed integers are the null pointers of integers.

#### But signed has optimizations that unsigned does not?
It's true there are some optimizations compilers can make assuming signed
integers cannot underflow or overflow that you would otherwise not get to
participate in had you used unsigned. The less known reality is that [value range
analysis](https://en.wikipedia.org/wiki/Value_range_analysis) is an optimization
that can apply to any numeric type of any numeric range in modern [optimizing
compilers](https://developers.redhat.com/blog/2021/04/28/value-range-propagation-in-gcc-with-project-ranger). The `enum` example from
earlier is something where this would apply, despite that being an unsigned type.
You can define numeric ranges as compiler hints with simple guiding branches,
e.g the expression `b = a + 2` immediately establishes that `b > a`, which the
compiler can use to optimize.