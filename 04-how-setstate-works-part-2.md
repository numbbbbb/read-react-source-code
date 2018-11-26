# How `setState` works - part 2

This article belongs to the series [Read React Source Code](https://github.com/numbbbbb/read-react-source-code).

Okay, back to our `componentDidMount()`. In this article we will:

- execute our `componentDidMount()`
- understand how `Batching` works

## Execute Our `componentDidMount()`

After being enqueued and waiting for the mount process, finally our `componentDidMount()` is executed.

```
componentDidMount() {
  this.setState({
    name: 'Jack'
  })
}
```

Here we just call `this.setState()` with new value. Let's search `setState` and find its definition first.

Scroll down a little bit, you can see this.

![](http://i.imgur.com/L8PiHmZ.jpg)

Double click it.

![](http://i.imgur.com/ZFbjDqA.jpg)

What's the `updater`?

![](http://i.imgur.com/iiDcaGz.jpg)

It's a parameter of `ReactComponent()`, injected by other classes. We don't know where it's injected(because we don't have an injection method here for searching), but we can search `enqueueSetState` to see who has that method. There is a `SetState` in that method name, so seems it's not a generic method, maybe only used in `setState()`.

![](http://i.imgur.com/zdUUW8J.jpg)

Six matches, first is the calling, second is an abstract definition, next three are real definitions of server render, fiber, and stack. Recall that fiber is the new generation, so we go into the last one.

![](http://i.imgur.com/thLZIrj.jpg)

It pushes partial state(the parameter of `setState()`) into the `_pendingStateQueue` and calls `enqueueUpdate(internalInstance)`.

![](http://i.imgur.com/K6hoNpB.jpg)

Open `ReactUpdates.js` and search `enqueueUpdate`.

![](http://i.imgur.com/vx08OI6.jpg)

These two `if` clauses are more complicated than you think.

Let's simulate the execution with different `batchingStrategy.isBatchingUpdates` value.

### `isBatchingUpdates = false`

Now the first `if` is true, so it calls `batchedUpdates()`. Let's read `batchedUpdates()`(Recall it's inside `ReactDefaultBatchingStrategy.js` and injected to `ReactUpdate`).

![](http://i.imgur.com/fXXKnIM.jpg)

Here checks whether inside a batching update again. It's false, so we set it to `true`, then wraps and executes it with a transaction.

This transaction is a `ReactDefaultBatchingStrategyTransaction`, it will call `ReactUpdates.flushBatchedUpdates()` at the close time.

It's important, **batching update is implemented by Transaction**.

Remember what's the input function? Yes, the `enqueueUpdate()`. So inside transaction, we go back to `enqueueUpdate()` again. But this time we are inside a batching update, the `isBatchingUpdates` is `true`, so we skip the first if and go on. 

The component is pushed into `dirtyComponents`. If it doesn't have `_updateBatchNumber`, set it to `updateBatchNumber + 1`.

After doing this, the transaction will run those `close()`, inside that we run `flushBatchedUpdates()` which does the real updating.

Now let's see what will happen in the other situation.

### `isBatchingUpdates = true`

When would this be `true` when we call `setState()`?

Combine with previous articles, we know this situation will happen if we call `setState()` during mount process. Because the whole mount process is executed inside a batching update, so when we meet `setState()` and come to `enqueueUpdate()`,
`isBatchingUpdates` is `true`.

This is what happens to our demo code. The `componentDidMount()` is called after `ReactReconcileTransaction` finish it's main job. **But**, the `ReactReconcileTransaction` itself is the wrapper in a batching update! So at the time our `componentDidMount()` is executed, we are still inside the batching update.

If that confuses you, you can read the render article again. In render process, we first go into the batching update, then call `mountComponent()` which use `ReactReconcileTransaction` to separate DOM operation and hooks.

Okay, now `isBatchingUpdates` is `true`, so we skip the first if, push our component into `dirtyComponents` and set `_updateBatchNumber`.

That's all.

You may ask, so when will that component be updated?

Recall that we are already inside a batching update now, so when we finish the main job(render), it will call `flushBatchedUpdates()` at close time.

### Picture Time

A picture is worth a thousand words.

![](http://i.imgur.com/E9hCJYJ.jpg)

Above is the `render()` process with `setState()` in `componentDidMount()`.

![](http://i.imgur.com/xxOuiY7.jpg)

Above is the normal `setState()` process. Normal means not inside some batching update and the update will be performed immediately. Notice that most time `setState()` will be executed inside some batching transactions, like this demo, it's executed inside the batching transaction performed by `render()`. This makes sure that `setState()` will be batched and not executed immediately.

You may ask, why we need batching update in the second situation? In first situation it exists because we want to put off updates like `setState()`, but in the second situation seems we have no need to push some operations.

In order to answer this question, we need to go into `flushBatchedUpdates()` in next article.

## Next Step

Now we know how batching updates and nested transaction works. Wrapping operations in different custom transactions can decouple them from each other and easier to nest. Transaction can also help us put off operations and batching them.

In next article, we will go until the end of our `setState()` journey. Come on, we have almost made it!

## Practice

Write a small demo to show when `setState()` is sync and when is async. You can read the state immediately after `setState()` to show whether it changes.






