---
title: "Almost Always Unsigned"
draft: false
---

# Almost Always Unsigned

Written by Dale Weiler

* [Twitter](https://twitter.com/actualGraphite)
* [GitHub](https://github.com/graphitemaster)

The need for signed integer arithmetic is often misplaced as most integers
never represent negative values within a program. The indexing of arrays and
iteration count of a loop reflects this concept as well. There should be a
propensity to use unsigned integers more often than signed, yet despite this,
most code incorrectly choses to use signed integers almost exclusively.

Most of the motivation of this article applies to C and C++, but examples for
other languages such as Go, Rust and [Odin](https://odin-lang.org/) will also be
presented in an attempt to establish that this concept applies to all
languages, regardless of their choices (for instance C and C++ leave signed
integer wrap undefined), but rather is intrinsic to the arithmetic itself.

## The arguments against unsigned
There are a lot of arguments against the use of unsigned integers. Let me
explain why I think they're mostly incorrect.

### The safety argument
The most typical argument against the use of unsigned integers is that it's
more error prone since it's far easier for an expression to underflow than it
is to overflow. This advice is so common that the official
[Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html#Integer_Types)
outright discourages the use of unsigned types.

We'll see in the following arguments where these safety issues come from and how
to easily avoid them with trivial idioms that are easier to understand than
using signed everywhere. We'll also see that these arguments are incorrect most
of the time as they encourage continuing to write and use unsafe code.

### The loop in reverse argument
When the counter of a for loop needs to count in reverse and the body of
the loop needs to execute when the counter is also zero, most programmers will
find unsigned difficult to use because `i >= 0` will always evaluate `true`.

The temptation is to cast the unsigned value to a signed one, e.g:
```cpp
for (int64_t i = int64_t(size) - 1; i >= 0; i--) {
    // ...
}
```
Of course this is dangerous as it's a narrowing conversion, with a cast which
silences a legitimate warning. In C and C++ it invokes undefined behavior when
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
overflow is undefined.

> In languages that define wrap behavior like Go and Odin, this loop just won't
work correctly.

The correct approach here is that unsigned underflow is well-defined in C and
C++ and we should be teaching the behavior of wrapping arithmetic as it's
genuinely useful in general, but it also makes reverse iteration as easy as
forward.
```cpp
for (size_t i = size - 1; i < size; i--) {
    // ...
}
```

In general, the approach here is to begin from `size - 1` and count down on each
iteration. When the counter reaches zero, the decrement causes the counter to
underflow and wrap around to the max possible value of the unsigned type. This
value is far larger than `size`, so the condition `i < size` evaluates false and
the loop stops.

Languages like Rust chose to make even unsigned underflow a trap representation
in Debug builds, but specific features like `Range` will let you safely achieve
the same efficient wrapping behavior on underflow with much cleaner syntax.
```rust
for i in (0..size).rev() {
    // ...
}
```

With this approach, no casts were needed, no silent bugs were introduced, and
the "pathological" input still works correctly. In fact, this form permits every
possible value from `[0, 0xffffffffffffffff)`, covering the entire range.

It should be noted that if `size == 0` these loops stil works because `0 - 1`
produces the largest possible value of the unsigned type which is larger than
`size` (still `0`) and so the loop never enters.

### The difference of two numbers can become negative
When you want to compute the difference (or delta) between two numbers, it's
often the case to want to express that as:
```cpp
delta = x - y;
```
Although most of the time the sign isn't needed so you tend to write and see:
```cpp
delta = abs(x - y);
```
The argument is that unsigned is dangerous here because if `y > x` then you get
underflow. The problem with this argument is it's not valid because the code
itself is simply incorrect regardless of the signedness of `x` and `y`. There
are values for both `x` and `y` which will lead to signed integer underflow.
So like before, in languages like C and C++, you just unconditionally invoked
undefined behavior since signed integer underflow is undefined.

It turns out that computing differences safely is actually quite hard for
signed integers because of underflow, even in languages which support wrapping
behavior for them, e.g `INT_MAX - INT_MIN` is still going to be incorrect. 
There just isn't a trivial way to do this safely, this is the best technique
I currently know of.
```cpp
if ((y > 0 && x < INT_MIN + y) || (y < 0 && x > INT_MAX + y)) {
    // error
} else {
    delta = abs(x - y);
}
```

For unsigned integers however, it's so much easier to just write:
```cpp
delta = max(x, y) - min(x, y);
```
Which will always give you the absolute difference safely. It may just be me,
but it also reads better too, you don't need to name the variable `delta` anymore
as the context is self documenting.

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
sum overflows (invoking undefined behavior) to a negative value and the negative
stays negative when divided. Using a larger signed integer type here does not
save you either because it's easy for the sum to exceed `2^64-1` too.

Like the previous example, this code is just wrong.

In fact, computing the mid-point of two variables safely is pretty much
impossible in any language without the help of a library function because
as wrap or otherwise, there is specific inputs that will fail.

One common solution for signed is to rewrite it to this idiom, which still fails
for `high = INT_MAX` and `low = INT_MIN`.
```cpp
int mid = low + (high - low) / 2;
```

Sticking with unsigned integers, you might think you can write it the obvious
way you originally intended to precisely because underflow is well-defined
but we'll see that in the following:
```cpp
size_t mid = (low + high) / 2;
```
This doesn't actually work for e.g: `low = 0x80000000` and `high = 0x80000002`,
which would underflow and produce `2` which divided by `2` produces `1`, when
the correct value is actually `0x80000001`.

Where unsigned does benefit here is when these are used as indices into an array.
The signed behavior will almost certainly produce invalid indices which leads to
memory unsafety issues. The unsigned way will never do that, it'll stay bounded,
even if the index it produces is actually wrong, which is a less-severe logic
bug, but can still be used maliciously.

The correct safe way to do this regardless of signedness is to convert
everything to unsigned and account for the wrapping with masked addition.
This should really just be a general function in your codebase though.
```cpp
// U is the unsigned version of your type T, or same as T if T already unsigned
U shift = digits - 1; // numeric digits (not including sign) in your type T.
                      // std::numeric_limits<T>::digits in C++ is helpful here.
U difference = (U)x - (U)y;
U sign = y < x;
U half = (difference / 2) + (sign << shift) + (sign & difference);
T mid = (T)(x + half);
```
> C++ actually has `std::midpoint` which does precisely this.

> I picked this as real-world, non-contrived example. You can expect to find
this in binary searches, merge sort, and pretty much any other divide-and-conquer
algorithm.

### Unsigned multiplication can overflow
This is a very common complaint for memory safety specifically because unsigned
multiplication is most often seen when allocating arrays. It's tempting to write
the following in C.
```c
malloc(sizeof(Object) * n)
```
If such an expression were to overflow then `malloc` will allocate and return
a pointer for memory not actually sufficiently large for all `n` `Object`. Again,
signed here does not save you, since overflow is undefined in C and C++. In
practice this will silently avoid the memory safety issue by over-allocating
around 4 GiB of memory had you used `int` since a negative casted to `size_t`
becomes about that large. Random resource exhaustion is not exactly a better
situation to be in here.

There are a couple better ways to write this. The first obvious one is just
use `calloc`. That does have a zeroing cost though and doesn't help if you need
to `realloc`. In C++ you should just be using array `new[]` since you don't
need to multiply anything.

Those don't apply generally though. It's very trivial to check if `x * y` would
overflow though and you should just learn the extremely simple and obvious test
```cpp
if (y && x > (T)-1/y)
```
> Where `(T)-1` here is your unsigned type and casting `-1` just gives you the max
value, i.e every bit set. You can also use `~((T)0)`, `UINT{8,16,32,64}?_MAX`,
or `numeric_limits<T>::max()` in C++, there's a lot of ways to compute this value.

### Sentinel values
One extremely common use of signed integers is using the negative range to encode
an error code or some [sentinel value](https://en.wikipedia.org/wiki/Sentinel_value).
This is a terrible programming practice and a literal category error. It's pretty
unavoidable when working with existing or legacy code that is designed around it,
but it's not a strong argument of signed integers for continuing this practice.
You can still have sentinels with unsigned too, not that you should.

We should not be encouraging this and using it as an argument in favor of signed.
```cpp
int result = connect();
if (result >= 0) // ...
```
Where anything but a positive value is an error.
> This is typical of early C.

There are multiple better and safer ways to express this, for instance.
* We could have an out param for the result instead.
    ```cpp
    if (uint result; connect(&result)) {
        // ...
    }
    ```
* We can return a tuple or pair.
    ```cpp
    tuple<bool, uint> result = connect();
    pair<bool, uint> result = connect();
    if (result.first) {
        // ...
    }
    ```
* We could use an option type.
    ```cpp
    option<uint> result = connect();
    maybe<uint> result = connect();
    if (result) {
        // ...
    }
    ```

* However, if the numeric range is well-defined to begin with, just define
the entire domain and avoid using full-range integers to represent a subset
of all the possible values.

    ```cpp
    enum class ConnectionStatus : unsigned {
        Connected,
        NoRouteToHost,
        Disconnected,
        TimedOut,
    };

    ConnectionStatus status = connect();
    if (status == ConnectionStatus::Connected) {
        // ...
    }
    ```

### It's the default
This is a weak argument, but it's at least the only one that is hard to refute.
C and C++ default `int` to signed and that's a historical thing that has
persisted in many languages that are descendants of C. It's generally agreed
upon now that a lot of defaults in C were bad, maybe we should add this too?

## What if signed was defined to wrap?
Some languages like Go and [Odin](https://odin-lang.org/) claim to avoid these
problems because signed integer arithmetic is defined to wrap on underflow and
overflow. The safety arguments there are incorrect as well. In all the previous
examples, if the signed integers wrapped, they would almost always produce
negative values which would either introduce silent logic bugs, or worse,
memory safety issues if used as indices into arrays as an example.

> The only way to ever make safe use of signed integers in this manner is to
bounds check all array accesses, which has a appreciable, non-negligible runtime
cost. Bounds checking is also quite error prone if not automated as it's easy
to forget to check at all and often unmaintained under code refactoring.

Unfortunately, for languages based on LLVM, no amount of hand-waving and wanting
signed wrapping to be defined will work no matter how hard you try, so such
statements about safety are factually incorrect anyways. Here's a somewhat
exhaustive list of all the undefined behavior of signed integer arithmetic 
in LLVM which applies to all languages which use LLVM:
* `x / 0`
* `INT_MIN / -1`, or `INT_MAX % -1`
* `INT_MAX - INT_MIN`

> These are certain to produce invalid results in languages like Rust and Odin.

## What about trapping?
Some languages such as Rust instead take a different approach where any
integer underflow or overflow in debug builds will lead to a trap representation
where your program will panic. Sanitizers for C and C++ also exist to help
detect these problems and because C and C++ define unsigned underflow to wrap,
it's actually the case that the use of signed integers is better as it's the
only way you can get trap behavior for integers, as using it on unsigned would
trigger trap representations for valid code that relies on that behavior.

Trap representations are actually quite terrible since they can only trigger at
runtime when those paths are successfully executed with the correct trap-producing
inputs. This coverage is impossible to expect in any non-trivial program even
with tons of unit tests. The idea is also incompatible in many contexts such as
library code where you almost never want the library to panic, but rather all
errors be recoverable by the calling application code. Or in service-availability
sensitive code which must not be susceptible to denial of service attack vectors,
where a panic is pretty much game over.

## Your counter arguments are about pathological inputs
It's been my experience that our intuition of what is and isn't a pathological
or malicious input is about as accurate as time estimates. Requirements change
and any and all attack vectors will be found and exploited. The mental burden
of remembering the assumptions made to correctly check for pathological or
malicious inputs in all cases and keep them updated during refactoring is far
too enormous to successfully maintain.

One of the most famous cases of this stubborn attitude is the [infamous qmail
64-bit remote code execution exploit](https://www.guninski.com/where_do_you_want_billg_to_go_today_4.html) which Daniel J. Bernstein denied a bounty
for and can still be [exploited as of 2020](https://www.qualys.com/2020/05/19/cve-2005-1513/)

These are:
> "classical **signed** integer overflow, pointer with **signed**
index and **signedness** problems"

Writes the very exploit author.

The use of unsigned integer arithmetic not only prevents these bugs, it forces
you to think about pathological and malicious inputs more directly because it
becomes more evident.

## The arguments for unsigned

### Most integers in a program never represent negative values
The use of unsigned is a good type indication of the numeric range of the
integer, in much the same way sized integer types are too. The immediate
ability to disregard negative quantities is one of the largest benefits to
actually using unsigned variables. It's a simple observation to make that most
values in a program never actually are negative and never can become negative,
we should be encoding that intent and behavior within the type system for the
added safety and benefits it provides.

I cannot find a research paper I once read from Intel which claimed from their
observations that **only 3% of the integers** in an entire desktop x86 Windows
system **ever represented negative values**. Regardless, if that 3% figure is
correct, then given the above opinion, I would expect to see ~97% of integer
types in a codebase being unsigned.

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

The reality is that the use of signed and unsigned paints all your integers red
or blue, respectively. [What color is Your Function](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/). The more of one you use, the more likely it is
everything will also share the same signedness regardless of if it's appropriate.
Since most integers never require representing negative values, I personally
think it's more appropriate to paint everything blue in this case. The exception
is negative integer values. The rule is mostly positive integer values. The
default of any programming language should align with the rule, rather than the
exception.

### Checking for overflow and underflow is easier and safer
Since C and C++ make signed integer overflow and underflow undefined, it's
almost impossible to write safe, correct, and obvious code to check for it.

So simple, obvious tests like the following:
```cpp
if (x + y < x) // addition overflows
if (x - y > x) // subtraction underflows
```
Will either get compiled away, or miscompiled as it invokes undefined behavior.
> Yet this is perfectly fine and safe for unsigned integers, always.

For a fun laugh, this is the only correct way (that I know of) to detect for
signed integer overflow and underflow in standard C or C++.
```cpp
if ((b > 0 && a > INT_MAX - b) || (b < 0 && a < INT_MIN - b)) // addition overflows
if ((b > 0 && a < INT_MIN + b) || (b < 0 && a > INT_MAX + b)) // subtraction underflows
```
> Good luck remembering and typing these monstrosities when you need it.

### Your code will be simpler and faster
In addition to all the examples I've already shown where unsigned just does the
right thing, almost all code that uses signed integers to represent values that
will never be negative, tends to have a cacophony of range assertions and other
tests which are just as error-prone as bounds checking to remembering to write,
but also maintain when refactoring. It's truly underappreciated how much those
tests can just go if your integer can never actually become negative due to the
type system itself. It's extremely similar to not having raw pointers, in that
you never have to check for null pointers. In many ways signed integers are the
null pointers of integers.

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