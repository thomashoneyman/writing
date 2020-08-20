+++
title = "How to Write PureScript React Components to Replace JavaScript"
category = "PureScript"
date = "2019-06-26"
slug = "replace-react-components-with-purescript"
discourse = "https://discourse.purescript.org/t/post-how-to-replace-react-components-using-purescripts-react-libraries/"
description = "PureScript helps reduce bugs and improve the stability of complex applications. Learn how to incrementally take over a small React application using Purescript's React Basic library while writing idiomatic code in both languages."
+++

I have twice helped migrate large JavaScript apps to PureScript. At CitizenNet we replaced Angular with Halogen and at Awake Security we've replaced a React application with PureScript React. Both companies have experienced a dramatic drop in bugs and a commensurate rise in maintainability.

It's relatively easy to replace React because PureScript has several React libraries (including support for Hooks and React Native). These libraries range from the low-level bindings of the `react` library to the higher-level `react-basic` and `concur` libraries (to name just a few). The React mental model fits well with a strongly-typed, pure functional language like PureScript -- after all, React was originally implemented in a functional language (Standard ML).

Using one of these libraries to write React applications in PureScript makes it easy to share components between languages with little-to-no modification. At Awake Security we share internationalization, a Redux store and middleware, and much more in a code base where PureScript regularly imports JavaScript and JavaScript regularly imports PureScript.

PureScript is an ideal choice for incrementally rewriting a React application in a typed language. In this article I will demonstrate how to replace part of a React application with simple components written in PureScript. Along the way I'll share best practices for making this interop convenient and dependable. The examples will be simple, but the same techniques also apply to complex components and libraries like Redux.

For a full application written with PureScript and React, I recommend {{< external-link "https://github.com/jonasbuntinx/purescript-react-realworld" "the Real World PureScript React application" >}}. The application we'll work on in this article is absurdly simple for demonstration purposes and doesn't capture what makes PureScript great for writing React applications as much as that larger example does.

We'll use `react-basic-hooks` in this article to demonstrate how you can transform the interface of idiomatic PureScript code into idiomatic JavaScript (and vice versa).

#### Sections

1. [Write a tiny React application in JavaScript](#let-s-write-a-react-app-in-javascript)
2. [Update the application to support PureScript](#setting-up-a-shared-purescript-javascript-project)
3. [Replace a React component with React Basic Hooks](#replacing-a-react-component-with-the-react-basic-hooks-library)
4. [Introduce more complex types](#introducing-more-complex-types)

I encourage you to code along with this article; no code is omitted and dependencies are pinned to help ensure the examples are reproducible. This code uses Node `v12.0.0` and NPM `v6.14.0` installed globally. All other tooling is installed locally.

There are two example applications written based on this article:

- Peter Murphy expanded on the previous version of this article {{< external-link "https://github.com/ptrfrncsmrph/purescript-react-tutorial" "using react-basic-hooks" >}}.
- Andys8 implemented the previous version of this article using the `create-react-app` TypeScript template. {{< external-link "https://github.com/andys8/create-react-app-purescript" "His implementation" >}} motivated an update to this article which uses the same setup.

The {{< external-link "https://web.archive.org/web/20200626090853/https://thomashoneyman.com/articles/replace-react-components-with-purescript" "previous version of this article" >}} used the `react` and `react-basic` libraries. I think `react` is a bit too low-level for most folks, and `react-basic` has now split into `react-basic-hooks` and `react-basic-classic`. Due to those changes I've updated this article to focus on `react-basic-hooks` alone.

{{< subscribe >}}

## Let's write a React app in JavaScript

We are going to write a tiny React application which shows a counter and then we'll replace the implementation with PureScript. The remaining JavaScript will only differ in imports from the original code but everything will be PureScript under the hood.

First, let's follow the official React docs and bootstrap a project with `create-react-app`. We'll leave everything as-is and introduce our PureScript code within this structure.

```sh
# Create the app
npx create-react-app my-app --template typescript && cd my-app
```

We're using the TypeScript template because it includes some conveniences for importing non-native JavaScript (like TypeScript or PureScript) into JavaScript. But we aren't going to write any TypeScript in this article, and you certainly don't need `create-react-app` or TypeScript to write your React applications in PureScript.

At the time of writing, `create-react-app` produces these React and TypeScript dependencies:

```json
"dependencies": {
    "react": "^16.13.1",
    "react-dom": "^16.13.1",
    "react-scripts": "3.4.3",
    "typescript": "^3.7.5"
  }
```

Feel free to delete the test and service workers files in `src` -- we won't be using them.

### Writing a React component

Let's write our first React component: a counter. This is likely the first example of a React component you ever encountered; it's the first example in the PureScript React libraries as well.

Start by creating a new `components` directory to hold our React components and a `Counter.js` file within it for our counter:

```sh
mkdir src/components && touch src/components/Counter.js
```

We're going to implement a counter component which maintains a count internally, displays it, and provides a button to increment the count. It's almost identical to the example used in the introduction to React's 'Hooks at a Glance' documentation. Copy this implementation into `Counter.js`:

```jsx
// src/components/Counter.js
import React, { useState } from "react";

const Counter = (props) => {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>{props.label}</button>
    </div>
  );
};

export default Counter;
```

Now, let's rename the `App.tsx` file which `create-react-app` generated for us to `App.js` and change its contents to import our new counter:

```js
// App.js
import React from "react";
import Counter from "./components/Counter";
import "./App.css";

const App = () => {
  return (
    <div className="App">
      <Counter label="Click me!" />
    </div>
  );
};

export default App;
```

With `npm start` we can run the dev server and see our app in action:

![running app](/images/2019/running-app.gif)

## Setting up a shared PureScript & JavaScript project

We've set up a new React project with JavaScript (and support for TypeScript, if we want). But it doesn't take much effort to support PureScript as well.

Our goal is to write code in either language and freely import in either direction without friction. To accomplish that we will add PureScript to our Webpack setup, install PureScript tooling, and rely on the compiler to generate JavaScript code we can import into our existing JavaScript app.

### 1. Install the compiler and package manager

First we need to install PureScript tooling: the compiler itself, `purs`, as well as the build tool and package manager {{< external-link "https://github.com/purescript/spago" "Spago" >}}. (You may also be interested in the {{< external-link "https://discourse.purescript.org/t/recommended-tooling-for-purescript-in-2020" "Recommended Tooling for PureScript in 2020" >}} post on Discourse for more tooling suggestions.)

```sh
# Install the compiler and the Spago package manager
npm install --save-dev purescript@0.13.8 spago@0.16.0
```

### 2. Initialize the PureScript project

We can create a new PureScript project with `spago init`. This will fetch the latest package set for PureScript.

```sh
npx spago init
```

Spago has created a `packages.dhall` file which points at the set of packages which can be installed and a `spago.dhall` file which lists the packages we've actually installed.

If you would like to use the same package set as I use in this article, update your `packages.dhall` file to point at this upstream:

```dhall
let upstream =
      https://github.com/purescript/package-sets/releases/download/psc-0.13.8-20200724/packages.dhall sha256:bb941d30820a49345a0e88937094d2b9983d939c9fd3a46969b85ce44953d7d9
```

Future versions of Spago will support initializing with this package set directly with the `--tag` flag, but this feature is unreleased at the time of writing:

```sh
npx spago init --tag psc-0.13.8-20200724
```

Next, before we install anything, add these entries to the existing `.gitignore` file to cover PureScript:

```sh
output .psc* .purs* .spago
```

It's common to namespace a PureScript application under the application name. We'll do that by creating a folder named `MyApp` to hold our PureScript application:

```sh
mkdir src/MyApp
```

That's all we need for a standard PureScript setup. But using `create-react-app` gives us the opportunity to hook in to the nice things the React team has already built. In the next section we're going to set things up so we can:

- Import PureScript modules directly into JavaScript (instead of importing build artifacts from the `output` directory)
- Support the same hot module replacement provided by `npm start`, except it will compile our PureScript, too

### 3. Integrate PureScript into our create-react-app setup

First, we're going to install {{< external-link "https://github.com/gsoft-inc/craco" "craco" >}} ("Create React App Configuration Override") and its PureScript loader, which will allow us to tweak our `create-react-app` settings to support PureScript.

```sh
npm install --save @craco/craco
npm install --save-dev craco-purescript-loader
```

Add a `craco.config.js` file in the root of the repository with these contents:

```js
const cracoPureScriptLoader = require("craco-purescript-loader");

module.exports = {
  plugins: [
    {
      plugin: cracoPureScriptLoader,
      options: {
        spago: true,
        pscIde: true,
      },
    },
  ],
};
```

Update your `package.json` to call `craco` instead of `react-scripts`:

```diff
--- a/package.json
+++ b/package.json
@@ -17,10 +17,10 @@
     "typescript": "^3.7.5"
   },
   "scripts": {
-    "start": "react-scripts start",
-    "build": "react-scripts build",
-    "test": "react-scripts test",
-    "eject": "react-scripts eject"
+    "start": "craco start",
+    "build": "craco build",
+    "test": "craco test",
+    "eject": "craco eject"
   },
   "eslintConfig": {
     "extends": "react-app"
```

Finally, add a TypeScript definition file that makes `create-react-app` aware of PureScript modules:

```ts
// src/purescript-module.d.ts
declare module "*.purs";
```

To be clear, it's completely unnecessary to use `create-react-app` or its TypeScript conveniences to write a PureScript React app. None of the projects I've worked on professionally do this (we import PureScript code directly out of the `output` directory created by the compiler, which is blocked by `create-react-app` by default). We're only using this setup because it's convenient for bootstrapping new hybrid JS / PS projects.

From this point on you can use `npm start` at any time to build the project and view a dev server running with hot module replacement. Any changes to your PureScript or JavaScript will be picked up.

## Replacing a React component with the React Basic Hooks Library

I like to follow four simple steps when replacing a JavaScript React component with a PureScript one:

1. Write the component in idiomatic PureScript.
2. Write an 'interop' module for the component, which will provide the JavaScript interface and conversion functions between PureScript and JavaScript types and idioms.
3. Compile the PureScript to generate JavaScript in the `output` directory
4. Import the resulting code from the interop module as if it were a regular JavaScript component.

This pattern lets us write components in ordinary PureScript, with a richer set of types than are available to us in JavaScript, and then convert the interface of that component into something compatible with regular JavaScript. Our goal is to write idiomatic code in both languages -- the core PureScript component shouldn't have to know that it's going to be used in JavaScript, and our JavaScript code shouldn't have to know we have any PureScript code at all.

Even better, the interop module serves as a canary to ensure we change our JavaScript if the PureScript component's interface changes.

We'll use the `react-basic-hooks` library to implement this pattern. There's also `react-basic-classic` for folks who prefer class components; both libraries can be used together.

As we take each step in this process I'll explain more about why it's necessary and some best practices to keep in mind. Let's start by installing the `react-basic-hooks` and `react-basic-dom` libraries:

```sh
# Install the react-basic-hooks and react-basic-dom libraries
npx spago install react-basic-hooks react-basic-dom

# Install the interpolate library for simple string interpolation
npx spago install interpolate
```

### 1. Write the React component in idiomatic PureScript

We're now going to implement the same counter in PureScript. This component will ultimately be imported and used in JavaScript, but fortunately we don't have to care about that just yet -- we're just going to write ordinary PureScript code. Our interop module will handle the conversion to an idiomatic JavaScript interface.

I've implemented the same counter (with the same props) below. Copy its contents into the PureScript tree at `src/MyApp/Components/Counter.purs`:

```hs
module MyApp.Components.Counter where

import Prelude

import Data.Interpolate (i)
import Data.Tuple.Nested ((/\))
import React.Basic.DOM (button, div_, p_, text)
import React.Basic.Events (handler_)
import React.Basic.Hooks (Component, component, useState')
import React.Basic.Hooks as Hooks

type Props = { label :: String }

mkCounter :: Component Props
mkCounter = component "Counter" \props -> Hooks.do
  count /\ setCount <- useState' 0

  pure do
    div_
      [ p_ [ text $ i "You clicked " count " times" ]
      , button
          { onClick: handler_ $ setCount (count + 1)
          , children: [ text props.label ]
          }
      ]
```

In a PureScript codebase this is all we need; we could use this component by importing `mkCounter` and providing it with its props. We can almost use this component in JavaScript, too, but we need to take one more step.

Component creation is inherently effectful, which means that we need to run this effect to have a component ready to import into JavaScript. We'll talk more about how PureScript represents effects later when we dive deeper into the interop module.

For the time being we'll use `unsafePerformEffect` to run the effect that creates our component. We won't ever use this function in our PureScript code, but it makes it possible to import this module directly into JavaScript:

```diff
--- a/src/MyApp/Components/Counter.purs
+++ b/src/MyApp/Components/Counter.purs
@@ -6,5 +6,6 @@ import Data.Interpolate (i)
 import Data.Tuple.Nested ((/\))
+import Effect.Unsafe (unsafePerformEffect)
 import React.Basic.DOM (button, div_, p_, text)
 import React.Basic.Events (handler_)
-import React.Basic.Hooks (Component, component, useState')
+import React.Basic.Hooks (Component, JSX, component, useState')
 import React.Basic.Hooks as Hooks
@@ -13,2 +14,5 @@ type Props = { label :: String }

+jsCounter :: Props -> JSX
+jsCounter = unsafePerformEffect counter
+
 mkCounter :: Component Props
```

Now we can import this component directly into our existing `App.js` file. Note how we only have to change the import, nothing else! PureScript compiles our code to the `output` directory, and our configuration makes those modules importable directly into our JavaScript files.

```diff
--- a/src/App.js
+++ b/src/App.js
@@ -1,5 +1,5 @@
 import React from "react";
-import Counter from "./components/Counter";
+import { jsCounter as Counter } from "./MyApp/Components/Counter.purs";
 import "./App.css";

 const App = () => {
```

Run `npm start` and you ought to see the same counter as before. With our component now implemented in PureScript we don't need our JavaScript version anymore:

```sh
rm src/components/Counter.js
```

We've successfully taken over part of our JavaScript app with PureScript.

### 2. Write an interop module for the component

This component is compatible with our JavaScript right away because it only uses simple JavaScript types as props and the users of our counter component provided the `label` prop that we require.

What if a user forgets to provide a label to the component, though?

```diff
--- a/src/App.js
+++ b/src/App.js
@@ -7,3 +7,3 @@ const App = () => {
     <div className="App">
-      <Counter label="Click me!" />
+      <Counter />
     </div>
```

Oops!

![forgotten label prop](/images/2019/forgot-prop.png)

Well, a button with no contents is not _good_, but it's not as bad as the entire app crashing -- which is what will happen if we tried to run functions expecting a `String` on an `undefined` value. The problem is that the `String` type doesn't quite capture what values are likely to arrive from JavaScript.

As a general rule I expect people to write JavaScript the way they usually do, which means using built in types, regular uncurried functions, and to sometimes omit information and supply `null` or `undefined` instead. This is why at Awake Security we provide an interop module for components that will be used in JavaScript code.

These modules:

1. Provide a mapping between PureScript types used in the component and a simple JavaScript representation
2. Add a layer of safety by marking all inputs that could reasonably be `null` or `undefined` with the `Nullable` type, which helps our code handle missing values gracefully
3. Translate functions in their curried form into usual JavaScript functions, and translates effectful functions (represented as thunks in generated code) into functions that run immediately when called
4. Serve as a canary for changes in PureScript code that will affect dependent JavaScript code so that you can be extra careful

Over the remainder of this article we'll explore each of these techniques.

First, let's deal with the possibility that a JavaScript consumer forgets to provide the `label` prop and explicitly handle what should happen when it is omitted. Let's create an interop module for our component:

```sh
mkdir src/MyApp/Components/Counter
touch src/MyApp/Components/Counter/Interop.purs
```

Typically, each interop module will contain at least these parts:

1. A JavaScript-compatible interface for our props (`JSProps`)
2. A function for converting from the JavaScript interface to richer PureScript types (`jsPropsToProps`)
3. A function to export our component for JavaScript, which will use the JavaScript interface and convert it internally to the PureScript interface (`counter`)

Delete `jsCounter` from the existing `Counter` component so that the PureScript file doesn't have any code to accommodate JavaScript. Then, write a new interop module with these contents:

```hs
module MyApp.Components.Counter.Interop where

import Prelude

import Data.Maybe (fromMaybe)
import Data.Nullable (Nullable, toMaybe)
import Effect.Unsafe (unsafePerformEffect)
import MyApp.Components.Counter (Props, counter)
import React.Basic (JSX)

type JSProps = { label :: Nullable String }

jsPropsToProps :: JSProps -> Props
jsPropsToProps { label } = { label: fromMaybe "Click away!" $ toMaybe label }

jsCounter :: JSProps -> JSX
jsCounter = unsafePerformEffect counter <<< jsPropsToProps
```

We have created a new interface for our component, `JSProps`, which will be used in JavaScript instead of our PureScript interface, `Props`. We've also created a function which translates between the two interfaces, and produced a new component which uses the JavaScript interface instead of the PureScript one.

Marking the `label` prop as `Nullable` makes the compiler aware the string may not exist. It then forces us to explicitly handle the `null` or `undefined` case before we can treat the prop as a usual `String`. We'll need to handle the null case in order to map our new `JSProps` type to our component's expected `Props` type. To do that, we convert `Nullable` to `Maybe` and then provide a fallback value to use when the prop does not exist.

The `Nullable` type is explicitly for interop with JavaScript, but it doesn't always behave exactly the way you would expect. It does not map directly to the ordinary `Maybe` type. You should usually convert any `Nullable` types to `Maybe` as soon as possible. {{< external-link "https://github.com/purescript-contrib/purescript-nullable" "Check out the nullable library if you'd like to learn more about this" >}}.

Now let's update our existing code to accommodate the change. Update the import in `App.js`:

```diff
--- a/src/App.js
+++ b/src/App.js
@@ -1,5 +1,5 @@
 import React from "react";
-import { jsCounter as Counter } from "./MyApp/Components/Counter.purs";
+import { jsCounter as Counter } from "./MyApp/Components/Counter/Interop.purs";
 import "./App.css";

 const App = () => {
```

We can verify that if the JavaScript side forgets to provide a `label` prop we still render something reasonable:

![forgot prop fixed](/images/2019/forgot-prop-fixed.png)

In this case our interop module simply marked a single field as `Nullable`. But it's common for the JavaScript interface to diverge slightly from the PureScript interface it is translating. Keeping a separate interop module makes it easy to do this without affecting the core component.

It also ensures that any changes to the underlying component are reflected as type errors in the interop file rather than (potentially) silently breaking JavaScript code. It's easy to become lazy about this when you're used to the compiler warning you of the effect changes in one file will have in another!

If you use TypeScript, Justin Woo has {{< external-link "https://qiita.com/kimagure/items/4847685d02d4b15a556c" "written a piece on transparently sharing types with Typescript from PureScript" >}} which is worth reading.

## Introducing more complex types

What happens when you have a more complicated type you need to construct from JavaScript? Let's say, for example, that our counter component needs two new pieces of information:

1. An effectful callback function to run after the counter is clicked
2. A type which represents whether the function should increment or decrement on click

We can apply the same process to accommodate the new features. We will write idiomatic PureScript in our component module and then write a translation in the interop module. The end result will be a component equally usable in PureScript code or in JavaScript code, without compromising how you write code in either language.

```diff
--- a/src/MyApp/Components/Counter.purs
+++ b/src/MyApp/Components/Counter.purs
@@ -3,23 +3,50 @@ module MyApp.Components.Counter where
 import Prelude

 import Data.Interpolate (i)
+import Data.Maybe (Maybe(..))
 import Data.Tuple.Nested ((/\))
+import Effect (Effect)
 import React.Basic.DOM (button, div_, p_, text)
 import React.Basic.Events (handler_)
 import React.Basic.Hooks (Component, component, useState')
 import React.Basic.Hooks as Hooks

-type Props = { label :: String }
+type Props =
+  { label :: String
+  , onClick :: Int -> Effect Unit
+  , counterType :: CounterType
+  }
+
+data CounterType = Incrementer | Decrementer
+
+counterTypeToString :: CounterType -> String
+counterTypeToString = case _ of
+  Incrementer -> "incrementer"
+  Decrementer -> "decrementer"
+
+counterTypeFromString :: String -> Maybe CounterType
+counterTypeFromString = case _ of
+  "incrementer" -> Just Incrementer
+  "decrementer" -> Just Decrementer
+  _ -> Nothing

 mkCounter :: Component Props
 mkCounter = component "Counter" \props -> Hooks.do
   count /\ setCount <- useState' 0

+  let
+    step n = case props.counterType of
+      Incrementer -> n + 1
+      Decrementer -> n - 1
+
   pure do
     div_
       [ p_ [ text $ i "You clicked " count " times" ]
       , button
-          { onClick: handler_ $ setCount (count + 1)
+          { onClick: handler_ do
+              let next = step count
+              setCount next
+              props.onClick next
           , children: [ text props.label ]
           }
       ]
```

With these changes our counter can decrement or increment and can run an arbitrary effectful function after the click event occurs. But we can't run this from JavaScript: there is no such thing as a `CounterType` in JavaScript, and a normal JavaScript function like...

```js
function onClick(ev) {
  console.log("clicked!");
}
```

will not work if supplied as the callback function. It's up to our interop module to smooth things out.

I'll make the code changes first and describe them afterwards:

```diff
--- a/src/MyApp/Components/Counter/Interop.purs
+++ b/src/MyApp/Components/Counter/Interop.purs
@@ -5,12 +5,24 @@ import Prelude
 import Data.Maybe (fromMaybe)
 import Data.Nullable (Nullable, toMaybe)
+import Effect.Uncurried (EffectFn1, runEffectFn1)
 import Effect.Unsafe (unsafePerformEffect)
-import MyApp.Components.Counter (Props, counter)
+import MyApp.Components.Counter (CounterType(..), Props, counter, counterTypeFromString)
 import React.Basic (JSX)

-type JSProps = { label :: Nullable String }
+type JSProps =
+  { label :: Nullable String
+  , onClick :: Nullable (EffectFn1 Int Unit)
+  , counterType :: Nullable String
+  }

 jsPropsToProps :: JSProps -> Props
-jsPropsToProps { label } = { label: fromMaybe "Click away!" $ toMaybe label }
+jsPropsToProps props =
+  { label:
+      fromMaybe "Click away!" $ toMaybe props.label
+  , onClick:
+      fromMaybe mempty $ map runEffectFn1 $ toMaybe props.onClick
+  , counterType:
+      fromMaybe Incrementer $ counterTypeFromString =<< toMaybe props.counterType
+  }

 jsCounter :: JSProps -> JSX
```

First, I updated the JavaScript interface to include the two new fields our component accepts.

I decided to represent `CounterType` as a lowercase string `"incrementer"` or `"decrementer"` and guard against both the case in which the value isn't provided (`Nullable`) or the provided value doesn't make sense (it can't be parsed by `counterTypeFromString`). In either case the component will default to incrementing.

I also decided to represent `onClick` as a potentially missing value. But instead of a usual function, I'm representing the value as an `EffectFn1`: an effectful, uncurried function of one argument.

That type merits a little extra explanation. In PureScript, functions are curried by default and _effectful_ functions are represented as a thunk. Therefore, these two PureScript functions:

```hs
add :: Int -> Int -> Int
log :: String -> Effect Unit
```

...do not correspond to functions that can be called in JavaScript as `add(a, b)` or `log(str)`. Instead, they translate more closely to:

```js
// each function has only one argument, and multiple arguments are represented
// by nested functions of one argument each.
const add = (a) => (b) => a + b;

// effectful functions are thunked so they can be passed around and manipulated
// without being evaluated.
const log = (str) => () => console.log(str);
```

This is an unusual programming style for JavaScript. So PureScript provides helpers for exporting functions that feel more natural.

- The `Fn*` family of functions handles pure functions of _N_ arguments
- The `EffectFn*` family of functions handles effectful functions of _N_ arguments
- A few other translating functions exist; for example, you can turn `Aff` asynchronous functions into JavaScript promises and vice versa.

If we rewrite our PureScript definitions to use these helpers:

```hs
add :: Fn2 Int Int Int
log :: EffectFn1 String Unit
```

then we will get a more usual JavaScript interface:

```js
const add = (a, b) => a + b;
const log = (str) => console.log(str);
```

Without using `EffectFn1`, JavaScript code using our counter component would have to supply a thunked callback function like this:

```jsx
<Counter onClick={(count) => () => console.log("clicked: ", n)} />
```

With `EffectFn1` in place, however, we can provide usual code:

```jsx
<Counter onClick={(count) => console.log("clicked: ", n)} />
```

Let's take advantage of our new component features by updating `App.js`. Our first component will omit all props except for the `onClick` callback, which will log the count to the console. The next one will specify a decrementing counter. The last component will stick to the original interface and just provide a label.

```diff
--- a/src/App.js
+++ b/src/App.js
@@ -7,4 +7,6 @@ const App = () => {
     <div className="App">
       <Counter />
+      <Counter onClick={(n) => console.log("clicked: ", n)} />
+      <Counter counterType="decrementer" label="Click me!" />
     </div>
   );
```

![running app with decrement and function](/images/2019/running-app-2.gif)

## Wrapping Up

We replaced a simple counter in this article, but the same steps apply to more complex components as well.

- Write the PureScript component using whatever types and libraries you would like.
- Then, write an interop module for the component which translates between JavaScript and PureScript.
- Compile the result.
- Import it into your JavaScript code like any other React component.

Interop between React and PureScript becomes more involved when you introduce global contexts like a Redux store, but this bootstrapping work remains largely out of view in day-to-day coding.

If you're faced with a tangled React app and wish you had better tools, try introducing PureScript.

{{< subscribe >}}
