# Domain-Specific Languages

## Chapter Goals

In this chapter, we will explore the implementation of _domain-specific languages_ (or _DSLs_) in PureScript, using a number of standard techniques. 

A domain-specific language is a language which is well-suited to development in a particular problem domain. Its syntax and functions are chosen to maximize readability of code used to express ideas in that domain. We have already seen a number of examples of domain-specific languages in this book:

- The `Game` monad and its associated actions, developed in chapter 11, constitute a domain-specific language for the domain of _text adventure game development_.
- The library of combinators which we wrote for the `ContT` and `Parallel` functors in chapter 12 could be considered an example of a domain-specific language for the domain of _asynchronous programming_.
- The `purescript-quickcheck` package, covered in chapter 13, is a domain-specific language for the domain of _generative testing_. Its combinators enable a particularly expressive notation for test properties.

This chapter will take a more structured approach to some of standard techniques in the implementation of domain-specific languages. It is by no means a complete exposition of the subject, but should provide you with enough knowledge to build some practical DSLs for your own tasks.

Our running example will be a domain-specific language for creating HTML documents. Our aim will be to develop a type-safe language for describing correct HTML documents, and we will work by improving a naive implementation in small steps.

## Project Setup

The project accompanying this chapter adds one new Bower dependency - the `purescript-free` library, which defines the _free monad_, one of the tools which we will by using.

The source code for this project can be built using Grunt.

## A HTML Data Type

The most basic version of our HTML library is defined in the `Data.DOM.Simple` module. The module contains the following type definitions:

```haskell
newtype Element = Element
  { name         :: String
  , attribs      :: [Attribute]
  , content      :: Maybe [Content]
  }

data Content
  = TextContent String
  | ElementContent Element

newtype Attribute = Attribute
  { key          :: String
  , value        :: String
  }
```

The `Element` type represents HTML elements. Each element consists of an element name, an array of attribute pairs and some content. The content property uses the `Maybe` type to indicate that an element might be open (containing other elements and text) or closed.

The key function of our library is a function

```haskell
render :: Element -> String
```

which renders HTML elements as HTML strings. We can try out this version of the library by constructing values of the appropriate types explicitly in `psci`:

```text
> import Data.DOM.Simple
> import Data.Maybe

> render $ Element 
    { name: "p"
    , attribs: [
        Attribute 
          { key: "class"
          , value: "main" 
          }
      ]
    , content: Just [
        TextContent "Hello World!"
      ] 
    }
  
"<p class=\"main\">Hello World!</p>"
```

As it stands, there are several problems with this library:

- Creating HTML documents is difficult - every new element requires at least one record and one data constructor.
- It is possible to represent invalid documents:
    - The developer might mistype the element name
    - The developer can associate an attribute with the wrong type of element
    - The developer can use a closed element when an open element is correct
    
In the remainder of the chapter, we will apply certain techniques to solve these problems and turn our library into a usable domain-specific language for creating HTML documents.

## Smart Constructors

The first technique we will apply is simple but can be very effective. Instead of exposing the representation of the data to the module's users, we can use the module exports list to hide the `Element`, `Content` and `Attribute` data constructors, and only export so-called _smart constructors_, which construct data which is known to be correct.

Here is an example. First, we provide a convenience function for creating HTML elements:

```haskell
element :: String -> [Attribute] -> Maybe [Content] -> Element
element name attribs content = Element
  { name:      name
  , attribs:   attribs
  , content:   content
  }
```

Next, we create smart constructors for those HTML elements we want our users to be able to create, by applying the `element` function:

```haskell
a :: [Attribute] -> [Content] -> Element
a attribs content = element "a" attribs (Just content)

div :: [Attribute] -> [Content] -> Element
div attribs content = element "div" attribs (Just content)

p :: [Attribute] -> [Content] -> Element
p attribs content = element "p" attribs (Just content)

img :: [Attribute] -> Element
img attribs = element "img" attribs Nothing
```

Finally, we update the module exports list to only export those functions which are known to construct correct data structures:

```haskell
module Data.DOM.Smart
  ( Element()
  , Attribute(..)
  , Content(..)

  , a
  , div
  , p
  , img

  , render
  ) where
```

The module exports list is provided immediately after the module name inside parentheses. Each module export can be one of three types:

- A value (or function), indicated by the name of the value,
- A type class, indicated by the name of the class, 
- A type constructor and any associated data constructors, indicated by the name of the type followed by a parenthesized list of exported data constructors.

Here, we export the `Element` _type_, but we do not export its data constructors. If we did, the user would be able to construct invalid HTML elements.

In the case of the `Attribute` and `Content` types, we still export all of the data constructors (indicated by the symbol `..` in the exports list). We will apply the technique of smart constructors to these types shortly.

Notice that we have already made some big improvements to our library:

- It is impossible to represent HTML elements with invalid names (of course, we are restricted to the set of element names provided by the library).
- Closed elements cannot contain content by construction.

We can apply this technique to the `Content` type very easily. We simply remove the data constructors for the `Content` type from the exports list, and provide the following smart constructors:

```haskell
text :: String -> Content
text = TextContent

elem :: Element -> Content
elem = ElementContent
```

Let's apply the same technique to the `Attribute` type. First, we provide a general-purpose smart constructor for attributes. Here is a first attempt:

```haskell
(:=) :: String -> String -> Attribute
(:=) key value = Attribute
  { key: key
  , value: value
  }
```

This representation suffers from the same problem as the original `Element` type - it is possible to represent attributes which do not exist or whose names were entered incorrectly. To solve this problem, we can create a newtype which represents attribute names:

```haskell
newtype AttributeKey = AttributeKey String
```

With that, we can modify our operator as follows:

```haskell
(:=) :: AttributeKey -> String -> Attribute
(:=) (AttributeKey key) value = Attribute
  { key: key
  , value: value
  }
```

If we do not export the `AttributeKey` data constructor, then the user has no way to construct values of type `AttributeKey` other than by using functions we explicitly export. Here are some examples:

```haskell
href :: AttributeKey
href = AttributeKey "href"

_class :: AttributeKey
_class = AttributeKey "class"

src :: AttributeKey
src = AttributeKey "src"

width :: AttributeKey
width = AttributeKey "width"

height :: AttributeKey
height = AttributeKey "height"
```

Here is the final exports list for our new module. Note that we no longer export any data constructors directly:

```haskell
module Data.DOM.Smart
  ( Element()
  , Attribute()
  , Content()
  , AttributeKey()

  , a
  , div
  , p
  , img

  , href
  , _class
  , src
  , width
  , height

  , (:=)
  , text
  , elem

  , render
  ) where
```

If we try this new module in `psci`, we can already see massive improvements in the conciseness of the user code:

```text
> import Data.DOM.Smart
> render $ p [ _class := "main" ] [ text "Hello World!" ]
  
"<p class=\"main\">Hello World!</p>"
```

Note, however, that no changes had to be made to the `render` function, because the underlying data representation never changed. This is one of the benefits of the smart constructors approach - it allows us to separate the internal data representation for a module from the representation which is perceived by users of its external API.

X> ## Exercises
X> 
X> 1. (Easy) Use the `Data.DOM.Smart` module to experiment by creating new HTML documents using `render`.
X> 1. (Medium) Some HTML attributes such as `checked` and `disabled` do not require values, and may be rendered as _empty attributes_:
X> 
X>     ```html
X>     <input disabled>
X>     ```
X> 
X>     Modify the representation of an `Attribute` to take empty attributes into account. Write a function which can be used in place of `(:=)` to add an empty attribute to an element. 

## Phantom Types

To motivate the next technique, consider the following code:

```text
> import Data.DOM.Phantom
> render $ img [ src    := "cat.jpg"
               , width  := "foo"
               , height := "bar" 
               ]
  
"<img src=\"cat.jpg\" width=\"foo\" height=\"bar\" />"
```

The problem here is that we have provided string values for the `width` and `height` attributes, where we should only be allowed to provide numeric values in units of pixels or percentage points.

To solve this problem, we can introduce a so-called _phantom type_ argument to our `AttributeKey` type:

```haskell
newtype AttributeKey a = AttributeKey String
```

The type variable `a` is called a _phantom type_ because there are no values of type `a` involved in the right-hand side of the definition. The type `a` only exists to provide more information at compile-time. Any value of type `AttributeKey a` is simply a string at runtime, but at compile-time, the type of the value tells us the desired type of the values associated with this key.

We can modify the type of our `(:=)` operator to take the new form of `AttributeKey` into account:

```haskell
(:=) :: forall a. (IsValue a) => AttributeKey a -> a -> Attribute
(:=) (AttributeKey key) value = Attribute
  { key: key
  , value: toValue value
  }
```

Here, the phantom type argument `a` is used to ensure that the attribute key and attribute value have compatible types. Since the user cannot create values of type `AttributeKey a` directly (only via the constants we provide in the library), every attribute will be correct by construction.

Note that the `IsValue` constraint ensures that whatever value type we associate to a key, its values can be converted to strings and displayed in the generated HTML. The `IsValue` type class is defined as follows:

```haskell
class IsValue a where
  toValue :: a -> String
```

We also provide type class instances for the `String` and `Number` types:

```haskell
instance stringIsValue :: IsValue String where
  toValue = id

instance numberIsValue :: IsValue Number where
  toValue = show
```

We also have to update our `AttributeKey` constants so that their types reflect the new type parameter:

```haskell
href :: AttributeKey String
href = AttributeKey "href"

_class :: AttributeKey String
_class = AttributeKey "class"

src :: AttributeKey String
src = AttributeKey "src"

width :: AttributeKey Number
width = AttributeKey "width"

height :: AttributeKey Number
height = AttributeKey "height"
```

Now we find it is impossible to represent these invalid HTML documents, and we are forced to use numbers to represent the `width` and `height` attributes instead:

```text
> import Data.DOM.Phantom
> render $ img [ src    := "cat.jpg"
               , width  := 100
               , height := 200 
               ]
  
"<img src=\"cat.jpg\" width=\"100\" height=\"200\" />"
```

X> ## Exercises
X> 
X> 1. (Easy) Create a data type which represents either pixel or percentage lengths. Write an instance of `IsValue` for your type. Modify the `width` and `height` attributes to use your new type.
X> 1. (Difficult) By defining type-level representatives for the Boolean values `true` and `false`, we can use a phantom type to encode whether an `AttributeKey` represents an _empty attribute_ such as `disabled` or `checked`.
X> 
X>     ```haskell
X>     data True
X>     data False
X>     ```
X> 
X>     Modify your solution to the previous exercise to use a phantom type to prevent the user from using the `(:=)` operator with an empty attribute.

## The Free Monad

In our final set of modifications to our API, we will use a construction called the _free monad_ to turn our `Content` type into a monad, enabling do notation. This will allow us to structure our HTML documents in a form in which the nesting of elements becomes clearer - instead of this:

```haskell
p [ _class := "main" ]
  [ elem $ img 
      [ src    := "cat.jpg"
      , width  := 100
      , height := 200 
      ]
  , text "A cat"
  ]
```

we will be able to write this:

```haskell
p [ _class := "main" ] $ do
  elem $ img 
    [ src    := "cat.jpg"
    , width  := 100
    , height := 200 
    ]
  text "A cat"
```

However, do notation is not the only benefit of a free monad. The free monad allows us to separate the _representation_ of our monadic actions from their _interpretation_, and even support _multiple interpretations_ of the same actions.

The `Free` monad is defined in the `purescript-free` library, in the `Control.Monad.Free` module. We can find out some basic information about it using `psci`, as follows:

```text
> import Control.Monad.Free
> :k Free
(* -> *) -> * -> *
```

The kind of `Free` indicates that it takes a type constructor as an argument, and returns another type constructor. In fact, the `Free` monad can be used to turn any `Functor` into a `Monad`!

We begin by defining the _representation_ of our monadic actions. To do this, we need to create a `Functor` with one data constructor for each monadic action we wish to support. In our case, our two monadic actions will be `elem` and `text`. In fact, we can simply modify our `Content` type as follows:

```haskell
data ContentF a
  = TextContent String a
  | ElementContent Element a

instance functorContentF :: Functor ContentF where
  (<$>) f (TextContent s a) = TextContent s (f a)
  (<$>) f (ElementContent e a) = ElementContent e (f a)
```

Here, the `ContentF` type constructor looks just like our old `Content` data type - however, it now takes a type argument `a`, and each data constructor has been modified to take a value of type `a` as an additional argument. The `Functor` instance simply applies the function `f` to the component of type `a` in each data constructor.

With that, we can define our new `Content` type constructor as a newtype around the `Free` monad, which we construct by using our `ContentF` type constructor as the first type argument:

```haskell
newtype Content a = Content (Free ContentF a)
```

Here, we use a newtype to avoid exposing the internal representation of our library to our users - by hiding the `Content` data constructor, we restrict our users to only using the monadic actions we provide.

Because `ContentF` is a `Functor`, we automatically get a `Monad` instance for `Free ContentF`, and we can lift this instance to a `Monad` instance on `Content`:

```haskell
runContent :: forall a. Content a -> Free ContentF a
runContent (Content x) = x

instance functorContent :: Functor Content where
  (<$>) f (Content x) = Content (f <$> x)

instance applyContent :: Apply Content where
  (<*>) (Content f) (Content x) = Content (f <*> x)

instance applicativeContent :: Applicative Content where
  pure = Content <<< pure

instance bindContent :: Bind Content where
  (>>=) (Content x) f = Content (x >>= (runContent <<< f))

instance monadContent :: Monad Content
```

We have to modify our `Element` data type slightly to take account of the new type argument on `Content`. We will simply require that the return type of our monadic computations be `Unit`:

```haskell
newtype Element = Element
  { name         :: String
  , attribs      :: [Attribute]
  , content      :: Maybe (Content Unit)
  }
```

In addition, we have to modify our `elem` and `text` functions, which become our new monadic actions for the `Content` monad. To do this, we can use the `liftF` function, provided by the `Control.Monad.Free` module. Here is its (simplified) type:

```haskell
liftF :: forall f a. (Functor f) => f a -> Free f a
```

`liftF` allows us to construct an action in our free monad from a value of type `f a` for some type `a`. In our case, we can simply use the data constructors of our `ContentF` type constructor directly: 

```haskell
text :: String -> Content Unit
text s = Content $ liftF $ TextContent s unit

elem :: Element -> Content Unit
elem e = Content $ liftF $ ElementContent e unit
```

Some other routine modifications have to be made, but the interesting changes are in the `render` function, where we have to _interpret_ our free monad.

## Interpreting the Monad

The `Control.Monad.Free` module provides a number of functions for interpreting a computation in a free monad:

```haskell
go :: forall f a. (Functor f) => 
  (f (Free f a) -> Free f a) -> 
  Free f a -> 
  a
  
goM :: forall f m a. (Functor f, Monad m) => 
  (f (Free f a) -> m (Free f a)) -> 
  Free f a -> 
  m a
  
iterM :: forall f m a. (Functor f, Monad m) => 
  (forall a. f (m a) -> m a) -> 
  Free f a -> 
  m a
```

The `go` function is useful when you want to use the free monad to compute a _pure_ result. The `goM` and `iterM` functions allow us to use a monad to interpret the actions of our free monad. The two functions differ slightly in the type of their interpretation functions, but we will use the `iterM` function. The interested reader might like to try reimplementing our code using the `goM` function instead.

First, we have to choose a monad in which we can interpret our actions. We will use the `Writer String` monad to accumulate a HTML string as our result.

Our new `render` method starts by delegating to a helper function, `renderElement`, and using `execWriter` to run our computation in the `Writer` monad:

```haskell
render :: Element -> String
render e = execWriter $ renderElement e
```

`renderElement` is defined in a where block:

```haskell
  where
  renderElement :: Element -> Writer String Unit
  renderElement (Element e) = do
```

The definition of `renderElement` is straightforward, using the `tell` action from the `Writer` monad to accumulate several small strings:

```haskell
    tell "<"
    tell e.name
    for_ e.attribs $ \a -> do
      tell " "
      renderAttribute a
    renderContent e.content
```

Next, we define the `renderAttribute` function, which is equally simple:

```haskell
    where
    renderAttribute :: Attribute -> Writer String Unit
    renderAttribute (Attribute a) = do
      tell a.key
      tell "=\""
      tell a.value
      tell "\""
```

The `renderContent` function is more interesting. Here, we use the `iterM` function to interpret the computation inside the free monad, delegating to a helper function, `renderContentItem`:

```haskell
    renderContent :: Maybe (Content Unit) -> Writer String Unit
    renderContent Nothing = tell " />"
    renderContent (Just (Content content)) = do
      tell ">"
      iterM renderContentItem content
      tell "</"
      tell e.name
      tell ">"
```

The type of `renderContentItem` can be deduced from the type signature of `iterM`. The functor `f` is our type constructor `ContentF`, and the monad `m` is the monad in which we are interpreting the computation, namely `Writer String`. This gives the following type signature for `renderContentItem`:

```haskell
    renderContentItem :: forall a. ContentF (Writer String a) -> Writer String a
```

We can implement this function by simply pattern matching on the two data constructors of `ContentF`:

```haskell
    renderContentItem (TextContent s rest) = do
      tell s
      rest
    renderContentItem (ElementContent e rest) = do
      renderElement e
      rest
```

In each case, the expression `rest` has the type `Writer String a`, and represents the remainder of the interpreted computation. We can complete each case by calling the `rest` action.

That's it! We can test our new monadic API in `psci`, as follows:

```text
> import Data.DOM.Free
> render $ p [] $ do
    elem $ img [ src := "cat.jpg" ]
    text "A cat"
  
"<p><img src=\"cat.jpg\" />A cat</p>"
```

X> ## Exercises
X> 
X> 1. (Medium) Add a new data constructor to the `ContentF` type to support a new action `comment`, which renders a comment in the generated HTML. Implement the new action using `liftF`. Update the interpretation `renderContentItem` to interpret your new constructor appropriately.
X> 1. (Difficult) One problem with the `goM` and `iterM` functions is that they are not _stack-safe_ - large monadic actions can result in stack overflows when interpreted. However, the `Control.Monad.Free` library provides the `go` and `goEff` functions which are stack-safe. Use the `goEff` function to interpret the `Content` monad by using the `ST` effect in place of the `Writer` monad. 

## Extending the Language

A monad in which every action returns something of type `Unit` is not particularly interesting. In fact, aside from an arguably nicer syntax, our monad adds no extra functionality over a `Monoid`.

Let's illustrate the power of the free monad construction by extending our language with a new monadic action which returns a non-trivial result.

Suppose we want to generate HTML documents which contain hyperlinks to different sections of the document using _anchors_. We can accomplish this already, by generating anchor names by hand and including them at least twice in the document: once at the definition of the anchor itself, and once in each hyperlink. However, this approach has some basic issues:

- The developer might fail to generate unique anchor names.
- The developer might mistype one or more instances of the anchor name.

In the interest of protecting the developer from their own mistakes, we can introduce a new type which represents anchor names, and provide a monadic action for generating new unique names.

The first step is to add a new type for names:

```haskell
newtype Name = Name String

runName :: Name -> String
runName (Name n) = n
```

Again, we define this as a newtype around `String`, but we must be careful not to export the data constructor in the module's export lists.

Next, we define an instance for the `IsValue` type class for our new type, so that we are able to use names in attribute values:

```haskell
instance nameIsValue :: IsValue Name where
  toValue (Name n) = n
```

We also define a new data type for hyperlinks which can appear in `a` elements, as follows:

```haskell
data Href
  = URLHref String
  | AnchorHref Name

instance hrefIsValue :: IsValue Href where
  toValue (URLHref url) = url
  toValue (AnchorHref (Name nm)) = "#" ++ nm
```

With this new type, we can modify the value type of the `href` attribute, forcing our users to use our new `Href` type. We can also create a new `name` attribute, which can be used to turn an element into an anchor:

```haskell
href :: AttributeKey Href
href = AttributeKey "href"

name :: AttributeKey Name
name = AttributeKey "name"
```

The remaining problem is that our users currently have no way to generate new names. We can provide this functionality in our `Content` monad. First, we need to add a new data constructor to our `ContentF` type constructor:

```haskell
data ContentF a
  = TextContent String a
  | ElementContent Element a
  | NewName (Name -> a)
```

The `NewName` data constructor corresponds to an action which returns a value of type `Name`. Notice that instead of requiring a `Name` as a data constructor argument, we require the user to provide a _function_ of type `Name -> a`. Remembering that the type `a` represents the _rest of the computation_, we can build an intuition that this function provides a way to continue computation after a value of type `Name` has been returned.

We also need to update the `Functor` instance for `ContentF`, taking into account the new data constructor, as follows:

```haskell
instance functorContentF :: Functor ContentF where
  (<$>) f (TextContent s a) = TextContent s (f a)
  (<$>) f (ElementContent e a) = ElementContent e (f a)
  (<$>) f (NewName k) = NewName (f <<< k)
```

Now we can build our new action by using the `liftF` function, as before:

```haskell
newName :: Content Name
newName = Content $ liftF $ NewName id
```

Notice that we provide the `id` function as our continuation, meaning that we return the result of type `Name` unchanged.

Finally, we need to update our interpretation function, to interpret the new action. We previously used the `Writer String` monad to interpret our computations, but that monad does not have the ability to generate new names, so we must switch to something else. The `RWS` monad can provide the capabilities of the `Writer`, but also enables pure state. We can define our interpretation monad as a type synonym to keep our type signatures short:

```haskell
type Interp = RWS Unit String Number
```

Remember that the `RWS` monad takes three type arguments: the first is the global configuration, which in our case is the trivial `Unit` type; the second is the type of the "log", which in our case is the HTML string we are accumulating; the final argument is the state type, and in our case we will use a number which will act as an incrementing counter, used to generate unique names.

Because the `Writer` and `RWS` monads use the same type class members to abstract their actions, we do not need to change any actions - we only need to replace every reference to `Writer String` with `Interp`. We do, however, need to modify the handler used to run our computation. Instead of `execWriter`, we now need to use `evalRWS`:

```haskell
render :: Element -> String
render e = snd $ evalRWS (renderElement e) unit 0
```

The call to `snd` makes sure that we only return the _second component_ of the `Tuple` returned from `evalRWS`, which in this case represents the accumulated HTML string.

We also need to add a new case to `renderContentItem`, to interpret the new `NewName` data constructor:

```haskell
    renderContentItem (NewName k) = do
      n <- get
      let name = Name $ "name" ++ show n
      put $ n + 1
      k name
```

Here, we are given a continuation `k` of type `Name -> Interp a`, and we need to construct an interpretation of type `Interp a`. Our interpretation is simple: we use `get` to read the state, and use that state to generate a unique name, then use `put` to increment the state. Finally, we pass our new name to the continuation to complete the computation.

With that, we can try out our new functionality in `psci`, by generating a unique name inside the `Content` monad, and using it as both the name of an element and the target of a hyperlink:

```text
> import Data.DOM.Name
> render $ p [ ] $ do
    top <- newName
    elem $ a [ name := top ] $ 
      text "Top"
    elem $ a [ href := AnchorHref top ] $ 
      text "Back to top"
  
"<p><a name=\"name0\">Top</a><a href=\"#name0\">Back to top</a></p>"
```

You can verify that multiple calls to `newName` do in fact result in unique names.

X> ## Exercises
X> 
X> 1. (Medium) We can simplify the API further by hiding the `Element` type from its users. Make these changes in the following steps:
X>     
X>     - Combine functions like `p` and `img` (with return type `Element`) with the `elem` action to create new actions with return type `Content Unit`. 
X>     - Change the `render` function to accept an argument of type `Content a` and return a result of type `Tuple a String`.
X> 1. (Difficult) Modify the `ContentF` type to support a new action
X> 
X>     ```haskell
X>     isMobile :: Content Boolean
X>     ```
X> 
X>     which returns a boolean value indicating whether or not the document is being rendered for display on a mobile device.
X> 
X>     _Hint_: use the `ask` action and the `Reader` component of the `RWS` monad to interpret this action.

## Conclusion

In this chapter, we developed a domain-specific language for creating HTML documents, by incrementally improving a naive implementation using some standard techniques:

- We used _smart constructors_ to hide the details of our data representation, only permitting the user to create documents which were _correct-by-construction_.
- We used an _user-defined infix binary operator_ to improve the syntax of the language.
- We used _phantom types_ to encode additional information in the types of our data, preventing the user from providing attribute values of the wrong type.
- We used the _free monad_ to turn our array representation of a collection of content into a monadic representation supporting do notation. We then extended this representation to support a new monadic action, and interpreted the monadic computations in the `Writer` and `RWS` monads.

These techniques all leverage PureScript's module and type systems, either to prevent the user from making mistakes or to improve the syntax of the domain-specific language.

The implementation of domain-specific languages in functional programming languages is an area of active research, but hopefully this provides a useful introduction some simple techniques, and illustrates the power of working in a language with expressive types.
