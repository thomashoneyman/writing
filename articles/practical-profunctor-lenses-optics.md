+++
title = "Practical Profunctor Lenses & Optics In PureScript"
category = "PureScript"
date = "2019-08-08"
slug = "practical-profunctor-lenses-optics"
discourse = "https://discourse.purescript.org/t/post-practical-profunctor-lenses-optics-in-purescript/904?u=thomashoneyman"
description = "Profunctor lenses and optics are among the most rewarding tools in PureScript. Armed with a handful of types and functions, you can manipulate even complex data structures with minimal effort."
reviewers = [
  { author = "Liam Goodacre", url = "https://github.com/liamgoodacre" },
  { author = "Jordan Martinez", url = "https://github.com/jordanmartinez" },
  { author = "Vance Palacio", url = "https://github.com/vanceism7" },
  { author = "Nicholas Scheel", url = "https://github.com/monoidmusician" },
  { author = "Colin Wahl", url = "https://github.com/colinwahl" }
]
+++

Optics are among the most rewarding tools in functional programming. You can use them to manipulate values within data structures without boilerplate -- you just need a few types and functions from an optics library like `profunctor-lenses`.

Consider working with a deeply-nested data structure like this one:

```hs
type NestedData =
  Maybe (Array { foo :: Tuple String (Either Int Boolean), bar :: String })
```

Faced with this type, ask yourself:

* What code would I write to update the deeply-nested `Int` value, given an index in the outer array? Would my code be terse, readable, and maintainable?
* How many functions would it take to get, set, and modify the `Array`, `foo`, or `bar` values? The inner values within `foo`?
* How quickly and easily could I update my code if this type changed -- if, for example, `Maybe (Array { foo :: ... })` became `Array { foo :: Maybe ... }`? How many functions would I have to update?

If you disliked your answers, read on: profunctor optics help you manipulate any layer of even complex structures like this one with ease.

This article will teach you to be productive with profunctor lenses, even if you haven't used them before. You'll learn how to use the four most common optics -- **lenses**, **prisms**, **traversals**, and **isos** -- to solve practical problems.

If you're looking for a more thorough introduction, I recommend Brian Marick's book {{< external-link "https://leanpub.com/lenses" "Lenses for the Mere Mortal" >}}.

#### Sections

1. [A brief demonstration of profunctor optics in action](#profunctor-lenses-optics-in-action)
2. [Cheatsheet: common types, functions, and constructors](#cheatsheet-types-functions-and-constructors)
3. [`Lens`, lens functions, constructing lenses, and `At` lenses](#lenses)
4. [`Prism`, prism functions, and constructing prisms](#prisms)
5. [Tips for composing lenses, prisms, and other optics](#tips-for-composing-optics)
6. [`Traversal`, traversal functions, and `Index` traversals](#traversals)
7. [`Iso`, iso functions, and constructing isos](#isos)
8. [Related resources & wrapping up](#wrapping-up)

## Profunctor Lenses & Optics In Action

Let's use our deeply-nested data type to prove the flexibility and power of profunctor optics.

```hs
type NestedData =
  Maybe (Array { foo :: Tuple String (Either Int Boolean), bar :: String })
```

To retrieve or update the `Int` value within `NestedData`, given an index in the outer array, we'd likely write a pair of functions like these:

```hs
getNestedInt :: Int -> NestedData -> Maybe Int
getNestedInt index nested = case nested >>= (_ !! index) of
  Nothing -> Nothing
  Just { foo: Tuple _ eitherInt } -> case eitherInt of
    Left int -> Just int
    Right _ -> Nothing

modifyNestedInt :: Int -> (Int -> Int) -> NestedData -> NestedData
modifyNestedInt index modifyFn nested = case _ of
  Nothing -> Nothing
  Just array -> Array.modifyAt index modifyFoo array
  where
  modifyFoo record@{ foo: Tuple str eitherInt } = case eitherInt of
    Left int -> record { foo = Tuple str (Left (modifyFn int)) }
    Right _ -> record
```

We can do better. Let's replace both functions with a single optic.

Our optic will support much more than just getting or setting values, and we'll construct it from several smaller optics in a single line of code.

Let's start with those small optics. Each optic below manages a single layer of structure. Except for `_foo`, they're all pre-defined in the `profunctor-lenses` library (along with most other common data types and type classes in PureScript). By convention, optics are prefixed with an underscore.

```hs
-- Access the second element of a tuple
_2 :: forall a b. Lens' (Tuple a b) b

-- Access the value of a record at the label "foo"
_foo :: forall a r. Lens' { foo :: a | r } a

-- Access the value within the `Just` constructor of a `Maybe`,
-- if it exists
_Just :: forall a. Prism' (Maybe a) a

-- Access the value within the `Left` constructor of an `Either`,
-- if it exists
_Left :: forall a b. Prism' (Either a b) a

-- Access the nth index of an `Array`, if it exists
ix :: forall a. Int -> Traversal' (Array a) a
```

We can use the above optics to deal with a single layer of structure at a time. To work with _many_ layers of structure, we compose optics together. Most people read this composition left-to-right, where `<<<` represents moving a single layer deeper.

For example, you can read the composed optic below as "go through the `Just` layer, then the `Left` layer." You can also read it right-to-left: "The optic focuses on the value in a `Left`, which is itself within a `Just`."

```hs
_LeftWithinJust :: forall a b. Prism' (Maybe (Either a b)) a
_LeftWithinJust = _Just <<< _Left
```

We now know how to compose optics, so let's write one which focuses on the  `Int` inside our `NestedData` type.

The function below produces the optic we want, given a particular index in the outer array. Notice how the composition, read left-to-right, lines up with the layers of structure in the type. `_Just` lines up with `Maybe`, `ix` lines up with `Array`, and so on.

```hs
-- the same type we've been working with, for reference
type NestedData =
  Maybe (Array { foo :: Tuple String (Either Int Boolean), bar :: String })

-- This function constructs an optic which accesses the `Int` value
-- within our deeply-nested type, if it exists
_NestedInt :: Int -> Traversal' NestedData Int
_NestedInt index = _Just <<< ix index <<< _foo <<< _2 <<< _Left
```

With this optic in hand, we can replace both of our previous functions with simple one-liners:

```hs
getNestedInt :: Int -> NestedData -> Maybe Int
getNestedInt index = preview (_NestedInt index)

modifyNestedInt :: Int -> (Int -> Int) -> NestedData -> NestedData
modifyNestedInt index = over (_NestedInt index)
```

...and that's it! We now have simple, maintainable replacements for our original functions.

Profunctor lenses also help us adapt to changes in our data types. Let's handle a significant change to `NestedData`:

```diff
 type NestedData =
-  Maybe (Array { foo :: Tuple String (Either Int Boolean), bar :: String })
+  Array { foo :: Maybe (Tuple String (Either Int Boolean)), bar :: String }
```

Our composition is adaptable -- unlike our original functions. When the underlying structure changes, we can simply shuffle around or replace the optics we composed. Even better, since we used functions that work with most optics (`preview` and `over`), we only have to update the underlying optic for our code to work.

One optic serves as the source of truth for an entire family of functions for manipulating a value within structure. Even this significant change to the `NestedData` type only requires a trivial change to our code.

```diff
 _NestedInt :: Int -> Traversal' NestedData Int
-_NestedInt index = _Just <<< ix index <<< _foo <<< _2 <<< _Left
+_NestedInt index = ix index <<< _foo <<< _Just <<< _2 <<< _Left
```

With this simple re-ordering, our optics-based `getNestedInt` and `modifyNestedInt` functions handle the new `NestedData` type. Compare this to the effort of refactoring the original functions!

Let's recap what we've learned so far:

* Optics are a terse, flexible way to manipulate values within structure.
* Individual optics usually deal with a single layer of structure.
* Compose optics together to deal with many layers of structure, where the rightmost optic represents the deepest layer.
* Adapt to changes in data structures by adjusting the composition used to work with that structure
* Use an optic with optic functions like `preview` and `over` to perform tasks like getting, setting, modifying, and more.
* By convention, optics are prefixed with an underscore.

Optics are a deep, fascinating topic, but this article focuses on their practical benefits. Over the next several sections you'll learn more about how to use them to write elegant solutions to practical problems in your code.

{{< subscribe >}}

## Cheatsheet: Types, Functions, and Constructors

PureScript's flagship optics library is `profunctor-lenses`, which encompasses a family of optic types, functions, and constructors. You should understand optics from each of these three angles.

#### 1. Types

Use the **type** of an optic to understand its meaning: what kind of structures, values, and functions you can use it with.

Optics describe a relationship between a structure `s` and zero, one, or many values of type `a`, called the focus (or foci) of the optic.

For example, the optic `Lens' s a` denotes access to the single value `a` within the structure `s`. The `_2` lens we used earlier had the type `Lens' (Tuple a b) b`; in this lens, the tuple is the structure and the tuple's second element is the focus.

These four optics are used most often:

{{< table src="optics-types.csv" class="opticsTypes" >}}

#### 2. Functions

Use **library functions** with a given optic to get, set, modify, insert, delete, fold, or otherwise manipulate the focus of the optic.

A given function is compatible with only some optic types, which is part of what gives an optic type its meaning. Functions denote what you can _do_ with the focus of an optic. These are the most common ones, along with the type of optics they are used with:

{{< table src="optics-functions.csv" class="opticsFunctions" >}}

These functions are more nuanced than the table allows for; they're explained in much greater depth (and with examples) in the later lens, prism, traversal, and iso sections.

#### 3. Constructors

Use **constructors** to create optics for your data types. These constructors work for the four most common optics:

{{< table src="optics-constructors.csv" class="opticsConstructors" >}}

The `profunctor-lenses` library pre-defines optics for most common data types and type classes in PureScript. They include `Tuple`, `Maybe`, `Either`, `Newtype`, `Traversable`, and many more.

Don't worry if these tables are incomprehensible to you now; by the end of the article, you'll feel comfortable using them as a reference.

#### Setup

I wrote this article with PureScript v0.13.2, Spago v0.8.5, and the July 25, 2019 package set (psc-0.13.2-20190725). Code along by initializing a new project and installing the libraries I used:

```sh
spago init
spago install profunctor-lenses remotedata
```

You can start a repl with `spago repl`. I recommend using a `.purs-repl` file in the root of the project to automatically import the modules you need. {{< external-link "https://gist.github.com/thomashoneyman/0d50c736503fc9a5c72f8796e70d77d1" "I used this one." >}}

I must warn you: lenses are highly general and, without type annotations, you're likely to run into obscure errors in the repl. I recommend you:

* Define new lenses with the :paste command so you can give them the same type I did in this article (use Ctrl+D to end the paste)
* Annotate the optic or result type inline if you get a type error
* Or, write optics in a module in your editor (with type annotations), and then import the module into the repl
* Review "Identifying an optic from type spewage" from Marick's {{<  external-link "https://leanpub.com/lenses" "Lenses for the Mere Mortal" >}}

In real-world code, this is typically unnecessary because there is enough context for the compiler to infer all types. Now, let's turn to our first optic -- the namesake of the `profunctor-lenses` library!

## Lenses

Lenses are for product types like records and tuples. The type `Lens' s a` means that the structure `s` contains a single value of type `a`. Lenses are used to get, set, or modify the value `a` within structure when you _know_ it exists.

```hs
-- This lens focuses on the second element of a tuple and is implemented
-- in the Data.Lens.Lens.Tuple module.
_2 :: forall a b. Lens' (Tuple a b) b

-- This lens focuses on the "name" field of a record; we have to construct
-- this one ourselves.
_name :: forall a r. Lens' { name :: a | r } a
_name = prop (SProxy :: SProxy "name")
```

I will use these to demonstrate the common lens functions `view`, `set`, and `over`. Then, I'll show you how to construct lenses for your types by implementing the above optics from scratch.

### Common Functions For Working With Lenses

Use `view` with a lens `Lens' s a` to get the value `a` from the structure `s`. Lenses always contain their focused value, so this operation will always succeed.

```hs
-- We can use our `_2` lens to retrieve the second element of a tuple
snd :: forall a b. Tuple a b -> b
snd = view _2

> view _2 (Tuple "five" 10)
10

-- We can use our `_name` lens to retrieve the value of the 'name' field
-- from a record.
getName :: forall a r. { name :: a | r } -> a
getName = view _name

> view _name { name: "Tim", age: 40 }
"Tim"
```

Use `set` with a lens `Lens' s a` and a value of type `a` to overwrite the focused value `a` within the structure `s`.

```hs
-- We can use our `_2` lens to replace the second element of a tuple.
setSnd :: forall a b. b -> Tuple a b -> Tuple a b
setSnd = set _2

> set _2 5 (Tuple "five" 10)
Tuple "five" 5

-- We can use our `_name` lens as an alternative to record update syntax when
-- replacing the value of a field
setName :: forall a r. a -> { name :: a | r } -> { name :: a | r }
setName = set _name

> set _name "John" { name: "Tim", age: 40 }
{ name: "John", age: 40 }
```

Use `over` with a lens `Lens' s a` and a modifying function `(a -> a)` to modify the value `a` within `s`.

```hs
-- We can use our `_2` lens to modify the second element of a tuple
modifySnd :: forall a b. (b -> b) -> Tuple a b -> Tuple a b
modifySnd = over _2

> over _2 (_ + 5) (Tuple "five" 10)
Tuple "five" 15

-- We can use our `_name` lens as an alternative to record update syntax when
-- modifying the value of a field
modifyName :: forall a r. (a -> a) -> { name :: a | r } -> { name :: a | r }
modifyName = over _name

> over _name (_ <> " Smitherson") { name: "Tim", age: 40 }
{ name: "Tim Smitherson", age: 40 }
```

### Constructing Lenses

The `lens'` constructor produces a valid lens from a getter/setter function. This getter/setter function takes the structure `s` as its argument and produces a tuple containing the focused value `a` (the "getter") and a function which places `a` back within the structure (the "setter").

```hs
-- Note: this is a more concrete version of the type in `profunctor-lenses`
lens' :: forall s a. (s -> Tuple a (a -> s)) -> Lens' s a
```

We can use `lens'` to construct the `_2` and `_name` lenses we've been using.

The first lens focuses on the second element of a tuple. That means our getter/setter function will take a tuple as its argument, the getter part of our function should retrieve the second element of the tuple, and the setter part should overwrite it.

```hs
_2 :: forall a b. Lens' (Tuple a b) b
_2 =
  lens' \(Tuple first second) ->
    Tuple
      second -- here we get the second element
      (\b -> Tuple first b) -- and here we set the second element
```

The second lens focuses on the "name" field of a record. Our getter/setter function will take a record containing this label as its argument. The getter should retrieve the value of the field and the setter should overwrite it, returning the record. We can accomplish both tasks with record syntax.

```hs
_name :: forall a r. Lens' { name :: a | r } a
_name = lens' \record -> Tuple record.name (\new -> record { name = new })
```

We now have two valid lenses, but we should improve the second. It's common to write lenses for custom record types, but `lens'` has a lot of boilerplate. Fortunately, the `profunctor-lenses` library has a much nicer helper function for record lenses called `prop`.

The `prop` function only requires the name of the field you want to focus on. The field name must be provided as a symbol proxy; if you're unfamiliar with proxies, don't worry -- you'll soon get used to the pattern. Almost all record lenses are written like these examples.

Let's construct a few record lenses, including `_name`, using `prop`.

```hs
_name :: forall a r. Lens' { name :: a | r } a
_name = prop (SProxy :: SProxy "name")

_age :: forall a r. Lens' { age :: a | r } a
_age = prop (SProxy :: SProxy "age")

_location :: forall a r. Lens' { location :: a | r } a
_location = prop (SProxy :: SProxy "location")
```

### Related Classes: The `At` Lens

The `At` type class describes a special kind of lens used for keyed structures like maps and sets. This lens is different from the ones we've seen before; an `At` lens:

* requires that the structure `s` is a keyed structure, like a map or set
* must be constructed using the `at` constructor, which requires a key of the correct type for the structure
* lets you get, set, and modify (like usual lenses)
* also enables you to insert and delete elements
* focuses on a value of type `Maybe a`, rather than the simple `a` we've seen so far

Consider this `At` lens, created using the `at` constructor:

```hs
-- This lens focuses on the specific key "Tim" in a string-keyed collection
_Tim :: forall a. At s String a => Lens' s (Maybe a)
_Tim = at "Tim"

-- We can simplify the type by specializing it
_Tim :: forall a. Lens' (Map String a) (Maybe a)
```

We can use this `At` lens with the same common functions like `view`:

```hs
-- We can use our `_Tim` lens as an alternative to the `lookup` function for
-- maps and sets.
getTim :: forall a. Map String a -> Maybe a
getTim = view _Tim

> view _Tim (Map.singleton "Tim" 40)
Just 40

-- We can also replace the `update` function for maps and sets with `over`
updateTim :: forall a. (a -> a) -> Map String a -> Map String a
updateTim f = over _Tim (map f)

> over _Tim (map (_ + 20)) (Map.singleton "Tim" 40)
fromFoldable [ Tuple "Tim" 60 ]
```

But we can now do more than we could with an ordinary lens: we can also insert and delete values from the structure. If the lens is used with a `Just a` value, then the value will be updated or inserted. If used with a `Nothing` value, then the key will be removed from the structure (if it existed).

```hs
-- We can use our `_Tim` lens as an alternative to `insert` function
insertTim :: forall a. a -> Map String a -> Map String a
insertTim = set _Tim <<< Just

> set _Tim (Just 45) (Map.singleton "John" 20)
fromFoldable [ Tuple "John" 20, Tuple "Tim" 45 ]

-- In contrast, providing a `Nothing` value will delete the key (if it exists).
deleteTim :: forall a. Map String a -> Map String a
deleteTim = set _Tim Nothing

> set _Tim Nothing (Map.singleton "Tim" 40)
fromFoldable []
```

We're still using common lens functions, but the `At` lens has given us some extra capabilities. While normal `Lens'` gives us type-independent get, set, and modify operations, the `At` lens goes even further with insertion and deletion.

## Prisms

Prisms are used for sum types like `Maybe` and `Either`. The type `Prism' s a` means that the type `s` might contain a value of type `a`. Consider this pair of simple prisms:

```hs
-- This prism focuses on the value within the `Left` data constructor. It's
-- defined in `Data.Lens.Either`.
_Left :: forall a b. Prism' (Either a b) a

-- The `RemoteData err a` data type is often used in PureScript to represent
-- requests. It's implemented like this in the `remotedata` library:
data RemoteData err a = NotAsked | Loading | Error err | Success a

-- The library exports prisms for the sum type. This prism focuses on the value
-- within the `Success` data constructor. If you are using the `.purs-repl` file
-- for this article then you already have this prism in scope.
_Success :: forall err a. Prism' (RemoteData err a) a
```

I'll use these to explore the `preview` and `review` functions for working with prisms, and then I'll demonstrate how to construct prisms for your types with the `prism'` function.


### Common Functions For Working With Prisms

Use `preview` with a prism `Prism' s a` to retrieve a value of type `a` from a structure `s`, if it exists. The value will be returned as `Just a` if it exists, and `Nothing` otherwise.

```hs
-- Our `_Left` prism focuses on the left branch of an `Either`, which frequently
-- represents errors or failure cases.
getLeft :: forall a b. Either a b -> Maybe a
getLeft = preview _Left

> preview _Left (Left "five")
Just "five"

-- We can use our `_Success` prism to retrieve a successful result from a
-- request, if there is one.
getSuccess :: forall err a. RemoteData err a -> Maybe a
getSuccess = preview _Success

> preview _Success (Success { name: "Tim" })
Just { name: "Tim" }
```

I also use the helpful `is` function to test whether the focused value exists in the structure. For example:

```hs
-- check if the request succeeded, specializing the result to Boolean so it
-- can be shown in the repl
> is _Success (Success { name: "Tim" }) :: Boolean
true

-- this request has not been sent yet, so it did not succeed
> is _Success NotAsked :: Boolean
false
```

Next, use `review` with a prism `Prism' s a` to produce a structure `s` given a value `a`. It's a little like using `pure` to lift a value into an applicative or monadic context. It generalizes the idea of a data constructor; in the below cases we could also have used the `Left` and `Success` data constructors directly.

```hs
mkLeft :: forall a b. a -> Either a b
mkLeft = review _Left

-- create a `Left` from a `String`, specializing the right branch to a concrete
-- type so it can be shown in the repl
> (review _Left "Resource not found") :: Either String Void
Left "Resource not found"

mkSuccess :: forall err a. a -> RemoteData err a
mkSuccess = review _Success

> review _Success { name: "Tim" } :: RemoteData Void { name :: String }
Success { name: "Tim" }
```

Prisms can also use the lens functions we've already discussed: `view`, `set`, and `over`. These functions behave a little differently when used with a prism instead of a lens:

* `view` must return the focused value from the structure, but the value may not exist in a prism. Therefore, `view` requires that the focused value `a` has a `Monoid` instance and returns the empty value if it does not exist.
* `over` and `set` will manipulate the focused value if it exists, but will leave the structure unchanged if it does not exist.

Let's see each of these in practice:

```hs
-- view will return the focus value when it exists
> view _Success (Success { name: "Tim" })
{ name: "Tim" }

-- or `mempty` if it does not
> view _Success NotAsked :: { name :: String }
{ name: "" }

-- set and over will manipulate the focus value if it exists
> over _Left (_ <> " Smitherson") (Left "Tim") :: Either String Void
Left "Tim Smitherson"

-- and will leave the structure unchanged if it does not; contrast this to
-- the behavior of the `At` lens
> set _Left "John" (Right 10)
Right 10
```

### Constructing Prisms

You can write prisms for your types using the `prism'` constructor. This function produces a `Prism' s a` given a constructor function `a -> s` and a retrieval function `s -> Maybe a`. Remember that the focused value, `a`, is not guaranteed to exist in a prism!

* The constructor function is usually a data constructor like `Left` or `Success`.
* The retrieval function is usually a case statement which uses pattern-matching to retrieve the focused value `a`

Almost every prism definition in the wild looks the same. Let's construct the two prisms we've been using so far:

```hs
_Left :: forall a b. Prism' (Either a b) a
_Left = prism' Left case _ of
  Left a -> Just a
  _ -> Nothing

_Success :: forall err a. Prism' (RemoteData err a) a
_Success = prism' Success case _ of
  Success a -> Just a
  _ -> Nothing
```

## Tips For Composing Optics

Composition is what makes optics so flexible. Each optic focuses on a value within a single layer of structure. You can compose them one layer at a time to drill down through many layers of structure, and if the structure changes, you can simply remove, replace, or re-order the composed optics to accommodate the change. As a general rule, I recommend that you:

* Define optics to focus on one layer of structure at a time, and compose them to work with multiple layers of structure
* Define pre-composed optics to get and set data I use often; for example, I might define `_uuid` to retrieve a user's unique ID from a larger `User` type rather than write the composition over and over.
* Don't define concrete getter or setter functions; a function like `getUUID :: User -> UUID` prevents composition with other optics. Instead, define optics, and then use common functions like `view`, `preview`, and `over` with the optic directly.

When reading composed optics, make sure to think of the optic as describing the location of the value, or how to access it, but not a as chain of operations like getting and setting. For example, consider this composition:

```hs
_Just <<< _name <<< _2
```

If you read this as "get the second element of the tuple, then get the `name` field of the record, then get the value within the `Just` constructor," then you are reading it backward.

Instead, think of the composition as describing the location of the value: "The focus is the second element of a tuple, which is within the `name` field of a record, which is within the `Just` constructor of a `Maybe`."

Three common compositions describe most optics:

* Compose two optics of the same type, and you'll get the same type.
* Compose a lens and a prism, and you'll get an affine traversal.
* Compose almost anything with a traversal, and you'll get a traversal.

Finally, let's turn to traversals.

## Traversals

The type `Traversal' s a` means that the type `s` contains zero, one, or many values of `a`. A traversal can refer to one of two things:

* A traversal is used for collections like lists and maps, where the optic has zero, one, or many foci.
* An **affine** traversal is a more general form of a lens, prism, or composed optic, which has only zero or one focus.

You have a traversal either way, but an affine traversal is more similar in its use to a prism. Affine traversals are more general than lenses or prisms, however, and can't be used with as many functions.

Traversals that operate on collections are especially useful for:

* Applying a function to every focus in the collection (like `map`)
* Collapsing every focus in the collection to a single value (like `fold`)

Affine traversals are useful when you compose lenses and prisms; you'll most often use prism functions like `preview` and `over` with them.

```hs
-- This pre-defined optic is valid for any `Traversable`, which includes
-- arrays, lists, and maps. It's defined in Data.Lens.Traversal.
traversed :: forall t a. Traversable t => Traversal' (t a) a

-- This affine traversal is the composition of lenses and prisms we've already
-- seen. Composing a lens and a prism will produce an affine traversal.
_city :: forall a b r. Traversal' (Maybe { city :: Maybe a | r }) a
_city = _Just <<< prop (SProxy :: _ "city") <<< _Just
```

### Common Functions For Working With Traversals

Traversals can be used with many of the same lens and prism functions we've already seen, but can also be used with library functions meant for working with collections of values. Most of the traversal-specific functions you'll use are defined in the `Data.Lens.Fold` module.

Let's start with the familiar functions we've already worked with.

Be careful with `view`, which is defined to only return a single value `a` from a structure: traversals support zero, one, or many values of type `a`. Like a prism, if the focus doesn't exist then the monoidal empty value is returned. Like a lens, if a single focus exists, it is returned. What about collections like arrays, which contain many foci? In that case, the values are concatenated together to produce a single value.

By convention I:

* use `view` with affine traversals, which have at most one focus.
* use `foldOf` with traversals which may have many foci, like an array; it behaves the same as `view`, but has a more intuitive name for collections.

I'll follow this convention in the below examples.

```hs
-- when no element exists, the empty value is returned
> foldOf traversed [] :: String
""

> view _city (Just { city: Nothing }) :: String
""

-- when one element exists it is returned
> foldOf traversed [ "Gjelina" ]
"Gjelina"

> view _city (Just { city: Just "Los Angeles" })
"Los Angeles"

-- when many elements exist, they are concatenated
> foldOf traversed [ "Tim", " Smitherson" ]
"Tim Smitherson"
```

We can continue to use `set`, and `over` as we did with lenses and prisms; this time, however, the modification will apply to everything focused by the traversal, not just a single value of type `a`.

```hs
-- when no focus exists, the structure is unchanged
> set traversed 10 []
[]

> over _city (_ <> ", CA") Nothing
Nothing

-- when one focus exists, `set` and `over` behave exactly as they did with
-- lenses and prisms
> set traversed 10 [ 4 ]
[ 10 ]

> over _city (_ <> ", CA") (Just { city: Just "Los Angeles" })
Just { city: Just "Los Angeles, CA" }

-- when multiple foci exist, `set` and `over` will apply to every one
> set traversed 10 (Array.range 0 4)
[ 10, 10, 10, 10, 10 ]

> over traversed (\x -> x * x) (Array.range 0 4)
[ 0, 1, 4, 9, 16 ]
```

We can use `preview`, like with prisms, but we can't use `review` with a traversal. Be careful with `preview`: like `view`, it's designed to return at most one focus. When many foci exist, as they may in a traversal, it will return only the _first_ one. I prefer to:

* use `preview` for affine traversals, which have at most one focus
* use `firstOf` for traversals which have many foci; it behaves the same as `preview`, but has a more intuitive name for collections

I'll use that convention with our two traversals.

```hs
-- when no focus exists, `preview` and `toFirstOf` return `Nothing`
> firstOf traversed [] :: Maybe Void
Nothing

> preview _city Nothing :: Maybe Void
Nothing

-- when one focus exists, it is returned in `Just`
> firstOf traversed [ 1 ]
Just 1

> preview _city (Just { city: Just "Los Angeles" })
Just "Los Angeles"

-- when many foci exist, only the first is returned
> firstOf traversed [ 1, 2, 3, 4, 5 ]
Just 1
```

Among the many library functions defined in `Data.Lens.Fold`, the most often used are `lastOf`, which returns the last of many foci, and `toListOf`, which places the foci in an list. For example:

```hs
> lastOf traversed [ 1, 2 ]
Just 2

> toListOf _city (Just { city: Just "Los Angeles" })
"Los Angeles" : Nil
```

### Constructing Traversals

You usually don't need to construct a traversal directly, because:

* the `traversed` optic works for any member of the `Traversable` type class
* lenses and prisms are already traversals
* many other optics become traversals when composed together.

The `wander` function helps you construct a traversal for a data type which does not conform to `Traversable`, but still has parts you want to focus. It's rarely used outside the `profunctor-lenses` library, but it's good to know it exists.

### Related Classes: The `Index` Traversal

The `Index` type class describes a special kind of traversal which is used to focus on a single element within an indexed collection. This traversal is different from the ones we've seen before; an `Index` traversal:

* requires that the structure `s` is indexed, like an array, list, or map
* must be constructed using the `ix` constructor, which requires an index of the correct type for the structure
* lets you get, set, and modify just one focus among many in the collection -- but does not let you insert or remove elements, as `At` does, which makes it well-suited for fixed-size collections

You can use all the regular traversal functions with an `Index` traversal, but remember that there is only one focus. Here's a sample:

```hs
-- This index traversal focuses on the third index of a Traversable
_3 :: Index t Int a => Traversal' t a
_3 = ix 3

-- We can simplify the type by specializing to an array of strings
_3 :: Traversal' (Array String) String
```

The most common operations I take with an index traversal include:

* get values with `preview`
* set and modify values with `set` and `over`

```hs
-- We can use `preview` to look up the value by its index
getIndex3 :: Array String -> Maybe String
getIndex3 = preview _3

> preview _3 [ "Preux & Proper", "Gjelina", "Sonoratown", "Republique", "Bavel" ]
Just "Republique"

-- Use `set` and `over` to modify the value (if it exists)
setIndex3 :: String -> Array String -> Array String
setIndex3 = set _3

> set _3 "Bavel" []
[]

> over _3 (_ <> "!") [ "Gjelina", "Republique", "Bavel", "Sonoratown" ]
[ "Gjelina", "Republique", "Bavel", "Sonoratown!" ]
```

## Isos

Isos are the most constrained optic, simply enabling you to transform back and forth between two types without losing information. The type `Iso' s a` means that `s` and `a` are isomorphic -- the two types represent the same information. They're especially useful when you need to unwrap a newtype or transform an array to a list or otherwise convert between types. This optic is also useful for backward compatibility without boilerplate.

```hs
-- The most common iso used in practice is _Newtype, which allows you to
-- unwrap or wrap a newtype, usually composed with other optics. As usual,
-- this is simplified from the type in `profunctor-lenses`. The optic is
-- defined in Data.Lens.Iso.Newtype.
_Newtype :: forall s a. Newtype s a => Iso' s a
```

### Common Functions

Isos share similarities with lenses and prisms. If you have an iso `Iso' s a` then you can use `view` to produce an `a` given an `s`, like a lens, and you can use `review` to produce an `s` given an `a`, like a prism.

```hs
-- `view` can act as a replacement for the `unwrap` function from the `Newtype`
-- type class
unwrap :: forall s a. Newtype s a => s -> a
unwrap = view _Newtype

-- We'll need a type annotation here
> view (_Newtype :: Iso' (Additive Int) Int) (Additive 10)
10

-- `review` can act as a replacement for the `wrap` function
wrap :: forall s a. Newtype s a => s -> a
wrap = review _Newtype

> review (_Newtype :: Iso' (Additive Int) Int) 10
Additive 10
```

### Constructing Isos

Isos are constructed with the `iso` function. For a given `Iso' s a` it expects two conversion functions as arguments: one which converts an `s` to an `a`, and one which converts an `a` to an `s`. If you have written any conversion functions already, like `fromFoldable` or `unwrap`, then this is trivial; if not, start by writing conversion functions before constructing the iso.

```hs
-- we can re-use the `unwrap` and `wrap` functions from `Data.Newtype` to create
-- this iso
_Newtype :: forall s a. Newtype s a => Iso' s a
_Newtype = iso unwrap wrap
```

## Wrapping up

Optics are a powerful tool. With just a few functions for constructing and using optics, you can write more elegant code to manage complex structures. As you continue using optics, consider referring back [to the optics cheat sheet](#cheatsheet-types-functions-and-constructors) from the beginning of the article.

If you're curious to learn more about optics, I recommend these resources:

- {{< external-link "https://leanpub.com/lenses" "Lenses for the Mere Mortal" >}}, a book by Brian Marick.
- {{< external-link "http://oleg.fi/gists/posts/2017-04-18-glassery.html" "Glassery" >}}, detailed notes on profunctor optics by Oleg Grenrus.
- {{< external-link "https://artyom.me/#lens-over-tea" "Lens Over Tea" >}}, a series of tutorials on the Haskell `lens` library by Artyom Kazak (note: this uses a different formulation of lenses, but most of the ideas are the same)

If you've been helped by other resources, please suggest them for this list {{< external-link "https://discourse.purescript.org/t/post-practical-profunctor-lenses-optics-in-purescript/904?u=thomashoneyman" "on this post's discussion on the PureScript Discourse" >}}.

{{< subscribe >}}
