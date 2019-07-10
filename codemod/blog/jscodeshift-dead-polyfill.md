# jscodeshift Recipe: Remove a dead polyfill function

Let's say I have some dead polyfill function that I want to remove. I have two code changes that need to happen all over the place. I want to remove all the import statements, and I want to not call the polyfill function anymore, instead leaving behind its single argument. A diff might look like this:

```diff
  import foo from 'foo';
- import { deadPolyfill } from 'path/to/dead_polyfill';
  import bar from 'bar';
  ...
  function renderThingy () {
    let qux = 'baz';
+   return render(qux);
-   return render(deadPolyfill(qux));
  }
```

This is a good candidate for codemod. Why?

- This is a one-time refactor.
- The refactor will remove all references to the polyfill file/package so I will remove also remove the polyfill in the same PR.
- Once that PR is landed nobody can even try to use it without having a compile error.
- Additionally, while it might be possible to replace the import using somewhat fancy regex, unwrapping the polyfill call would be quite hard unless you have some other special knowledge because there could be any manner of things as the argment: strings, vars, numbers, other function calls, inline functions, etc.

## Transform 1: Remove a specific import statement entirely

In my case, `deadPolyfill` is the only export from the polyfill file, so I can make a simplifying assumption that I can always remove the whole import statement. I'll add a detail about how to handle this if that was not true later. Here's the first version of the transform I came up with to remove the imports:

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

note that `ImportSpecifier` is a separate AST type from `ImportDefaultSpecifier`, so this will not find and change default imports ie. `import deadPolyfill from 'path/to/dead_polyfill'`. That's not my particular pattern so it's ok.

### OK, it worked! Let's make improvements and learn more!

I was writing and testing in AST Explorer and once this worked I wanted to improve it and learn about some of the other utility of JSCodeShift along the way. Once working it can be refactored iteratively to make the code shorter, less confusing, more efficient, and less fragile.

1. Remove unnecessary nested `.find()`s

   We only look for `ImportSpecifier`s that are inside of `ImportDeclaration`s... but logically they can't appear anywhere else. So, we can just skip that `find()`.

   ```diff
   root
   - .find(j.ImportDeclaration)
     .find(j.ImportSpecifier)
     .filter(p => p.value.imported.name === 'deadPolyfill')
     .forEach(path => {
       path.parentPath.parentPath.replace();
     });
   ```

1. Use find's filter parameter instead of a chained `.filter()`

   In this case, because our filter condition is simple, we can replace `.filter()` with a filter parameter on `.find()`. You can do this when you're filtering by equality matching nested properties. If you're doing something fancier (like filtering to multiple possible values) you may need to stick with `.filter()`.

   ```diff
   root
   + .find(j.ImportSpecifier, { imported: { name: 'deadPolyfill' } })
   - .find(j.ImportSpecifier)
   - .filter(p => p.value.imported.name === 'deadPolyfill')
     .forEach(path => {
       path.parentPath.parentPath.replace();
     });
   ```

1. Use `.closest()` + `.remove()` instead of replacing the parent path in `.forEach()`.

   `forEach` gives us the freedom to do whatever we want to the paths... and we need it currently to get the right `parent` of the `ImportSpecifier` to remove. Collection has a `.remove()` function that will remove all the paths in the collection but we're finding `ImportSpecifier`, so if we called `.remove()` we would only remove the specifiers. That would change the line to `import 'path/to/dead_polyfill';` instead of removing the whole import line. That _is_ valid javascript... but it's _not_ what we want.

   If we could map the collection into a collection of the parent `ImportDeclarations` then we can just call `remove()`. Since we already know from our inspection of the AST that the `ImportSpecifier` is always inside the `ImportDeclaration` we can use `.closest(j.ImportDeclaration)` to get a collection of the parent imports. Then we can just call `.remove()`.

   This is extremely preferable to `forEach ... path.parentPath.parentPath.replace()` because `closest()` automatically figures out how many `.parents` to go up to find the right path. In our case was luckily always two, but in other transforms we may have a variable number of `parent` steps which would be a pain to write out.

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

   If we could write a find filter parameter on `ImportDeclaration` that specified the imports we wanted then we wouldn't need to use `.closest()` up from the `ImportSpecifier`s. This one wasn't too hard to figure out but at some point this could become trickier than it is worth and the previous solution would be easier to write. Looking at the structure of the AST I was able to figure out the filter parameter object that has the desired effect. **NOTE**: this filter will only match imports that only have one specifier where the specifier is named `deadPolyfill`. It won't match imports with multiple specifiers where one is named `deadPolyfill`.

   ```diff
   root
   + .find(j.ImportDeclaration, { specifiers: [{ imported: { name: 'deadPolyfill' } }] })
   - .find(j.ImportSpecifier, { imported: { name: 'deadPolyfill' } })
   - .closest(j.ImportDeclaration)
     .remove();
   ```

1. (optional) `filter()` using something with less structure

   I considered that my import source might be a simpler way to filter than the specifier. **Note that this would not be easier** if the import source might have different variations (various relative paths, absolute, etc.). Because of that, in my case this wasn't useful, but I leave it here as an example of a different approach.

   ```diff
   root
   + .find(j.ImportDeclaration, { source: { value: 'path/to/dead_polyfill' } })
   - .find(j.ImportDeclaration, { specifiers: [{ imported: { name: 'deadPolyfill' } }] })
     .remove();
   ```

## Transform 2: Remove a wrapping function call

The second piece of the deprecated polyfill I want to remove is the calls. Fortunately my case is somewhat simple: the polyfill always takes one argument, and the refactor I want is to remove the call and leave the argument behind, alone and unwrapped. An example would be this:

```diff
  function renderThingy () {
    let qux = 'baz';
+   return render(qux);
-   return render(deadPolyfill(qux));
  }
```

This can be done using the `.replaceWith()` function on collection. We want to find the appropriate `CallExpression`s and call `replaceWith()` on the collection, returning the argument in the callback. I used the AST explorer to figure out the filter condition and where the argument to the call lives. The following transform will find all `deadPolyfill(someArg)` calls and replace it with `someArg` in the context.

```js
export default function transformer(file, api) {
  const j = api.jscodeshift;
  let root = j(file.source);

  root
    .find(j.CallExpression, { callee: { name: 'deadPolyfill' } })
    .replaceWith(path => {
      return path.value.arguments[0];
    });

  return root.toSource();
}
```

As I mentioned in the why, the power of AST codemods is really kicks in here because this would work for any kind of argument. It would be nearly impossible to handle all variations of this with regex. Here are just a few examples of things it would handle:

```js
deadPolyfill(someVar));

render(deadPolyfill(5));

{ outer: 'object', thing: deadPolyfill({ some: { deep: 'object' } } }

const func = deadPolyfill(() => {
  return 'anon function';
}));
```

## Result

Now we can put the two transforms together:

```js
export default function transformer(file, api) {
  const j = api.jscodeshift;
  let root = j(file.source);

  root
    .find(j.ImportDeclaration, {
      specifiers: [{ imported: { name: 'deadPolyfill' } }]
    })
    .remove();

  root
    .find(j.CallExpression, { callee: { name: 'deadPolyfill' } })
    .replaceWith(path => {
      return path.value.arguments[0];
    });

  return root.toSource();
}
```

One small additional optimization we could make would be to leverage our knowledge of the problem to speed up the transform a bit. We know that if we don't find the import we won't have any call expressions either so we don't have to look for them and there is nothing to modify.

```js
export default function transformer(file, api) {
  const j = api.jscodeshift;
  let root = j(file.source);

  const imports = root.find(j.ImportDeclaration, {
    specifiers: [{ imported: { name: 'deadPolyfill' } }]
  });

  if (imports.size() === 0) return;

  imports.remove();

  root
    .find(j.CallExpression, { callee: { name: 'deadPolyfill' } })
    .replaceWith(path => {
      return path.value.arguments[0];
    });

  return root.toSource();
}
```

If there are no matched imports this does two things:

1. It doesn't search read every `CallExpression` in the AST and check if it matches the filter.
2. It returns `undefined` which jscodeshift understands as "no transformation required". This saves turning the unmodified AST back into the source code AND comparing it with the original to see if there was a difference.

So, there we go, we have our desired transform and we learned some things along the way.

:sparkles:

## appendix

### handling one of many imports

If we didn't have the simplification that the import was always the only import from the source but instead it could either be the only one or one of multiple we'd have to write more flexible transform

example refactors:

```diff
// solo import
- import { deadPolyfill } from 'path/to/helpers';

// multiple imports
+ import { helperA, helperB } from 'path/to/helpers';
- import { helperA, deadPolyfill, helperB } from 'path/to/helpers';
```

The transform that we have written will remove the entire line only if there is one named import specifier. We'll have to roll back filter optimization #4 if we want to match imports with multiple named import specifiers. Once we find the imports it might be tempting to go back to the flexibility of `.forEach()` so we can handle them differently, but I found it easier to filter the collection into the two types we want to handle.

```js
export default function transformer(file, api) {
  const j = api.jscodeshift;
  let root = j(file.source);

  const imports = root
    .find(j.ImportSpecifier, { imported: { name: 'deadPolyfill' } })
    .closest(j.ImportDeclaration);

  if (imports.size() === 0) return;

  // If there is only one specifier, remove the entire import.
  imports.filter(p => p.value.specifiers.length === 1).remove();
  // For others, remove only the one specifier
  imports
    .filter(p => p.value) // this will match any ones not removed by the previous line
    .find(j.ImportSpecifier, { imported: { name: 'deadPolyfill' } })
    .remove();

  root
    .find(j.CallExpression, { callee: { name: 'deadPolyfill' } })
    .replaceWith(path => {
      return path.value.arguments[0];
    });

  return root.toSource();
}
```
