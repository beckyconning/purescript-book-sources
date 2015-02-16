# Recursion, Maps And Folds

## Chapter Goals

In this chapter, we will look at how recursive functions can be used to structure algorithms. Recursion is a basic technique used in functional programming, which we will use throughout this book.

We will also cover some standard functions from PureScript's standard libraries. We will see the `map` and `fold` functions, as well as some useful special cases, like `filter` and `concatMap`.

The motivating example for this chapter is a library of functions for working with a virtual filesystem. We will apply the techniques learned in this chapter to write functions which compute properties of the files represented by a model of a filesystem.

## Project Setup

The source code for this chapter is contained in the two files `src/Data/Path.purs` and `src/FileOperations.purs`.

The `Data.Path` module contains a model of a virtual filesystem. You do not need to modify the contents of this module. 

The `FileOperations` module contains functions which use the `Data.Path` API. Solutions to the exercises can be completed in this file.

The project has the following Bower dependencies:

- `purescript-maybe`, which defines the `Maybe` type constructor
- `purescript-arrays`, which defines functions for working with arrays
- `purescript-foldable-traversable`, which defines functions for folding arrays and other data structures

## Introduction

Recursion is an important technique in programming in general, but particularly common in pure functional programming, because, as we will see in this chapter, recursion helps to reduce the mutable state in our programs.

Recursion is closely linked to the _divide and conquer_ strategy: to solve a problem on certain inputs, we can break down the inputs into smaller parts, solve the problem on those parts, and then assemble a solution from the partial solutions.

Let's see some simple examples of recursion in PureScript.

Here is the usual _factorial function_ example:

```haskell
fact :: Number -> Number
fact 0 = 1
fact n = n * fact (n - 1)
```

Here, we can see how the factorial function is computed by reducing the problem to a subproblem - that of computing the factorial of a smaller number. When we reach zero, the answer is immediate.

Here is another common example, which computes the _Fibonnacci function_:

```haskell
fib :: Number -> Number
fib 0 = 1
fib 1 = 1
fib n = fib (n - 1) + fib (n - 2)
```

Again, this problem is solved by considering the solutions to subproblems. In this case, there are two subproblems, corresponding to the expressions `fib (n - 1)` and `fib (n - 2)`. When these two subproblems are solved, we assemble the result by adding the partial results.

## Recursion on Arrays

We are not limited to defining recursive functions over the `Number` type! We will see recursive functions defined over a wide array of data types when we cover _pattern matching_ later in the book, but for now, we will restrict ourselves to numbers and arrays.

Just as we branch based on whether the input is non-zero, in the array case, we will branch based on whether the input is non-empty. Consider this function, which computes the length of an array using recursion:

```haskell
import Data.Array (null)
import Data.Array.Unsafe (tail)

length :: forall a. [a] -> Number
length arr = 
  if null arr
  then 0 
  else 1 + length (tail arr)
```

In this function, we use an `if .. then .. else` expression to branch based on the emptiness of the array. The `null` function returns `true` on an empty array. Empty arrays have length zero, and a non-empty array has a length that is one more than the length of its tail.

This example is obviously a very impractical way to find the length of an array in JavaScript, but should provide enough help to allow you to complete the following exercises.

X> ## Exercises
X> 
X> 1. (Easy) Write a recursive function which returns `true` if and only if its input is an even number.
X> 2. (Medium) Write a recursive function which counts the number of even numbers in an array. _Hint_: use the `head` function from `Data.Array.Unsafe`.

## Maps

The `map` function is an example of a recursive function on arrays. It is used to transform the elements of an array by applying a function to each element in turn. Therefore, it changes the _contents_ of the array, but preserves its _shape_ (i.e. its length).

When we cover _type classes_ later in the book we will see that the `map` function is an example of a more general pattern of shape-preserving functions which transform a class of type constructors called _functors_.

Let's try out the `map` function in `psci`:

```text
$ psci

> :i Data.Array
> map (\n -> n + 1) [1, 2, 3, 4, 5]

[2, 3, 4, 5, 6]
```

Notice how `map` is used: we provide a function which should be "mapped over" the array in the first argument, and the array itself in its second.

## Infix Operators

The `map` function can also be written between the mapping function and the array, by wrapping the function name in backticks:

```text
> (\n -> n + 1) `map` [1, 2, 3, 4, 5]

[2, 3, 4, 5, 6]
```

This syntax is called _infix function application_, and any function can be made infix in this way. It is usually most appropriate for functions with two arguments.

There is an operator which is equivalent to the `map` function when used with arrays, called `<$>`. This operator can be used infix like any other binary operator:

```text
> (\n -> n + 1) <$> [1, 2, 3, 4, 5]

[2, 3, 4, 5, 6]
```

_Note_: the type of `<$>` is actually more general than that of `map`, but for the most part it is safe to use `<$>` in place of `map` if infix application seems more natural.

Let's look at the type of `map`:

```text
> :t map
forall a b. (a -> b) -> [a] -> [b]
```

This type says that we can choose any two types, `a` and `b`, with which to apply the `map` function. `a` is the type of elements in the source array, and `b` is the type of elements in the target array. In particular, there is no reason why `map` has to preserve the type of the array elements. We can use map to transform numbers to strings, for example:

```text
> show <$> [1, 2, 3, 4, 5]

["1","2","3","4","5"]
```

Even though the infix operator `<$>` looks like special syntax, it is in fact a regular PureScript function. It is simply _applied_ using infix syntax. In fact, the function can be used like a regular function by enclosing its name in parentheses. This means that we can used the parenthesized name `(<$>)` in place of `map` on arrays:

```text
> (<$>) show [1, 2, 3, 4, 5]

["1","2","3","4","5"]
```

Defining a new infix operator uses the same notation: define infix operators just like regular functions, enclosing the operator name in parentheses. For example, the `Data.Array` module defines an infix operator `(..)` as a synonym for the `range` function, as follows:

```haskell
(..) :: Number -> Number -> [Number]
(..) = range
```

We can use this operator as follows:

```text
> 1 .. 5

[1, 2, 3, 4, 5]

> show <$> (1 .. 5)
  
["1","2","3","4","5"]
```

_Note_: User-defined infix operators can be a great tool for defining domain-specific languages with a natural syntax. However, used carelessly, they can render code unreadable to beginners, so it is wise to exercise caution when defining any new operators.

In the example above, we parenthesized the expression `1 .. 5`, but this was actually not necessary, because the `Data.Array` module assigns a higher precedence level to the `..` operator than that assigned to the `<$>` operator. PureScript provides a way to assign a precedence level to user-defined operators, using the `infix` keyword:

```text
infix 5 ..
```

This assigns a precedence level of 5 to the `(..)` operator, which is higher than the precedence level of `<$>`, meaning that we do not need to add parentheses:

```text
> show <$> 1 .. 5
  
["1","2","3","4","5"]
```

If we wanted to assign an associativity (left or right) to an infix operator, we could do so with the `infixl` and `infixr` keywords instead.

## Filtering Arrays

The `Data.Array` module provides another function `filter`, which is commonly used together with `map`. It provides the ability to create a new array from an existing array, keeping only those elements which match a predicate function.

For example, suppose we wanted to compute an array of all numbers between 1 and 10 which were even. We could do so as follows:

```text
> :i Data.Array

> filter (\n -> n % 2 == 0) (1 .. 10)
[2,4,6,8,10]
```

X> ## Exercises
X> 
X> 1. (Easy) Use the `map` or `<$>` function to write a function which calculates the squares of an array of numbers.
X> 1. (Easy) Use the `filter` function to write a function which removes the negative numbers from an array of numbers.
X> 1. (Medium) Define an infix synonym `<$?>` for `filter`. Rewrite your answer to the previous question to use your new operator. Experiment with the precedence level and associativity of your operator in `psci`.

## Flattening Arrays

Another standard function on arrays is the `concat` function, defined in `Data.Array`. `concat` flattens an array of arrays into a single array:

```text
> :i Data.Array
> :t concat 

forall a. [[a]] -> [a]

> concat [[1, 2, 3], [4, 5], [6]]

[1, 2, 3, 4, 5, 6]
```

There is a related function called `concatMap` which is like a combination of the `concat` and `map` functions. Where `map` takes a function from values to values (possibly of a different type), `concatMap` takes a function from values to arrays of values. 

Let's see it in action:

```text
> :i Data.Array

> :t concatMap
forall a b. (a -> [b]) -> [a] -> [b]

> concatMap (\n -> [n, n * n]) (1 .. 5)
  
[1,1,2,4,3,9,4,16,5,25]
```

Here, we call `concatMap` with the function `\n -> [n, n * n]` which sends a number to the array of two elements consisting of that number and its square. The result is a list of ten numbers: the numbers from 1 to 5 along with their squares.

Note how `concatMap` concatenates its results. It calls the provided function once for each element of the original array, generating an array for each. Finally, it collapses all of those arrays into a single array, which is its result.

`map`, `filter` and `concatMap` form the basis for a whole range of functions over arrays called "array comprehensions".

## Array Comprehensions

Suppose we wanted to find the factors of a number `n`. One simple way to do this would be by brute force: we could generate all pairs of numbers between 1 and `n`, and try multiplying them together. If the product was `n`, we would have found a pair of factors of `n`.

We can perform this computation using an array comprehension. We will do so in steps, using `psci` as our interactive development environment.

The first step is to generate an array of pairs of numbers below `n`, which we can do using `concatMap`. 

Let's start by mapping each number to the array `1 .. n`:

```text
> let pairs n = concatMap (\i -> 1 .. n) (1 .. n)
```

We can test our function

```text
> pairs 3
[1,2,3,1,2,3,1,2,3]
```

This is not quite what we want. Instead of just returning the second element of each pair, we need to map a function over the inner copy of `1 .. n` which will allow us to keep the entire pair:

```text
> let pairs n = concatMap (\i -> map (\j -> [i, j]) (1 .. n)) (1 .. n)

> pairs 3
[[1,1],[1,2],[1,3],[2,1],[2,2],[2,3],[3,1],[3,2],[3,3]]
```

This is looking better. However, we are generating too many pairs: we keep both [1, 2] and [2, 1] for example. We can exclude the second case by making sure that `j` only ranges from `i` to `n`:

```text
> let pairs n = concatMap (\i -> map (\j -> [i, j]) (i .. n)) (1 .. n)

> pairs 3
[[1,1],[1,2],[1,3],[2,2],[2,3],[3,3]]
```

Great! Now that we have all of the pairs of potential factors, we can use `filter` to choose the pairs which multiply to give `n`:

```text
> :i Data.Foldable

> let factors n = filter (\pair -> product pair == n) (pairs n)
  
> factors 10
[[1,10],[2,5]]
```

This code uses the `product` function from the `Data.Foldable` module in the `purescript-foldable-traversable` library.

Excellent! We've managed to find the correct set of factor pairs without duplicates.

## Do Notation

However, we can improve the readability of our code considerably. `map` and `concatMap` are so fundamental, that they form the basis (or rather, their generalizations `<$>` and `>>=` form the basis) of a special syntax called _do notation_.

_Note_: Just like `map` and `concatMap` allowed us to write _array comprehensions_, the more general operators `<$>` and `>>=` allow us to write so-called _monad comprehensions_. We'll see plenty more examples of _monads_ later in the book, but in this chapter, we will only consider arrays.

We can rewrite our `factors` function using do notation as follows:

```haskell
factors :: Number -> [[Number]]
factors n = filter (\xs -> product xs == n) $ do
  i <- 1 .. n
  j <- i .. n
  return [i, j]
```

The keyword `do` introduces a block of code which uses do notation. The block consists of expressions of a few types:

- Expressions which bind elements of an array to a name. These are indicated with the backwards-facing arrow `<-`, with a name on the left, and an expression on the right whose type is an array.
- Expressions which do not bind elements of the array to names. The last line `return [i, j]` is an example of this kind of expression.
- Expressions which give names to expressions, using the `let` keyword.

This new notation hopefully makes the structure of the algorithm clearer. If you mentally replace the array `<-` with the word "choose", you might read it as follows: "choose an element `i` between 1 and n, then choose an element `j` between `i` and `n`, and return `[i, j]`".

Note that the `return` function in the last line is _not_ a keyword. It is a regular function, which can be evaluated in `psci` like any other. However, we have to provide a type:

```text
> return [1, 2] :: [[Number]]
[[1, 2]]
```

In the case of arrays, `return` simply constructs a singleton array. In fact, we could modify our `factors` function to use this form, instead of using `return`:

```haskell
factors :: Number -> [[Number]]
factors n = filter (\xs -> product xs == n) $ do
  i <- 1 .. n
  j <- i .. n
  [[i, j]]
```

and the result would be the same.

## Guards

One further improvement we can make to the `factors` function is to move the filter inside the array comprehension. This is possible using the `guard` function from the `Control.MonadPlus` module (from the `purescript-control` package):

```haskell
factors :: Number -> [[Number]]
factors n = do
  i <- range 1 n
  j <- range i n
  guard $ i * j == n
  return [i, j]
```

Just like `return`, the `guard` function is _not_ a keyword. We can apply it like a regular function in `psci` to understand how it works.

The type of the `guard` function is more general than we need here:

```text
> :i Control.MonadPlus
> :t guard

forall m. (MonadPlus m) => Boolean -> m Unit
```

In our case, we can assume that `psci` reported the following type:

```haskell
Boolean -> [Unit]
```

For our purposes, the following calculations tell us everything we need to know about the `guard` function on arrays:

```text
> :i Data.Array
> length $ guard true

1

> length $ guard false
  
0
```

That is, if `guard` is passed an expression which evaluates to `true`, then it returns an array with a single element. If the expression evaluates to `false`, then its result is empty. 

This means that if the guard fails, then the current branch of the array comprehension will terminate early with no results. This means that a call to `guard` is equivalent to using `filter` on the intermediate array. Try the two definitions of `factors` to verify that they give the same results.

X> ## Exercises
X> 
X> 1. (Easy) Use the `factors` function to define a function `isPrime` which tests if its integer argument is prime or not.
X> 1. (Medium) Write function which uses do notation to find the _cartesian product_ of two arrays, i.e. the set of all tuples of elements `a`, `b`, where `a` is an element of the first array, and `b` is an element of the second.
X> 1. (Medium) A _Pythagorean triple_ is an array of numbers `[a, b, c]` such that `a² + b² = c²`. Use the `guard` function in an array comprehension to write a function `triples` which takes a number `n` and calculates all Pythagorean triples whose components are less than `n`. Your function should have type `Number -> [[Number]]`.
X> 1. (Difficult) Rewrite the `factors` function from scratch using the `any` function from `Data.Foldable`. Do not use `concat`, `concatMap`, `do` or `>>=`. _Note_: the type of `any` as reported by `psci` is more general than you need. For the purposes of this exercise, you can assume the type of `any` is `forall a. (a -> Boolean) -> [a] -> Boolean`.
X> 1. (Diabolical) Use the `factors` function to define a function `factorizations` which produces all _factorizations_ of a number `n`, i.e. arrays of integers whose product is `n`. _Hint_: consider the factorizations of 1 separately. Be careful to avoid infinite recursion.

## Folds

Left and right folds over arrays provide another class of interesting functions which can be implemented using recursion.

Start by importing the `Data.Foldable` module, and inspecting the types of the `foldl` and `foldr` functions using `psci`:

```text
> :i Data.Foldable

> :t foldl
forall a b f. (Foldable f) => (b -> a -> b) -> b -> f a -> b

> :t foldr
forall a b f. (Foldable f) => (a -> b -> b) -> b -> f a -> b
``` 

These types are actually more general than we are interested in right now. For the purposes of this chapter, we can assume that `psci` had given the following (more specific) answer:

```text
> :t foldl
forall a b. (b -> a -> b) -> b -> [a] -> b

> :t foldr
forall a b. (a -> b -> b) -> b -> [a] -> b
```

In both of these cases, the type `a` corresponds to the type of elements of our array. The type `b` can be thought of as the type of an "accumulator", which will accumulate a result as we traverse the array.

The difference between the `foldl` and `foldr` functions is the direction of the traversal. `foldl` folds the array "from the left", whereas `foldr` folds the array "from the right".

Let's see these functions in action. Let's use `foldl` to sum an array of numbers. The type `a` will be `Number`, and we can also choose the result type `b` to be `Number`. We need to provide three arguments: a function `Number -> Number -> Number`, which will add the next element to the accumulator, an initial value for the accumulator of type `Number`, and an array of `Number`s to add. For the first argument, we can just use the addition operator, and the initial value of the accumulator will be zero:

```text
> foldl (+) 0 (1 .. 5)
15
```

In this case, it didn't matter whether we used `foldl` or `foldr`, because the `(+)` function returns the same result if its arguments are reversed:

```text
> foldr (+) 0 (1 .. 5)
15
```

Let's write an example where the choice of folding function does matter, in order to illustrate the difference. Instead of the addition function, let's use string concatenation to build a string:

```text
> foldl (\acc n -> acc ++ show n) "" [1,2,3,4,5]
"12345"

> foldr (\n acc -> acc ++ show n) "" [1,2,3,4,5]
"54321"
```

This illustrates the difference between the two functions. The left fold expression is equivalent to the following application:

```text
((((("" ++ show 1) ++ show 2) ++ show 3) ++ show 4) ++ show 5)
```

whereas the right fold is equivalent to this:

```text
((((("" ++ show 5) ++ show 4) ++ show 3) ++ show 2) ++ show 1)
```

## Tail Recursion

Recursion is a powerful technique for specifying algorithms, but comes with a problem: evaluating recursive functions in JavaScript can lead to stack overflow errors if our inputs are too large.

It is easy to verify this problem, with the following code in `psci`:

```text
> let f 0 = 0
      f n = 1 + f (n - 1)

> f 10
10
  
> f 10000
RangeError: Maximum call stack size exceeded
```

This is a problem. If we are going to adopt recursion as a standard technique from functional programming, then we need a way to deal with possibly unbounded recursion.

PureScript provides a partial solution to this problem in the form of _tail recursion optimization_. 

_Note_: more complete solutions to the problem can be implemented in libraries using so-called _trampolining_, but that is beyond the scope of this chapter.

The key observation which enables tail recursion optimization is the following: a recursive call in _tail position_ to a function can be replaced with a _jump_, which does not allocate a stack frame. A call is in _tail position_ when it is the last call made before a function returns. This is the reason why we observed a stack overflow in the example - the recursive call to `f` was _not_ in tail position.

In practice, the PureScript compiler does not replace the recursive call with a jump, but rather replaces the entire recursive function with a _while loop_.

Here is an example of a recursive function with all recursive calls in tail position:

```haskell
fact :: Number -> Number -> Number
fact 0 acc = acc
fact n acc = fact (n - 1) (acc * n)
```

Notice that the recursive call to `fact` is the last thing that happens in this function - it is in tail position.

## Accumulators

One common way to turn a function which is not tail recursive into a tail recursive function is to use an _accumulator parameter_. An accumulator parameter is an additional parameter which is added to a function which _accumulates_ a return value, as opposed to using the return value to accumulate the result, which can prevent tail recursion.

For example, consider this array recursion which reverses the input array:

```haskell
reverse :: forall a. [a] -> [a]
reverse [] = []
reverse (x : xs) = reverse xs ++ [x]
```

This implementation is not tail recursive, so the generated JavaScript will cause a stack overflow when executed on a large input array. However, we can make it tail recursive, by introducing a second function argument to accumulate the result instead:

```haskell
reverse :: forall a. [a] -> [a]
reverse = reverse' []
  where
  reverse' acc [] = acc
  reverse' acc (x : xs) = reverse' (x : acc) xs
```

In this case, we delegate to the helper function `reverse'`, which performs the heavy lifting of reversing the array. Notice though that the function `reverse'` is tail recursive - its only recursive call is in the last case, and is in tail position. This means that the generated code will be a _while loop_, and will not blow the stack for large inputs.

To understand the second implementation of `reverse`, note that the helper function `reverse'` essentially uses the accumulator parameter to maintain an additional piece of state - the partially constructed result. The result starts out empty, and grows by one element for every element in the input array. However, because later elements are added at the front of the array, the result is the original array in reverse! 

Note also that while we might think of the accumulator as "state", there is no direct mutation going on. The accumulator is an immutable array, and we simply use function arguments to thread the state through the computation.

## Prefer Folds to Explicit Recursion

If we can write our recursive functions using tail recursion, then we can benefit from tail recursion optimization, so it becomes tempting to try to write all of our functions in this form. However, it is often easy to forget that many functions can be written directly as a fold over an array or similar data structure. Writing algorithms directly in terms of combinators such as `map` and `fold` has the added advantage of code simplicity - these combinators are well-understood, and as such, communicate the _intent_ of the algorithm much better than explicit recursion.

For example, the `reverse` example can be written as a fold in at least two ways. Here is a version which uses `foldr`:

```text
> :i Data.Foldable
> let reverse :: forall a. [a] -> [a]
      reverse = foldr (\x xs -> xs ++ [x]) []
  
> reverse [1, 2, 3]
  
[3,2,1]
```

Writing `reverse` in terms of `foldl` will be left as an exercise for the reader.

X> ## Exercises
X> 
X> 1. (Easy) Use `foldl` to test whether an array of boolean values are all true. 
X> 2. (Medium) Characterize those arrays `xs` for which the function `foldl (==) false xs` returns true.
X> 3. (Medium) Rewrite the following function in tail recursive form using an accumulator parameter:
X> 
X>     ```haskell
X>     count :: forall a. (a -> Boolean) -> [a] -> Number
X>     count _ [] = 0
X>     count p (x : xs) = if p x then 1 + count p xs else count p xs
X>     ```
X> 
X> 4. (Medium) Write `reverse` in terms of `foldl`.

## A Virtual Filesystem

In this section, we're going to apply what we've learned, writing functions which will work with a model of a filesystem. We will use maps, folds and filters to work with a predefined API.

The `Data.Path` module defines an API for a virtual filesystem, as follows:

- There is a type `Path` which represents a path in the filesystem.
- There is a path `root` which represents the root directory.
- The `ls` function enumerates the files in a directory.
- The `filename` function returns the file name for a `Path`.
- The `size` function returns the file size for a `Path` which represents a file.
- The `isDirectory` function tests whether a function is a file or a directory.

In terms of types, we have the following type definitions:

```haskell
root :: Path

ls :: Path -> [Path]

filename :: Path -> String

size :: Path -> Maybe Number

isDirectory :: Path -> Boolean
```

We can try out the API in `psci`:

```text
> :i Data.Path

> root
/

> isDirectory root
true

> ls root
[/bin/,/etc/,/home/]
```

The `FileOperations` module defines functions which use the `Data.Path` API. You do not need to modify the `Data.Path` module, or understand its implementation. We will work entirely in the `FileOperations` module.

## Listing All Files

Let's write a function which performs a deep enumeration of all files inside a directory. This function will have the following type:

```haskell
allFiles :: Path -> [Path]
```

We can define this function by recursion. First, we can use `ls` to enumerate the immediate children of the directory. For each child, we can recursively apply `allFiles`, which will return an array of paths. `concatMap` will allow us to apply `allFiles` and flatten the results at the same time.

Finally, we use the cons operator `:` to include the current file:

```haskell
allFiles file = file : concatMap allFiles (ls file)
```

Let's try this function in `psci`:

```text
> :i FileOperations
> :i Data.Path

> allFiles root
  
[/,/bin/,/bin/cp,/bin/ls,/bin/mv,/etc/,/etc/hosts, ...]
```

Great! Now let's see if we can write this function using an array comprehension using do notation.

Recall that a backwards arrow corresponds to choosing an element from an array. The first step is to choose an element from the immediate children of the argument. Then we simply call the function recursively for that file. Since we are using do notation, there is an implicit call to `concatMap` which concatenates all of the recursive results.

Here is the new version:

```haskell
allFiles' :: Path -> [Path]
allFiles' file = file : do
  child <- ls file
  allFiles' child
```

Try out the new version in `psci` - you should get the same result. I'll let you decide which version you find clearer.

X> ## Exercises
X> 
X> 1. (Easy) Write a function `onlyFiles` which returns all _files_ (not directories) in all subdirectories of a directory.
X> 1. (Medium) Write a fold to determine the largest and smallest files in the filesystem. 
X> 1. (Difficult) Write a function `whereIs` to search for a file by name. The function should return a value of type `Maybe Path`, indicating the directory containing the file, if it exists. It should behave as follows:
X> 
X>     ```text
X>     > whereIs "/bin/ls"
X>     Just (/bin/)
X>     
X>     > whereIs "/bin/cat"
X>     Nothing
X>     ```
X> 
X>     _Hint_: Try to write this function as an array comprehension using do notation.

## Conclusion

In this chapter, we covered the basics of recursion in PureScript, as a means of expressing algorithsm concisely. We also introduced user-defined infix operators, standard functions on arrays such as maps, filters and folds, and array comprehensions which combine these ideas. Finally, we showed the importance of using tail recursion in order to avoid stack overflow errors, and how to use accumulator parameters to convert functions to tail recursive form.

