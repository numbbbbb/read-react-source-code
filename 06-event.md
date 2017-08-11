# Event

This article belongs to the series [Read React Source Code](https://github.com/numbbbbb/read-react-source-code).

In this article, we will:

- find the call spot of event functions
- understand how events are managed

Compared with previous articles, this one is much easier. The key point of this article is not the event itself, but **how to read source code**.

We have solid the understanding of `render()` and `setState()` processes, thus we can find whatever we want based on them.

Here we go.

## Find the Call Spot

Okay, now our target is understanding how React manages events. Where to start?

Even if you can't remember each line in `mountComponent()`, you can guess it must deal with events.

But we have many `mountComponent()`, which one should we choose? Obviously, we should go to `ReactDOMComponent`, which will create the real DOM elements.

Open it, search `mountComponent()` and read it.

![](http://i.imgur.com/x24ZWWG.jpg)

These two calls may have something to do with events. Here creates the real DOM element, and if I want to listen to events, this is the best place.

Let's read `_updateDOMProperties()` first.

![](http://i.imgur.com/JTwwZKn.jpg)

Yes, it does call functions related to events. After trace, we find `registrationNameModules` is injected and generated. 

![](http://i.imgur.com/TyzCSTQ.jpg)

These are events we need to register. You can go into them for more details.

Okay, let's read `_createOpenTagMarkupAndPutListeners()`.

![](http://i.imgur.com/IG4G88A.jpg)

No `deleteListener()` here, because this function return pure string. When the string is mounted to DOM, old elements will be replaced, so we don't need to care old event listeners on them.

Both methods come to `
enqueuePutListener()`, read it.

![](http://i.imgur.com/lpLKtnx.jpg)

It first calls `listenTo()`, then get a queue and enqueue `putListener()` with parameters. Recall that this queue will be executed after mount process, it's the same queue to store `componentDidMount()`.

### `listenTo()`

![](http://i.imgur.com/mdI6M7B.jpg)

![](http://i.imgur.com/X8wOpj6.jpg)

React doesn't listen to target directly, instead, it listens to `document` and triggers your listener when the event bubbles up. So for each event, there is only one real listener on `document`.

Let's see how `ReactBrowserEventEmitter.ReactEventListener.trapBubbledEvent()` works.

`ReactEventListener` is injected, it's in `ReactEventListener.js`.

![](http://i.imgur.com/Y0VCQGX.jpg)

It uses `EventListener.listen()` to do the real listen.

Search for `EventListener` and...what? No results?

![](http://i.imgur.com/HwSPPUr.jpg)

Recall that during the build process, the module map also contains `fbjs`, let's search inside `node_modules/fbjs`.

![](http://i.imgur.com/FrKFTQ7.jpg)


Yes, it's here.

![](http://i.imgur.com/k8RPq09.jpg)

Now we know where and how React does the event listening.

Next, go back to our `putListener()`.

### `putListener()`

Inside `putListener()`, it calls `EventPluginHub.putListener()` with instance, event name and listener. Let's open `EventPluginHub` and read its `putListener()`.

![](http://i.imgur.com/lLZIsjR.jpg)

It gets the special key of this instance, gets or initialize the listener store object and store the listener.

Then it checks if this event has `didPutListener()` hook and calls it.

## Next Step

The pattern React uses to do event listening is subtle but efficient. In fact, React has a concept called `SyntheticEvent`, it's a wrapper to encapsulation and enhances the real events.

With `SyntheticEvent`, React can apply its event system to different platforms and add more custom event plugins to make your app stronger.

Now you have learned the most important parts of React and how to use them to understand other codes. In next article, I will compare React with Vue from source code view.

## Practice

Find where React uses `SyntheticEvent` and compare its implementation with native events.

Hint: trace the `listenTo()` and simulate an event to see how listener works.


