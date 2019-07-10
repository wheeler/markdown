# Codemods and automatic refactoring

## Should I codemod or should I lint rule?

\<flow chart\>

## Guessing how hard it's going to be

Editing small things in line is much easier than manipulating things.

Ex: change an identifier, change a literal, remove a statement, remove a comment.

Structural things are harder since you need to utilize API that is terribly documented, confusing, and you will have to account for many possible different structures.

Ex: return the last statement in a block,

Adding and moving line comments is perplexing. The line comments are stored in a separate structure and ordering them is currently confusing to me.

## JSCodeShift: Getting Started

1. Install jscodeshift (only needed the first time if you don't have it)
   ```shell
   $ npm install -g jscodeshift
   ```
1. Write a transform file
   - use https://astexplorer.net/
     - pick an appropriate language like `babel-eslint`
     - under transform pick `jscodeshift`
     - paste your code to modify in the top left box
     - write your transform in the bottom left box
     - preview the output in the bottom right box
   - Take the transform and save it in a file somewhere. I'll call mine `my_transform.js` and I saved it in `~/my_project/script/jscodeshift/`
1. Run your transform on a single file to test. Inspect the result to make sure you like the output
   ```shell
   $ cd ~/my_project
   $ jscodeshift -t script/jscodeshift/my_transform.js frontend/javascripts/components/**/*.jsx
   ```
1. Run your transform on all your target files

   ```shell
   $ jscodeshift -t script/jscodeshift/my_transform.js frontend/javascripts/components/**/*.jsx
   ```

   > **NOTE**: `/**/` is glob notation for "any amount of subdirectories". This won't work by default in Mac terminals. Apple doesn't like GPL3 so they haven't upgraded bash versions since 2007. That old version doesn't support \*_ (it treats it as just two _ wildcards and you'll only match files with exactly one folder depth). You can work around this by installing your own bash, running it in the normal terminal, then running your command.
   >
   > ```shell
   > $ brew install bash
   > ...
   > $ bash
   > bash-5.0$ jscodeshift -t script/jscodeshift/my_transform.js frontend/javascripts/components/**/>*.jsx
   > ```

1. Bask in the glory

Unfortunately... the "Write a transform file" step is really tricky.

## Ingredients:

Your file should `export default` a function that takes two parameters: `(file, api)`. The jscodeshift api is in `api.jscodeshift`. `file.source` contains the source code from the file beint transformed. Calling `j(file.source)` will parse the source into an AST that you can then manipulate. At the end of your transform you need to return the source. Every transform I write starts with this boilerplate outline:

```js
// my_empty_transform.js

export default function transformer(file, api) {
  const j = api.jscodeshift;
  let root = j(file.source);

  // Manipulate the AST

  return root.toSource({ quote: 'single' });
}
```

When working with jscodeshift you're mostly going to interact with these three special types: `Collection`, `Path`, and `Node`. In the above example `root` is a `Collection` containing one `Path` which has a `Node` that represents the root of the file.

A Collection is a collection of zero, one, or many `Path`s. Typically, you will be working with the `root` collection or with a collection you create by calling `.find` on another collection.

A `Node` is the description of a piece of code structure. It contains context on what the particular piece of code is and what structure it may contain.

A `Path` is a wrapper around `Node` that additionally contains information about it's parent and scope

---

## Collection Methods:

Here are a few basic descriptions of essential collection functions lifted from the [Collection docs](https://rawgit.com/facebook/jscodeshift/master/docs/Collection.html).

> #### `find(type, filter)` → {Collection}
>
> Find nodes of a specific type within the nodes of this collection.
>
> #### `findJSXElements(name)` → {Collection}
>
> Finds all JSXElements optionally filtered by name
>
> #### `filter(callback)` → {Collection}
>
> Returns a new collection containing the nodes for which the callback returns true.
>
> #### `forEach(callback)` → {Collection}
>
> Executes callback for each node/path in the collection.
>
> #### `replaceWith(nodes)` → {Collection}
>
> Simply replaces the selected nodes with the provided node. If a function is provided it is executed for every node and the node is replaced with the functions return value.

#### remove()

prunes all paths in collection

---

## Path Methods:

#### replace()

replaces the path with something else

#### prune()

removes the path from the tree

### Node Methods:

`find` - nope this isn't a thing.

---

# Cookbook (aka Examples):

## Remove an entire import statement based on imported name

Let's say I have some deprecated polyfill that I want to remove. One piece of the transform I want to do is remove all the import statements. I wan the code change to look like this:

```diff
  import foo from 'foo';
- import { deadPolyfill } from 'path/to/dead_polyfill';
  import bar from 'bar';
```

I came up with the following transform to do this:

```js
export default function transformer(file, api) {
  const j = api.jscodeshift;
  let root = j(file.source);

  root
    .find(j.ImportDeclaration)
    .find(j.ImportSpecifier)
    .filter(p => p.value.imported.name === 'deadPolyfill')
    .forEach(path => {
      path.parentPath.parentPath.replace();
    });

  return root.toSource();
}
```

note that `ImportSpecifier` is different than `ImportDefaultSpecifier` so a default import would not be changed. In my case that's ok.

```
import deadPolyfill from 'path/to/dead_polyfill'
```

### OK, it worked. Let's improve it!

The above transform was the first thing I got working when I was trying to figure out how to do this. Once I got it working I wanted to improve it and learn about some of the other tools along the way. Once you have something working you can try to make the code shorter, less confusing, more efficient, and less fragile.

1. Remove unnecessary nested `.find()`s

   It's we only want to look for `ImportSpecifier`s that are inside of `ImportDeclaration`s... but I'm fairly certain it's not possible for them to appear anywhere else. So, we can just skip that `find`.

   ```diff
   root
   - .find(j.ImportDeclaration)
     .find(j.ImportSpecifier)
     .filter(p => p.value.imported.name === 'deadPolyfill')
     .forEach(path => {
       path.parentPath.parentPath.replace();
     });
   ```

1. Use find's filter parameter instead of chaining `.filter()`

   In this case, because our filter condition is simple, we can replace `.fitler()` with a filter parameter on `.find()`. You can do this if you're simply filtering by matching nested properties. If you're doing something fancier (or want to debug each path) you may want to stick with `.filter()`.

   ```diff
   root
   + .find(j.ImportSpecifier, { imported: { name: 'deadPolyfill' } })
   - .find(j.ImportSpecifier)
   - .filter(p => p.value.imported.name === 'deadPolyfill')
     .forEach(path => {
       path.parentPath.parentPath.replace();
     });
   ```

1. Use `.closest()` & `.remove()` instead of replacing the parent path in `.forEach()`.

   `forEach` gives us the freedom to do whatever we want to the paths... and we need it because we're finding the right parent of the ImportSpecifier to remove. Collection has a `.remove()` function that will remove all the paths in the collection but we're finding ImportSpecifier, so if we called `.remove()` we would only remove the specifiers. Instead of removing the import line, it would change it to `import 'path/to/dead_polyfill';`. It's valid javascript... but it's not what we want.

   If we can change the collection into a collection of the ImportDeclarations then we can just call remove(). Since we already know from our inspection of the AST that the ImportSpecifier is always inside the ImportDeclaration we can use `.closest(j.ImportDeclaration)` to get a collection of the imports. Then we can just call remove.

   This is extremely preferable to `path.parentPath.parentPath.replace()` because we don't have to figure out how many parents up we have to go. In this case it happens to be always two but in other transforms we may have a variable number of parent steps which would be a pain to write out.

   ```diff
   root
     .find(j.ImportSpecifier, { imported: { name: 'deadPolyfill' } })
   + .closest(j.ImportDeclaration)
   + .remove();
   - .forEach(path => {
   -   path.parentPath.parentPath.replace();
   - });
   ```

1. (optional) Yolo the ImportDeclaration with a more specific find filter parameter

   If we could write a find filter parameter on ImportDeclaration that specified the imports we wanted then we wouldn't need to `.closest()` back up from the `ImportSpecifier`s. At some point this could become trickier than it is worth and the previous solution would be easier to write. I continued to look at the structure of the AST and was able to figure out the filter parameter object that has the same effect.

   ```diff
   root
   + .find(j.ImportDeclaration, { specifiers: [{ imported: { name: 'deadPolyfill' } }] })
   - .find(j.ImportSpecifier, { imported: { name: 'deadPolyfill' } })
   - .closest(j.ImportDeclaration)
     .remove();
   ```

1. (optional) Find using something simpler

   In this case I realized that my import source might be a simpler way to filter. Note that this might not work if the import source might have different variations (relative paths vs. absolute, etc.)

   ```diff
   root
   + .find(j.ImportDeclaration, { source: { value: 'path/to/dead_polyfill' } })
   - .find(j.ImportDeclaration, { specifiers: [{ imported: { name: 'deadPolyfill' } }] })
     .remove();
   ```

## Refactor away a wrapping function call

The second piece of the deprecated polyfill I want to remove is the calls. Fortunately my case is somewhat simple: the polyfill always takes one argument, and the refactor I want is to remove the call and leave the argument behind, alone and unwrapped.

```diff
  function renderThingy () {
    let qux = 'baz';
+   return render(qux);
-   return render(deadPolyfill(qux));
  }
```

This can be done using the `.replaceWith()` function on collection. I used the AST explorer to figure out the filter condition and where the argument to the call is. So, this will find all `deadPolyfill(someArg)` calls and replace it with `someArg` in the context.

```js
s.find(j.CallExpression, { callee: { name: 'deadPolyfill' } }).replaceWith(
  path => {
    return path.value.arguments[0];
  }
);
```

The power of AST codemods is starting to show because this would work for any kind of argument. It might have been possible to to the `import` removal using a regex, it would be nearly impossible to handle all variations of this. Here are just a few examples of things it would handle:

```js
deadPolyfill(someVar));

render(deadPolyfill(5));

{ outer: 'object', thing: deadPolyfill({ some: { deep: 'object' } } }

const func = deadPolyfill(() => {
  return 'anon function';
}));
```

:sparkle:
