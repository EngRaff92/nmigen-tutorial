# Splitting and combining signals

## Slicing signals

We can use the array indexing operator to extract bits from a signal. For example, given a 16-bit signal `s`, we can get the least significant bit via `s[0]` or the most significant bit via `s[15]`.

```python
>>> x = Signal(16)
>>> x.shape()
Shape(width=16, signed=False)
>>> x[15].shape()
Shape(width=1, signed=False)
```

```python
>>> x = Signal(signed(16))
>>> x.shape()
Shape(width=16, signed=True)
>>> x[15].shape()
Shape(width=1, signed=False)
```

Just like Python arrays, the bits in a signal are always ordered in one and only one way. This can cause a bit of confusion for those familiar with indexing in HDL. While `s[7:0]` might seem to extract the eight least significant bits of a signal, this is not the way Python works, and _it is Python that we are programming in_. The correct way to extract the eight least significant bits of a signal would be `s[0:8]` or `s[:8]`, and this is the same way we would extract the first eight elements of a Python array. In essence, the _first_ N bits of a signal, when treated like an array of bits, are the N _least significant_ bits of that signal.

Remember that since this is Python, negative slice indices are offsets from the end, so a way of getting the most significant bit ("last bit") out of a signal is just `x[-1]`.

You can even use strides: `x[0:8:2]` is simply bits 0, 2, 4, and 6 of `x`.

Note that taking bits out of a signal always results in an unsigned signal.

```python
>>> x = Signal(signed(16))
>>> x.shape()
Shape(width=16, signed=True)
>>> x[:8].shape()
Shape(width=8, signed=False)
```

You can even assign to a piece of a signal:

```python
m.d.comb += x[:8].eq(y)
```


### Tip: Using a slice when comparing

In a situation like this:

```python
a = Signal(unsigned(16))
b = Signal(unsigned(16))
c = Signal(unsigned(16))

m.d.comb += c.eq(a+b)
```

We expect that if `a+b` overflows, `c` will just be the lower 16 bits of the result. However, consider this:

```python
a = Signal(unsigned(16))
b = Signal(unsigned(16))
z = Signal()

m.d.comb += z.eq((a+b) == 0)
```

Here, `a+b` must be a 17-bit signal because in Python, integers are as wide as they need to be. So a 16-bit overflow is not a 17-bit overflow, and this comparison will fail for values such as `a=0xFFFF` and `b=1`. The addition would be `0x10000`, which is obviously not 0.

Therefore, be careful to slice the result:

```python
m.d.comb += z.eq((a+b)[:16] == 0)
```

Alternatively, just use an intermediate signal:

```python
tmp = Signal(unsigned(16))

m.d.comb += tmp.eq(a+b)
m.d.comb += z.eq(tmp == 0)
```

This becomes especially insidious when combining unsigned and signed signals. Consider:

```python
ptr = Signal(unsigned(16))
addr = Signal(unsigned(16))
offset = Signal(signed(5)) # -16 to +15

m.d.comb += ptr.eq(addr + offset)
```

we expect `ptr` to be a 16-bit value, since that is what we set it to be. However, what happens here?

```python
y = Signal()

m.d.comb += y.eq((addr + offset) == 0xFFFF)
```

Suppose `addr` is 0 and `offset` is -1. Will this comparison work? It seems like it should, but it will not! Consider that `addr`, an unsigned 16-bit value which goes from 0 to 0xFFFF, plus `offset`, a 2's complement 5-bit value which goes from -16 to +15, results in a signal that needs to be signed and wide enough to accept any number from -0x10 to 0x1000E. We can see that this requires a 2's complement 18-bit value -- roughly 17 bits for the magnitude, and one for the sign.

So the result of `addr+offset` in this case is -1, which in 2's complement 18-bits is `0x3FFFF` which would be `0xFFFF` if we sliced it, but without the slice, the comparison will not work.

## Concatenating signals

You can create a new signal out of other signals using `Cat`:

```python
m.d.comb += x.eq(Cat(a, b, ...))
```

This concatenates the given signals _first element last_. This is important: it may be somewhat surprising that `a` in the example above ends up as the least significant bits of `x`. That is, the concatenation of `a` and `b` is not `ab` but `ba`.

It is now easy to swap the bytes of a 16-bit signal:

```python
m.d.sync += x.eq(Cat(x[8:], x[:8]))
```

You can also assign to a `Cat`, so swapping the bytes can be accomplished in this way also:

```python
m.d.sync += Cat(x[8:], x[:8]).eq(x)
```

## Replicating signals

You can replicate a signal by concatenating it to itself via `Cat(x, x)`. But you can also replicate the signal via `Repl(x, 2)`.

`Repl` with `Cat` can be used together to, for example, sign-extend a value:

```python
uint16 = Signal(unsigned(16)) # Yes, note *un*signed
int32 = Signal(signed(32))

m.d.comb += int32.eq(Cat(uint16, Repl(uint16[15], 16)))
```

Of course, the same can be done by simply using the right signal types:

```python
uint16 = Signal(unsigned(16)) # Yes, note *un*signed
int32 = Signal(signed(32))

m.d.comb += int32.eq(uint16)
```

The generated code will do the right thing.

## Arrays

You can create an array of signals like this:

```python
# All of these create an array of 3 16-bit elements:

# Creates an array from a, b, c:
a = Signal(unsigned(16))
b = Signal(unsigned(16))
c = Signal(unsigned(16))

abc = Array([a, b, c])

# Creates an array of 16-bit signals:
x = Array([Signal(unsigned(16)), Signal(unsigned(16)), Signal(unsigned(16))])

# Also creates an array of 16-bit signals, taking advantage of Python's list comprehension:
y = Array([Signal(unsigned(16)) for _ in range(3)])
```

You can even create multidimensional arrays:

```python
# Creates a 3 by 5 array of 16-bit signals:
yy = Array([Array[Signal(unsigned(16)) for _ in range(5)] for _ in range(3)])
```

You can index into the array with a constant:

```python
z = y[2]
```

This will result in an "elaborate time" error if the index is out of bounds.

However, you can also index with another signal:

```python
i = Signal(unsigned(16))

z = y[i]
```

Of course, during elaboration this will not result in an error. The actual result depends on your simulator or HDL compiler. It is best to ensure as much as possible that your access is not invalid. One way is to declare the index to only have a valid range:

```python
y = Array([Signal(unsigned(16)) for _ in range(5)])
i = Signal.range(5)

z = y[i]
```

Of course, there is nothing preventing `i` from being 5, 6, or 7, since it is a 3-bit signal, and this will lead to surprising results.

Another way is to simply deal with invalid values:

```python
y = Array([Signal(unsigned(16)) for _ in range(5)])
i = Signal.range(5)

z = y[i % 4]
```

This isn't great because it can still lead to surprising results.

In the end, you will have to *formally verify* that `i` will only contain valid values. We will talk about formal verification extensively in later sections.