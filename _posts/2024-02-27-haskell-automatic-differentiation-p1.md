# Automatic Differentiation in Haskell: Part 1
This is going to be a series of spread-out blog posts on automatic
differentiation and implementations of different methods in Haskell.

This first part will cover **forward-mode automatic differentiation**.

&nbsp;

## Automatic Differentiation and the Chain Rule
If you ended up on this post, it's because you either searched for it or I
linked it to you. Regardless, you likely know *why* we want automatic
differentation, but if not I'll briefly summarize...

We can optimize things (models, algorithms, objectives, etc.) by gradient-based
methods, like [stochastic gradient descent][3] or [L-BFGS][4]. It's fairly
obvious that these methods need to calculate derivatives to get gradients, but
we don't want to do this by hand (not always practical) or symbolically (way
too slow *most of the time*). Automatic differentiation gives us the ability
to compute gradients while only defining derivatives for certain component
parts.

The power behind most of this computation is the chain rule, which you may be
familiar with from calculus classes. Here it is in two forms:

$$\dfrac{\mathrm{d}z}{\mathrm{d}x}=\dfrac{\mathrm{d}z}{\mathrm{d}y}
\dfrac{\mathrm{d}y}{\mathrm{d}x},\ \ \ \left(g\circ f\right)'\!\left(x\right)=
f'\!\left(x\right)g'\!\left(f\!\left(x\right)\right)$$

Essentially, provided we know the derivatives for both parts, we can compute
the derivative for their chain. From this, we can compute gradients for large
and quite complex functions with ease! There's quite a few methods for doing
this, and the primary method this post looks into is using **dual numbers**.

&nbsp;

## Dual Numbers
I'm going to assume you're familiar with [complex numbers][1], essentially

$$z=a+ib,\ \ \ i^2=-1,\ \ \ a,\,b\in\mathbb{R}.$$

Complex numbers have distant cousins that live in a different [ring][2], and
these cousins are the **dual numbers**. The concept is similar, except instead
of $i$ we have $\varepsilon$ where

$$z=a+\varepsilon b,\ \ \ \varepsilon^2=0,\ \varepsilon\neq 0,\ \ \
a,\,b\in\mathbb{R}.$$

Seasoned mathematicians will see this and think "hmm yes, makes sense". For the
rest of you thinking "how on ***Earth*** can something not be zero yet when
squared *is* zero?!", I have good news:

<center>Don't worry about it.</center>

It's just a relationship. We shrug and say "okay" and treat it like any other
value with certain relationships - that is, if we end up with $\varepsilon^2$
at any point we just replace it with $0$ if we want.

Dual numbers become interesting when we find out that they can perform
automatic differentiation. Let's consider a function
$f:\mathbb{R}\to\mathbb{R}$. In order to extend $f$ to the dual numbers
(producing $f:\mathbb{D}\to\mathbb{D}$), we can use $f$'s Taylor series
expansion:

$$f\!\left(a+\varepsilon b\right)=\sum_{n=0}^\infty\dfrac{
\varepsilon^nb^nf^{\left(n\right)}\!\left(a\right)}{n!}=f\!\left(a\right)+
\varepsilon bf'\!\left(a\right)+\frac{1}{2}\varepsilon^2b^2f''\!\left(a
\right)+\cdots$$

But since $\varepsilon^2=0$, the $n=2$ term and anything after it must also be
$0$. This means our extension is

$$f\!\left(a+\varepsilon b\right)=f\!\left(a\right)+\varepsilon
bf'\!\left(a\right)$$

and by considering the dual term $bf'\!\left(a\right)$, we can see this
performs the chain rule by applying a second function:

$$g\!\left(f\!\left(a+\varepsilon b\right)\right)=
g\!\left(f\!\left(a\right)+\varepsilon bf'\!\left(a\right)\right)=
g\!\left(f\!\left(a\right)\right)+\varepsilon bf'\!\left(a\right)g'\!\left(
f\!\left(a\right)\right)$$

where the chain rule appears as $f'\!\left(a\right)g'\!\left(f\!\left(a
\right)\right)$! From this, we can also see that setting $b=1$ in the original
value will give us the gradient with respect to $a$ in the dual term.

&nbsp;

### Implementing Dual Numbers in Haskell
The data type for a dual number can be as simple as
```haskell
data Dual a =
  Dual a a
```
since a value of `Dual` just contains two values of the same type - the real
part and the dual part.

We may also wish to pretty-print this, so `Dual` should be made an instance of
`Show`:
```haskell
instance Show a => Show (Dual a) where
  show :: Dual a -> String
  show (Dual x dx) =
    concat [show x, " + ", show dx, "ε"]
```
where here I have chosen to place $\varepsilon$ *after* the dual term instead,
otherwise we may have `ε-1.0` which doesn't look as good as `-1.0ε`.

**Now we can work on implementing dual number math!** The best place to start
is by making `Dual` an instance of `Num`, which means we have to implement
the following:
```haskell
(+)         :: (Num a) => Dual a  -> Dual a -> Dual a
(-)         :: (Num a) => Dual a  -> Dual a -> Dual a
(*)         :: (Num a) => Dual a  -> Dual a -> Dual a
abs         :: (Num a) => Dual a  -> Dual a
signum      :: (Num a) => Dual a  -> Dual a
fromInteger :: (Num a) => Integer -> Dual a
```
or, instead of `(-)`, you can implement `negate`, but we're not going to do that
here.

Let's start with `abs`. If we replace $f$ in our dual extension with
$\mathrm{abs}$, we get $\mathrm{abs}\!\left(a+\varepsilon b\right)=
\mathrm{abs}\!\left(a\right)+\varepsilon b\mathrm{abs}'\!\left(a\right)$, but
the derivative of $\mathrm{abs}$ is simply [the sign function][5] already
implemented for us as `signum`. This means `abs` can be written as
```haskell
-- abs(a + εb) = abs(a) + εb*sgn(a)
abs (Dual x dx) =
  Dual (abs x) (dx * signum x)
```

This same process can be applied for all the functions of one variable. For
`signum`, the derivative is *just* `0` (except at 0 which we're conveniently
ignoring). As a result, we don't even need to multiply by `dx` and can just
write
```haskell
-- sgn(a + εb) = sgn(a) (+ 0ε)
signum (Dual x dx) =
  Dual (signum x) 0
```

For `(+)`, `(-)`, and `(*)` we can just implement the sum and product rules
from differentiation respectively. Similar to complex numbers, addition and
subtraction just applies to the real parts and dual parts separately:
```haskell
-- (a + εb) + (c + εd) = (a + c) + ε(b + d)
(Dual x dx) + (Dual y dy) =
  Dual (x + y) (dx + dy)

-- (a + εb) - (c + εd) = (a - c) + ε(b - d)
(Dual x dx) - (Dual y dy) =
  Dual (x - y) (dx - dy)
```
Multiplying out the parts for both dual numbers

$$\left(a+\varepsilon b\right)\left(c+\varepsilon d\right)=ac+\varepsilon ad+
\varepsilon bc+\varepsilon^2 bc=ac+\varepsilon\left(ad+bc\right)$$

gives us the product rule as the dual part, which we can implement:
```haskell
-- (a + εb) * (c + εd) = a * c + ε(a * d + b * c)
(Dual x dx) * (Dual y dy) =
  Dual (x * y) (x * dy + dx * y)
```

Finally, we can assume any use of the `fromInteger` function produces a
*constant* - a dual number with a dual part of 0:
```haskell
fromInteger x =
  Dual (fromInteger x) 0
```

With all of these added to our instance of `Num`, we should be able to use
any functions which require a `Num` type, such as the following:
```haskell
sum $ scanr (*) (Dual 3 1) [2, (-1), 3]
```
which gives us the result of $-15-5\varepsilon$. That's a value of $-15$ and
a gradient of $-5$.

&nbsp;

The next class to implement is the `Fractional` class, which requires
```haskell
(/)          :: a        -> a -> a
fromRational :: Rational -> a
```
(or `recip` instead of `(/)`). For division, we can use [Wikipedia's entry][6]
on the division of two dual numbers

$$\dfrac{a+\varepsilon b}{c+\varepsilon d}=\dfrac{a}{c}+\varepsilon\cdot
\dfrac{bc-ad}{c^2}$$

while keeping in mind that we have to use `^` or `*` instead of `**` as we can
only guarantee that $c$ will be a `Fractional` type and not a `Floating` type:
```haskell
(Dual x dx) / (Dual y dy) =
  Dual (x / y) ((dx * y - x * dy) / (y * y))
```
After that, `fromRational` is the same as `fromInteger` but using `fromRational`
instead:
```haskell
fromRational x =
  Dual (fromRational x) 0
```

&nbsp;

The final class we're going to implement for our dual numbers is the `Floating`
class, which requires *quite a lot*:
```haskell
pi    :: Dual a
exp   :: Dual a -> Dual a
log   :: Dual a -> Dual a
sin   :: Dual a -> Dual a
cos   :: Dual a -> Dual a
asin  :: Dual a -> Dual a
acos  :: Dual a -> Dual a
atan  :: Dual a -> Dual a
sinh  :: Dual a -> Dual a
cosh  :: Dual a -> Dual a
asinh :: Dual a -> Dual a
acosh :: Dual a -> Dual a
atanh :: Dual a -> Dual a
```
Once these are all defined, Haskell will automatically generate additional
functions like `(**)`, `sqrt`, and `logBase`.

`pi` is easy, we can just create the value for it:
```haskell
pi =
  Dual pi 0
```
After that, all functions take a single value, and we can just apply the same
rule as we did for `abs` earlier:

$$f\!\left(a+\varepsilon b\right)=f\!\left(a\right)+\varepsilon bf'\!\left(
a\right)$$

This will be a lot of filling in values, so here it is for every function:

$\mathrm{exp}'\!\left(x\right)=\mathrm{exp}\!\left(x\right)$
```haskell
exp (Dual x dx) =
  let x' = exp x in Dual x' (dx * x')
```

$\mathrm{log}'\!\left(x\right)=\dfrac{1}{x}$
```haskell
log (Dual x dx) =
  Dual (log x) (dx / x)
```

$\mathrm{sin}'\!\left(x\right)=\mathrm{cos}\!\left(x\right)$
```haskell
sin (Dual x dx) =
  Dual (sin x) (dx * cos x)
```

$\mathrm{cos}'\!\left(x\right)=-\mathrm{sin}\!\left(x\right)$
```haskell
cos (Dual x dx) =
  Dual (cos x) (-dx * sin x)
```

$\mathrm{asin}'\!\left(x\right)=\dfrac{1}{\sqrt{1-x^2}}$
```haskell
asin (Dual x dx) =
  Dual (asin x) (dx / sqrt (1 - x * x))
```

$\mathrm{acos}'\!\left(x\right)=-\dfrac{1}{\sqrt{1-x^2}}$
```haskell
acos (Dual x dx) =
  Dual (acos x) (-dx / sqrt (1 - x * x))
```

$\mathrm{atan}'\!\left(x\right)=\dfrac{1}{x^2+1}$
```haskell
atan (Dual x dx) =
  Dual (atan x) (dx / (1 + x * x))
```

$\mathrm{sinh}'\!\left(x\right)=\mathrm{cosh}\!\left(x\right)$
```haskell
sinh (Dual x dx) =
  Dual (sinh x) (dx * cosh x)
```

$\mathrm{cosh}'\!\left(x\right)=\mathrm{sinh}\!\left(x\right)$
```haskell
cosh (Dual x dx) =
  Dual (cosh x) (dx * sinh x)
```

$\mathrm{asinh}'\!\left(x\right)=\dfrac{1}{\sqrt{x^2+1}}$
```haskell
asinh (Dual x dx) =
  Dual (asinh x) (dx / sqrt (1 + x * x))
```

$\mathrm{acosh}'\!\left(x\right)=\dfrac{1}{\sqrt{x-1}\sqrt{x+1}}$
```haskell
acosh (Dual x dx) =
  Dual (acosh x) (dx / (sqrt (x - 1) * sqrt (x + 1)))
```

$\mathrm{atanh}'\!\left(x\right)=\dfrac{1}{1-x^2}$
```haskell
atanh (Dual x dx) =
  Dual (atanh x) (dx / (1 - x * x))
```

&nbsp;

Feel free to add anything to the implementation! But this should be enough to
allow strong usage of dual numbers for automatic differentiation.

&nbsp;

### Using the Dual Numbers
To test out using the dual numbers for automatic differentiation, let's
implement [Newton's method for finding roots][7]:

$$x_{n+1}=x_n-\dfrac{f\!\left(x_n\right)}{f'\!\left(x_n\right)}$$

In order to do this, we need a higher-order function which takes a function and
a starting point. The type for this should be:
```haskell
newtonRoot :: (Fractional a)
           => (Dual a -> Dual a) -- The function to find a root for.
           -> a                  -- The starting point.
           -> a                  -- The numeric solution.
```
`Fractional` is chosen as a class constraint on `a` because we need to use the
`(/)` operator, which all `Fractional` types will have defined.

It's easiest to start by defining a single step. One step will take our
current `x` value, make it a dual number with a dual part of `1` (so we can
calculate the gradient), extract the real and dual parts, then update `x` by
subtracting the division.

Luckily pattern matching in a `let` statement makes that very easy to do, and
our update step will have the form:
```haskell
\x -> let (Dual x' dx) = f (Dual x 1) in x - (x' / dx)
```

To repeat this infinitely many times, we just use the `iterate` function,
resulting in our function being:
```haskell
newtonRoot f x0 =
  iterate (\x -> let (Dual x' dx) = f (Dual x 1) in x - (x' / dx)) x0
```

&nbsp;

Applying this to a function is simple, let's define

$$f\!\left(x\right)=\cos\!\left(3x\right)+x^2+x$$

as our function, with a starting point of $x=-1$. We also need to define how
many steps our method should take. $n=30$ should be fine:
```haskell
myFunc :: (Floating a)
       => a
       -> a
myFunc x =
  cos (3 * x) + x^2 + x
```
we can test this method by running `(newtonRoot myFunc (-1)) !! 30`, which gives
us a result of `-0.440587...`.

This matches one of the expected roots. Try starting with $x=1$ instead, but
this time you may observe something different. Try using
`take 30 $ newtonRoot myFunc 1` to see the change in values over each step and
see what's happening.

&nbsp;

Experiment! Try different functions, or other things like implementing gradient
descent for finding minima. So many choices!

&nbsp;

## More to Come
Eventually I'll write the next post on **reverse-mode automatic
differentiation**, but for now I hope this is enough to try something out and
have fun. Let me know if there's any mistakes or errors!

&nbsp;

### The Module
Here's the entire code for the dual numbers module, feel free to adapt as
needed:
```haskell
{-# LANGUAGE InstanceSigs #-}
module Numeric.Differentiable.Dual (
    Dual(..)
  ) where

-- A dual number just consists of a real part and a dual part.
data Dual a =
  Dual a a

instance Num a => Num (Dual a) where
  (+) :: Dual a -> Dual a -> Dual a
  (Dual x dx) + (Dual y dy) =
    Dual (x + y) (dx + dy)

  (-) :: Dual a -> Dual a -> Dual a
  (Dual x dx) - (Dual y dy) =
    Dual (x - y) (dx - dy)

  (*) :: Dual a -> Dual a -> Dual a
  (Dual x dx) * (Dual y dy) =
    Dual (x * y) (x * dy + dx * y)

  abs :: Dual a -> Dual a
  abs (Dual x dx) =
    Dual (abs x) (dx * signum x)

  signum :: Dual a -> Dual a
  signum (Dual x _) =
    Dual (signum x) 0

  fromInteger :: Integer -> Dual a
  fromInteger x =
    Dual (fromInteger x) 0

instance Fractional a => Fractional (Dual a) where
  (/) :: Dual a -> Dual a -> Dual a
  (Dual x dx) / (Dual y dy) =
    Dual (x / y) ((dx * y - x * dy) / (y * y))

  fromRational :: Rational -> Dual a
  fromRational x =
    Dual (fromRational x) 0

instance Floating a => Floating (Dual a) where
  pi :: Dual a
  pi =
    Dual pi 0

  exp :: Dual a -> Dual a
  exp (Dual x dx) =
    let x' = exp x in Dual x' (dx * x')

  log :: Dual a -> Dual a
  log (Dual x dx) =
    Dual (log x) (dx / x)

  sin :: Dual a -> Dual a
  sin (Dual x dx) =
    Dual (sin x) (dx * cos x)

  cos :: Dual a -> Dual a
  cos (Dual x dx) =
    Dual (cos x) (-dx * sin x)

  asin :: Dual a -> Dual a
  asin (Dual x dx) =
    Dual (asin x) (dx / sqrt (1 - x * x))

  acos :: Dual a -> Dual a
  acos (Dual x dx) =
    Dual (acos x) (-dx / sqrt (1 - x * x))

  atan :: Dual a -> Dual a
  atan (Dual x dx) =
    Dual (atan x) (dx / (1 + x * x))

  sinh :: Dual a -> Dual a
  sinh (Dual x dx) =
    Dual (sinh x) (dx * cosh x)

  cosh :: Dual a -> Dual a
  cosh (Dual x dx) =
    Dual (cosh x) (dx * sinh x)

  asinh :: Dual a -> Dual a
  asinh (Dual x dx) =
    Dual (asinh x) (dx / sqrt (1 + x * x))

  acosh :: Dual a -> Dual a
  acosh (Dual x dx) =
    Dual (acosh x) (dx / (sqrt (x - 1) * sqrt (x + 1)))

  atanh :: Dual a -> Dual a
  atanh (Dual x dx) =
    Dual (atanh x) (dx / (1 - x * x))

instance Show a => Show (Dual a) where
  show :: Dual a -> String
  show (Dual x dx) =
    concat [show x, " + ", show dx, "ε"]
```

[1]: https://en.wikipedia.org/wiki/Complex_number
[2]: https://en.wikipedia.org/wiki/Ring_(mathematics)
[3]: https://en.wikipedia.org/wiki/Stochastic_gradient_descent
[4]: https://en.wikipedia.org/wiki/Limited-memory_BFGS
[5]: https://en.wikipedia.org/wiki/Sign_function
[6]: https://en.wikipedia.org/wiki/Dual_number#Division
[7]: https://en.wikipedia.org/wiki/Newton%27s_method
