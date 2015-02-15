# Generative Testing

## Chapter Goals

In this chapter, we will see a particularly elegant application of type classes to the problem of testing. Instead of testing our code by telling the compiler _how_ to test, we simply assert _what_ properties our code should have. Test cases can be generated randomly from this specification, using type classes to hide the boilerplate code of random data generation. This is called _generative testing_ (or _property-based testing_), a technique made popular by the [QuickCheck](http://www.haskell.org/haskellwiki/Introduction_to_QuickCheck1) library in Haskell.

The `purescript-quickcheck` package is a port of Haskell's QuickCheck library to PureScript, and for the most part, it preserves the types and syntax of the original library. We will see how to use `purescript-quickcheck` to test a simple library, using Grunt to integrate our test suite into our automated build.

## Project Setup

This chapter's project adds `purescript-quickcheck` as a Bower dependency. In fact, in `bower.json`, `purescript-quickcheck` is added to the `devDependencies` section:

```javascript
  "devDependencies": {
    "purescript-quickcheck": "~0.1.3"
  }
```

This indicates that `purescript-quickcheck` is only required during development. For production builds, to avoid including the QuickCheck library code in your output, use the `--production` flag on the `bower` executable:

```text
$ bower update --production

$ grunt build
```

## Automating Tests

The `Gruntfile.js` file for this project has been modified slightly to support our test suite.

Firstly, a new section has been added to the `psc` task, which will build the source code along with the code for the test suite, and generate a secondary JavaScript file:

```javascript
psc: {
  lib: {
    src: ["<%=srcFiles%>"],
    dest: "dist/Main.js"
  },
  tests: {
    options: {
      module: ["Main"],
      main: true
    },
    src: ["tests/Main.purs", "<%=srcFiles%>"],
    dest: "dist/tests.js"
  }
}
```

The `psc:tests` task now generates the additional `dist/tests.js` file, which can be executed to run our test suite. The next step is to automate this process, which we can do by using the `grunt-execute` package:

```javascript
execute: {
  tests: {
    src: "dist/tests.js"
  }
}
```

The `grunt-execute` package is also added as an NPM dependency. Finally, we need to add it as a task to our Grunt task list:

```javascript
grunt.loadNpmTasks("grunt-execute");

grunt.registerTask("build", 
  ["psc:lib", "dotPsci"]);
grunt.registerTask("test", 
  ["build", "psc:tests", "execute:tests"]);
```

This makes two new tasks available: `build` which only builds the library source code, and `test`, which builds the library and test suite, and executes the tests. 

## Writing Properties

The `Merge.purs` file implements a simple function `merge`, which we will use to demonstrate the features of the `purescript-quickcheck` library.

```haskell
merge :: [Number] -> [Number] -> [Number]
```

`merge` takes two sorted arrays of numbers, and merges their elements so that the result is also sorted. For example:

```text
> :i Merge
> merge [1, 3, 5] [2, 4, 6]

[1, 2, 3, 4, 5, 6]
```

In a typical test suite, we might test `merge` by generating a few small test cases like this by hand, and asserting that the results were equal to the appropriate values. However, everything we need to know about the `merge` function can be summarized in two properties:

- (Sortedness) If `xs` and `ys` are sorted, then `merge xs ys` is also sorted.
- (Subarray) `xs` and `ys` are both subarrays of `merge xs ys`, and their elements appear in the same order.

`purescript-quickcheck` allows us to test these properties directly, by generating random test cases. We simply state the properties that we want our code to have, as functions:

```haskell
main = do
  quickCheck $ \xs ys -> 
    isSorted $ merge (sort xs) (sort ys)
  quickCheck $ \xs ys -> 
    xs `isSubarrayOf` merge xs ys
```

Here, `isSorted` and `isSubarrayOf` are implemented as helper functions with the following types:

```haskell
isSorted :: forall a. (Ord a) => [a] -> Boolean
isSubarrayOf :: forall a. (Eq a) => [a] -> [a] -> Boolean
```

When we run this code, `purescript-quickcheck` will attempt to disprove the properties we claimed, by generating random inputs `xs` and `ys`, and passing them to our functions. If our function returns `false` for any inputs, the property will be incorrect, and the library will raise an error. Fortunately, the library is unable to disprove our properties after generating 100 random test cases:

```text
$ grunt

Running "execute:tests" (execute) task
-> executing dist/tests.js
100/100 test(s) passed.
100/100 test(s) passed.
-> completed dist/tests.js

>> 1 file and 0 calls executed

Done, without errors.
```

If we deliberately introduce a bug into the `merge` function (for example, by changing the less-than check for a greater-than check), then an exception is thrown at runtime after the first failed test case:

```text
Error: Test 1 failed: 
Test returned false
```

As we can see, this error message is not very helpful, but it can be improved with a little work.

## Improving Error Messages

To provide error messages along with our failed test cases, `purescript-quickcheck` provides the `<?>` operator. Simply separate the property definition from the error message using `<?>`, as follows:

```haskell
quickCheck $ \xs ys -> 
  let
    result = merge xs ys 
  in 
    xs `isSubarrayOf` result <?> show xs ++ " not a subarray of " ++ show result
```

This time, if we modify the code to introduce a bug, we see our improved error message after the first failed test case:

```text
Error: Test 6 failed: 
[0.85] not a subarray of [0.89,0.82,0.44,0.01]
```

Notice how the input `xs` was generated as an array of randomly-selected numbers.

X> ## Exercises
X> 
X> 1. (Easy) Write a property which asserts that merging an array with the empty array does not modify the original array.
X> 1. (Easy) Add an appropriate error message to the remaining property for `merge`.

## Testing Polymorphic Code

The `Merge` module defines a generalization of the `merge` function, called `mergePoly`, which works not only with arrays of numbers, but also arrays of any type belonging to the `Ord` type class:

```haskell
mergePoly :: forall a. (Ord a) => [a] -> [a] -> [a]
```

If we modify our original tests to use `mergePoly` in place of `merge`, we see the following error message:

```text
Error in declaration main
No instance found for Testable ([u1] -> [u1] -> Boolean)
```

This error message indicates that the compiler could not generate random test cases, because it did not know what type of elements we wanted our arrays to have. In these sorts of cases, we can use a helper function to force the compiler to infer a particular type. For example, if we define a function `numbers` as a synonym for the identity function:

```haskell
numbers :: [Number] -> [Number]
numbers = id
```

then we can modify our tests so that the compiler infers the type `[Number]` for our two array arguments:

```haskell
quickCheck $ \xs ys -> 
  isSorted $ numbers $ mergePoly (sort xs) (sort ys)
quickCheck $ \xs ys -> 
  numbers xs `isSubarrayOf` mergePoly xs ys
```

Here, `xs` and `ys` both have type `[Number]`, since the `numbers` function has been used to disambiguate the unknown type.

X> ## Exercises
X> 
X> 1. (Easy) Write a function `bools` which forces the types of `xs` and `ys` to be `[Boolean]`, and add additional properties which test `mergePoly` at that type.
X> 1. (Medium) Choose a function from the standard library (for example, from the `purescript-arrays` package), and write a QuickCheck property for it, including an appropriate error message. Your property should use a helper function to fix any polymorphic type arguments to either `Number` or `Boolean`.

## Generating Arbitrary Data

Now we will see how the `purescript-quickcheck` library is able to randomly generate test cases for our properties.

Those types whose values can be randomly generated are captured by the `Arbitrary` type class:

```haskell
class Arbitrary t where
  arbitrary :: Gen t
```

The `Gen` type constructor represents the side-effects of _deterministic random data generation_. It uses a pseudo-random number generator to generate deterministic random function arguments from a seed value. `Gen` is both a monad and an applicative functor, so we have the usual collection of combinators at our disposal for creating new instances of the `Arbitrary` type class.

For example, the `Arbitrary` instance for the `Number` type, provided in the `purescript-quickcheck` library, generates uniformly distributed values between 0 and 1. If instead, we wanted to define a type whose `Arbitrary` instance generated numbers with a different distribution, we could use the `Applicative` instance to transform a uniform random variable by applying a function to the randomly-generated values:

```haskell
newtype Byte = Byte Number

instance arbitraryByte :: Arbitary Byte where
  arbitrary = uniformToByte <$> arbitrary
    where
    uniformToByte n = Math.floor (n * 256)
```

Here, we define a type `Byte` of integral values between 0 and 255. The `Arbitrary` instance uses the `<$>` operator to lift the `uniformToByte` function over the `arbitrary` action. The type of the inner `arbitrary` action is inferred as `Gen Number`, which means that it is generates a uniformly distributed number between 0 and 1.

We can also use this idea to improve our sortedness test for `merge`:

```haskell
quickCheck $ \xs ys -> 
  isSorted $ numbers $ mergePoly (sort xs) (sort ys)
```

In this test, we generated arbitrary arrays `xs` and `ys`, but had to sort them, since `merge` expects sorted input. On the other hand, we could create a newtype representing sorted arrays, and write an `Arbitrary` instance which generates sorted data:

```haskell
newtype Sorted a = Sorted [a]

sorted :: forall a. Sorted a -> [a]
sorted (Sorted xs) = xs

instance arbSorted :: (Arbitrary a, Ord a) => Arbitrary (Sorted a) where
  arbitrary = Sorted <<< sort <$> arbitrary
```

With this type constructor, we can modify our test as follows:

```haskell
quickCheck $ \xs ys -> 
  isSorted $ numbers $ mergePoly (sorted xs) (sorted ys)
```

This may look like a small change, but the types of `xs` and `ys` have changed to `Sorted Number`, instead of just `[Number]`. This communicates our _intent_ in a clearer way - the `mergePoly` function takes sorted input. Ideally, the type of the `mergePoly` function itself would be updated to use the `Sorted` type constructor.

As a more interesting example, the `Tree.purs` file defines a type of sorted binary trees with values at the branches:

```haskell
data Tree a
  = Leaf
  | Branch (Tree a) a (Tree a)
```

The `Tree` module defines the following API:

```haskell
insert :: forall a. (Ord a) => a -> Tree a -> Tree a
member :: forall a. (Ord a) => a -> Tree a -> Boolean

toArray :: forall a. Tree a -> [a]
fromArray :: forall a. (Ord a) => [a] -> Tree a
```

The `insert` function is used to insert a new element into a sorted tree, and the `member` function can be used to query a tree for a particular value. For example:

```text
> :i Tree

> member 2 $ insert 1 $ insert 2 Leaf
true

> member 1 Leaf
false
```

The `toArray` and `fromArray` functions can be used to convert sorted trees to and from (sorted) arrays. We can use `fromArray` to write an `Arbitrary` instance for trees:

```haskell
instance arbTree :: (Arbitrary a, Ord a) => Arbitrary (Tree a) where
  arbitrary = fromArray <<< sorted <$> arbitrary
```

Note that we are using the `sorted` function here to ensure that the input to `fromArray` is a sorted array.

We can now use `Tree a` as the type of an argument to our test properties, whenever there is an `Arbitrary` instance available for the type `a`. For example, we can test that the `member` test always returns `true` after inserting a value:

```haskell
quickCheck $ \t a -> 
  member a $ insert a (t :: Tree Number) 
```

Here, the argument `t` is a randomly-generated tree of type `Tree Number`.

X> ## Exercises
X> 
X> 1. (Medium) Create a newtype for `String` with an associated `Arbitrary` instance which generates collections of randomly-selected characters in the range `a-z`. _Hint_: use the `Arbitrary` instance for the `[Number]` and a function of type `[Number] -> String`.
X> 1. (Difficult) Write a property which asserts that a value inserted into a tree is still a member of that tree after arbitrarily many more insertions.

## Testing Higher-Order Functions

The `Merge` module defines another generalization of the `merge` function - the `mergeWith` function takes an additional function as an argument which is used to determine the order in which elements should be merged. That is, `mergeWith` is a higher-order function.

For example, we can pass the `length` function as the first argument, to merge two arrays which are already in length-increasing order. The result should also be in length-increasing order:

```haskell
> :i Data.String
> mergeWith length 
    ["", "ab", "abcd"] 
    ["x", "xyz"]

["","x","ab","xyz","abcd"]
```

How might we test such a function? Ideally, we would like to generate values for all three arguments, including the first argument which is a function.

There is a second type class which allows us to create randomly-generated functions. It is called `CoArbitrary`, and it is defined as follows:

```haskell
class CoArbitrary t where
  coarbitrary :: forall r. t -> Gen r -> Gen r
```

The `coarbitrary` function takes a function argument of type `t`, and a random generator for a function result of type `r`, and uses the function argument to _perturb_ the random generator. That is, it uses the function argument to modify the random output of the random generator for the result. 

In addition, there is a type class instance which gives us `Arbitrary` functions if the function domain is `CoArbitrary` and the function codomain is `Arbitrary`:

```haskell
instance arbFunction :: (CoArbitrary a, Arbitrary b) => Arbitrary (a -> b)
```

In practice, this means that we can write properties which take functions as arguments. In the case of the `mergeWith` function, we can generate the first argument randomly, modifying our tests to take account of the new argument.

In the case of the sortedness property, we cannot guarantee that the result will be sorted - we do not even necessarily have an `Ord` instance - but we can expect that the result be sorted with respect to the function `f` that we pass in as an argument. In addition, we need the two input arrays to be sorted with respect to `f`, so we use the `sortBy` function to sort `xs` and `ys` based on comparison after the function `f` has been applied:

```haskell
quickCheck $ \xs ys f -> 
  isSorted $ 
    map f $ 
      mergeWith (numberToBool f) 
                (sortBy (compare `on` f) xs) 
                (sortBy (compare `on` f) ys)
```

Here, we use a function `numberToBool` to disambiguate the type of the function `f`:

```haskell
numberToBool :: (Number -> Boolean) -> Number -> Boolean
numberToBool = id
```

In the case of the subarray property, we simply have to change the name of the function to `mergeWith` - we still expect our input arrays to be subarrays of the result:

```haskell
quickCheck $ \xs ys f -> 
  xs `isSubarrayOf` mergeWith (numberToBool f) xs ys
```

In addition to being `Arbitrary`, functions are also `CoArbitrary`:

```haskell
instance coarbFunction :: (Arbitrary a, CoArbitrary b) => CoArbitrary (a -> b)
```

This means that we are not limited to just values and functions - we can also randomly generate _higher-order functions_, or functions whose arguments are higher-order functions, and so on.

## Writing CoArbitrary Instances

Just as we can write `Arbitrary` instances for our data types by using the `Monad` and `Applicative` instances of `Gen`, we can write our own `CoArbitrary` instances as well. This allows us to use our own data types as the domain of randomly-generated functions.

Let's write a `CoArbitrary` instance for our `Tree` type. We will need a `CoArbitrary` instance for the type of the elements stored in the branches:

```haskell
instance coarbTree :: (CoArbitrary a) => CoArbitrary (Tree a) where
```

We have to write a function which perturbs a random generator given a value of type `Tree a`. If the input value is a `Leaf`, then we will just return the generator unchanged:

```haskell
  coarbitrary Leaf = id
```

If the tree is a `Branch`, then we will perturb the generator using the left subtree, the value and the right subtree, using function composition to create our perturbation function:

```haskell
  coarbitrary (Branch l a r) =
    coarbitrary l <<<
    coarbitrary a <<<
    coarbitrary r
```

Now we are free to write properties whose arguments include functions taking trees as arguments. For example, the `Tree` module defines a function `anywhere`, which tests if a predicate holds on any subtree of its argument:

```haskell
anywhere :: forall a. (Tree a -> Boolean) -> Tree a -> Boolean
```

Now we are able to generate the predicate function randomly. For example, we expect the `anywhere` function to be _invariant under disjunction_:

```haskell
quickCheck $ \f g t ->
  anywhere (\s -> f s || g s) t ==
    anywhere f (treeOfNumber t) || anywhere g t
```

Here, the `treeOfNumber` function is used to fix the type of values contained in the tree to the type `Number`:

```haskell
treeOfNumber :: Tree Number -> Tree Number
treeOfNumber = id
```

## Testing Without Side-Effects

For the purposes of testing, we usually include calls to the `quickCheck` function in the `main` action of our test suite. However, there is a variant of the `quickCheck` function, called `quickCheckPure` which does not use side-effects. Instead, it is a pure function which takes a random seed as an input, and returns an array of test results.

We can test `quickCheckPure` using `psci`. Here, we test that the `merge` operation is associative:

```text
> :i Test.QuickCheck
> :i Merge

> quickCheckPure 12345 10 $ \xs ys zs -> 
    ((xs `merge` ys) `merge` zs) == 
      (xs `merge` (ys `merge` zs))
  
[Success, Success, ..., Success]
```

`quickCheckPure` takes three arguments: the random seed, the number of test cases to generate, and the property to test. If all tests pass, you should see an array of `Success` data constructors printed to the console.

`quickCheckPure` might be useful in other situations, such as generating random input data for performance benchmarks, or generating sample form data for web applications.

X> ## Exercises
X> 
X> 1. (Easy) Write `CoArbitrary` instances for the `Byte` and `Sorted` type constructors.
X> 1. (Medium) Write a (higher-order) property which asserts associativity of the `mergeWith f` function for any function `f`. Test your property in `psci` using `quickCheckPure`. 
X> 1. (Medium) Write a `CoArbitrary` instance for the following data type:
X> 
X>     ```haskell
X>     data OneTwoThree a = One a | Two a a | Three a a a
X>     ```
X> 1. (Medium) Use the `all` function to simplify the result of the `quickCheckPure` function - your function should return `true` if every test passes, and `false` otherwise. Try using the `First` monoid, defined in `purescript-monoids` with the `foldMap` function to preserve the first error in case of failure.

## Conclusion

In this chapter, we met the `purescript-quickcheck` package, which can be used to write tests in a declarative way using the paradigm of _generative testing_. In particular:

- We saw how to automate QuickCheck tests using Grunt.
- We saw how to write properties as functions, and how to use the `<?>` operator to improve error messages.
- We saw how the `Arbitrary` and `CoArbitrary` type classes enable generation of boilerplate testing code, and how they allow us to test higher-order properties.
- We saw how to implement custom `Arbitrary` and `CoArbitrary` instances for our own data types.