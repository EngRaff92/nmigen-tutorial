# Values

A `Value` is a basic type. You can think of it as a literal value, but you generally don't use `Value` directly. Instead, you use subtypes such as `Const` or `Signal`.

## Const

A `Const` is a literal value, with a given number of bits, that does not change. 

## Shapes: widths and signedness

But how does nMigen know how many bits the value has? If you simply construct a `Const` out of an integer, like `a = Const(10)` then you'll get exactly as many bits as needed to fit the integer. In this example, four bits. However, you can specify the number of bits you want by adding its width and signedness (its *shape*): `a = Const(10, unsigned(16))` will give you a 16-bit unsigned value.

You can specify that the value is signed, which affects things like magnitude comparisons. `a = Const(-10)` will automatically create a signed value. But `a = Const(10, signed(16))` explicitly creates a 16-bit signed value.

Note that signed values are 2's complement values. `Const(-10)` is a _5-bit value_, because the minimum representation of `-10` in 2's complement is `11010`.

You can also retrieve the shape of a `Const`:

```python
>>> from nmigen import *
>>> a = Const(10)
>>> a.shape()
(width=4, signed=False)
>>> a.width
4
>>> a.signed
False
```

```python
>>> from nmigen import *
>>> a = Const(-10)
>>> a.shape()
(width=5, signed=True)
>>> a.width
5
>>> a.signed
True
```

In general, a `Const` can be created using `Const(<value>, <shape>)`. We have already seen the shapes for `signed(n)` and `unsigned(n)`.

## Shapes: ranges

Just like `Const(10, unsigned(16))` creates a constant of 10 whose size is large enough to hold any 16-bit unsigned integer, `Const(10, range(51))` creates constant of 10 whose size is large enough to hold any integer between 0 and 50 inclusive, which in Python is `range(51)`.

If the range contains a negative number, then the constant will be signed: `Const(3, range(-5, 11))` is of a size to hold any value between -5 and +10 inclusive, which makes it a 2's complement 5-bit signal:

```python
>>> x = Const(3, range(-5, 11))
>>> x.shape()
Shape(width=5, signed=True)
```

### Shapes: enums

Given a Python enum with all integer values, you can create a constant out of it:

```python
from enum import Enum, unique

@unique
class Func(Enum):
    NONE = 0
    ADD = 1
    SUB = 2
    MUL = 3
    DIV = 4

...

>>> x = Const(2, Func)
>>> x.shape()                                                                                                   Shape(width=3, signed=False)
```

This is the equivalent of finding the minimum and maximum value of the enum, and then using that as the range for the constant. In the example above, it would be the same as `Const(2, range(0, 5))`.

The enumerated values of the enum can be used anywhere a `Const` can be used, so that `Func.SUB` is equivalent to `Const(2, range(0, 5))`.

## Signal

A `Signal` is a value that does change. Signals may be implemented as flipflops or wires. The backend will figure it out based on how they are used. It's the equivalent of `wire`, `reg`, or `logic` in Verilog.

Just like a `Const`, a `Signal` has a shape: a width and a signedness. You can create a 16-bit unsigned `Signal`, for example, by `a = Signal(16)` or `a = Signal(unsigned(16))`. You can specify a signed 16-bit `Signal` via `a = Signal(signed(17))` -- you need the extra bit for the sign.

If you don't specify a shape at all: `a = Signal()`, the shape of such a `Signal` is one bit, unsigned.

Again, like with `Const`, you can retrieve the shape of a `Signal`:

```python
>>> from nmigen import *
>>> a = Signal(signed(4))
>>> a.shape()
(width=4, signed=True)
>>> a.width
4
>>> a.signed
True
>>>
```

Again, note that `signed(4)` gives you a 4-bit 2's complement signal which can represent anything from -8 to +7. If you meant to get a Signal that is 4 bits plus a sign in order to represent -16 to +15, then you need `signed(5)`.

### Signal.range

`Signal.range(11)` creates an unsigned signal large enough to hold any integer between 0 and 10 inclusive, which in Python is `range(11)`. As with `Const`, if the range contains a negative number, the resulting signal will be signed:

```python
>>> x = Signal.range(-5, 11)
>>> x.shape()
Shape(width=5, signed=True)
```

### Signal.enum

Given a Python enum with all integer values, you can create a signal out of it:

```python
from enum import Enum, unique

@unique
class Func(Enum):
    NONE = 0
    ADD = 1
    SUB = 2
    MUL = 3
    DIV = 4

...

x = Signal.enum(Func)
```

Again, as with `Const`, this is the equivalent of finding the minimum and maximum value of the enum, and then using that as the range for the signal. In the example above, it would be the same as `Signal.range(0, 5)`.
