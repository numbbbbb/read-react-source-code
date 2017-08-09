# How setState works - part 3

This article belongs to the series [Read React Source Code](https://github.com/numbbbbb/read-react-source-code).

Finally, we come to `flushBatchedUpdates()`. In this article we will: 

- trace it until the end
- talk about the famous DOM diff


## `flushBatchedUpdates()`

It's in `ReactUpdates.js`.

![](http://i.imgur.com/L05CmrN.jpg)

It uses `ReactUpdatesFlushTransaction` to run `runBatchedUpdates()`. 

Read `ReactUpdatesFlushTransaction` first. It has two wrappers, first will deal with nested updates, it makes sure nested updates will be executed before running callbacks. The second wrapper is simply a callback executer.

Okay, let's read `runBatchedUpdates()`.

It first sorts dirty components to make sure parent is updated before children. Children may use their parent's props, so they must be updated later to get the latest prop values.

> It's interesting that `render()` build the component from bottom to top, while updates go from top to bottom.

Then it iterates dirty components and calls `ReactReconciler.performUpdateIfNecessary()`. Open `ReactReconciler.js` and search that.

![](http://i.imgur.com/IlkyMCE.jpg)

It calls `internalInstance.performUpdateIfNecessary()`. `internalInstance` in this demo is created by `ReactCompositeComponent`, so we go to that file and search the method.

![](http://i.imgur.com/lTWQGJj.jpg)

Recall that our parameter of `setState()` is stored in `_pendingStateQueue`, so we search `updateComponent()`.

This is a big method. In short, it generates new data and updates the component while calling hooks at the right time. It uses `_performComponentUpdate()` to perform updating, let's see it.

The key line in this method is `_updateRenderedComponent()`, search again.

![](http://i.imgur.com/KhXcBnq.jpg)

Oh! Our old friend `ReactReconciler.mountComponent()` again!

We can stop here, things after that call are all the same as `render()` process, just generate all markups and update the DOM.

Now we can answer the question in [previous article](): why should we wrap `flushBatchedUpdates()` with batching update? 

Let's imagine if we don't wrap it what would happen.

So now we run `flushBatchedUpdates()` directly. It will updates the component and corresponding DOM **while** calling hooks. If, for example, we call `setState()` again in `componentDidUpdate()`, what would happen?

Yes, we will come back to `flushBatchedUpdates()`. That's the problem.

If our dirty components are `[A, B]`, and `A` will call `setState()` in `componentDidUpdate()`, then `B` has to wait forever before `A` finishes. If we have more components, it means some unlucky components must wait for a long time before it can be updated.

So we must use batching updates here. Dirty components are updated one by one if someone triggers new updates, store and put off them until next batch.

![](http://i.imgur.com/W6dXQLr.jpg)

Above is the calling stack of `flushBatchedUpdates()`. It may be recursive if new updates exist, but they need to wait after this batch is finished.

## DOM diff

Now we have traveled to the end of `setState()`. But there is one thing left, the famous DOM diff.

Where is it?

In fact, it's not implemented as a standalone function. Instead, the comparing and updating just happen in `mountComponent()`.

You can read the related document at [Reconciliation](https://facebook.github.io/react/docs/reconciliation.html).

In my opinion, the concept behind that algorithm is such simple. React's view is built with `ReactElement`s and normal DOM elements. And all `ReactElement`s are eventually made of normal DOM elements. So basically, the algorithm just walk through the DOM tree with this comparing:

- are these two DOM types the same?
  - `true`, then just update props, attributes and so on. Then compare its children
  - `false`, drop the old element and generate a new one

Simple, right? React just combine this with its own `ReactElement` and `ReactComponent`. They also got their nickname, the virtual DOM.

## Next Step

Finally, we made it!

`setState()` is a complicated call. Different situations need different behaviors, many transactions are involved in this process.

You may still feel confused about this process. Don't worry, reading source code is hard, you have to be patient. `render()` and `setState()` are the most important parts of React, so make sure you really understand how they work. Give your brain some time to figure them out.

We have passed the hardest parts, in next article, we will see how React deals with events.

## Practice

Find out the real DOM operations and see where they are executed.


