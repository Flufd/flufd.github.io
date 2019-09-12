---
title: Avoiding useEffect race conditions with a custom hook
date: "2019-10-09T22:40:32.169Z"
description: A description
---

If you have been using React hooks, specifically `useEffect` then you may have come across the same problem that I did when making asynchronous calls such as data fetching. I first encountered this when finding a warning in my console:

> Warning: Can't perform a React state update on an unmounted component.
> <cite>React</cite>

... when doing something like this:

```jsx
const [article, setArticle] = useState(null);
useEffect(() => {
  async function fetchData() {
    const article = await API.fetchArticle(id);
    setArticle(article);
  }
  fetchData();
}, [id]);
```

This will happen when the component is unmounted before the data is returned. But there is also a separate issue here too; if our `id` dependency changes while the first fetch is still happening, the effect will run again with the new `id` and there's no guarantee that the data will come back in the correct order. Additionally, if the `id` changed then are we really interested in the value returned from the server any more? The state will still be updated with the old data once the request has returned.

To fix this you may have seen a pattern where the lifecycle of the effect is tracked with a `didCancel` flag. The call to the state setter is then guarded with this variable.

```jsx
function Article({ id }) {
  const [article, setArticle] = useState(null);

  useEffect(() => {
    let didCancel = false;

    async function fetchData() {
      const article = await API.fetchArticle(id);
      if (!didCancel) {
        setArticle(article);
      }
    }

    fetchData();

    return () => {
      didCancel = true;
    };
  }, [id]);

  // ...
}
```

I first found that pattern on [Dan Abramov's blog post](https://overreacted.io/a-complete-guide-to-useeffect/#speaking-of-race-conditions) and it looks like that pattern came from [Robin Wieruch's blog post](https://www.robinwieruch.de/react-hooks-fetch-data). While the examples there call the variable `didCancel`, really that flag tells us one of a few things:

- One of the dependencies of the hook has changed
- The component was unmounted
- The effect wasn't passed any dependencies array at all, but you probably aren't fetching data on each render.

I like this method of tracking the "lifecycle" of the `useEffect` hook, but feel like that variable could have been included by the React team as part of the interface of `useEffect`, there's an empty parameter list passed to the effect right there!

```jsx
useEffect((/* effect lifecycle state goes here */) => {
  // Do my effect and be sure that the dependencies haven't changed
}, []);
```

I decided to create a custom hook that had a similar interface to the regular `useEffect` hook, but had this functionality built in.

As I mentioned before, the `useEffect` hook takes a single parameterless function as the first argument. We can use this fact to our advantage, as any current use of `useEffect` would not be using the arguments list, so it should be easy to slot this new hook in. We can pass in a parameter that does the same job as the `didCancel` variable.

This is how I'd like it to look when used:

```jsx
function Article({ id }) {
  const [article, setArticle] = useState(null);

  useCancelledEffect(
    didCancel => {
      async function fetchData() {
        const article = await API.fetchArticle(id);
        if (!didCancel) {
          setArticle(article);
        }
      }

      fetchData();
    },
    [id]
  );

  // ...
}
```

And here is a quick naïve first implementation:

```jsx
function useCancelledEffect(effect, deps) {
  useEffect(() => {
    let didCancel = false;
    // Call the effect
    // we pass the didCancel bool to be used by the effect
    effect(didCancel);
    return () => {
      didCancel = true;
    };
  }, deps);
}
```

You may notice an issue here. As a javascript boolean is a primitive type, the `didCancel` variable will be passed by value to the effect function, so when we set the variable to true in the clean up function, its value will not be reflected inside the effect when the asynchronous effect continues.

We can fix this by passing a function that returns the `didCancel` instead of the variable itself. The definition of the function will capture the `isCurrent` variable, so when it's changed, the return value of the function will also be updated. I'm also going to flip the boolean so that it's true by default (because I think it looks better when used) and change the name to "current".

```jsx
function useCurrentEffect(effect, deps) {
  useEffect(() => {
    let isCurrent = true;
    const checkCurrent = () => isCurrent;
    effect(checkCurrent);
    return () => {
      isCurrent = false;
    };
  }, deps);
}
```

And now we use it like this:

```jsx
useCurrentEffect(
  isCurrent => {
    async function fetchData() {
      const article = await API.fetchArticle(id);
      // We can check if the dependencies have changed
      if (isCurrent()) {
        setArticle(article);
      }
    }

    fetchData();
  },
  [id]
);
```

## What about clean up?

So far our custom hook ignores the return value of the effect, so a returned clean up function won't be called:

```jsx
useCurrentEffect(
  isCurrent => {
    async function fetchData() {
      const article = await API.fetchArticle(id);
      if (isCurrent()) {
        setArticle(article);
      }
    }

    fetchData();

    return () => {
      // This won't be called :'(
      API.doSomeCleanup();
    };
  },
  [id]
);
```

Let's fix that by taking any returned clean up function and calling it in the outer clean up function:

```jsx
function useCurrentEffect(effect, deps) {
  useEffect(() => {
    let isCurrent = true;
    const checkCurrent = () => isCurrent;

    // Get the clean up function if the effect uses one
    const cleanup = effect(checkCurrent);
    return () => {
      isCurrent = false;
      // Call the clean up function
      cleanup && cleanup();
    };
  }, deps);
}
```

And that's it. This custom hook has a very similar interface to the original `useEffect`, but now we can track if the dependencies have changed in any asynchronous callbacks.

## Lint rules for our custom hook

After using this hook for a while, I realised that I missed the `exhaustive-deps` warning I got from eslint when I missed a dependency. Searching around the documentation, I couldn't find any nice way to get eslint to give me the same warnings I'd get with `useEffect`. I did find that I could do something like this:

```jsx
import { useCurrentEffect as useEffect } from "./hooks/useCurrentEffect";
```

... this kind of works, but it conflicted with when I wanted to use the regular useEffect.

Eventually I found within the react-hooks/exhaustive-deps source, that you could configure custom hooks in the .eslint config file, with the `additionalHooks` option.
`"react-hooks/exhaustive-deps": ["warn", { "additionalHooks": "useCurrentEffect" }],`
*Note: This option takes a regular expression.*

### Obligatory npm package

I've published this hook, written in TypeScript, along with a similar hook, `useCurrentCallback` on [GitHub](https://github.com/Flufd/use-current-effect) as well as published an npm package `use-current-effect`.