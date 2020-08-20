+++
title = "Introducing Halogen Hooks"
category = "PureScript"
date = "2020-03-31"
slug = "introducing-halogen-hooks"
discourse = "https://discourse.purescript.org/t/introducing-halogen-hooks/1279?u=thomashoneyman"
description = "Hooks are a new approach to stateful logic in Halogen. Hooks are a better alternative to higher-order and renderless components, and they offer a more convenient way to write many ordinary Halogen components."
reviewers = [
  { author = "Gary Burgess", url = "https://github.com/garyb" },
  { author = "Ben Hart", url = "https://github.com/benjmhart" },
  { author = "Jordan Martinez", url = "https://github.com/jordanmartinez" },
  { author = "Dave Zuch", url = "https://github.com/davezuch" }
]
+++

Components are the only way to use local state in Halogen, but they come with strings attached: you have to render something. Components are therefore ideal for writing stateful UI code, but ill-suited for writing reusable stateful logic.

Stateful logic shows up everywhere in Halogen. State is used for UI concerns like persisting how many times a button has been clicked or whether a modal is open. But state is needed for many use cases which have nothing directly to do with rendering. That includes interacting with external data sources, managing subscriptions, handling forms, and many other things.

If you try to handle these non-UI use cases for state using components you usually end up with:

* Types and logic duplicated among different components, which can be only reduced -- not eliminated -- with helper functions
* Complex patterns like higher-order and renderless components, solutions so unwieldy that most Halogen developers use them only as a last resort

{{< external-link "https://github.com/thomashoneyman/purescript-halogen-hooks" "Halogen Hooks" >}} are a new solution for writing reusable, stateful logic. Hooks are simple functions with access to Halogen features like local state. These stateful functions can produce values of any type, not just UI code. But they're no less powerful than components: we can turn a Hook that returns Halogen's `ComponentHTML` type into an ordinary component with a single function call.

**Hooks are a simpler mental model for writing code in Halogen.** In the Hooks model, applications are made up of ordinary PureScript functions and stateful Hooks functions. Components, in this model, are simply stateful functions that produce `ComponentHTML`.

You can start using Hooks today with the {{< external-link "https://github.com/thomashoneyman/purescript-halogen-hooks" "Halogen Hooks library">}}.

## Hooks In Action: UseWindowWidth

Let's say we need the current browser window width.

We will need to register an event listener on the window, store the width in state, update our state when the window resizes, and clean up our event listener when the component unmounts.

To implement this code we need component-only features like local state, initializers, and finalizers. But this code doesn't make sense as a component -- it's meant to be used *by* a component.

We do have some options: we could implement this code as a collection of helper functions and types for a component to import, or we could write a higher-order or renderless component.

But no existing solution in Halogen today can match this for convenience and readability:

```hs
myComponent :: forall q i o m. MonadAff m => H.Component HH.HTML q i o m
myComponent = Hooks.component \_ _ -> Hooks.do
  width <- useWindowWidth -- our custom Hook
  Hooks.pure do
    HH.p_ [ HH.text $ "Window width is " <> maybe "" show width ]
```

This code is readable: we use the window width and render it as paragraph text. When the width changes the text will re-render.

We've written a simple hook that returns `ComponentHTML`, so we can use the `Hooks.component` function to turn it into an ordinary component.

The underlying `useWindowWidth` hook takes care of all the complicated logic necessary to subscribe to the window width and it simply returns the width itself. One function call is all we need to use it.

We've now seen how to use a hook to reuse stateful logic, but how would you actually implement one?

## Implementing UseWindowWidth

Hooks are functions that can opt in to component features like state, side effects, and queries. Let's break down what our `useWindowWidth` hook will need:

* We need to use local state to persist the window width
* We need to use a side effect to subscribe to window events when the component initializes and unsubscribe when it finalizes.

We can capture those two features using a newtype, which will also be used to uniquely identify our new Hook.

```hs
newtype UseWindowWidth hooks =
  UseWindowWidth (UseEffect (UseState (Maybe Int) hooks))

derive instance newtypeUseWindowWidth :: Newtype (UseWindowWidth hooks) _
```

This type represents using local state of the type `Maybe Int`, and then using a side effect. If we needed to, we could use more independent states and effects, or mix in other Hook types.

Next, let's turn to our new Hook's type signature.

```hs
useWindowWidth
  :: forall m
   . MonadAff m
  => Hook m UseWindowWidth (Maybe Int)
--   [1]    [2]            [3]
```

1. A `Hook` is a (possibly) stateful function which can run effects from some monad `m`, uses a particular set of hooks, and returns a value.
2. Our Hook type, `UseWindowWidth`, uniquely identifies this Hook and specifies what Hooks are used internally. The Hooks library will unwrap this newtype and verify the correct Hooks were used in the correct order in the implementation.
3. This `Hook` returns a `Maybe Int`: the current window width.

Now let's turn to our implementation, taken from the {{< external-link "https://github.com/thomashoneyman/purescript-halogen-hooks/blob/master/examples/Example/Hooks/UseWindowWidth.purs" "full implementation" >}} in the Hooks examples:

```hs
useWindowWidth = Hooks.wrap Hooks.do
  width /\ widthId <- Hooks.useState Nothing -- [1]

  Hooks.useLifecycleEffect do -- [2]
    subscriptionId <- subscribeToWindow (H.modify_ widthId)
    pure $ Just $ Hooks.unsubscribe subscriptionId -- [3]

  Hooks.pure width -- [4]
  where
  -- we'll define the `subscribeToWindow` function in the next section, as it's
  -- ordinary effectful code and not Hooks specific.
  subscribeToWindow modifyWidth = ...
```

Our new Hook is built out of other hooks, provided by the Hooks library as primitives:

1. First, we use `useState` to produce a new independent state which will hold the window width. Its initial state is `Nothing`, because we don't yet have a window width. We specified in our `UseWindowWidth` type that this Hook should return `Maybe Int`, so the compiler will ensure we use that type. The Hook returns the current value in state to us, and also a unique identifier we can use to update the state -- more on this soon.
2. Next, we use `useLifecycleEffect` to run an effect when the component initializes and another one when the component finalizes. Our initializer function subscribes to the window using `subscribeToWindow`, an effectful function we defined in a where block underneath the body of the Hook.
3. Here, we return our optional 'disposal' function to run when the component finalizes. (It's technically unnecessary to end Halogen subscriptions in a finalizer because they are automatically cleaned up when a component unmounts. But that's a special case: you *would* need to unsubscribe when using the other effect hook, `useTickEffect`, and it's common to run a cleanup function when a component is finalized.)
4. Finally, we return the window width from the hook.

The built-in `useState` and `useLifecycleEffect` hooks are basic building blocks you can use directly in Hooks components or to implement your own custom hooks like this one. There are {{< external-link "https://github.com/thomashoneyman/purescript-halogen-hooks/blob/master/docs/07-Hooks-API.md" "several built-in Hooks" >}} you can use.

I omitted the definition of `subscribeToWindow` to keep our implementation concise, but we can now take a look:

```hs
subscribeToWindow
  :: ((Maybe Int -> Maybe Int) -> HookM m Unit)
  -- this is the same type variable `m` introduced by `useWindowWidth`
  -> HookM m H.SubscriptionId
subscribeToWindow modifyWidth = do
  let
    readWidth :: Window -> HookM _ _ _ Unit
    readWidth =
      modifyWidth <<< const <<< Just <=< liftEffect <<< Window.innerWidth

  window <- liftEffect HTML.window
  subscriptionId <- Hooks.subscribe do
    ES.eventListenerEventSource
      (EventType "resize")
      (Window.toEventTarget window)
      (Event.target >>> map (fromEventTarget >>> readWidth))

  readWidth window
  pure subscriptionId
```

This function sets up the subscription and ensures our state is updated every time the window resizes. It's *almost* identical to what you would write in `HalogenM`, but you may have noticed some differences:

* This function runs in the `HookM` monad, not `HalogenM`. This monad is almost identical to `HalogenM` and it's used to implement effectful code in Hooks. You can do anything in `HookM` that you can do in `HalogenM`, such as start subscriptions, query child components, or fork threads.
* There is no state type in the `HookM` monad, but we can still update state using the unique identifier returned by `useState`. You can pass this identifier to the `modify`, `modify_`, `put`, and `get` functions you're familiar with from `HalogenM`. This is a feature of Hooks that lets you have as many independent states as you would like, each with its own modify function.
* There is no action type because Hooks don't need actions. Where you write actions in Halogen, you write `HookM` functions in Hooks. However, you can still manually implement the action / handler pattern from Halogen if you want to.
* There is no slot type because slots only make sense in the context of components. You can only use functions that use a `slot` type if you have used the `component` function to turn your Hook into a component first.
* There is no output type because outputs also only make sense in the context of components. Like the slot type, you must turn your Hook into a component before you can raise output messages.

If you are ready to learn more about using and implementing Hooks, please see {{< external-link "https://github.com/thomashoneyman/purescript-halogen-hooks/tree/master/docs" "the official Halogen Hooks guide" >}}.

## What About Components?

Halogen Hooks is implemented on top of Halogen and it makes no changes to the underlying library. Components are here to stay, and Hooks make no effort to move away from them. Hooks-based components are still ordinary Halogen components.

In fact, while you can combine primitive and custom Hooks to your heart's delight, the only way to actually _run_ a Hook is to interpret it into a Halogen component. This can be done for any Hook that returns Halogen's `ComponentHTML` type.

Halogen components are still the foundation everything stands on. Hooks themselves can only be executed as components. But you will likely find nested Hooks are much nicer to use than the equivalent tree of components, and that it's more convenient to write most components the Hooks way.

This means that Hooks can be adopted incrementally: you don't need to use Hooks everywhere in your code and Hooks-based components are still ordinary Halogen components. You don't have to update your existing components to begin using Hooks in new ones.

## Next Steps

The {{< external-link "https://github.com/thomashoneyman/purescript-halogen-hooks" "Halogen Hooks repository" >}} contains plenty of documentation on how to get started with Hooks. You may want to start with:

* {{< external-link "https://github.com/thomashoneyman/purescript-halogen-hooks/blob/master/docs/01-Hooks-At-A-Glance.md" "Hooks at a Glance" >}}: a crash course in using and writing Hooks to get you started.
* {{< external-link "https://github.com/thomashoneyman/purescript-halogen-hooks/blob/master/docs/07-Hooks-API.md" "Hooks API Reference" >}}: see all the primitive Hooks the library provides!
* {{< external-link "https://github.com/thomashoneyman/purescript-halogen-hooks/blob/master/docs/08-Hooks-FAQ.md" "Hooks FAQ" >}}: a set of questions and answers -- if you don't see your question here, please open an issue and ask away!

I'm excited about the potential for Hooks to sand off Halogen's roughest edge. My next steps are to rewrite {{< external-link "https://github.com/citizennet/purescript-halogen-select" "Select" >}} and {{< external-link "https://github.com/thomashoneyman/purescript-halogen-formless" "Formless" >}} to use Hooks -- I suspect the resulting libraries will be much simpler.

Hooks are brand-new for Halogen, and if you encounter trouble using them I hope you take the time to {{< external-link "https://github.com/thomashoneyman/purescript-halogen-hooks/issues" "stop by the issue tracker" >}} and we can work together to make the library better for everyone.

{{< subscribe >}}
