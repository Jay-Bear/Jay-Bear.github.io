# Notes on Haskell's Happy LALR(1) Parser

The main intention for this is to help students understand a few potentially
confusing things about LALR(1) parsing, in particular the one used by Happy
(which is basically Haskell's YACC). This serves as more of a collection of
notes – *mostly those I've written in Discord messages* – but it can be
anything else.

## Braces in Production Rules are just Haskell

For anyone who might be confused about how the braces operate in Happy
production rules, for example,\
`let var '=' Exp in Exp {Let $2 $4 $6}`,\
just note that for both Alex and Happy, pretty much anything in the braces is
Haskell and is almost entirely inserted into the generated files verbatim. *Of
course* you may then argue that `$2` is not valid Haskell – *and you are
correct*, I'm hiding a lot of extra technical detail here, but this `$2` is
most likely going to be converted to `happy_var_2` in the generated file, which
is an argument passed in place of the `$2`:\
`Let happy_var_2 happy_var_4 happy_var_6`

This `$n` format just tells Happy to replace whatever is in that position with
whatever the `n`th term produces as part of a reduction. For example, with a
simple grammar that encodes Peano naturals:
```
Nat :: {Int}
  : Succ Nat {1 + $2}
  | Zero     {0     }
```
the `Zero` expansion produces literally a `0 :: Int` in the Happy-generated
function. The `Succ Nat` expansion takes the value produced by reducing `Nat`
and adds `1` to it, because `Nat` appears in the 2nd position (`$2`). When the
token sequence `Succ Succ Nat` is parsed, this performs equivalent to
`1 + 1 + 0`, which produces `2`.

If you look in the generated file, you'll find the term encased
in some other generated functions (most likely called `happyReduction_n` where
`n` is some integer). For me, that generates:
```haskell
happyReduction_2 _ =
  HappyAbsSyn4 (0)
```
where `HappyAbsSyn4 :: HappyAbsSyn` is a generated data constructor. In most
cases where we use such a parser, we wish to generate an abstract syntax tree,
and often this is done by defining custom data types. In the case of the `Let`
statement at the beginning of this section, that `Let` is a data constructor
for a certain type which is defined in a separate bit of Haskell (usually in
the postamble or in a separate imported file).

### Getting Values from Tokens

When you state your tokens in Happy, you may use `$$` for certain values
contained in the token type, such as a string or integer. This is so Happy can
use these values when you do `$n` later on. This works equivalently to
pattern matching – for example you may be familiar with
```haskell
fromJust :: Maybe a
         -> a
-- Error case for `Nothing` here.
fromJust (Just x) =
  x
```
which you can view as *'extracting'* the value contained in the value of type
`Maybe a`. This is how the `$$` works in your token statements. If I have a
token data constructor `TokenVar :: String -> Token` from the data type
definition
```haskell
data Token =
  ...
  | TokenVar String
  ...
```
then the token statement `var {TokenVar $$}` performs this pattern matching by
producing a reduction that converts `var` to the string contained in the
token.
