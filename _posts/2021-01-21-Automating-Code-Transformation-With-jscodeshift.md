---
title: "Automating Code Transformation With jscodeshift"
date: 2021-01-21
author: Mark Skelton
categories: JavaScript codemod jscodeshift
excerpt: "Tired of manual refactoring? This article shows how to automate code transformations using jscodeshift to reduce manual work when performing jrefactors."
---

The JavaScript ecosystem is fast paced and continuously changing. By the time you finish adopting some fancy new library, another library has come along with people telling you to drop everything and adopt it. Even if you pick a library and stick with it, breaking changes happen from time to time, which add great new features but sometimes mean a long or painful refactor.

As a developer, there are a few ways of dealing with this problem. The first approach would be to never update your libraries so you don't have to spend time on updates. While this might save you some time, it also means you won't be getting security updates or access to new features that could save time in other ways. Another approach would be to always update your libraries to the latest versions soon after they come out. While this is better than the previous approach, it also means you will be spending a lot of extra time refactoring your applications.

In this article, I'll discuss a third approach, which uses a tool called jscodeshift to automate code transformation to speed up refactoring so you can take advantage of new libraries or updates without having to manually migrate your project(s).

_You can find the code from this article, as well as an example Jest test suite, in the [accompanying GitHub repo](https://github.com/Widen/jscodeshift-example)._

## Introduction to jscodeshift

[jscodeshift](https://github.com/facebook/jscodeshift) is a tool created by Facebook that provides a simple API to perform abstract syntax tree (AST) transformations on your project's source files. We can use these APIs to create custom "codemods" to transform our code for whatever needs we have. An example of this might be removing an imported variable from a library and instead importing it from a file in your project. The code block below demonstrates this example, which I'll use for the remainder of this article.

```js
// Input
import { useQuery, queryCache } from "react-query";

// Output
import { useQuery } from "react-query";
import { queryClient as queryCache } from "common/queryClient";
```

## Getting started with jscodeshift

Hopefully, by now, you have an idea of the purpose of jscodeshift, so let's dive in and write an example codemod from scratch to solve the use case I showed in the last code block.

First, we need to install jscodeshift.

```sh
npm install -g jscodeshift
```

This will install jscodeshift globally so we can run it for any project. Next, let's create a file called `transform.js` with the following boilerplate for our codemod.

```js
// transform.js
module.exports = function (fileInfo, api) {
  const j = api.jscodeshift;
  const source = j(fileInfo.source);

  // Your transformations go here

  return source.toSource();
};
```

You don't need to understand this boilerplate in great depth, but a quick explanation of its purpose doesn't hurt. jscodeshift expects that transform scripts export a function that will accept two arguments with the file information and the jscodeshift API. We can then call the `api.jscodeshift` method (which we alias as `j`), passing it the input file source code. Our transformations would be the next step of the codemod, but for now we won't do anything. Finally, we'll call `toSource(),` which will convert the transformed AST back into a string, which jscodeshift will write back to the input file.

If any of that was confusing, don't worry. This boilerplate is pretty much the same for all codemods, so the important part to learn is the actual code transformation portion.

## Applying transformations

When writing codemods, it's often best to break up the problem into each part and then tackle each part separately. This is especially helpful when writing a large codemod with lots of steps and options. For this codemod, there are two steps we need to take:

1. Remove the `queryCache` variable from the `react-query` import, if it exists.
1. If `queryCache` was imported, add a new import declaration to import it from the new location.

We'll start with step 1, but first let's do a quick review of the AST for our example file. Below is a rough outline of the AST, which shows that our file has an `ImportDeclaration` node that contains two `ImportSpecifier` nodes. Each import specifier has an `imported` name and a `local` name that will be important for step 2.

```md
- ImportDeclaration:
  - source
    - Literal
      - value: react-query
  - specifiers [2]
    - ImportSpecifier
      - imported
        - Identifier
          - name: useQuery
      - local
        - Identifier
          - name: useQuery
    - ImportSpecifier
      - imported
        - Identifier
          - name: queryCache
      - local
        - Identifier
          - name: queryCache
```

_To see the AST of the source code in your project(s), check out [AST Explorer](https://astexplorer.net) which provides a fantastic AST visualizer._

### Removing the `queryCache` variable

Now that we know what our AST looks like, we can get down to business writing our transformation. To remove the `queryCache` import specifier from the `react-query` import, use the jscodeshift `find`, `filter`, and `remove` methods using the following three steps.

1. Find all the import specifiers in the file.
1. Filter to just the `queryCache` import specifier from the `react-query` import.
1. Remove the `queryCache` import specifier.

This would look like this in our `transform.js` file.

```js
// Your transformations go here
source
  .find(j.ImportSpecifier)
  .filter((path) => path.parent.value.source.value === "react-query")
  .filter((path) => path.value.imported.name === "queryCache")
  .remove();
```

### Testing our first change

Before we move on, let's test our change to verify it's correctly removing the `queryCache` variable. To do this, create a file called `test.js` and add the following content:

```js
import { useQuery, queryCache } from "react-query";
```

Next, transform the file using the following shell command:

```sh
jscodeshift -t transform.js test.js
```

After running that command, we should see that `test.js` has changed to the following:

```js
import { useQuery } from "react-query";
```

Awesome! We've now successfully completed the first step of removing the `queryCache` variable from the `react-query` import. Now, let's move on to step 2.

### Adding the new import

To add the new import, we first need to track if we removed a `queryCache` variable from the `react-query` import. Do this by checking the `length` property of the collection returned from our code in step 1.

```js
const specifiers = source
  .find(j.ImportSpecifier)
  .filter((path) => path.parent.value.source.value === "react-query")
  .filter((path) => path.value.imported.name === "queryCache")
  .remove();

if (specifiers.length) {
  // Add new import
}
```

Next, we need to find the `react-query` import so we can insert our new import after it. This logic will look very similar to our logic in step 1 where we found the import specifier.

```js
// Add new import
source
  .find(j.ImportDeclaration)
  .filter((path) => path.value.source.value === "react-query");
```

Finally, we can call the `insertAfter` function to insert our new import after the existing `react-query` import. The value that we pass to `insertAfter` can be any valid AST node, which we can easily create thanks to the helper functions provided by jscodeshift.

```js
source
  .find(j.ImportDeclaration)
  .filter((path) => path.value.source.value === "react-query")
  .insertAfter(
    j.importDeclaration(
      [
        j.importSpecifier(
          j.identifier("queryClient"),
          j.identifier("queryCache")
        ),
      ],
      j.stringLiteral("common/queryClient")
    )
  );
```

_When creating new AST nodes, using AST Explorer helps to understand the shape of nodes so you can more easily create them using the jscodeshift helper methods._

## Putting it all together

If we put all of this together, our final file should look something like this:

```js
module.exports = function (file, api) {
  const j = api.jscodeshift;
  const source = j(file.source);

  const specifiers = source
    .find(j.ImportSpecifier)
    .filter((path) => path.parent.value.source.value === "react-query")
    .filter((path) => path.value.imported.name === "queryCache")
    .remove();

  if (specifiers.length) {
    source
      .find(j.ImportDeclaration)
      .filter((path) => path.value.source.value === "react-query")
      .insertAfter(
        j.importDeclaration(
          [
            j.importSpecifier(
              j.identifier("queryClient"),
              j.identifier("queryCache")
            ),
          ],
          j.stringLiteral("common/queryClient")
        )
      );
  }

  return source.toSource();
};
```

Congratulations! We've just created our first codemod using jscodeshift!

## Conclusion

In this article, we explored how to use jscodeshift to automate code transformations to simplify refactors. While the example I showed might not be useful to you, it should give you an idea of what you can do with jscodeshift. If you want to see a more complex codemod, check out the [PropTypes to TS](https://github.com/mskelton/prop-types-to-ts) project I created.

Thanks for reading! If you have any questions or comments, feel free to leave them below. Also, if you want the chance to work with some cool people and technology, come [join us](https://www.widen.com/careers)!

## Bonus: Type definitions

Since remembering all the available jscodeshift methods is not easy, jscodeshift provides type definitions out of the box! All you have to do is add a JSDoc comment to specify the types of the parameters and your editor will now autocomplete the available methods!

```js
/**
 * @param {import('jscodeshift').FileInfo} fileInfo
 * @param {import('jscodeshift').API} api
 */
module.exports = function (fileInfo, api) {};
```
