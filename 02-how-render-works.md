# How render works

This article belongs to the series [Read React Source Code](https://github.com/numbbbbb/read-react-source-code).

`ReactDOM.render()` is the most familiar function to React developers. In this article, we will:

- Prepare a small demo
- Trace the `render()` call

## Prepare a Small Demo

There are many demos in React's official website, I prefer this one in [Introducing JSX](https://facebook.github.io/react/docs/introducing-jsx.html):

```
function formatName(user) {
  return user.firstName + ' ' + user.lastName;
}

const user = {
  firstName: 'Harper',
  lastName: 'Perez'
};

const element = (
  <h1>
    Hello, {formatName(user)}!
  </h1>
);

ReactDOM.render(
  element,
  document.getElementById('root')
);
```

You can [try it on CodePen](http://codepen.io/gaearon/pen/PGEjdG?editors=0010).

Since `render()` is executed after compilation, we have to get the compiled JS code before jump into it.

How?

Easy. Open that [CodePen link](http://codepen.io/gaearon/pen/PGEjdG?editors=0010):

![](http://i.imgur.com/LSi8y9r.jpg)

Right click on that black hello and choose `Inspect`:

![](http://i.imgur.com/FKt1C1i.jpg)

You can see, CodePen just wrap the real code with an iframe. Click the bottom `<script>`:

![](http://i.imgur.com/R5XDxPW.jpg)

Got it!

```
function formatName(user) {
  return user.firstName + ' ' + user.lastName;
}

var user = {
  firstName: 'Harper',
  lastName: 'Perez'
};

var element = React.createElement(
  'h1',
  null,
  'Hello, ',
  formatName(user),
  '!'
);

ReactDOM.render(element, document.getElementById('root'));
```

At this time, JSX is compiled to pure JS, so we can start now.

> The compilation from JSX to JS is not complicated, you see, they just convert a JSX statement to a `React.createElement()` call and do that recursively. If you are interested in that process, you can [read the babel plugin](https://github.com/babel/babel/blob/master/packages/babel-helper-builder-react-jsx/src/index.js).

Okay, are you ready? Here we go!

## Trace the `render()` Call

First, we need to find its location.

The call is `ReactDOM.render()`, so we search `@providesModule ReactDOM` in `src/`:

![](http://i.imgur.com/a8w7cvv.jpg)

That special comment is useful when you do searching.

Double click it to open the file:

![](http://i.imgur.com/xh7Gnqf.jpg)

Search `@providesModule ReactMount`, jump into the file and search `render`.

So many `render` matches in this file! We have no idea how it's defined, so we must go through each match until...

![](http://i.imgur.com/e7CVuYv.jpg)

Read those comments first! This `render()` function can do both mount and update. Here we focus on the mount, in later articles we will come back and see how it does the updating(with the `domdiff` algorithm).

Go on, search `_renderSubtreeIntoContainer`. This function is too long to fit in the article, just list what it does:

- validate the callback
- use `invariant()` to do validation and print error message if it fails
- use `warning()` to do validation and print warning message if it fails
- wrap the `nextElement`, add some properties and methods
- get or create `nextContext`
- get `prevComponent`; if it exists and `shouldUpdateReactComponent()`, update it and return; if it exists and shouldn't update, unmount the component and go on
- defines `reactRootElement`, `containerHasReactMarkup`, `containerHasNonRootReactChild` and `shouldReuseMarkup`
- call `_renderNewRootComponent()`
- call callback if it exists
- return component

Jump to `_renderNewRootComponent()`, basically it just create the component instance with `instantiateReactComponent(nextElement, false)` and calls `ReactUpdates.batchedUpdates()`.

`nextElement` is created by `React.createElement` which is compiled from JSX. It's a React virtual dom node, contains many data and flags. You will reveal it's detailed in practice section by yourself.

![](http://i.imgur.com/kxtbfo2.jpg)

Here the important thing is the comment. Notice that the render process is synchronous and **ANY** updates happen during `componentWillMount()` and `componentDidMount()` will be batched.

**Batch**? What's that?

Okay, let me tell you now. Batch just means put off.

In next article we will trace a `setState()` call inside `componentDidMount()` and see how batch works.

Search and find `ReactUpdates.batchedUpdates()`:

![](http://i.imgur.com/Zps0PDT.jpg)

It passes all parameters to `batchingStrategy.batchedUpdates()`.

Search `batchingStrategy`.

![](http://i.imgur.com/6zmqn5G.jpg)

It's injected here. Why not write the implement here instead of injection? Because the batching strategy is dynamic, different users may choose or implement different strategies. So React use this injection pattern to extract the implement and inject it at runtime.

That also explains why we need to `ensureInjected()` before calling `batchingStrategy.batchedUpdates()`. The user can inject any objects they like, so we have to validate it before using.

Okay, now we need to search `injectBatchingStrategy()` globally and find who do the injection.

![](http://i.imgur.com/KaKFyOa.jpg)

Read those paths, it's clear we should go into the second result.

```
ReactInjection.Updates.injectBatchingStrategy(ReactDefaultBatchingStrategy);
```

Search `@providesModule ReactDefaultBatchingStrategy`, open the file and search `batchedUpdates`:

![](http://i.imgur.com/qBiFT9i.jpg)

If `alreadyBatchingUpdates`, run `callback`; otherwise run `transaction.perform()`.

Here we meet transaction first time. It's a useful pattern and used across the whole React project. We will cover it in next article, here you can treat it as a wrapper which will do some preparations before running `callback()` and do some cleanups after that.

Now it's time to run `callback()`. Wait, what's `callback()`? Let me scroll back and copy that for you.

![](http://i.imgur.com/kxtbfo2.jpg)

Yeah, it's `batchedMountComponentIntoNode()`! Open `ReactMount.js` and search it.

![](http://i.imgur.com/NoOSjOu.jpg)

Transaction again. Recall it's just a wrapper, focus on `mountComponentIntoNode()` now.

![](http://i.imgur.com/2BAO8pn.jpg)

This function generates the `markup` and mounts it to the container. 

`_mountImageIntoNode()` just mount the generated markup into DOM, I'll leave it to you. Here we search `@providesModule ReactReconciler`, open it and search `mountComponent`.

The most important line of `mountComponent()` is:

```
var markup = internalInstance.mountComponent(transaction, hostParent, hostContainerInfo, context, parentDebugID);
```

What is `internalInstance`? If you trace back the call stack, you can find it's an instance of `ReactDOMComponent`. 

![](http://i.imgur.com/qxKSIH6.jpg)

Here we have three instance types:

- if the element is basic DOM, it's type is string, like `h1`, it's a `ReactDOMComponent`
- if the element type is not string, it may be the internal component(check the `isInternalComponentType` function for details); this kind of element is rarely used in our projects, so you can ignore them
- if the element type is not string and not internal component type, it's a `ReactCompositeComponent`

You can treat `ReactCompositeComponent` as the enhanced `ReactDOMComponent`. Compared with `ReactDOMComponent`, `ReactCompositeComponent`:

- has many addon things like `state`, `props`, `render()`, `lifecycle hooks`
- has children, children can be both `ReactCompositeComponent` and `ReactDOMComponent`

If a `ReactDOMComponent` is the root, it will be wrapped by an empty `ReactCompositeComponent`.

Here because our element type is `h1`, we go to `ReactDOMComponent`. Search that, open the file and read `mountComponent()`.

It's a long function. But basically, it just does some initializations and generates the HTML markup based on the component.

Is this the end of our journey?

No. We have one question last. How to deal with children?

Generate the markup is easy, but children can be another component, so there must be some recursive process to generate markup of children. Let's try to find that process.

After looking through the whole function body, there are two lines seem related to children:

```
this._createInitialChildren(transaction, props, context, lazyTree);

var tagContent = this._createContentMarkup(transaction, props, context);
```

They are in an if and else clause where `mountImage` is set, so they must deal with the children.

After reading their implementations, we find that children's markup is generated by calling `this.mountChildren()`. It's defined in `ReactMultiChild` and mixed into `ReactDOMComponent`. Open `ReactMultiChild.js` and search `mountChildren`.

![](http://i.imgur.com/ikf8pbI.jpg)

There it is, our old friend `ReactReconciler.mountComponent`! That's the recursive process.

Notice that children's markup is generated before the parent, you can't build something before build parts included in it.

## Sum up

After such a long journey, we need to sum all things up.

![](http://i.imgur.com/WEtqCHj.jpg)


This picture can help you to understand and organize the entire render process.

## Next Step

Now we have rendered our component into DOM, but it's just a `ReactDOMComponent`. In next article, we will use the class to refactor our demo and see how `setState()` works.

## Practice

The `_mountImageIntoNode()` method in `ReactMount.js` can do the DOM operation. Read it implementation, find a friend and try to explain to him.



