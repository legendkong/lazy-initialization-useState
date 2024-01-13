## Note
This eslint rule was not implemented in the `facebook/react` repository as the [eslint-plugin-react-hooks] is no longer being actively maintained. </br>
I just want to preface that I made the below rule to "solve" the lazy-initialization issues within the React codebase during my tenure as an intern at Zendesk, and received positive feedback both from the public as well as internally in my company, hence I've decided to make this public.
Feel free to copy and paste to use the rule. More information below. </br></br>
[eslint-plugin-react-hooks] See github issue opened on facebook/react repo here: https://github.com/facebook/react/issues/26520

## Context
It is common to have instances where `React.useState()` code is unnecessarily re-creating its initial state:

Example:
```js
const Component = () => {
 const [state, setState] = useState(getInitialHundredItems());
}
```

React's `useState` hooks accepts an `initialState` argument e.g. `React.useState(initialState)`.
The `initialState` argument is the state used during the initial render. In subsequent renders, it is disregarded.
If the initial state is the result of an expensive computation, we may provide a function instead, which will be executed only on the initial render.

In the above example, `getInitialHundredItems` is called on each re-render, but its result is only needed on initial render.

More details `useState` and  `Lazy initial state` can be found here: [link1](https://reactjs.org/docs/hooks-reference.html#lazy-initial-state), [link2](https://beta.reactjs.org/reference/react/useState#avoiding-recreating-the-initial-state).

## Description

To address the problem mentioned above, we can make use of an [initializer function](https://beta.reactjs.org/reference/react/useState#avoiding-recreating-the-initial-state) in `React.useState()`.
This function will only be executed once (on initial render) and not on each re-render like the above code will.

Example of how one would use lazy initialization:
```js
const Component = () => {
 const [state, setState] = useState(() => {
    return getInitialHundredItems(x);
  })
}
```

## Rule
```js
...
  /**
   * This rule is meant to detect the following anti-pattern:
   *
   * ```js
   * const Component = () => {
   *  const [state, setState] = useState(getInitialHundredItems());
   * }
   * ```
   *
   * The initialState argument is the state used during the initial render. In subsequent renders, it is disregarded.
   * This initial value can be the result of calling a function as in the above example.
   * But note that getInitialHundredItems is unconditionally and needlessly called on each render cycle.
   *
   * To avoid the problem mentioned above, instead of just calling a function that returns a value,
   * you can pass a function which returns the initial state.
   * This function will only be executed once (initial render) and not on each render like the above code will.
   * See [Lazy Initial State] for details.
   *
   * [Lazy Initial State]: https://reactjs.org/docs/hooks-reference.html#lazy-initial-state
   *
   * The above code can be changed to the below, with lazy initilization:
   *
   * ```js
   * const Component = () => {
   *  const [state, setState] = useState(() => {
   *    return getInitialHundredItems(x);
   *  })
   * }
   * ```
   */
  'prefer-react-use-state-lazy-initialization': {
    meta: {
      type: 'problem',
      fixable: 'code',
      schema: [], // no options
    },
    create(context) {
      const ALLOW_LIST = ['Boolean', 'String'];

      return {
        'CallExpression[callee.name="useState"]'(node) {
          if (node.arguments.length > 0) {
            const arg = node.arguments[0];
            // If arg is a call expression and is not in `allowList`
            // e.g `Boolean(x)` is allowed, but `expensiveFunc(x)` is not allowed
            if (arg.type == 'CallExpression' && ALLOW_LIST.indexOf(arg.callee.name) === -1) {
              context.report({
                node: arg,
                message:
                  'To prevent expensive re-computation, consider using lazy initial state for React.useState().',
              });
            }
          }
        },
      };
    },
  }
...
```
