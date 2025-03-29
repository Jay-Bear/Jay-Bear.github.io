# The Trifuncta: Functors, Applicative Functors, and Monads

***This version is an early draft, and will be updated.***

Oh *boy* do these confuse people. There's a plethora of memes about *monads* in
particular as well as that classic phrase:

<center>A monad is just a monoid in the category of endofunctors!</center>

But realistically these topics aren't too difficult to understand. I believe
the main issues stem from how these topics are often taught (especially in
relation to imperative programming) and the names being confusing.

This serves as a way of going through – in *some* detail – my approach to
teaching these topics. In particular, this way of teaching them is
**sequential**. Each builds on the last through the same logic.

### A Foreword on Style

I'm going to be writing most blocks of code in my style. **This includes
already existing code**. Deal with it.

I've also taken the liberty of writing *some* bits of code in a non-optimal
way. This is intended to mimic the observed coding style of students at the
stage they'd roughly be at when learning about the "trifuncta". If you'd like
to complain about this, you can email me, but just know that I'll completely
ignore it.

&nbsp;

## Parameterized Data Types

*– sometimes called 'polymorphic data types' –*

Before we get into the *bread and butter* of decent Haskell code, we first need
to acknowledge the existence of **parameterized data types**. You *should* be
familiar with these as simply a data type that depends on some type
parameter (or *type variable* as some people refer to them). It's a type
that's constructed by giving it different types.

First, an example of a type that *isn't* parameterized; `Bool`:
```haskell
data Bool =
    False
  | True
```
`Bool` isn't parameterized. It cannot produce a new type from taking another
type. It cannot contain values of some arbitrary unspecified type.

Then, an example that may be familiar to you is `Maybe`:
```haskell
data Maybe a =
    Just a
  | Nothing
```
`Maybe` *is* parameterized precisely because it *can* produce a new type from
taking another type. We can give it `Int` to produce `Maybe Int`, `String` to
produce `Maybe String`, or even itself to make `Maybe (Maybe a)` for some type
`a`.

*But then* the types `Maybe Int` or `Maybe String` *aren't* parameterized
because the one parameter for `Maybe` has been consumed. Think of parameterized
data types like functions; which they kind of *are*. The "type" of a type is
called its **kind**, where a single concrete (monomorphic) type has kind `*`.
`Bool` has kind `*`, `String` has kind `*`, `Int` has kind – *you guessed it* –
`*`.

However `Maybe` has kind `* -> *`. It maps from a type to a new type. It
*constructs* a new type from a given type – hence the name "type constructor".
Applying `Maybe` to a concrete type gives me another concrete type;
`Maybe ___`.

These parameterized types are structures. They contain values of another type
and give them context. Lists, maps, sets, *etc.*. Now with *that* out of the
way...

&nbsp;

## Functors

Being able to contain values and types is nice, but it sounds *tedious*
because we now have to define functions for these. We can use `reverse` to
reverse a list, but if the list is in a `Maybe` type then do we have to define
`maybeReverse`? Do we have to do this for *every* function? **For every
parameterized type?!**

Not quite; for every parameterized type we care about *yeah*, but not for
every function. Consider what we actually need here, specifically in the case
of `Maybe`:

We want to convert any function `a -> b` to work on `Maybe` values,
specifically `Maybe a -> Maybe b`. This is referred to as *lifting* the
function to `Maybe`. To do this, we can construct a *new* function that uses
pattern matching to "extract" the value from `Just`, apply it to the original
function, then wrap it back in `Just`, *or* return `Nothing` if `Nothing`[^1]:
```haskell
-- Lift a function to work on `Maybe` type values.
liftMaybe :: (a -> b)           -- Function.
          -> Maybe a -> Maybe b -- Lifted function.
liftMaybe f =
  fx -- Return new function.
  where
    fx (Just x) = Just $ f x -- Extract value, apply function, re-wrap.
    fx Nothing  = Nothing    -- Do nothing.
```
Now we can just do `liftMaybe reverse` and we get a function that reverses
lists, but from within a `Maybe` value!
```haskell
liftMaybe reverse $ Just [1, 2, 3] -- Just [3, 2, 1]
liftMaybe reverse $ Nothing        -- Nothing
```

What if we now want to do this for lists? We'd like to define a function\
`liftList :: (a -> b) -> [a] -> [b]`\
It should take a function and produce a
new function that applies the original to ea– *oh wait* that's called `map`...
```haskell
-- Lift a function to work on list values.
liftList :: (a -> b)   -- Function.
         -> [a] -> [b] -- Lifted function.
liftList =
  map
```

The observant few of you may recognize a pattern here. Each of these lifting
functions has a similar type signature. For a given parameterized type `m`, the
lifting function has type\
`(a -> b) -> m a -> m b`\
so we could just define a **type class of liftable types**; if `m` is
"liftable", then it should have a function that lifts functions to that type.

**This exists in the form of** `Functor`**!**\
If `m` is "liftable", it is a functor. If `m` is a member of the `Functor` type
class, then functions can be lifted to it by using `fmap`, as type defined in
the type class definition:
```haskell
class Functor m where
  -- Lifts a function to act on type 'm'.
  fmap :: (a -> b)
       -> m a -> m b
```
Then both `liftMaybe` and `liftList` can be replaced by making `Maybe` and `[]`
members of `Functor`;
```haskell
instance Functor Maybe where
  fmap f (Just x) = Just $ f x
  fmap f Nothing  = Nothing

instance Functor ([]) where
  fmap = map
```
where `fmap` can be used to lift functions *(note: these instances are already
defined)*.

With any type that's a member of `Functor` you can use `fmap` to lift a
function to it. `Map` from `Data.Map` is a functor:
```haskell
k :: Map String Int
k =
  fromList [("hello", 1), ("world", 2)]

fmap (+ 1) $ k -- [("hello", 2), ("world", 3)]
```

Usually the usage of `fmap` is that we immediately apply it to a value. Due
to currying, we just append the argument (`fmap f x`). Alternatively, one can
use the operator form of `fmap`: `(<$>)`...
```haskell
(+ 1) <$> [1, 2, 3]          -- [2, 3, 4]
length <$> Just [1, 3, 3, 7] -- Just 4
```

&nbsp;

## Applicative Functors

Thanks to `Functor`, we have a way of lifting functions from `a -> b` to
`m a -> m b` via `fmap`, but what if the function we're trying to lift is
*contained* in a functor? We'd like something to do\
`m (a -> b) -> m a -> m b`.

This is a real possibility. Let's say we are making a calculator and have a
function lookup table represented by a map `Map String (Double -> Double)`; a
mapping between a string representing the name of a math function and the
function itself, such as `"cos"` to `cos`. The function in `Data.Map` to
obtain a value from this is\
`lookup :: Ord k => k -> Map k a -> Maybe a`\
Given our lookup table, using this function would give us a value of type
`Maybe (Double -> Double)`; a function contained in a functor.

First of all, we can't just remove the `Maybe`. Okay, technically we *can* but
it's probably not a good idea. We'll have to handle the
`Just (Double -> Double)` and `Nothing` cases separately and that gets messy.
Given this problem, it might be ideal to keep propagating `Nothing` until we
get our answer, then if we get a `Nothing` value we can just say there's an
error, otherwise we produce the value.

Similarly to defining `fmap`, we can define a lifting operation on `Maybe`
values, but in the specific case where those functions are contained in the
functor itself. Here's how we might logically do that:
```haskell
liftMaybe' :: Maybe (a -> b)     -- Function in 'Maybe'.
           -> Maybe a -> Maybe b -- Lifted function.
liftMaybe' (Just f) = fmap f        -- Function exists; lift it.
liftMaybe' Nothing  = const Nothing -- Otherwise only return 'Nothing'.
```
Now before moving on to define the same for `liftList'`, we can just jump
ahead and acknowledge that a **type class** is probably best here. That type
class is called `Applicative` and it represents applicative functors:
```haskell
class Functor m => Applicative m where
  -- Lifts a value to type 'm'.
  pure :: a
       -> m a
  -- Lifts the function contained in type 'm' to act on type 'm' itself.
  (<*>) :: m (a -> b)
        -> m a -> m b
```
You *may* notice a little extra thing here; `pure :: a -> m a`. From a
practical point of view, `pure` exists to map a value to one in that type.
Here's `pure` for `Maybe` and `[]`:
```haskell
-- Maybe:
pure = Just
-- List:
pure = (: [])
```
In both of these cases, it just puts the value into the structure in the
simplest way possible. With `Maybe`, it simply uses the `Just` constructor on
it. With lists, it creates a singleton list of that value. This is useful for
when we may have on argument contained in an applicative, but another that
isn't. There needs to be consistency, so we'd use `pure` to contain that value
in the applicative.

This can also be used to lift functions of multiple arguments. Consider the
operator `(+) :: Num a => a -> a -> a`. What if we wanted to make this work on 
numbers contained in applicative functors instead (`[]`, `Maybe`, *etc.*)? We
can chain `fmap` and `<*>`:
```haskell
-- Apply 'fmap' to '(+)':
fmap (+) :: (Functor m, Num a) => m a -> m (a -> a)
-- Compose with '(<*>)' to convert 'm (a -> a)' to 'm a -> m a':
(<*>) . fmap (+) :: (Applicative m, Num a) => m a -> m a -> m a
```
The generic version of this function is `liftA2`, and it has type\
`liftA2 :: Applicative m => (a -> b -> c) -> m a -> m b -> m c`\
Alternatively, for some function with multiple arguments, you can chain
`<$>` and `<*>` if you have arguments contained in the applicative functor:
```haskell
-- Just [(1, 3, 5), (2, 4, 6)]:
zip3 <$> (Just [1, 2]) <*> (Just [3, 4]) <*> (Just [5, 6])

-- [[(1, 3, 6)], [], [(2, 3, 6)], []]:
zip3 <$> [[1], [2]] <*> (pure [3, 4]) <*> [[6], []]
```

&nbsp;

## Monads

We can now convert `a -> b` to `m a -> m b` **and** we can now convert
`m (a -> b)` to `m a -> m b`, but there's *one more important case*:

**Converting** `a -> m b` **to** `m a -> m b`**.**

Can we do this with the tools we have? Let's start by trying both functions:
```haskell
-- Original function:
f :: Applicative m => a -> m b
-- Trying 'fmap':
fmap f :: Applicative m => m a -> m (m b)
-- Trying '(<*>)':
(<*>) f :: Applicative m => ... -- That won't work.
```
The only option here to get anywhere is `fmap`, but that leaves us with a
*gross* doubled-up applicative as an output. It's giving us `m (m b)`, but we
want `m b`.

Going back to examples, let's think about what this means for `Maybe`. A value
of type `Maybe (Maybe a)` can be only one of three different values (given a
value `x :: a`):

 - `Just (Just x)`
 - `Just Nothing`
 - `Nothing`

Similarly for a list, a value of type `[[a]]` is just a list containing lists.

At this point it should seem fairly obvious we want a function that "squashes"
these. A function `m (m a) -> m a`. Let's call this function `join`, and
define it for both `Maybe` and lists:
```haskell
-- 'Just (Just x) -> Just x', anything else to 'Nothing'.
joinMaybe (Just x) = x       -- Covers 'Just (Just x)' and 'Just Nothing'.
joinMaybe Nothing  = Nothing -- Covers 'Nothing'.

-- 'join' on lists is just concatenation.
joinList =
  concat
```
Now, with `join` defined, we have a function that performs this lift properly:
```haskell
-- Original function:
f :: Applicative m => a -> m b
-- Lifted ('Maybe'):
joinMaybe . fmap f :: Maybe a -> Maybe b
-- Lifted (list):
joinList . fmap f :: [a] -> [b]
```
...but let's just make `join` part of a type class instead.

**Unfortunately** Haskell decided on something different, so we have to take
a *small detour*. By defining `join` on a type, we have a **monad**, but types
are made instances of `Monad` in Haskell by defining a *different* operator
called "bind"; `(>>=)`. Using `join`, bind can be defined as\
`x >>= f = join $ fmap f x`\
which tells us that bind lifts the function `f`, applies it to `x`, then uses
`join` to squash the output.

*However*, often in Haskell we just define `>>=` directly as an operator which
takes a value in a monadic type and applies it to a function that *produces*
a monadic type. This is what's mainly expected in the definition of `Monad`:
```haskell
class Applicative m => Monad m where
  -- Lifts a value to type 'm'.
  return :: a
         -> m a
  -- Applies a lifted value to a lifting function.
  (>>=) :: m a
        -> (a -> m b)
        -> m b
```
Where `return` here basically exists for the same reason `pure` does; it
maps a value into a monadic one.

The primary reason for Haskell using bind instead of `join` is that bind
naturally lends itself to convenient chaining. For example:
```haskell
main :: IO ()
main =
  getLine >>= readFile >>= putStrLn
```
can be thought of as first converting\
`readFile :: String -> IO String` to `IO String -> IO String`\
then applying this to `getLine :: IO String` to produce an `IO String`. Then it
converts\
`putStrLn :: String -> IO ()` to `IO String -> IO ()`\
and applies *this* to the `IO String` produced before.\
This chain thus reads the line from the terminal, reads the file with the path
given in that line, then prints the contents of the file.

### Don't Monads Make Applicative Functors Obsolete?

Think about it: if I have a function `m (a -> b)` and I want to convert it to
`m a -> m b`, I could just use the monad functions:
```haskell
-- Original function:
mf :: Monad m => m (a -> b)
-- Convert to 'm a -> m b':
\mx -> mf >>= \f -> mx >>= \x -> return (f x) :: Monad m => m a -> m b
-- Rewritten in 'do' notation for convenience:
\mx -> do
  f <- mf
  x <- mx
  return (f x)
```
So why do applicative functors exist?

The simple answer is that every monad is an applicative functor, but not
every applicative functor is a monad. There are some types that can be defined
as applicative functors but **not** monads. For example, the `ZipList` type
from `Control.Applicative` has a `(<*>)` implementation that applies each
function contained in the list zip-wise with the items in the other list,
however it **cannot** be a monad as any conceivable implementation will violate
some monad law.

### Editor Note on Join and Bind

In reality I find it easier to define `join` for a type then either just do\
`x >>= f = join $ fmap f x`\
when defining the instance, or inferring what it *would* be directly. This
might be the case for you too, so give it a go when you next define a monad.

Also it makes the monad laws easier:

 - `join` is a [natural transformation][1].
 - `join . fmap join` = `join . join`.
 - `join . fmap pure` = `join . pure` = `id`.

If I didn't make it clear, I believe Haskell picking `(>>=)` over `join` for
`Monad` was a mistake.

[^1]: Yes I'm *well aware* you can do `liftMaybe f (Just x)` or
      `liftMaybe f Nothing`, but as stated in the foreword I'm mimicking the
      observed style of students, and from what I've observed defining
      higher-order functions in such a way to be partially applied is rare.

[1]: https://en.wikipedia.org/wiki/Natural_transformation
