
# Start with the Messy Build Process

At the time I write this series, the latest React release version is `v15.6.1`.

Find your favorite directory and clone the React repo:

```
git clone git@github.com:facebook/react.git
cd react
```

In order to be consisted with documents, we checkout the branch for v15:

```
git checkout 15-stable
```

> Why not use `master`? Because master branch is used to develop the latest React code, it's different from what we use in the daily project. But `master` branch has many new features and refactors, so you can read it after reading this series.

Let's try to build it:

```
npm install
npm run build
```

You may get the error `Current node version is not supported for development`, that's because your node version is not supported by React. React only support `4.x || 5.x || 6.x || 7.x` at the time of this series. I use node v8.1.3, so I need to install 7.x:

```
npm install -g n
n lts
```

Okay, now run install and build again.

Open `build` folder, look through those subfolders, you can find many familiar files, like `react-with-addons.min.js`.

That's enough for warming up, now turn to our first mission.

## Find the Entry

React is a big project, so the first thing we need to do is finding its entry.

Open your favorite editor and drag `react` folder into it. I use Sublime Text 3.

For an npm project, we can simply open `package.json` and find `scripts`:

```
"scripts": {
  "build": "grunt build",
  "linc": "git diff --name-only --diff-filter=ACMRTUB `git merge-base HEAD master` | grep '\\.js$' | xargs eslint --",
  "lint": "grunt lint",
  "postinstall": "node node_modules/fbjs-scripts/node/check-dev-engines.js package.json",
  "test": "jest",
  "flow": "flow",
  "prettier": "node ./scripts/prettier/index.js write"
}
```

The `npm run build` command runs `grunt build`, so open `Gruntfile.js` and search `build`:

![](http://i.imgur.com/9JeRQ1l.jpg)

The task `build` contains many sub tasks, you can search their name to see the detail. 

For example, let's search `build-modules`:

![](http://i.imgur.com/7MgCCuj.jpg)

It calls gulp to do the work. Open `gulpfile.js` and search `react:modules`:

![](http://i.imgur.com/KUBB751.jpg)

Go back to `Gruntfile.js` and search `browserify:basic`, no `registerTask`? Search `browserify`:

![](http://i.imgur.com/d636hHu.jpg)

![](http://i.imgur.com/SNYDO2z.jpg)

The config of browserify tasks is imported from `grunt/config/browserify` and executed by `grunt/tasks/browserify`. Open `grunt/config/browserify` and search `basic`:

![](http://i.imgur.com/tFQKvme.jpg)

Wow! Here is the entry and other configure.

Now you know how to trace the build tasks, here is the list of them:

- **delete-build-modules** delete all modules generated in last build
- **build-modules** build all js files
- **version-check** check whether the version of `react`, `react-dom`, `react-test-renderer`, `src/ReactVersion.js` equals to the version in `package.json`
- **browserify:basic** build `react.js`
- **browserify:addons** build `react-with-addons.js`
- **browserify:min** build `react.min.js`
- **browserify:addonsMin** build `react-with-addons.min.js`
- **browserify:dom** build `react-dom.js`
- **browserify:domMin** build `react-dom.min.js`
- **browserify:domServer** build `react-dom-server.js`
- **browserify:domServerMin** build `react-dom-server.min.js`
- **browserify:domFiber** build `react-dom-fiber.js`
- **browserify:domFiberMin** build `react-dom-fiber.min.js`
- **npm-react:release** build the npm project of React
- **npm-react:pack** pack the react npm project into a `.tgz` file
- **npm-react-dom:release** build the npm project of ReactDOM
- **npm-react-dom:pack** pack the react-dom npm project into a `.tgz` file
- **npm-react-test:release** build the npm project of React test render
- **npm-react-test:pack** pack the react-test-renderer npm project into a `.tgz` file
- **compare_size'** compare and output the size before and after gzip

There are 3 main projects:

- **React**: with or without addons
- **ReactDOM**: including `ReactDOMServer` and `ReactDOMFiber`
- **ReactTest**: used for testing

This series focuses on React itself, so we ignore the `ReactTest` parts. As for fiber, it's the next generation architecture of React and still in progress now, so we will also ignore that part. If you want to learn fiber, read [this article](https://github.com/acdlite/react-fiber-architecture) **after** reading this series.

The main targets for this series are `React` and `ReactDOM`(without fiber).

Let's sum up the architecture of build process.

![](http://i.imgur.com/x5akcyy.jpg)


Is that enough for build process?

No. Let's look at `build-modules` again.

## `build-modules`

Why should React build modules? Why not just set an entry and generate the final JS file? There must be some reasons to do this.

Let's figure it out.

Go back to `gulpfile.js` and find the `react:modules` task:

![](http://i.imgur.com/KUBB751.jpg)

Scroll up, you can find the definition of `paths.react.src`:

![](http://i.imgur.com/mnjZPEw.jpg)

Let's open the first one `src/umd/ReactUDMEntry.js`:

![](http://i.imgur.com/NLCVNFV.jpg)


Uh, a simple file, nothing strange.

But...wait! Look at that `require`:

![](http://i.imgur.com/miuCvaE.jpg)


Remember that in Node.js's document, it says:

> Without a leading '/', './', or '../' to indicate a file, the module must either be a core module or is loaded from a node_modules folder.

Obviously, `React` is neither a core module nor an npm project
inside `node_modules`!

Maybe we can find the generated file after run `build-modules` and compare them.

Read the gulp task `react:modules` again, it stores the output file in `paths.react.lib`, which comes to be `build/node_modules/react/lib`. Go to that directory:

![](http://i.imgur.com/P0oWamm.jpg)

Oh man, seems all files are generated to this `lib` directory. Open `ReactUDMEntry.js`:

![](http://i.imgur.com/OW26R4D.jpg)

It's `./React` now! And yes, there **is** a `React.js` file in the same directory.

We can guess what happens here. The `build-modules` task extracts all JS files that match the `path.react.src`, modify their `require()` part according to **some rules** and put them in the `path.react.lib` directory.

We need to figure out those rules.

Let's try to find the source `React.js` file. After some clicks, I find it's in `src/isomorphic` directory. Open it:

![](http://i.imgur.com/dutiemn.jpg)

What's that in the comment?

```
@providesModule React
```

Hey, I bet this is the **rule** we are looking for. Each module has this line in their comments. the task will walk through all JS files, build a map based on this and rewrite the path in `require()`.

Read the task again:

![](http://i.imgur.com/8GiDadd.jpg)

It reads all files, passes them through babel, calls `stripProvidesModule()`, calls  `flatten()` and writes them to the destination.

After reading the code, we know `stripProvidesModule()` just remove that special comment line and `flatten()` flat all files into the same directory.

So the key part is `babel(babelOptsReact)`. Search `babelOptsReact`:

![](http://i.imgur.com/GaejhIs.jpg)

`babelPluginModules` is imported from `node_modules/fbjs-scripts/babel-6/rewrite-modules.js`. Open it:

![](http://i.imgur.com/lxQe2K3.jpg)

This function does the map. It first checks if the input module is in the map if so returns the corresponding value, otherwise, adds the prefix which is `./` by default.

So what's that map?

Go back to `babelOptsReact`, the map is `moduleMapReact`:

![](http://i.imgur.com/GEkQ248.jpg)

It combines `moduleMapBase` with five addons. The `moduleMapBase` contains five modules for compatibility and `require(fbjs/module-map)`. Open `mode_modules/fbjs/module-map`:

![](http://i.imgur.com/5Z3tgNX.jpg)

It's a map for all utils inside `fbjs` package.

So the `moduleMapReact` contains:

- five modules for compatibility
- fbjs utils
- five React addons

No modules marked by `@providesModule`?!

Short answer: yes.

I have the same feeling as you that maybe I ignore some function calls or `require()` statements. But in the end, I understand what happens here.

All modules are flattened to the same directory, so there is no need to map them, just use the default `./` prefix!

Okay, so you Facebook guys just add those special comments and remove them during building.

Now we know how `build-modules` works. But why it exists?

I will leave this as a practice for you, just read another task, and see how simple it is with the files generated by `build-modules`.

Is that enough for build process?

Yes.

## Next Step

We have figured out the build architecture and how `build-modules` works. In next article, we will see a small demo and try to understand how `render()` works.


## Practice

Choose a task to trace and tell what it does, where are the output files. For example, `browserify:addons`.

If there are no `build-modules`, how to modify those `require()`s and how to do the building?


