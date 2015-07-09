# Callback Hell

## Chapter Goals

In this chapter, we will see how the tools we have seen so far - namely monad transformers and applicative functors - can be put to use to solve real-world problems. In particular, we will see how we can solve the problem of _callback hell_. 

## Project Setup

The source code for this chapter can be compiled using `grunt` and run using NodeJS.

## The Problem

Asynchronous code in JavaScript typically uses _callbacks_ to structure program flow. For example, to read text from a file, the preferred approach is to use the `readFile` function and to pass a callback - a function that will be called when the text is available:

```javascript
function readText(onSuccess, onFailure) {
  var fs = require('fs');
  fs.readFile('file1.txt', { encoding: 'utf-8' }, function (error, data) {
    if (error) {
      onFailure(error.code);
    } else {
      onSuccess(data);
    }   
  });
}
```

However, if multiple asynchronous operations are involved, this can quickly lead to nested callbacks, which can result in unreadable code:

```javascript
function copyFile(onSuccess, onFailure) {
  var fs = require('fs');
  fs.readFile('file1.txt', { encoding: 'utf-8' }, function (error, data1) {
    if (error) {
      onFailure(error.code);
    } else {
      fs.writeFile('file2.txt', data, { encoding: 'utf-8' }, function (error) {
        if (error) {
          onFailure(error.code);
        } else {
          onSuccess();
        }
      });
    }   
  });
} 
```

One solution to this problem is to break out individual asynchronous calls into their own functions:

```javascript
function writeCopy(data, onSuccess, onFailure) {
  var fs = require('fs');
  fs.writeFile('file2.txt', data, { encoding: 'utf-8' }, function (error) {
    if (error) {
      onFailure(error.code);
    } else {
      onSuccess();
    }
  });
}

function copyFile(onSuccess, onFailure) {
  var fs = require('fs');
  fs.readFile('file1.txt', { encoding: 'utf-8' }, function (error, data) {
    if (error) {
      onFailure(error.code);
    } else {
      writeCopy(data, onSuccess, onFailure);
    }   
  });
} 
```

This solution works but has some issues:

- It is necessary to pass intermediate results to asynchronous functions as function arguments, in the same way that we passed `data` to `writeCopy` above. This is fine for small functions, but if there are many callbacks involved, the data dependencies can become complex, resulting in many additional function arguments.
- There is a common pattern - the callbacks `onSuccess` and `onFailure` are usually specified as arguments to every asynchronous function - but this pattern has to be documented in module documentation which accompanies the source code. It is better to capture this pattern in the type system, and to use the type system to enforce its use.

Next, we will see how to use the techniques we have learned so far to solve these issues.

## The Continuation Monad

Let's translate the `copyFile` example above into PureScript by using the FFI. In doing so, the structure of the computation will become apparent, and we will be led naturally to a monad transformer which is defined in the `purescript-transformers` package - the continuation monad transformer `ContT`.

First, we need to gives types to `readFile` and `writeFile` using the FFI. Let's start by defining some type synonyms, and a new effect for file IO:

```haskell
foreign import data FS :: !

type ErrorCode = String
type FilePath = String
```

`readFile` takes a filename and a callback which takes two arguments. If the file was read successfully, the second argument will contain the file contents, and if not, the first argument will be used to indicate the error.

In our case, we will wrap `readFile` with a function which takes two callbacks: an error callback (`onFailure`) and a result callback (`onSuccess`), much like we did with `copyFile` and `writeCopy` above. Using the multiple-argument function support from `Data.Function` for simplicity, our wrapped function `readFileImpl` might look like this:

```haskell
foreign import readFileImpl
  "function readFileImpl(path, onSuccess, onFailure) {\
  \  return function() {\
  \    require('fs').readFile(path, \
  \      { encoding: 'utf-8' }, \
  \      function(error, data) {\
  \        if (error) {\
  \          onFailure(error.code)();\
  \        } else {\
  \          onSuccess(data)();\
  \        }\
  \      }\
  \    );\
  \  };\
  \}" :: forall eff. Fn3 FilePath
                         (String -> Eff (fs :: FS | eff) Unit)
                         (ErrorCode -> Eff (fs :: FS | eff) Unit)
                         (Eff (fs :: FS | eff) Unit)
```

This type signature indicates that `readFileImpl` takes three arguments: a file path, a success callback and an error callback, and returns an effectful computation which returns an empty (`Unit`) result. Notice that the callbacks themselves are given types which use the `Eff` monad to track their effects.

You should try to understand why this implementation has the correct runtime representation for its type.

`writeFileImpl` is very similar - it is different only in that the file content is passed to the function itself, not to the callback. Its implementation looks like this:

```haskell
foreign import writeFileImpl
  "function writeFileImpl(path, data, onSuccess, onFailure) {\
  \  return function() {\
  \    require('fs').writeFile(path, data, \
  \      { encoding: 'utf-8' }, \
  \      function(error) {\
  \        if (error) {\
  \          onFailure(error.code)();\
  \        } else {\
  \          onSuccess();\
  \        }\
  \      }\
  \    );\
  \  };\
  \}" :: forall eff. Fn4 FilePath
                         String
                         (Eff (fs :: FS | eff) Unit)
                         (ErrorCode -> Eff (fs :: FS | eff) Unit)
                         (Eff (fs :: FS | eff) Unit)
```

Given these FFI declarations, we can write the implementations of `readFile` and `writeFile`. These will use the `Data.Function` library to turn the multiple-argument FFI bindings into regular (curried) PureScript functions, and therefore have slightly more readable types.

In addition, instead of requiring two callbacks, one for successes and one for failures, we can require only a single callback which responds to _either_ successes or failures. That is, the new callback takes a value in the `Either ErrorCode` monad as its argument:

```haskell
readFile :: forall eff. 
  FilePath -> 
  (Either ErrorCode String -> Eff (fs :: FS | eff) Unit) -> 
  Eff (fs :: FS | eff) Unit
readFile path k = 
  runFn3 readFileImpl 
         path 
         (k <<< Right) 
         (k <<< Left)

writeFile :: forall eff. 
  FilePath -> 
  String -> 
  (Either ErrorCode Unit -> Eff (fs :: FS | eff) Unit) -> 
  Eff (fs :: FS | eff) Unit
writeFile path text k = 
  runFn4 writeFileImpl 
         path 
         text 
         (k $ Right unit) 
         (k <<< Left)
```

The `Eff` monad appears in both of these type signatures. We could simplify the types by introducing a new type synonym as follows:

```haskell
type M eff = Eff (fs :: FS | eff)

readFile :: forall eff. 
  FilePath -> 
  (Either ErrorCode String -> M eff Unit) -> 
  M eff Unit

writeFile :: forall eff. 
  FilePath -> 
  String -> 
  (Either ErrorCode Unit -> M eff Unit) -> 
  M eff Unit
```

Now we can spot an important pattern. Each of these functions takes a callback which returns a value in some monad (in this case `M eff`) and returns a value in _the same monad_. This means that when the first callback returns a result, that monad can be used to bind the result to the input of the next asynchronous function. In fact, that's exactly what we did by hand in the `copyFile` example.

This is the basis of the _continuation monad transformer_, which is defined in the `Control.Monad.Cont.Trans` module in `purescript-transformers`.

`ContT` is defined as a newtype as follows:

```haskell
newtype ContT r m a = ContT ((a -> m r) -> m r)
```

A _continuation_ is just another name for a callback. A continuation captures the _remainder_ of a computation - in our case, what happens after a result has been provided after an asynchronous call.

The argument to the `ContT` data constructor looks remarkably similar to the types of `readFile` and `writeFile`. In fact, if we take the type `a` to be the type `Either ErrorCode String`, `r` to be `Unit` and `m` to be the monad `M eff`, we recover the right-hand side of the type of `readFile`. 

For our purposes, we will always use `ContT` to transform the `Eff` monad, and the type `r` will always be `Unit`, but this is not required.

We can treat `readFile` and `writeFile` as computations in the `ContT Unit (M eff)` monad, by simply applying the `ContT` data constructor:

```haskell
type C eff = ContT Unit (M eff)

readFileCont :: forall eff. 
  FilePath -> 
  C eff (Either ErrorCode String)
readFileCont path = ContT $ readFile path

writeFileCont :: forall eff. 
  FilePath -> 
  String -> 
  C eff (Either ErrorCode Unit)
writeFileCont path text = ContT $ writeFile path text
```

With that, we can write our copy-file routine by simply using do notation for the `ContT` monad transformer:

```haskell
copyFileCont :: forall eff. FilePath -> FilePath -> C eff (Either ErrorCode Unit)
copyFileCont src dest = do
  e <- readFileCont src
  case e of
    Left err -> return $ Left err
    Right content -> writeFileCont dest content
```

Note how the asynchronous nature of `readFileCont` is hidden by the monadic bind expressed using do notation - this looks just like synchronous code, but the `ContT` monad is taking care of wiring our asynchronous functions together.

We can run this computation using the `runContT` handler by providing a continuation. The continuation represents _what to do next_, i.e. what to do when the asynchronous copy-file routine completes. For our simple example, we can just choose the `print` function as the continuation, which will print the result of type `Either ErrorCode Unit` to the console:

```haskell
import Debug.Trace

import Control.Monad.Eff
import Control.Monad.Cont.Trans

main = runContT 
  (copyFileCont "/tmp/1.txt" "/tmp/2.txt") 
  print
```

X> ## Exercises
X> 
X> 1. (Easy) Use `readFileCont` and `writeFileCont` to write a function which concatenates two text files.
X> 1. (Medium) Use the FFI to give an appropriate type to the `setTimeout` function. Write a wrapper function which uses the `ContT` monad transformer:
X> 
X>     ```haskell
X>     type Milliseconds = Number
X> 
X>     foreign import data Timeout :: !
X> 
X>     setTimeoutCont :: forall eff. 
X>       Milliseconds -> 
X>       ContT Unit (Eff (timeout :: Timeout | eff)) Unit
X>     ```

## Putting ErrorT To Work

This solution works, but it can be improved.

In the implementation of `copyFileCont`, we had to use pattern matching to analyse the result of the `readFileCont` computation (of type `Either ErrorCode String`) to determine what to do next. However, we know that the `Either` monad has a corresponding monad transformer, `ErrorT`, so it is reasonable to expect that we should be able to use `ErrorT` with `ContT` to combine the two effects of asynchronous computation and error handling.

In fact, it is possible, and we can see why if we look at the definition of `ErrorT`:

```haskell
newtype ErrorT e m a = ErrorT (m (Either e a))
```

`ErrorT` simply changes the result of the underlying monad from `a` to `Either e a`. This means that we can rewrite `copyFileCont` by transforming our current monad stack with the `ErrorT ErrorCode` transformer. It is as simple as applying the `ErrorT` data constructor to our existing solution. Again, we can provide a type synonym to tidy up the type signatures:

```haskell
type EC eff = ErrorT ErrorCode (C eff)

readFileContErr :: forall eff. FilePath -> EC eff String
readFileContErr path = ErrorT $ readFileCont path

writeFileContErr :: forall eff. FilePath -> String -> EC eff Unit
writeFileContErr path text = ErrorT $ writeFileCont path text
```

Now, our copy-file routine is much simpler, since the asynchronous error handling is hidden inside the `ErrorT` monad transformer:

```haskell
copyFileContErr :: forall eff. FilePath -> FilePath -> EC eff Unit
copyFileContErr src dest = do
  content <- readFileContErr src
  writeFileContErr dest content
```

X> ## Exercises
X> 
X> 1. (Medium) Rewrite your previous solution which concatenated two files, using `ErrorT` to handle any errors.

## A HTTP Client

As another example of using `ContT` to handle asynchronous functions, we'll now look at the `Network.HTTP.Client` module from this chapter's source code. This module uses continuations to wrap NodeJS' asynchronous chunked HTTP requests.

A typical chunked `GET` request using the `http` module looks something like this:

```javascript
function getRequest(onChunk, onComplete) {
  return function() {
    require('http').request({
      host: 'www.purescript.org',
      path: '/' 
    }, function(res) {
      res.setEncoding('utf8');
      res.on('data', function (chunk) {
        onChunk(chunk);
      });
      res.on('end', function () {
        onComplete();
      });
    }).end();
  };
}
```

The `request` method from the `http` module takes an object which specifies the host and path, and returns a response object. The response object emits two types of event that we are interested in:

- The `data` event, which indicates that a new chunk of the response is available.
- The `end` event, which indicates that the response is complete.

In the example above, we pass two callbacks `onChunk` and `onComplete`, which are invoked when the `data` and `end` events fire, respectively.

In the `Network.HTTP.Client` module, the `request` method is wrapped with a function `getImpl` which has the following API:

```haskell
foreign import data HTTP :: !

type WithHTTP eff = Eff (http :: HTTP | eff)

newtype Request = Request
  { host :: String
  , path :: String
  }

newtype Chunk = Chunk String

getImpl :: forall eff. 
  Fn3 Request
      (Chunk -> WithHTTP eff Unit)
      (WithHTTP eff Unit)
      (WithHTTP eff Unit)
```

Again, we can use the `Data.Function` module to turn this into a regular, curried PureScript function. As before, we turn the two callbacks into a single callback, this time accepting a value of type `Maybe Chunk`. A value of `Nothing` passed to the callback corresponds to the `end` event, and a value of `Just chunk` corresponds to a `data` event:

```haskell
getChunk :: forall eff. 
  Request ->
  (Maybe Chunk -> WithHTTP eff Unit) ->
  WithHTTP eff Unit
getChunk req k = 
  runFn3 getImpl 
         req 
         (k <<< Just) 
         (k Nothing)
```

Again, this asynchronous function can be turned into a computation in the continuation monad by applying the `ContT` data constructor:

```haskell
getCont :: forall eff. 
  Request -> 
  ContT Unit (WithHTTP eff) (Maybe Chunk)
getCont req = ContT $ getChunk req
```

In the `readFile` example, the callback was only called once, when the file contents were available (or an error occurred). This time, however, we expect the callback to be called multiple times, once for each chunk of the response.

X> ## Exercises
X> 
X> 1. (Medium) Use `runContT` to test `getCont` by printing each chunk of the response to the console.
X> 1. (Difficult) The `getImpl` and `getCont` functions do not handle asynchronous errors. Modify `getImpl` to respond to the `error` event, and use `ErrorT` to write a variant of `getCont` which expresses asynchronous errors. 
X> 
X>     _Hint_: you can follow the same approach that we took in the `readFile` example.

## Folding Chunked Responses

We can now collect the individual chunks of the response, but it would also be useful to create an asynchronous function whose continuation is called only when the full response is available. One way to implement such a function would be to write a _fold_ over the incoming chunks of the response.

We will write a function `foldC` which folds the multiple results passed to a continuation. This function is defined in the `Control.Monad.Cont.Extras` module in this chapter's source code. 

To keep track of the accumulated value, we will use the `Ref` effect in the `Eff` monad. We will use the following type synonyms to tidy up the type signatures:

```haskell
type WithRef eff = Eff (ref :: Ref | eff)

type ContRef eff = ContT Unit (WithRef eff)
```

With those synonyms, `foldC` can be given the following type:

```haskell
foldC :: forall eff a b r. 
  (b -> a -> Either b r) -> 
  b -> ContRef eff a -> ContRef eff r
```

The function passed to `foldC` takes the current accumulator value and the value passed to the continuation and returns either a new accumulator value or a result to be passed to the new continuation.

Once `foldC` is implemented, we can write a simple function `collect` which will allow us to collect the various chunks of the response body:

```haskell
collect :: forall eff a. 
  ContRef eff (Maybe a) -> 
  ContRef eff [a]
collect = foldC f []
  where
  f xs Nothing = Right xs
  f xs (Just x) = Left (xs ++ [x])
```

The implementation of `foldC` starts by creating a new reference containing the initial accumulator value. This reference will be used to keep track of the accumulator as it is modified in the body of the callback.

```haskell
foldC f b0 c = do
  current <- lift $ newRef b0
```

`foldC` also uses a function called `callCC`, which is an abbreviation for _call with current continuation_. `callCC` takes a single argument, which is a function which represents the _current continuation_ - that is, the code _after_ `callCC` in the current do notation block. We can pass a return value to the current continuation at any time to return early from the block of code inside `callCC`.

```haskell
  callCC $ \k -> quietly $ do
```

Here, `k` is the current continuation. Since this is the last expression in the do notation block defining `foldC`, the current continuation is actually just the continuation passed to `foldC`. We will use it to return the final value from `foldC` when the result of the fold function indicates that we are done accumulating a result.

The `quietly` combinator is defined in a `where` declaration, and we will see its definition soon. Its responsibility is to prevent the code inside `callCC` from returning a value to its continuation in the case where we do not call `k` explicitly. It should become clear soon why this is necessary.

Next, `foldC` binds the result of the asynchronous function `c` to the name `a`:

```haskell
    a <- c
```

Any code after this line will be executed when a new value has been produced asynchronously by the original computation (in our case, when a new chunk of the response is available). When that happens, we want to apply the fold function, so we need to read the current value of the accumulator:

```haskell
    r <- lift $ readRef current
```

Finally, we evaluate the fold function, and split into two cases depending on its result. If the fold function returns a new accumulator value then we update the reference with the new value. If the fold function returns a result, we pass it to the continuation `k`:

```haskell
    case f b a of
      Left next -> lift $ writeRef current next
      Right r -> k r
```

It should now be clear why the `quietly` function was necessary - if we did not silence the result of the code inside `callCC`, then we would have to produce a result of type `r` in the case where the fold function returned a value wrapped with the `Left` constructor - but we wouldn't have any way to produce such a result! 

Here is the definition of the `quietly` function. It allows us to change the result type of the asynchronous function. It is written using the `withContT` function, which allows us to transform the continuation function:

```haskell
  where
  quietly :: forall m a b. (Monad m) => ContT Unit m a -> ContT Unit m b
  quietly = withContT (\_ _ -> return unit)
```

The `foldC` function, and its variant `collect` in particular, allows us to accumulate the full response body by concatenating chunks as they become available:

```haskell
newtype Response = Response [Chunk]

getAll :: forall eff. 
  Request -> 
  ContT Unit (WithHTTP (ref :: Ref | eff)) Response
getAll req = Response <$> collect (getCont req)
```

With that, we can get the response body as a `String`, as follows:

```haskell
getResponseText :: forall eff. 
  Request -> 
  ContT Unit (WithHTTP (ref :: Ref | eff)) String
getResponseText req = responseToString <$> getAll req
  where
  responseToString :: Response -> String
  responseToString (Response chunks) = joinWith "" $ map runChunk chunks
```

We could, for example, print the length of the response body to the console by using `getResponseText` and the `trace` action in the continuation:

```main
main = runContT (getResponseText request) $ \response -> do
  let responseLength = length response
  trace responseLength
  
  where
  request :: Request
  request = Request
    { host: "www.purescript.org"
    , path: "/"
    }
```

This works, but we can improve our solution by using `foldC` more carefully, as we will see in the following exercises.

X> ## Exercises
X> 
X> 1. (Medium) Write a function which saves the response body of a request to a file on disk using `writeFileCont`.
X> 1. (Difficult) It is not necessary to concatenate the entire response body in memory in order to determine its length. By finding the size in bytes of each chunk as it becomes available, we can reduce the memory requirements of our function from the size of the response to the size of a single chunk.
X> 
X>     Rewrite the example to use `foldC` directly, instead of `collect`.

## Parallel Computations

We've seen how to use the `ContT` monad and do notation to compose asynchronous computations in sequence. It would also be useful to be able to compose asynchronous computations _in parallel_.

If we are using `ContT` to transform the `Eff` monad, then we can compute in parallel simply by initiating our two computations one after the other. 

We will write a function with the following type signature:

```haskell
par :: forall a b r eff. 
  (a -> b -> r) -> 
  ContRef eff a -> ContRef eff b -> ContRef eff r
```

`par` takes two asynchronous computations, and a function to combine their results, and returns a single computation which computes and combines the results in parallel. 

We can use mutable references (with the `Ref` effect) to keep track of which of the two continuations has been called. When both results have been returned, we can compute the final result and pass it to the main continuation.

It is simplest to implement `par` by constructing a value with the `ContT` data constructor directly:

```haskell
par f ca cb = ContT $ \k -> do
```

Here, `f` is the combining function, and `ca` and `cb` are are computations, asynchronously returning values of types `a` and `b` respectively. `k` is the continuation, which we will use to return a value of type `r` when both `ca` and `cb` complete.

We begin by creating two new references, to hold the results of `ca` and `cb` as they become available:

```haskell
  ra <- newRef Nothing
  rb <- newRef Nothing
```

The references `ra` and `rb` hold values of type `Maybe a` and `Maybe b` - they both contain the value `Nothing` initially, and as their computations complete, their values will be updated.

Next, we start the first asynchronous computation using `runContT`:

```haskell
  runContT ca $ \a -> do
    mb <- readRef rb
    case mb of
      Nothing -> writeRef ra $ Just a
      Just b -> k (f a b)
```

We provide a continuation, which checks whether the second value is available yet. If it is, we use the combining function to pass the final result to the continuation `k`. If not, we simply update the reference `ra` to contain the first value.

The case for the second computation is identical:

```haskell
  runContT cb $ \b -> do
    ma <- readRef ra
    case ma of
      Nothing -> writeRef rb $ Just b
      Just a -> k (f a b)
```

We can use the `par` combinator to read two files in parallel, or to issue two HTTP requests and wait on their results in parallel.

Here is a simple example which reads two text files in parallel, and concatenates and prints their results.

```haskell
import Control.Apply (lift2)

main = flip runContT print $
  par (lift2 (++)) (readFileCont "/tmp/1.txt")
                   (readFileCont "/tmp/2.txt")
```

Note that, since `readFileCont` returns a value of type `Either ErrorCode String`, we need to lift the operator `(++)` over the `Either` type constructor using `lift2` to form our combining function.

X> ## Exercises
X> 
X> 1. (Easy) Use `par` to make two HTTP requests and collect their response bodies in parallel. Your combining function should concatenate the two response bodies, and your continuation should use `trace` to print the result to the console.
X> 1. (Medium) Write a function
X> 
X>     ```haskell
X>     race :: forall a eff. 
X>       ContRef eff a -> 
X>       ContRef eff a -> 
X>       ContRef eff a
X>     ```
X>
X>     which executes two computations in parallel, and returns the result from the computation completes first.
X> 
X>     _Hint_: use a reference to store a `Boolean` indicating whether or not a result has been returned yet.
X>
X> 1. (Medium) Use your `race` function in conjunction with your `setTimeoutCont` function to define a function
X> 
X>     ```haskell
X>     timeout :: forall a eff. 
X>       Milliseconds -> 
X>       ContRef (timeout :: Timeout | eff) a -> 
X>       ContRef (timeout :: Timeout | eff) (Maybe a)
X>     ```
X> 
X>     which returns `Nothing` if the specified computation does not provide a result within the given number of milliseconds.

## An Applicative Functor For Parallelism

The type of the `par` combinator looks a lot like the type of `lift2` for the `ContRef eff` monad. In fact, it is possible to define a new applicative functor for which `par` is _exactly_ `lift2`, and we can define it simply in terms of `par` and `ContRef eff`.

You might be wondering why we don't define an `Applicative` instance for `ContRef eff` directly in terms of `par`. There are two reasons:

- If a type constructor has a `Monad` instance, then it is usually expected that the `Monad` and `Applicative` instances agree, in the sense that `(<*>)` is equivalent to the following function:

    ```haskell
    ap :: forall m a b. (Monad m) => m (a -> b) -> m a -> m b
    ap mf ma = do
      f <- mf
      a <- ma
      return (f a)
    ``` 
      
    However, our hypothetical `Applicative` instance would differ from the `Monad` instance in the amount of parallelism: `(<*>)` evaluates its arguments in parallel, whereas `ap` waits for its first computation to complete before executing the second.
- PureScript does not allow type class instances for types which involve row types. Since the `Eff` monad is parameterized by a row of effects, and in this case that row has to contain the `Ref` effect, it would be impossible to define an `Applicative` instance for `ContRef`.

Instead, we will create a newtype wrapper for `ContRef eff`, called `Parallel eff`, as follows:

```haskell
newtype Parallel eff a = Parallel (ContRef eff a)
```

We can write a function which turns a `Parallel` computation into a computation in the `ContRef eff` monad, by simply removing the outer data constructor:

```haskell
runParallel :: forall eff a. Parallel eff a -> ContRef eff a
runParallel (Parallel c) = c
```

The type class instances can, for the most part, be copied from the corresponding instances for `ContT`. However, in the case of the `Apply` type class, we make sure to use `par` to redefine `(<*>)`:

```haskell
instance functorParallel :: Functor (Parallel eff) where
  (<$>) f (Parallel c) = Parallel (f <$> c)

instance applyParallel :: Apply (Parallel eff) where
  (<*>) (Parallel f) (Parallel x) = Parallel (par ($) f x)

instance applicativeParallel :: Applicative (Parallel eff) where
  pure a = Parallel $ pure a
```

In the definition of the `Apply` instance, we use `par` to combine a function with its argument by using function application `($)` as the combining function.

We can now rewrite our example above to read two files in parallel by using the `Parallel` type constructor:

```haskell
import Control.Apply (lift2)

main = flip runContT print $ runParallel $
  lift2 (++) <$> Parallel (readFileCont "/tmp/1.txt")
             <*> Parallel (readFileCont "/tmp/2.txt")
```

Because applicative functors support lifting of functions of arbitrary arity, we can perform more computations in parallel by using the applicative combinators. We can also benefit from all of the standard library functions which work with applicative functors, such as `traverse` and `sequence`!

We can also combine parallel computations with sequential portions of code, by using applicative combinators in a do notation block, or vice versa, using `Parallel` and `runParallel` to change type constructors where appropriate. 

X> ## Exercises
X> 
X> 1. (Easy) Use the `traverse` function to write a function `readMany` which, given an array of file names, reads their contents in parallel and returns an array of strings representing their contents.
X> 1. (Easy) Use your `race` combinator to write an `Alt` instance for `Parallel eff`. Can you write an instance for `Alternative`?
X> 1. (Medium) Rewrite the parallel file IO example to use `ErrorT` for error handling, instead of lifting `(++)` with `lift2`. Your solution should use the `ErrorT` transformer to transform the `Parallel` functor.
X>
X>     Use this approach to modify your `readMany` function.
X> 1. (Difficult, Extended) Suppose we are given a collection of JSON documents on disk, such that each document contains an array of references to other files on disk:
X> 
X>     ```javascript
X>     { references: ['/tmp/1.json', '/tmp/2.json'] }
X>     ```
X>     
X>     Write a utility which takes a single filename as input, and spiders the JSON files on disk referenced transitively by that file, collecting a list of all referenced files.
X> 
X>     Your utility should use the `purescript-foreign` library to parse the JSON documents, and should fetch files referenced by a single file in parallel.

## Conclusion

In this chapter, we have seen a practical demonstration of monad transformers:

- We saw how the common JavaScript idiom of callback-passing can be captured by the `ContT` monad transformer.
- We saw how the problem of callback hell can be solved by using do notation to express sequential asynchronous computations, and an applicative functor to express parallelism.
- We used `ErrorT` to express _asynchronous errors_.
