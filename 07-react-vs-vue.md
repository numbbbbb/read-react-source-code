# React vs Vue

Before React, I also read and wrote a series called [read Vue source code](https://github.com/numbbbbb/read-vue-source-code). So in this article, I want to:

- briefly introduce Vue, and
- compare Vue and React from source code view

## Vue Introduction

Vue is a rapidly growing JavaScript framework for building UI on the web. It has about 60,000 stars on GitHub and is chosen by many companies to build their web projects.

Vue is similar to React in many aspects. Let's see a small Vue demo:

```
<div id="app">
  {{ message }}
</div>

var app = new Vue({
  el: '#app',
  data: {
    message: 'Hello Vue!'
  }
})
```

This demo creates a Vue component and mounts it to the page.

A bigger demo:

```
<div id="example">
  <p>Original message: "{{ message }}"</p>
  <p>Computed reversed message: "{{ reversedMessage }}"</p>
</div>

var vm = new Vue({
  el: '#example',
  created () {
    console.log('now I'm created')
  },
  data: {
    message: 'Hello'
  },
  computed: {
    // a computed getter
    reversedMessage: function () {
      // `this` points to the vm instance
      return this.message.split('').reverse().join('')
    }
  },
  watch: {
    message () {
      console.log('My message is changed to ' + this.message)
    }
  }
})
```

`data` in Vue is `state` in React. But you don't need to use `setData()`, just modify it directly like `this.message = 'new message'`, Vue will deal with the view updating.

This is not a Vue tutorial, if you want to read more about Vue, just go to its [official documents](https://vuejs.org/).

## React vs Vue

Vue has a [section](https://vuejs.org/v2/guide/comparison.html#React) in its official document to compare React and Vue, you can read that first.

That's the author's words from the user's point of view. I also used React and Vue in my projects, but Here I want to compare them from the source code angle.

Before reading React's source code and writing this series, I have done the same thing and written [Read Vue Source Code](https://github.com/numbbbbb/read-vue-source-code). So now I have both of them in my mind for comparing.

Okay, let's go.

#### Build Process

Vue uses rollup in build process, this makes it cleaner and easier for maintaining.

React uses gulp, grunt, and browserify. It makes the build process complicated and hard to maintain.

I see React also uses rollup in its master branch, a wise choice.

#### Code Organization

![](media/14994212700475/14994791004583.jpg)

![](media/14994212700475/14994791097792.jpg)

Which one is better?

First is Vue's source directory, second is React.

In my opinion, I think Vue is clearer. When I was reading their source code, most time it's easier to find something in Vue than React.

I don't know what's the rules Facebook guys use to split their code, but I do find some similar modules located in different places(like those event plugins). 

#### JSX vs Template

Some people don't like JSX, but I think the syntax is just a small thing. The core difference is the concept behind the syntax.

JSX inserts HTML tags into your JS codes. this may seem convenient at first, but when your project gets bigger and bigger, your component will contain many HTML tags. If you want to modify something, you have to find where is it and understand the JS around it. It also sucks that you have to press `<`, `>`, `/`, `\` when you are writing JS.

Vue use the traditional template but combines it with scripts and styles into a single `.vue` file.

So most time your component looks like this:

```
<template>
  ...
</template>

<script>
  ...
</script>

<style>
  ...
</style>
```

It doesn't mix them, but make them close enough for you to read and modify.

I don't hate the JSX syntax, but I prefer the way Vue organizing your code.

#### State Change

In React, you must call `setState()`. And, it may be async, so if you want to make sure getting the new value, you have to wrapper it in a callback.

In Vue, you just modify the state in the easiest way `this.name = 'new name'`. You can also read it immediately, no need to worry about that. Besides, you can modify state at any time, no rules you must follow.

#### View Render

Both React and Vue will update your view based on your new states. But they go with completely different ways.

React will mark dirty components and call their `render()` function to get new DOM, then compare them or call your `shouldComponentUpdate()` function. In other words, React doesn't know whether the view needs updating, it just know **maybe** the view needs updating. So you have to be very carefully to your states and `render()` function, otherwise you may trigger unnecessary render and compare.

Vue is smarter. You don't need to define that `render()` function, Vue will compile your template into a render function and build the dependency during render process. Thus, Vue **know** which part of your view need updating when some state is changed. I really love this design.

In short, React may give you more freedom to organize your HTML tags, but it also forces you to care more about data and render.

> "Why my project runs such slow?"
> "Cause you write the render in the **wrong** way."
> "My fault? Damn it!"

## Sum up

The core concept of Vue and React is the same, they both want to make it easier for you to build big web applications. They both choose the component model, virtual DOM and update DOM automatically.

However, in some detail things, Vue do more for you. When I write Vue projects, I focus more on my own logic. But when I write React projects, I have to figure out what data is state and how to make my render function works well. I think a good framework should be smart and do most things for their user. That's the reason why we learn and use them.

I also have to say that in some fields React is better than Vue. One of my friends tells me that the reason they choose React is it's easier to test. I think that because React asks you to write many important functions(like render), so you can compare it's return value directly. Vue does compilation and optimization so you must read DOM to do the test.

In conclusion, I think Vue is easier and more nature to use than React. But if you are really stuck in testing, maybe you can try React.


