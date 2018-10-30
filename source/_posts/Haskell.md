---
title: Haskell
date: 2018-10-30 10:57:01
tags:
---

![Haskell](http://www.cis.upenn.edu/~cis194/spring13/images/haskell-logo-small.png)

> Haskell is a *lazy, functional* programming language created in the late 1980’s by a committee of academics. 

- **functional**: functions are *first-class*, centered around *evaluating expressions* rather than *executing instructions*.
- **pure**: *immutable*, no *side effects*, calling the same function with the same arguments results in the same output every time.
- **lazy**: *call-by-need*, expressions are *not evaluated until their results are actually needed*.Alone with *Memoization*
- **statically typed**: as Swift.Every Haskell expression has a type, and types are all checked at *compile-time*.
- **abstraction**: *parametric polymorphism*, *higher-order functions*
- **wholemeal programming**: 

> “Functional languages excel at wholemeal programming, a term coined by Geraint Jones. Wholemeal programming means to think big: work with an entire list, rather than a sequence of elements; develop a solution space, rather than an individual solution; imagine a graph, rather than a single path. The wholemeal approach often offers new insights or provides new perspectives on a given problem. It is nicely complemented by the idea of projective programming: first solve a more general problem, then extract the interesting bits and pieces by transforming the general program into more specialised ones.” --- Ralf Hinze
> 
> ```
> int acc = 0;
for ( int i = 0; i < lst.length; i++ ) {
  acc = acc + 3 * lst[i];
}
> ```
> 
> to
> 
> ```
> sum (map (3*) lst)
> ```

## Lazy evaluation

#### Strict evaluation

```
f (release_monkeys(), increment_counter())
```

If the releasing of monkeys and incrementing of the counter could independently happen, or not, in either order, depending on whether f happens to use their results, it would be extremely confusing. When such *“side effects”* are allowed, strict evaluation is really what you want.

#### Side effects and purity

By “side effect” we mean anything that *causes evaluation of an expression to interact with something outside itself*.The root issue is that such outside interactions are time-sensitive.

- No global variable
- No output to screen
- No reading from a file or the network

WTF......

The solution is IO monad.

#### Consequences

- Purity
- Tricky space usage

```
-- Standard library function foldl, provided for reference
foldl :: (b -> a -> b) -> b -> [a] -> b
foldl _ z []     = z
foldl f z (x:xs) = foldl f (f z x) xs

foldl (+) 0 [1,2,3]
= foldl (+) (0+1) [2,3]
= foldl (+) ((0+1)+2) [3]
= foldl (+) (((0+1)+2)+3) []
= (((0+1)+2)+3)
= ((1+2)+3)
= (3+3)
= 6

foldl' (+) 0 [1,2,3]
= foldl' (+) (0+1) [2,3]
= foldl' (+) 1 [2,3]
= foldl' (+) (1+2) [3]
= foldl' (+) 3 [3]
= foldl' (+) (3+3) []
= foldl' (+) 6 []
= 6
```
Consider if there is a very long list, it leads to stack overflow.

- Short-circuiting operators

```
(&&) :: Bool -> Bool -> Bool
True  && x = x
False && _ = False
```

- Infinite data structures
- Pipelining/wholemeal programming
	- due to laziness, each stage of the pipeline can operate in lockstep, only generating each bit of the result as it is demanded by the next stage in the pipeline.
- Thunk.
- Confilct with parallel computing.

## Functors

- map :: (a -> b) -> [a] -> [b]
- treeMap :: (a -> b) -> Tree a -> Tree b
- maybeMap :: (a -> b) -> Maybe a -> Maybe b

why not

``thingMap :: (a -> b) -> f a -> f b``

Since f is type variable, we can make a type class, which is traditionally called **Functor**:

```
class Functor f where
  fmap :: (a -> b) -> f a -> f b
  
(<$>) :: Functor f => (a -> b) -> f a -> f b
(<$>) = fmap
  
instance Functor [] where
  fmap _ []     = []
  fmap f (x:xs) = f x : fmap f xs
  
instance Functor Maybe where
  fmap _ Nothing  = Nothing
  fmap h (Just a) = Just (h a)
```

## Applicative functors

```
type Name = String

data Employee = Employee { name    :: Name
                         , phone   :: String }
```

So Employee is 

``Employee :: Name -> String -> Employee``

However, sometimes we want this:

``(Name -> String -> Employee) -> Maybe Name -> Maybe String -> Maybe Employee``

and why not provide this as well:

``Name -> String -> Employee) -> [Name] -> [String] -> [Employee]``

Seems wec can use `fmap` to do this job, 

```
class Functor f where
  fmap2 :: (a -> b -> c) -> f a -> f b -> f c
  
fmap2 :: Functor f => (a -> b -> c) -> (f a -> f b -> f c)
fmap2 h fa fb = ???

h :: a -> (b -> c)
fmap h :: fa -> f(b -> c)
fmap h fa :: f(b -> c)
```

The reason why we cannot implement fmap2 is `fmap h fa :: f (b -> c)`, and we cannot handle `f (b -> c)` with `f b`

So it's time to introduce *Applicative*

```
class Functor f => Applicative f where
  pure  :: a -> f a
  (<*>) :: f (a -> b) -> f a -> f b
```

```
liftA2 :: Applicative f => (a -> b -> c) -> f a -> f b -> f c
liftA2 h fa fb = (h `fmap` fa) <*> fb

liftA3 :: Applicative f => (a -> b -> c -> d) -> f a -> f b -> f c -> f d
liftA3 h fa fb fc = ((h <$> fa) <*> fb) <*> fc
```

> Law for `Applicative`:
> 
> ```
> f `fmap` x === pure f <*> x
> ```

## Monads

From *functor* and *applicative*, we can see computations with a fixed structure.

However, sometimes we want to be able to decide what to do based on some intermediate results.

```
newtype Parser a = Parser { runParser :: String -> Maybe (a, String) }
```

Suppose we are trying to parse a file containing a sequence of numbers, like this:
> 4 78 19 3 44 3 1 7 5 2 3 2

So the example above could be broken up into groups like this:
> 78 19 3 44   -- first group  
> 1 7 5        -- second group  
> 3 2          -- third group

So what we need is some thing like this:
``parseFile :: Parser [[Int]]`` 

*Applicative* gives us no way to decide what to do next based on previous results: we must decide in advance what parsing operations we are going to run, before we see the results.

```
class Monad m where
  return :: a -> m a

  (>>=) :: m a -> (a -> m b) -> m b

  (>>)  :: m a -> m b -> m b
  m1 >> m2 = m1 >>= \_ -> m2
```

`m` refer to *monads* and `m a` refer to *mobits*. A mobit of type `m` a represents a computation which results in a value (or several values, or no values) of type `a`.

A function which will choose the next computation to run based on the result(s) of the first computation.

So all (>>=) really does is put together two mobits to produce a larger one, which first runs one and then the other, returning the result of the second one. 


## Resource

[CIS 194](http://www.seas.upenn.edu/~cis194/spring13/lectures.html)