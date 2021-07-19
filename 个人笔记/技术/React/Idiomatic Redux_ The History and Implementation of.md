Idiomatic Redux: The History and Implementation of React-Redux

*Some history and explanations of how React-Redux got its API, and how it works internally*

## Intro [🔗︎](#intro)

React-Redux is conceptually pretty simple. It subscribes to the Redux store, checks to see if the data your component wants has changed, and re-renders your component.

However, there's a lot of internal complexity to make that happen, and most people aren't aware of all the work that React-Redux does internally. I'd like to dig through some of the design decisions and implementation details of how React-Redux works, and how those implementation details have changed over time.

> **Note**: This post was originally written before the release of React-Redux v6. It has since been updated to cover the final release of v6, and the development and release of v7.0 and v7.1.

#### Table of Contents [🔗︎](#table-of-contents)

- [Integrating Redux with a UI](#integrating-redux-with-a-ui)
- [`connect` in a Nutshell](#connect-in-a-nutshell)
- [Development History of the React-Redux API](#development-history-of-the-react-redux-api)
- [v4.x](#v4-x)
- [v5.x](#v5-x)
- [v6.0](#v6-0)
- [v7.0](#v7-0)
- [v7.1: Hooks?!?!](#v7-1-hooks)
- [The Future](#the-future)
- [Final Thoughts](#final-thoughts)

## Integrating Redux with a UI [🔗︎](#integrating-redux-with-a-ui)

### Understanding Redux Store Subscriptions [🔗︎](#understanding-redux-store-subscriptions)

It's been said that "Redux is just a \[dumb\] event emitter". There's actually a fair amount of truth in that statement. Earlier MVC frameworks like Backbone would allow triggering any string as an event, and automatically trigger events like `"change:firstName"` in models. Redux, on the other hand, only has a single event type: "some action was dispatched".

As a reminder, here's what a miniaturized (but valid) implementation of the Redux store looks like:

```
function createStore(reducer) {
    var state;
    var listeners = []

    function getState() {
        return state
    }
    
    function subscribe(listener) {
        listeners.push(listener)
        return function unsubscribe() {
            var index = listeners.indexOf(listener)
            listeners.splice(index, 1)
        }
    }
    
    function dispatch(action) {
        state = reducer(state, action)
        listeners.forEach(listener => listener())
    }

    dispatch({})

    return { dispatch, subscribe, getState }
}

```

Let's focus in particular on the `dispatch()` implementation. Notice that it doesn't check to see if the root state actually changed or not - **it runs *every* subscriber callback after *every* dispatched action, regardless of whether there was any meaningful change to the state or not**.

In addition, **the store state is not passed to the subscriber callbacks** \- it's up to each subscriber to call `store.getState()` if desired to retrieve the latest state. (See the [Redux FAQ entry on why the state isn't passed to subscribers](https://redux.js.org/faq/designdecisions#why-doesnt-redux-pass-the-state-and-action-to-subscribers) for more details.)

### The Standard UI Update Cycle [🔗︎](#the-standard-ui-update-cycle)

Using Redux with *any* UI layer requires [the same consistent set of steps](https://blog.isquaredsoftware.com/presentations/workshops/redux-fundamentals/ui-layer.html#/4):

1.  Create a Redux store
2.  Subscribe to updates
3.  Inside the subscription callback:
    1.  Get the current store state
    2.  Extract the data needed by this piece of UI
    3.  Update the UI with the data
4.  If necessary, render the UI with initial state
5.  Respond to UI inputs by dispatching Redux actions

Here's an example with a vanilla JS "counter" app, where the "state" is a single number:

```

const store = createStore(counter)


store.subscribe(render);

const valueEl = document.getElementById('value');


function render() {
    
    const state = store.getState();

    
    const newValue = state.toString();

    
    valueEl.innerHTML = newValue;
}


render();


document.getElementById("increment")
    .addEventListener('click', () => {
        store.dispatch({type : "INCREMENT"});
    })

```

**Every Redux UI integration layer is simply a fancier version of those steps.**

You *could* do this manually, in every React, component, [but it would quickly get out of hand](https://blog.isquaredsoftware.com/presentations/workshops/redux-fundamentals/ui-layer.html#/7), especially once you [start trying to cut down on unnecessary UI updates](https://blog.isquaredsoftware.com/presentations/workshops/redux-fundamentals/ui-layer.html#/9).

Clearly, the process of subscribing to the store, checking for updated data, and triggering a re-render can be made more generic and reusable. That's where React-Redux and the `connect` API come in.

## `connect` in a Nutshell [🔗︎](#connect-in-a-nutshell)

About a year after Redux came out, Dan Abramov wrote a gist entitled [connect.js explained](https://gist.github.com/gaearon/1d19088790e70ac32ea636c025ba424e). It contains a miniature version of `connect` to illustrate how it works conceptually. It's worth pasting that here for emphasis:

```


function connect(mapStateToProps, mapDispatchToProps) {
  
  
  return function (WrappedComponent) {
    
    return class extends React.Component {
      render() {
        return (
          
          <WrappedComponent
            {}
            {...this.props}
            {}
            {...mapStateToProps(store.getState(), this.props)}
            {...mapDispatchToProps(store.dispatch, this.props)}
          />
        )
      }
      
      componentDidMount() {
        
        this.unsubscribe = store.subscribe(this.handleChange.bind(this))
      }
      
      componentWillUnmount() {
        
        this.unsubscribe()
      }
    
      handleChange() {
        
        this.forceUpdate()
      }
    }
  }
}









```

We can see the key aspects of the API here:

- `connect` is a function returning a function returning a wrapper component.
- The wrapped component's props are a combination of the wrapper component's props, the values from `mapState`, and the values from `mapDispatch`.
- Each wrapper component is an individual subscriber to the Redux store.
- The wrapper component abstracts away the details of *which* store you're using, *how* it's interacting with the store, and optimizing performance so that your own component only re-renders when it needs to.
- You simply specify how to extract the data your component needs based on the store state, and the functions it can call to dispatch actions to the store

But, this miniature example hand-waves a lot of the details. In particular, as the comments point out:

- Where does the store come from?
- How does `connect` check to see if your component needs to actually update?
- How does it implement the optimizations?

And beyond that, how did we even end up with an API that looks like this in the first place?

## Development History of the React-Redux API [🔗︎](#development-history-of-the-react-redux-api)

### Early Iterations [🔗︎](#early-iterations)

The first few versions of Redux included React bindings as part of the main package. I won't go through those versions specifically, but you can see the changes in the release notes and READMEs for these early releases:

- [v0.2.0](https://github.com/reduxjs/redux/tree/v0.2.0)
- [v0.3.0](https://github.com/reduxjs/redux/releases/tag/v0.3.0)
- [v0.5.0](https://github.com/reduxjs/redux/releases/tag/v0.5.0)
- [v0.6.0](https://github.com/reduxjs/redux/releases/tag/v0.6.0)
- [v0.8.0](https://github.com/reduxjs/redux/releases/tag/v0.8.0)
- [v0.9.0](https://github.com/reduxjs/redux/releases/tag/v0.9.0)

Fun side note: amazingly, all that iteration took place over a single week!

It took another three weeks to get up to [v1.0.0-rc](https://github.com/reduxjs/redux/releases/tag/v1.0.0-rc), which is where the meaningful history begins.

### Original API Design Constraints [🔗︎](#original-api-design-constraints)

React-Redux was split out as a separate repo in July 2015, just before Redux v1.0.0-rc came out.

Dan filed [React-Redux issue #1: Alternative API Proposals](https://github.com/reduxjs/react-redux/issues/1) to discuss what the final API should look like. In that issue, he listed a number of design constraints that should guide how the final API worked:

> Common pain points:  
> \- Not intuitive how way to separate smart and dumb components with `<Connector>`, `@connect`  
> \- You have to manually bind action creators with `bindActionCreators` helper which [some don't like](https://github.com/gaearon/redux/pull/86)  
> \- Too much nesting for small examples (`<Provider>`, `<Connector>` both need function children)
> 
> Let's go wild here. Post your alternative API suggestions.
> 
> They should satisfy the following criteria:  
> \- Some component at the root must hold the `store` instance. (Akin to `<Provider>`)  
> \- It should be possible to connect to state no matter how deep in the tree  
> \- It should be possible to select the state you're interested in with a `select` function  
> \- Smart / dumb components separation needs to be encouraged  
> \- There should be one obvious way to separate smart / dumb components  
> \- It should be obvious how to turn your functions into action creators  
> \- Smart components should probably be able to react to updates to the state in `componentDidUpdate`  
> \- Smart components' `select` function needs to be able to take their props into account  
> \- Smart component should be able to do something before/after dumb component dispatches an action  
> \- We should have `shouldComponentUpdate` wherever we can

These criteria form the basis for the React-Redux API that we have today, and help explain why it works the way it does.

The biggest points that came out of that discussion were using `connect` as a function instead of a decorator, and how to handle binding action creators.

Another fun side note: the ["object shorthand" for binding action creators](https://react-redux.js.org/docs/using-react-redux/connect-dispatching-actions-with-mapdispatchtoprops#defining-mapdispatchtoprops-as-an-object) was one of Dan's first suggestions for the API:

> Perhaps we can even go further and bind automatically if an object is passed.

### Finalizing the API [🔗︎](#finalizing-the-api)

The next few releases continued iterating on the `connect` API. Highlights of the changes:

- [v0.5.0](https://github.com/reduxjs/react-redux/releases/tag/v0.5.0): introduced the `mapState`, `mapDispatch`, and `mergeProps` arguments
- [v0.6.0](https://github.com/reduxjs/react-redux/releases/tag/v0.6.0): used immutability and reference checks to determine if re-rendering is necessary
- [v0.8.0](https://github.com/reduxjs/react-redux/releases/tag/v0.8.0): added the ability to pass `store` as a prop
- [v0.9.0](https://github.com/reduxjs/react-redux/releases/tag/v0.9.0): added the `ownProps` arguments to `mapState/mapDispatch`
- [v1.0.0](https://github.com/reduxjs/react-redux/releases/tag/v1.0.0): `<Provider>`s single child had to be a function that returned an element
- [v2.0.0](https://github.com/reduxjs/react-redux/releases/tag/v2.0.0): no longer supported "magically" hot reloading reducers
- [v2.1.0](https://github.com/reduxjs/react-redux/releases/tag/v2.1.0): added an `options` argument
- [v3.0.0](https://github.com/reduxjs/react-redux/releases/tag/v3.0.0): moved the initial `mapState/mapDispatch` calls to be part of the first render, to avoid state staleness issues

Continuing the observations, at one point [Dan warned against connecting leaf components](https://github.com/reduxjs/react-redux/issues/86#issuecomment-137237015):

> We do warn in the documentation that we encourage you to follow React flow and avoid connect()ing leaf components.

This is particularly amusing, given that [we now recommend connecting components anywhere in the tree you feel it would be useful](https://redux.js.org/faq/reactredux#should-i-only-connect-my-top-component-or-can-i-connect-multiple-components-in-my-tree).

That brings us up to what I'd say is the "modern era" of the React-Redux API, starting with version 4.

## v4.x [🔗︎](#v4-x)

**[React-Redux v4.0.0](https://github.com/reduxjs/react-redux/releases/tag/v4.0.0)** came out in October 2015, and had four major changes:

- React 0.14 as a minimum and a peer dependency
- No more React native-specific entry point
- `<Provider>` no longer accepted a function as a child, but rather a standard React element
- Refs to the child component now required the `withRef` option, instead of always being enabled

In addition, [v4.3.0](https://github.com/reduxjs/react-redux/releases/tag/v4.3.0) added the "factory function" syntax of `mapState/mapDispatch` to allow per-component-instance memoization of selectors.

I would consider 4.x the first "complete" version of the React-Redux API. As such, it's worth digging in to some of its implementation details, as well as some common points that define the overall behavior of the API across versions.

### API Behavior [🔗︎](#api-behavior)

#### Every Wrapper Component Instance is a Separate Store Subscriber [🔗︎](#every-wrapper-component-instance-is-a-separate-store-subscriber)

This one's pretty simple. If I have a connected list component, and it renders 10 connected list item children, that's 11 separate calls to `store.subscribe()`. That *also* means that **if I have N connected components in the tree, then N subscriber callbacks are run for *every* dispatched action!**.

Every single subscriber callback is then responsible for doing its own checks to see if that particular component actually needs to update or not, based on the store state and props.

#### UI Updates Require Store Immutability [🔗︎](#ui-updates-require-store-immutability)

We've already established that the Redux store will run *all* subscriber callbacks after *every* dispatched action, regardless of whether the state actually changed or not.

**In order to implement efficient UI updates, React-Redux assumes you have updated the store state immutably, so it can use reference comparisons to determine if the state changed.**

This occurs in three stages:

1.  When a `connect` wrapper component's subscriber callback runs, it first calls `store.getState()`, and checks to see if `prevStoreState !== storeState`. If the store state did *not* change by reference, then it will stop right there and bail out of any further update work, because it assumes that no other part of the store state changed.  
    
2.  If the root state *has* changed, the wrapper component then runs your `mapState` function, and does a "shallow equality" comparison between the current result and the last result. If any of the fields have changed by reference, then your component probably needs to be updated.
3.  Assuming that there was some change in the `mapState` or `mapDispatch` results, the `mergeProps()` function is run to combine the `stateProps` from `mapState`, `dispatchProps` from `mapDispatch`, and `ownProps` from the wrapper component itself. A final check is done to see if the merged props result has changed since the last time, and if there's no change, the wrapped component will not be re-rendered.

> **Note**: The root state comparison relies on a specific optimization in `combineReducers`, which checks to see if any state slices were changed while processing an action, and if not, returns the previous state object instead of a new one. **This is a key reason why mutating your state results in your React UI components not updating!**

#### The Store Instance is Passed Down via Legacy Context [🔗︎](#the-store-instance-is-passed-down-via-legacy-context)

In v4, rendering `<Provider store={store}>` puts that store instance into the legacy context API, as `context.store`. Any nested class component could then request that `context.store` be attached to the component.

The biggest reason why the entire store was put into context was because context itself was flawed. If you put an initial value into context in a parent, and some child requested that value, it would receive the correct value. However, if the parent component put an updated value by that name into context, and there was an intervening component that skipped rendering using `shouldComponentUpdate -> false`, the nested component would never see the updated value. That's one of the reasons why the React team always discouraged people from using legacy context directly, suggesting that any uses should be wrapped up in an abstraction so that when a "future replacement for context" came out, it would be easier to migrate. (See the [React "Legacy Context" docs page](https://reactjs.org/docs/legacy-context.html) and Michel Westrate's article [How to Safely Use React Context](https://medium.com/@mweststrate/how-to-safely-use-react-context-b7e343eff076) for more details.)

So, one workaround was to put an *event emitter instance* into context, instead of an actual value. The nested components could grab the emitter instance on first render and subscribe directly, thus bypassing the `sCU` "roadblocks" to receive updates. React-Redux did this, as did libraries like `react-broadcast`.

#### Connected Components Accept a Store as a Prop [🔗︎](#connected-components-accept-a-store-as-a-prop)

In addition to checking for the store instance in context, the wrapper components can optionally accept a store instance as a prop named `store` instead, like `<MyConnectedComponent store={store}>`. This works because each component subscribes to the store individually, so this component just gets that store a different way.

This is mostly useful for rendering connected components in a test without putting a `<Provider>` around it, but does mean you could have a connected component in the middle of your tree that uses a different store than all the other components around it.

### Implementation Details [🔗︎](#implementation-details)

#### Update Logic was Directly in the Wrapper Component [🔗︎](#update-logic-was-directly-in-the-wrapper-component)

Up through v4, all of the logic was part of the `connect` component class itself, including handling store updates, calling `mapState`, and determining if the wrapped component needed to re-render.

There's a couple interesting observations out of this. First, [the wrapper component always called `setState()` whenever the root store state had changed](https://github.com/reduxjs/react-redux/blob/v4.3.0/src/components/connect.js#L213-L216), before it tries to do anything else.

Second, as a result, the *real* work of running `mapState` and determining if anything has changed [was actually done directly in the `render` method](https://github.com/reduxjs/react-redux/blob/v4.3.0/src/components/connect.js#L228-L285).

This meant that any update to the Redux store required *all* wrapper components to re-render themselves in order to determine if their wrapped components actually need to update or not. That's a lot of components re-rendering every time, and also meant React was *always* getting called on every meaningful store update.

#### Child Component Rendering was Optimized Via Memoized React Elements [🔗︎](#child-component-rendering-was-optimized-via-memoized-react-elements)

It's not well documented, but React has a particular performance optimization built-in. Normally, when a component renders, it creates new child elements every time:

```
render() {
    
    return <ChildComponent a={42} />;
}

```

Every call to `React.createElement()` returns a new element object like `{type : ChildComponent, props : {a : 42}, children : []}`. So, normally every re-render results in all-new element objects being created.

However, if you return the exact same element objects as before, React will skip updating those specific elements as an optimization. (This is the basis for [@babel/plugin-transform-react-constant-elements](https://babeljs.io/docs/en/babel-plugin-transform-react-constant-elements), and the behavior is discussed some in [React issue #3226](https://github.com/facebook/react/issues/3226)).

The v4 implementation takes advantage of this, by [memoizing the element for the child component](https://github.com/reduxjs/react-redux/blob/v4.3.0/src/components/connect.js#L268-L283) to skip updating it if not needed. This is necessary because the wrapper component is itself already re-rendering, so it needs some way to *not* have the child re-render.

## v5.x [🔗︎](#v5-x)

Up through v4, React-Redux was still primarily the work of Dan Abramov, albeit with a number of external contributions. Dan had given me commit rights after I wrote the Redux FAQ, and I'd spent enough time working on issues and looking the code that I felt comfortable giving more feedback beyond just how to use Redux. However, by mid 2016 Dan had joined the React team at Facebook, and was getting busy there, so he told Tim Dorr and I that we were now the primary maintainers.

About that time, a user named [Jim Bolla](https://github.com/jimbolla) filed an issue asking about [an unusual use of `connect`](https://github.com/reduxjs/react-redux/issues/403). During the discussion, Jim commented that he was working on "an alternate version of `connect`", and I mentally dismissed that.

A few days after that, though, Jim filed [a follow-up issue asking for feedback on his alternate implementation](https://github.com/reduxjs/react-redux/issues/405). We discussed some of the complexities in `connect`'s implementation, and how those related to the use cases it was trying to solve, but I again otherwise didn't think much of it.

To my surprise, a couple days later Jim created **[issue #407: Completely rewrite connect() to offer advanced API, separate concerns, and (probably) resolve a lot of those pesky edge cases](https://github.com/reduxjs/react-redux/issues/407)** as a precursor to filing an actual PR. I was still really skeptical and began pointing out concerns and edge cases, but to my (pleasant) surprise, Jim kept taking my feedback and improving his WIP branch. This included producing some benchmarks which indicated that his version was noticeably faster than v4 in some particular scenarios.

Jim's efforts [eventually won me over](https://github.com/reduxjs/react-redux/issues/407#issuecomment-227925119), and we began seriously collaborating on pushing his rewrite forward. That became **[PR #416: Rewrite connect() for better performance and extensibility](https://github.com/reduxjs/react-redux/pull/416)** .

The rewrite was released as **[v5.0.0](https://github.com/reduxjs/react-redux/releases/tag/v5.0.0)** in December 2016. The biggest changes were:

- Logic was moved from the wrapper component into memoized selectors
- Enforceed top-down subscription updates
- Added a new `connectAdvanced` API
- More customization of comparison options
- Overall performance improvements

All of this while keeping the same public `connect` API compatible with v4.

v5 also resolved [a large number of existing issues](https://github.com/reduxjs/react-redux/pull/416#issuecomment-233144992) as well.

There were various bugfixes up through [v5.0.7](https://github.com/reduxjs/react-redux/releases/tag/v5.0.7), and [v5.1.0](https://github.com/reduxjs/react-redux/releases/tag/v5.1.0) recently added support for passing React's new built-in component types like `memo` and `lazy` into `connect`.

Let's dig through some of the details.

### API Behavior [🔗︎](#api-behavior-1)

#### Top-Down Updates [🔗︎](#top-down-updates)

The `connect` wrapper components subscribe to the store in `componentDidMount`. However, because that lifecycle fires bottom-to-top in a new component tree, it was possible for child components to subscribe to the store before their parents did. Up through v4, that resulted in some [nasty recurring bugs](https://github.com/reduxjs/react-redux/issues/292).

As an example, imagine a connected list with 10 connected list item children. If they all render right away, the list items will subscribe before the list parent. If you then delete the data for one of the items from the store, the list item component's `mapState` would run before the parent's did. This usually meant that the list item's `mapState` would throw an error and break the component tree.

v5 enforced the idea of top-down updates. Components higher in the tree *always* subscribe to the store before their children do. That way, in a scenario like the connected list, deleting an item from the store will result in the parent updating first, and re-rendering without that list item child before the child even has a chance to run its own `mapState`. This gives much more predictable behavior, and aligns with how React itself works.

We'll discuss the specifics of how this is implemented separately.

#### `connectAdvanced` [🔗︎](#connectadvanced)

`connect` is fairly opinionated. It lets you extract data from the store via `mapState`, and prepare functions that dispatch actions via `mapDispatch`, but it doesn't let you use store state data in `mapDispatch` to prevent performance footguns. It does provide the `mergeProps` argument as an escape hatch, but that's separate.

However, for users that want more flexibility (such as Jim himself), v5 adds a new `connectAdvanced` API. Rather than taking `(mapState, mapDispatch)`, it asks you to pass in a "selector factory". A selector instance will be created for each component and given a reference to `dispatch`, and the selectors will be called with `(state, ownProps)` on all future updates from the store or the wrapper component. That way, you can customize exactly how you want to handle derived props based on those inputs.

The original `connect` API is now actually implemented as a specific set of selector functions and options to `connectAdvanced`.

### Implementation Notes [🔗︎](#implementation-notes)

#### Logic is Implemented in Memoized Selectors [🔗︎](#logic-is-implemented-in-memoized-selectors)

v5 moves all of the state derivation logic out of the wrapper component and into a separate set of homegrown memoized selector functions. These selectors specifically implement all of the `connect` API behavior, like:

- Checking if the root state has changed
- Handling the various forms of `mapState` and `mapDispatch` ( `(state)` vs `(state, ownProps)`, `mapDispatch` as an object vs function, etc)
- Calling `mapState`, `mapDispatch`, and `mergeProps`
- Calculating the new child props and determining if a re-render is actually necessary

As a result, the subscriber callbacks can run extremely quickly, without involving React at all. In fact, React will only get involved once the wrapper component *knows* the child should re-render, and it uses a dummy `this.setState({})` call to queue up that re-render. (We probably could have used `forceUpdate()` instead, but I don't think it makes any difference in this case.)

This is the biggest reason why v5 is generally faster than v4.

#### Custom Top-Down Subscription Logic [🔗︎](#custom-top-down-subscription-logic)

In order to enforce top-down subscriptions, v5 introduced [a custom `Subscription` class](https://github.com/reduxjs/react-redux/blob/v5.1.1/src/utils/Subscription.js). Internally, `connect` actually puts both the store instance *and* an instance of `Subscription` into legacy context. If no subscription exists in context, that component will subscribe to the store directly, as it must be high up in the tree. Otherwise, it subscribes to the `Subscription` instance. This means that each connected component is effectively subscribing to its nearest connected ancestor.

When an action is dispatched, the uppermost connected components will have their callbacks be triggered right away. If they do need to re-render, they call `setState()`, and wait until `componentDidUpdate` to trigger notification of the next tier of connected components. If no update is necessary, the next tier is notified right away.

This works, but it also requires some very tricky logic in both the `Subscription` class and the wrapper component itself (including dynamically adding and removing a `componentDidUpdate` function to micro-optimize perf).

## v6.0 [🔗︎](#v6-0)

### Motivations [🔗︎](#motivations)

v5 was great. It performed faster than v4 in almost all scenarios we've seen, and it added more flexibility.

However, the React team has continued to innovate. In particular, React 16.3 introduced the new `React.createContext()` API, which is an officially supported replacement for the legacy context API, and encouraged for production use. With `createContext` now available, they've been [encouraging the community to migrate away from legacy context](https://github.com/facebook/react/pull/13728).

They've also been working on "concurrent React", an umbrella that describes future capabilities like ["time-slicing" and "Suspense"](https://reactjs.org/blog/2018/03/01/sneak-peek-beyond-react-16.html). Long-term, there are questions about how synchronous external stores like Redux will work correctly when React is running in concurrent mode.

With that in mind, we had several discussion threads about how React-Redux should work with concurrent React ([#890](https://github.com/reduxjs/react-redux/issues/890), [#950](https://github.com/reduxjs/react-redux/issues/950)), as well as how to deal with the deprecation warnings when used in `<StrictMode>`

We originally planned on releasing a 5.1.0 release to fix `<StrictMode>` issues, but [that test release turned out to be very broken](https://github.com/reduxjs/react-redux/releases/tag/v5.1.0-test.1). When we tried to fix the breakage, [our attempts turned out to drastically hurt performance](https://github.com/reduxjs/react-redux/pull/980), as well as add way too much complexity.

We ultimately decided to [not put out a direct fix for `<StrictMode>` warnings in 5.x](https://github.com/reduxjs/react-redux/pull/980#issuecomment-410924304), and instead moved on to work on v6.

The primary drivers for v6 development were:

- **Use `createContext` instead of legacy context**
- **Fix `<StrictMode>` warnings**
- **Be more compatible with concurrent React behavior going forward**

We went through several experimental PRs (particularly [#898](https://github.com/reactjs/react-redux/pull/898) and [#995](https://github.com/reduxjs/react-redux/pull/995)) before settling on **[PR #1000: Use React.createContext()](https://github.com/reduxjs/react-redux/pull/1000)** as the best approach. Another contributor named [Greg Beaver](https://github.com/cellog) had been working with us on the `<StrictMode>` issues, and he and I submitted "competing" candidate PRs for v6 with varying internal implementations. His approach turned out to be *slightly* faster than mine, so we went with that PR, and I was then able to further optimize the PR.

### API Changes [🔗︎](#api-changes)

We released **[React-Redux v6.0](https://github.com/reduxjs/react-redux/releases/tag/v6.0.0)** in early December 2018. The primary changes were:

- **Internal**: Uses `createContext` internally instead of legacy context
- **Internal**: Changes to how the components subscribe and receive the updated state from the store
- **Breaking**: The `withRef` option has been removed in favor of using React's `forwardRef` capability
- **Breaking**: Passing a store as a prop is no longer supported

Note that **there were only two minor breaking changes to the public API!**. React-Redux has a fairly comprehensive [suite of unit tests for `connect` and `<Provider`>](https://github.com/reduxjs/react-redux/tree/v6.0.0-beta.1/test/components), and v6 passed the same unit tests as v5 (with appropriate changes in the tests to match some of the implementation changes). v6 also ran safely inside a `<StrictMode>` tag without any warnings.

Because of that, for most apps, React-Redux v6 was a drop-in upgrade! We did require React 16.4 as a minimum because of using `createContext`, but other than that, many apps were able to just bump to the new version. The biggest set of issues came from various community libraries that relied on accessing the store instance directly from legacy context, which broke.

However, **the implementation changes did result in different behavior tradeoffs**. Let's look at the changes in detail.

### Implementation Notes [🔗︎](#implementation-notes-1)

#### v6: The Store State is Passed Down via `createContext` [🔗︎](#v6-the-store-state-is-passed-down-via-createcontext)

In every version up through v5.x, the Redux store instance itself was put into context, and every connected component subscribed directly. In v6, that changed drastically.

**In v6:**

- **The Redux *store state* is put into an instance of the new `createContext` API**
- **There is only *one* store subscriber: the `<Provider>` component**

This had all kinds of ripple effects across the implementation.

It's fair to ask why we chose to change this aspect. We certainly *could* have put the store instance into `createContext`, but there's several reasons why it made sense to put the store *state* into context instead.

**The largest reason was to improve compatibility with "concurrent React", because the entire tree will see a single consistent state value**. The very short explanation on this is that React's "time-slicing" and "Suspense" features can potentially cause problems when used with external synchronous state management tools. As one example, Andrew Clark has described "tearing" as a possible problem, where different parts of the component tree see different values during the same component tree re-render pass. By passing down the current state via context, we can ensure that the entire tree sees the same state value, because React takes care of that for us.

The long-term goal was to hopefully prevent weird bugs when React-Redux is used with concurrent-mode React. (We do have other questions we need to solve regarding how to fully make use of Suspense - [I wrote an extensive Reddit comment describing the aspects we might need to solve](https://www.reddit.com/r/reactjs/comments/9xnvs2/how_does_concurrent_react_update_priorities_work/e9toant/?context=3).)

Related to this, React-Redux has previously faced numerous problems around dispatching in constructors and `componentWillMount` (see [some related issues](https://github.com/reduxjs/react-redux/search?q=dispatch+componentwillmount&type=Issues) ). Switching to passing the state via context was meant to eliminate those edge cases.

**Another big reason was that we got "top-down updates" for free!** Context inherently propagates top-down and ties into the render process. So, if the data for a list item is deleted, the list parent will naturally re-render before the list item does. As a result, for v6 we were able to delete that custom `Subscription` logic - we no longer needed it! That was less code that we had to maintain, and a slightly smaller package size as a result.

In addition, the original reason for passing the store instance no longer exists, because **`createContext` correctly propagates value changes past `shouldComponentUpdate` blockers**.

Finally, while this would have changed either way we handled the state-vs-store question, **switching to `createContext` fixed bugs when mixing old and new context together**. There's already been a number of bugs filed that indicate weird things happen if you use both forms of context in the same component, and Dan has also said that [having old context being used anywhere in a component tree slows things down some](https://github.com/reduxjs/react-redux/issues/1083#issuecomment-439958259).

Putting the store state into context did have some interesting implications around performance, which we'll get to in a bit.

#### Update Logic is Selectors, Used In Rendering [🔗︎](#update-logic-is-selectors-used-in-rendering)

The new context API relies on a "render props" approach for receiving the values put into context, like:

```
<MyContext.Consumer>
{ (contextValue) => {
    
}}
</MyContext.Consumer>

```

This means that context updates are directly tied to the wrapper component's `render` function.

v6 still used the exact same set of selector functions for `connect` as v5 did. However, there was also some additional memoization logic built into the wrapper component itself to help with the rendering process. (I initially tried adding a second inner wrapper component and doing tricks with `getDerivedStateFromProps`, but adding an additional selector in the one wrapper component proved to be more efficient.)

As part of that, **v6 re-used the "memoized React child element" trick** to indicate that the wrapped component shouldn't be re-rendered. As with v4, this is because updates are tied to the wrapper component re-rendering, so we need a way to bail out if the child doesn't need to update. (In fact, v6 doesn't even actually implement `shouldComponentUpdate`, because this trick is equivalent in terms of when the child updates.)

#### The `withRef` Option is Replaced by `forwardRef` [🔗︎](#the-withref-option-is-replaced-by-forwardref)

One of the acknowledged downsides to Higher-Order Components is that they don't easily allow you to get access to the wrapped component inside. React 16.3 introduced a new [`React.forwardRef` API](https://reactjs.org/docs/forwarding-refs.html) as a solution. Libraries can use this to allow end users to put a `ref` on the HOC, but actually get back the real wrapped component instance.

We've added that in v6, which means that the old `withRef` option is no longer needed. Since this does add an additional layer of wrapping (and therefore a bit more work for React to do), it's still opt-in via the new `{forwardRef : true}` option.

#### No More `store` as a Prop [🔗︎](#no-more-store-as-a-prop)

This was a consequence of changing from individual subscriptions per component, to a single subscription in `<Provider>`. Since the components no longer subscribe, passing a store directly as a prop was meaningless, and so it was removed.

As mentioned earlier, the two main use cases for this were avoiding rendering `<Provider>` in tests, and allowing portions of the component tree to read from another store. The unit test use case was something that did require changes in your codebase, for folks who were actually rendering connected components in their unit tests. For the "alternate store" use case, we added the ability to pass your own custom context object as a prop to `<Provider>` and connected components, allowing them to read from a different store if desired, hoping that would be a sufficient alternative.

(I originally intended the API to involve passing a `Context.Provider` as a prop to `<Provider>` and passing a `Context.Consumer` as a prop to a connected component. However, the `useContext()` hook requires an entire context object, not just a consumer, despite my requests to the React team to allow it to work with just a consumer. So, the thought was that if we ever switched to using hooks internally to read from context, we'd need the whole context object in the wrapper component, so best to just require that as a prop now.)

#### Accessing the Store via Context Has Changed [🔗︎](#accessing-the-store-via-context-has-changed)

While it's never been part of our public API, it's common knowledge that any component could get a reference to the Redux store by declaring the appropriate `contextTypes` and using `this.context.store`. Many community libraries took advantage of this. For example, `connected-react-router` [adds an extra subscription to handle location changes](https://github.com/supasate/connected-react-router/blob/b197430be6315eb8c70f98be96ff67825653add5/src/ConnectedRouter.js#L23-L47), while `react-redux-subspace` [intercepts the store and passes down a wrapped-up version that presents an altered view of the state](https://github.com/ioof-holdings/redux-subspace/blob/d4c95fbf50fc93e4abfa2bcff742a51f249f0ad7/packages/react-redux-subspace/src/components/SubspaceProvider.js#L15-L21).

Obviously, this is unsupported, and any library that does this kind of thing is risking things breaking... and in v6, that all broke because we weren't using legacy context any more. However, we wanted to allow the community the ability to build customized solutions on top of React-Redux if desired.

Each connected component needs the current store state, *and* a reference to the store's `dispatch` function so that it can implement `mapDispatch` correctly. In one early PR, I had the `<Provider>` putting `{storeState, dispatch}` into context to handle that.

However, in v6 final, we actually put both the *store state* and the *store instance* into context, so the context value actually looks like `{storeState, store}`. That way, the components could reference `store.dispatch`. In addition, we're exporting our default instance of `ReactReduxContext`. *If* someone wants to, they can render that context consumer, retrieve the store instance, and do something with it.

Again, it wasn't an official API, but the goal was to make it possible for folks to build on that if needed.

### Performance Implications [🔗︎](#performance-implications)

When we were working on the attempt to fix the initial failed 5.1.0 version, we ran some benchmarks to see how the altered version compared to 5.0.7. The large perf slowdown was a big reason why we abandoned that attempt.

In response, [I set up a benchmarks repo that could compare multiple versions of React-Redux together](https://github.com/reduxjs/react-redux-benchmarks). We used that throughout the development of v6, comparing our various WIP builds against each other and v6.

Based on those benchmarks, **we expected that React-Redux v6 would be sufficiently fast enough for almost all real-world apps**.

Having said that, there's some caveats.

When I first envisioned switching over to `createContext`, I had hoped that it would be a potential boost to performance. After all, we would be going from N subscriber calls on every action down to just 1. Unfortunately, that isn't the case.

**In *artificial stress test benchmarks*, v6 was generally slower than v5.... but the amount varies, and the reasons are complex.**

#### Understanding the Performance Differences [🔗︎](#understanding-the-performance-differences)

In v5, a dispatched action would result in N subscriber callbacks executing. But, thanks to the heavily memoized selector functions used with `connect`, only wrapper components whose data had changed would actually call `this.setState()` to trigger a re-render. That meant that React only got involved when updates were needed.

In v6, `<Provider>` has the only subscriber callback. However, in order to safely handle state changes, it immediately calls `setState()` using the functional updater form:

```
    this.unsubscribe = store.subscribe(() => {
      const newStoreState = store.getState()

      if (!this._isMounted) {
        return
      }

      this.setState(providerState => {
        
        if (providerState.storeState === newStoreState) {
          return null
        }

        return { storeState: newStoreState }
      })
    })

```

It does try to optimize some by skipping any further work if the store state hasn't changed, but this means that **React will immediately and always get involved after each dispatched action.**

The next issue is that **React has to trace through the component tree to find all of the matching context consumers**. In a simple app structure, React would sort of do this automatically anyway, because calling `setState()` in the root component would recursively cause the entire component tree to be re-rendered.

However, many components in the tree may be blocking updates, whether it be manually-implemented `shouldComponentUpdate -> false`, instances of `PureComponent` or `React.memo()`, or `connect` wrappers that are skipping re-renders of their children. Let's assume for sake of the example that the topmost `<App>` component simply has `shouldComponentUpdate -> false`, thus blocking updates further down when `<Provider>` calls `setState()`. In this case, React still has to traverse the entire rendered component tree to find all the consumers.

React is fast, but that work does take time. The speed of context updates affects more than just React-Redux. The maintainer of `react-beautiful-dnd` opened up **[React issue #13739: React Context value propagation performance](https://github.com/facebook/react/issues/13739)** to discuss some of the perf implications. In that thread, Dan and Dominic suggested that the current handling of nested context updates is somewhat naive, and could potentially be optimized further down the road.

#### Performance Benchmarks [🔗︎](#performance-benchmarks)

When I finished cleanup and optimization on the PR that became v6 beta, I did a final set of runs against our benchmarks. [You can view those benchmark results here](https://github.com/reduxjs/react-redux/pull/1000#issuecomment-436140264). Summarizing:

- Both an earlier WIP v6 iteration and the final version of the v6 PR were slower than v5, in all benchmark scenarios
- That said, the final v6 build was by far the fastest of the v6 versions
- The amount of relative slowdown varied based on the benchmark scenario. It was most pronounced with a totally flat tree of rapidly-updating components (~20% slower), much less so with deeper trees and other update patterns (2% slower).

I'd like to re-emphasize that **these are totally artificial stress-test benchmarks!**. We needed some way to objectively compare different builds for performance, and so we set up scenarios that deliberately cranked up the numbers of components and frequency of dispatched actions until all the builds began slowing down.

(Note: I'd happily accept more help from the community in fleshing out the benchmarks suite, to help us come up with some more realistic scenarios. Also, anyone ought to be able to clone the benchmarks repo, drop in a particular build of React-Redux, and replicate the approximate results on your own machine.)

## v7.0 [🔗︎](#v7-0)

### Motivations [🔗︎](#motivations-1)

I've observed a number of times that the single best way to get feedback on a piece of software is to release it as a final version. Doesn't matter how much you advertise alphas and betas, and beg people to try them out, A) most people won't try them out, and B) the few folks who do simply can't match the breadth of ways that people are using your code.

v6 fit that pattern to a T. Soon after we released it, folks began filing issues listing various issues they encountered trying to upgrade. The most common issue was actually with third-party libraries that were trying (and now failing) to access the store directly. Notable examples included `connected-react-router`, `react-redux-firebase`, `react-redux-subspace`, and even `redux-form`. We couldn't do anything about that, beyond poke maintainers and offer some suggestions on an upgrade path.

Other issues, however, were more concerning. The biggest one was performance. Despite my hopes that v6 would be "fast enough" in real-world apps, several users [complained of noticeable slowdowns in various scenarios](https://github.com/reduxjs/react-redux/issues/1164).

Another major issue was that our use of context turned out to be a bad foundation for eventually creating a hooks-based API. There was a giant React discussion issue on [potential ways to bail out of updates caused by context in function components](https://github.com/facebook/react/issues/14110). Despite some early promising comments, it ultimately resulted in the React team saying that they weren't going to address that particular use case any time soon. In addition, Sebastian Markbage specifically described new context as ["not meant to be used for Flux-like state propagation"](https://github.com/facebook/react/issues/14110#issuecomment-448074060).

Finally, a few specific (but vocal) users raised concerns about [the removal of `store` as a prop](https://github.com/reduxjs/react-redux/issues/1161). The combination of Enzyme's implementation and limitations, and our use of context, effectively made it impossible for them to continue shallow-testing connected components.

In response, in early February 2019 I filed **[issue #1177: React-Redux Roadmap: v6, Context, Subscriptions, and Hooks](https://github.com/reduxjs/react-redux/issues/1177)**. It's a monster post, laying out those concerns in greater detail, and trying to establish a direction for what a potential v7 might look like that would solve those issues. Rather than duplicate it here, please read through it to understand the challenges and how the process evolved.

In particular, the React team (and specifically Dan Abramov and Sebastian Markbage) encouraged us to move back to using direct store subscriptions in components to improve performance. Dan also encouraged us to make use of React's `unstable_batchedUpdates()` API as well.

### Development [🔗︎](#development)

When I filed that roadmap issue, I didn't have any specific ideas on how we were going to reimplement `connect` to achieve them. Fortunately, I had some free time on my hands, and jumped right into experimenting with ideas.

Given the constraints we'd seen with accessing context in class component lifecycles, [I opted to try a brand-new implementation using React's new hooks APIs](https://github.com/reduxjs/react-redux/issues/1177#issuecomment-461185008). I ended up bringing back the custom `Subscription` class from v5, but the [initial benchmark results weren't promising](https://github.com/reduxjs/react-redux/issues/1177#issuecomment-461678923).

A day later, though, I made a breakthrough: [wrapping `connect` in `React.memo()`](https://github.com/reduxjs/react-redux/issues/1177#issuecomment-461699799) drastically improved performance! Comparisons showed it was at *least* as fast as v5, and even faster in some scenarios.

I began publishing alphas for people to play with. Over the next few weeks, the community poked and prodded, and found several issues that we fixed. At one point, I wrote [an insanely detailed step-by-step data flow analysis comparing how v5, v6, v7-alpha.1, and v7-alpha.2 process updates and re-renders](https://github.com/reduxjs/react-redux/issues/1177#issuecomment-468959889).

The mile-long issue thread had a number of digressions and debates on various topics, including the need for tiered subscriptions, if a class component approach could still work, whether peer dependency bumps require a new major version of this package, and much more.

Finally, in early April, we published **[React-Redux v7.0](https://github.com/reduxjs/react-redux/releases/tag/v7.0.1)** as final. The reception has been universally positive. Folks who were seeing slowdowns with v6 reported that v7 was vastly faster across the board.

### Implementation Notes [🔗︎](#implementation-notes-2)

#### `connect` Is Implemented Using Hooks [🔗︎](#connect-is-implemented-using-hooks)

The `connect` wrapper component had always been a class component. As of v7, [`connect` is now a function component that uses hooks inside](https://github.com/reduxjs/react-redux/blob/5c69baf817527ee9a742c9dc4d541945cb7d1719/src/components/connectAdvanced.js#L158). This made some aspects simpler (access to values from context, easy memoization of child elements), but complicated others (timing of effect callbacks).

We initially executed subscriptions in `useEffect()`, but later concluded we needed to use `useLayoutEffect()` to ensure they're added synchronously. Unfortunately, the React team chose to [print warnings when `useLayoutEffect()` is used in an SSR environment](https://github.com/facebook/react/issues/14927). We had to do [some hacky environment detection and call `useEffect()` in SSR instead](https://github.com/reduxjs/react-redux/blob/01966db91e33aca533bb62f9f7a69ccd16ba6282/src/components/connectAdvanced.js#L35-L45). Neither of them run, but at least `useEffect()` doesn't warn.

#### The Return of Direct Component Subscriptions [🔗︎](#the-return-of-direct-component-subscriptions)

As with v5 and earlier, v7 wrapper components all subscribe to the store directly, and only get React involved when the selector logic determines that the wrapper component needs to re-render. This was the first key step in bringing performance back to the level of v5.

#### Use of React's Batched Updates API [🔗︎](#use-of-react-s-batched-updates-api)

React has always had an API called `unstable_batchedUpdates()`. Internally, React wraps all your event handlers inside of that, which is what allows React to batch together multiple state updates from one event tick into a single render pass.

The React team urged us to use `unstable_batchedUpdates()` directly in React-Redux. This was tricky, because it's actually exported from renderers like ReactDOM and React Native, not the core React package. React-Redux *should* work with any React renderer, so we couldn't add a direct dependency on either of those. We had to write some different wrapper files so that the `"react-dom"` import would get loaded in a web environment, and the `"react-native"` import when used with RN. For apps that might be using React-Redux with an alternate renderer, we added an additional entry point that falls back to a dummy batching implementation.

There have been [other Redux addons that make use of batched updates](https://github.com/tappleby/redux-batched-subscribe). I didn't want to dictate anything about what enhancers and store setup were needed. Instead, I was able to [use the batching inside our custom `Subscription` class](https://github.com/reduxjs/react-redux/blob/5c69baf817527ee9a742c9dc4d541945cb7d1719/src/utils/Subscription.js#L25-L299), and [updated `<Provider>` to create a root subscription](https://github.com/reduxjs/react-redux/blob/5c69baf817527ee9a742c9dc4d541945cb7d1719/src/components/Provider.js#L13).

#### Use of `React.memo()` for Prop Optimizations [🔗︎](#use-of-react-memo-for-prop-optimizations)

`connect` has always implemented optimizations similar to what `React.PureComponent` now does, but more extensive. It does checks on incoming props from the parent component, but ultimately only renders if the *merged* `stateProps + dispatchProps + ownProps` have changed.

React 16.6 introduced [`React.memo()`](https://reactjs.org/docs/react-api.html#reactmemo) as an alternative to use of `shouldComponentUpdate` or `PureComponent`. Like `PureComponent`, it checks to see if updates are necessary by doing a shallow comparison of previous and current props. Unlike `PureComponent`, which is an alternate base class component, `React.memo()` is a new component type that can wrap around either class or function components. In addition, it returns a very special object that looks like `{$$typeof: REACT_MEMO_TYPE, type : WrappedComponent, compareFunction}` ([see implementation reference](https://github.com/facebook/react/blob/769b1f270e1251d9dbdce0fcbd9e92e502d059b8/packages/react/src/memo.js#L27-L31)). This is interesting, because up until now all React components have been functions of some kind (as JS classes are actually functions). For the first time, a React component type may actually be an object instead, and so code that tries to determine if a value is a component by seeing if it's a function is now wrong.

Naturally, this meant that once we released v7, we began getting issues saying that people's code had broken because there were checks expecting all components are functions.

### API Changes [🔗︎](#api-changes-1)

#### Return of `store` as a Prop [🔗︎](#return-of-store-as-a-prop)

Now that components are subscribing to the store themselves again, it was easy to re-add the ability for connected components to accept `store` as a prop once more. That resolved the issues from it being removed.

#### New `batch()` API [🔗︎](#new-batch-api)

Since we'd already gone to the trouble of ensuring that we could import `unstable_batchedUpdates()` in both web and RN environments, we decided to re-export it as a public API named `batch()`. This allows end users to wrap parts of their code that are triggering multiple state updates outside of React's event handlers (such as an async function or a thunk), and minimize the number of re-renders that occur.

## v7.1: Hooks?!?! [🔗︎](#v7-1-hooks)

### Motivations [🔗︎](#motivations-2)

As soon as React hooks were announced, [people began asking when React-Redux would include a hooks-based public API](https://github.com/reduxjs/react-redux/issues/1063). (The React Hooks FAQ [even mentioned "`useRedux()"` as a hypothetical hook](https://reactjs.org/docs/hooks-faq.html#what-do-hooks-mean-for-popular-apis-like-redux-connect-and-react-router).)

By the time React hooks were officially released, there were already [numerous third-party Redux hooks libraries](https://twitter.com/adamklein500/status/1072457324932067329). (I later put together [a spreadsheet comparing the various library APIs and their popularity](https://docs.google.com/spreadsheets/d/1JfjcpaZkNjRLzA35uuPQpDLfKc-Z8LqSP7GJaUjH19o).)

Clearly, it was a question of *when* we would develop and publish a hooks API, not "if".

### Development [🔗︎](#development-1)

The v7.0 work derailed the hooks API discussions for a while. As we found out in [React #14110: Provide more ways to bail out inside Hooks](https://github.com/facebook/react/issues/14110), there's no way to stop updates caused by `useContext()`. That incredibly long discussion led to the React team advising us to switch back to direct subscriptions, and that meant that getting v7.0 out the door was a prerequisite for any kind of a hooks API.

After the initial hooks discussion thread got derailed, I [opened up a new API discussion thread](https://github.com/reduxjs/react-redux/issues/1179) in February based on some of the v7 discussions. The discussion continued there for a while, but I didn't pay much attention. In fact, as of late March 2019, [I replied to an "ETA?" question saying I was busy, and it was unlikely to be any time soon](https://github.com/reduxjs/react-redux/issues/1179#issuecomment-475722015).

But, with v7 in beta and nearing release, my mind started poking at the hooks question. About that time, I [published a Twitter poll asking what I should focus my time on next](https://twitter.com/acemarke/status/1109520804860051456), and 82% selected "Hooks". Clearly, the people had spoken :)

I started actively participating in the discussion thread, and we soon figured out there were some major issues we'd have to deal with. In particular, we couldn't enforce top-down updates in a hooks environment, because v7 relies on overriding context values to pass down nested `Subscription` instances, and you can't render context values from a hook. This meant users would potentially encounter the "zombie child" issue again.

I finally spent a couple days doing some analysis of all the third-party hooks libs I'd seen and the various approaches that had been offered, and wrote [a summary of how I thought we might be able to move forward](https://github.com/reduxjs/react-redux/issues/1179#issuecomment-480547676).

There were some ensuing digressions on topics such as use of Proxies to track updates and whether we could use the v6 and v7 approaches in parallel. After much extended debate, we concluded that we'd basically just have to give up on a technical solution to the "zombie child" issue, document the potential concerns, and move on.

A user named Jonathan Ziller had put together a package implementing his proposed set of hooks APIs, and [I finally suggested we should take that implementation as a PR](https://github.com/reduxjs/react-redux/issues/1179#issuecomment-484559515). After more bikeshedding over hook names, [we finally published our first alpha](https://github.com/reduxjs/react-redux/releases/tag/v7.1.0-alpha.1), which contained five hooks:

- `useSelector`
- `useActions`
- `useDispatch`
- `useRedux`
- `useStore`

The alpha cycle led to [another monster discussion thread (250 comments)](https://github.com/reduxjs/react-redux/issues/1252). During that process, we made three major changes:

- [v7.1-alpha.3 dropped `useRedux`](https://github.com/reduxjs/react-redux/releases/tag/v7.1.0-alpha.3), on the grounds that it didn't provide anything useful
- [v7.1-alpha.4 dropped `useActions`](https://github.com/reduxjs/react-redux/releases/tag/v7.1.0-alpha.4), after Dan Abramov strongly argued that it added too much complexity (including naming and variable scoping clashes)
- [v7.1-alpha.5 changed from shallow to reference equality for determining updates](https://github.com/reduxjs/react-redux/releases/tag/v7.1.0-alpha.5), under the idea that users would be more likely to select specific values rather than returning grouped objects

After sitting in alpha for a couple months, we finally zoomed through the RC stage and put out **[React-Redux v7.1.0](https://github.com/reduxjs/react-redux/releases/tag/v7.1.0)** in early June. (I was about to give [a talk based on this post at ReactNext](https://blog.isquaredsoftware.com/2019/06/presentation-react-redux-deep-dive/), so we ended up publishing v7.1 the night before the talk so I could announce it was live.)

### API Changes [🔗︎](#api-changes-2)

##### Hooks! [🔗︎](#hooks)

As mentioned, we ultimately wound up shipping three hooks:

- `useSelector`: subscribes to the store and returns the selected value
- `useDispatch`: returns the store's `dispatch` function
- `useStore`: returns the store instance itself

Due to the lack of top-down subscription enforcement, we made sure to [document potential edge cases](https://react-redux.js.org/api/hooks#usage-warnings) so that people are aware of those issues.

We also ended up adding [copy-pasteable recipes for `useActions` and `useShallowEqualSelector`](https://react-redux.js.org/api/hooks#hooks-recipes) to the docs as well.

## The Future [🔗︎](#the-future)

The existing `connect` API has been incredibly successful overall, and there's hundreds of thousands of apps using it. Our hooks API is new and certainly not as battle-tested, but we iterated on it sufficiently that I'm confident in how it works.

I'm truly hoping that React-Redux has stabilized after all the v6/v7 churn. I'm thankful that we managed to keep the public APIs consistent, since that meant that most folks were able to upgrade v5->v6->v7 without any real breaking changes. Still, I dislike that we've had to bump through a couple major versions, and hopefully we can leave things alone for a while.

But, a maintainer's work is never done. Some possible future considerations:

### Alternate APIs [🔗︎](#alternate-apis)

Prior to the announcement of hooks, we were frequently asked to provide [a "render props" form of `connect`](https://github.com/reduxjs/react-redux/issues/799). Now that hooks are a thing, that's unlikely to ever happen.

Beyond that, perhaps there's some alternative API approach that would be easier to use and work better with concurrent React down the road, and we just haven't thought of it yet.

### Concurrent React [🔗︎](#concurrent-react)

The React world has been eagerly awaiting the release of React's "Concurrent Mode" since Dan helped publicize it in his [JSConf Iceland 2018 talk "Beyond React 16"](https://reactjs.org/blog/2018/03/01/sneak-peek-beyond-react-16.html). The React team published [a roadmap in late 2018](https://reactjs.org/blog/2018/11/27/react-16-roadmap.html) suggesting they hoped to have it out by mid-2019, but as of June that hasn't happened yet, and recent comments indicate it'll still be a while.

Flarnie Marchan (formerly on the React core team) did a great talk at ReactNext in June entitled ["Ready for Concurrent Mode?"](https://youtu.be/V1Ly-8Z1wQA), where she laid how a summary of how Concurrent Mode works and some potential issues with existing code. It's very worth watching.

Long-term, we don't know how React-Redux will be able to fully interact with Concurrent Mode, largely because it isn't out and documented yet for us to be able to understand it. Someone just filed [an issue asking about our Concurrent Mode compat status](https://github.com/reduxjs/react-redux/issues/1351), and the answer is: "we dunno, we'll figure it out down the road". Keep an eye on that issue for future discussions.

### Future Context Improvements? [🔗︎](#future-context-improvements)

The most disappointing thing about v6 is that it *worked*, it just wasn't fast enough for real-world usage. It's possible that React might someday make changes to the context API that would allow us to reconsider context-based state propagation as a viable approach again.

As a potential example, Josh Story just filed React RFCs describing two potential rewrites of context: [lazy context propagation](https://github.com/reactjs/rfcs/pull/118) for faster updates with less overhead, and [context selectors](https://github.com/reactjs/rfcs/pull/119) to determine if contexts should update, as a replacement for `observedBits`. He also filed a proof-of-concept React-Redux PR [to rewrite `connect` to use that context selectors implementation](https://github.com/reduxjs/react-redux/pull/1350). Obviously, a lot would have to happen before we could ever actually use that (RFCs accepted, changes merged into React, new versions published), and it would require a new major version of React-Redux, but there's some potential there.

### Magic? [🔗︎](#magic)

Back when v6 was in development, I wrote a long post on ways we could potentially use Proxies for tracking state dependencies and optimize context updates based on that info: [React-Redux issue #1018: Investigate use of context + observedBits for performance optimization](https://github.com/reduxjs/react-redux/issues/1018).

Since then, Daishi Kato has been experimenting with various similar approaches, and currently has a small library called [`reactive-react-redux`](https://github.com/dai-shi/reactive-react-redux) that implements a Proxy-based `useTrackedState()` hook as an alternative to React-Redux. Long-term, that kind of approach is very intriguing.

I'd certainly love to hear feedback from the community on what forms of "magic" are acceptable, especially around optimizing component updates.

## Final Thoughts [🔗︎](#final-thoughts)

Hopefully this journey through time and release notes has been informative. As you've seen, **React-Redux has never been "magic" - just smart about implementing optimizations so you don't have to**. For all the internal complexity, it's still just a matter of subscribing to the store, checking to see what data your component needs, and re-rendering it when necessary. The implementations have changed, but the goals haven't.

This should also help explain **why you should use React-Redux instead of trying to write subscription logic yourself in your components**. The Redux team has put countless hours into optimizing performance, handling edge cases, and dealing with changes in the ecosystem. You should take advantage of all the hard work that's gone into this API! :)

As always, if you've got questions, please leave a comment, file an issue, or ping me **@acemarke** on Reactiflux and Twitter.

## Further Information [🔗︎](#further-information)

- [ReactNext 2019 talk: A Deep Dive into React_Redux](https://blog.isquaredsoftware.com/2019/06/presentation-react-redux-deep-dive/)