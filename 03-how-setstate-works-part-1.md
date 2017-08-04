# How setState works - part 1

This article belongs to the series [Read React Source Code](https://github.com/numbbbbb/read-react-source-code).

Static component is boring. In this article we will:

- refactor the demo and add `setState()`
- trace the call and
- introduce Transaction

The whole call process of `setState()` is complex and long, so I break it into three articles. Enjoy it!

## Refactor the Demo and Add `setState()`

Refactor is not difficult for such a simple demo.

```
class Demo extends React.Component {
  constructor(props) {
    super(props);
    this.state = {name: 'John'};
  }
  
  render() {
    return (
      <h1>
        Hello, {this.state.name}!
      </h1>
    )
  }
  
  componentDidMount() {
    this.setState({
      name: 'Jack'
    })
  }
}

ReactDOM.render(
  <Demo />,
  document.getElementById('root')
);
```

Notice that I add `setState()` to `componentDidMount()` in order to show you how batching works.

Let's copy this to CodePen and get the compiled code with the skill we learn in [previous article](https://github.com/numbbbbb/read-react-source-code/blob/master/02-how-render-works.md).

![](http://i.imgur.com/Fhdm0dk.jpg)


We can ignore those helper functions. You see, our class is compiled to a function, `render()` and `componentDidMount()` is set to its prototype.

With our demo, we can start to trace the code execution.

## Trace the Call

In [previous article](https://github.com/numbbbbb/read-react-source-code/blob/master/02-how-render-works.md) we have seen the `render()` process of `ReactDOMComponent`. Now we change the plain DOM tag to our custom element, so it goes to create a `ReactCompositeComponent`.

The beginning calls are the same, but it diverges when you meet `internalInstance.mountComponent()` in `ReactReconciler.js`. Because the instance is different, now we should go into `ReactCompositeComponent` to look for `mountComponent()`.

The heart of this article is `setState()`, so I'll leave this different `mountComponent()` to you.

`setState()` is in `componentDidMount()`, so let's search it and start from that call.

![](http://i.imgur.com/4UbXH3v.jpg)

```
transaction.getReactMountReady().enqueue(inst.componentDidMount, inst);
```

We need to find out what is that `transaction`.

Go back through the call stack, we find it's set in `ReactMount.js`.

![](http://i.imgur.com/Gm6D7UE.jpg)

Jump into `ReactUpdates.js` and search `ReactReconcileTransaction`. Oh, it's injected. Search `injectReconcileTransaction(`(notice I add a `(` to filter out calls), we find the injected value is `ReactReconcileTransaction`.

Open `ReactReconcileTransaction.js`, here we will look transaction in detail.

## Transaction

In the previous article, we have said that Transaction is basically a wrapper. You can open `transaction.js` and read it's comments. Here is the behavior of Transaction:

- you call `transaction.perform(callback)`
- that transaction instance calls `getTransactionWrappers()` to get wrappers
- it iterates all wrappers and calls their `initialize()`
- then calls `callback()`
- finally, it iterates all wrappers and calls their `close()`

Transaction has two mainly usages:

1. initialize and cleanup, like a class `destructor()`
2. put of some operations and execute them during `close()`, `ReactDefaultBatchingStrategy` implements this pattern

Transaction itself is generic, most time you need to custom your own transaction class.

Let's see how `ReactReconcileTransaction` is defined.

First, it defines three wrappers, `SELECTION_RESTORATION`, `EVENT_SUPPRESSION` and `ON_DOM_READY_QUEUEING`. They all have `initialize()` and `close()`, comments can help you understand their function.

Then defines `ReactReconcileTransaction()` and it's mixin. `Transaction` and `Mixin` is assigned to `ReactReconcileTransaction.prototype`, so all methods of them can be called from the instance. That's how `Transaction.perform()` gets the wrappers you define.

Finally, use `PooledClass.addPoolingTo()` to add pooling to `ReactReconcileTransaction`.

What's `PooledClass`?

Search and open the file, it implements a generic pooling. When you call `addPoolingTo()`, it will define `getPooled()` and `release()` to target's prototype. So in your code you call `obj.getPooled()` instead of `new obj()` to get an instance.

Pooling is not complex, just store instances in an array, reinitialize and return them in `getPooled()`. The tricky part is how to do the reinitialize. This is a generic class, so it must delegate the reinitialization to its target class. Let's see the `oneArgumentPooler()`.

![](http://i.imgur.com/gsrg2Ne.jpg)

If the instance array is empty, it calls `new Klass()` to get a new instance. If the pooling is not empty, it just takes one and calls `Klass.call()` to do reinitialization.

Go back to `ReactReconcileTransaction`, we can see it calls `this.reinitializeTransaction()` which is a prototype method defined in `Transaction`.

With `Transaction` you can extract common parts in process and create a wrapper to use in the process. With `PooledClass` you can optimize your wrappers with pooling. It's important, because in real projects some process may be executed frequently(like `render()`), so reduce the need for creating a new instance is necessary.

Now we can go back to our start point.

```
transaction.getReactMountReady().enqueue(inst.componentDidMount, inst);
```

From `ReactReconcileTransaction.js` we know `getReactMountReady()` returns a queue. When we meet `componentDidMount()` and `componentDidUpdate()`, they will be enqueued and executed after the whole mounting process.

Why should we use a queue to store hooks? Because our component may have child components which also define hooks. The component is mounted to DOM as a whole, you can't mount child A, call its hooks, mount father, call its hook. It's unnecessary and inefficient, so we just mount the whole component with all its children and call any hooks defined in and inside this component.

The call of our `componentDidMount()` is in `ReactReconcileTransation.js`.

![](http://i.imgur.com/IHST864.jpg)

After the mount process, Transaction will iterate and call wrapper's `close()`, and this `close()` will iterate and call all hooks.

## Next Step

In this article, we have learned `Transaction` and `PooledClass`. We also know when and how our `componentDidMount()` is called.

In next article, we will go on and execute our `componentDidMount()` and see what will happen. Hope you still remember that we have a `setState()` inside it.

## Practice

Read the comments in `Transaction.js`, the character picture inside that will help you understand Transaction better.

Read `PooledClass` again, simulate the call `getPooled()` and `release()`, figure out how it manages the instance in the pool with the size limitation.


