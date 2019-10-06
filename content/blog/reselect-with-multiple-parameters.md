---
title: Creating reusable and composable paramaterized Reselect selectors
date: "2019-10-05T23:00:00.000Z"
description: Using parameter selectors to create reusable parameterized selectors in Redux
---

Often when using Redux we wish to select some state in our component based on a dynamic variable such as an ID, to do this using React Redux we can use the `connect` function, passing the `ownProps` parameter to `mapStateToProps`.

```jsx
// Given that our state looks like this
const initialState = {
  pokemon: {
    1: { name: "Bulbasaur", type: "Grass" },
    4: { name: "Charmander", type: "Fire" },
    7: { name: "Squirtle", type: "Water" }
  }
};

// We can select the correct item from the store by ID
const mapStateToProps = (state, ownProps) => {
  return {
    pokemon: state.pokemon[ownProps.id]
  };
};
```

These props could be passed in by a parent component, or maybe by a higher order component such as `withRouter` from React Router.

With version 7.1.0 of React Redux we have the `useSelector` hook instead, this simplifies a lot of the code from wiring up `connect`. We have access to the local variables and props of the functional component directly from the selector:

```jsx
function PokemonCard({ id }) {
  const pokemon = useSelector(state => {
    return state.pokemon[id];
  });

  return <div>{pokemon.name}</div>;
}
```

One of my favourite Redux libraries is Reselect, which lets us extract this logic into reusable and composable memoized selectors. With Reselect, your selectors can access those variables too:

```jsx
// Get the slice of state
const getPokemon = state => state.pokemon;

// The extra variables are accessible like so..
// We create a selector that ignores the state variable
// Returning just the passed id
const getId = (_, id) => id;

// ... and combine in createSelector
const getPokemonById = createSelector(
  getPokemon,
  getId,
  (pokemon, id) => {
    return pokemon[id];
  }
);
```

Now we can use the memoized selector in our component:

```jsx
function PokemonCard({ id }) {
  const pokemon = useSelector(state => getPokemonById(state, id));
  return <div>{pokemon.name}</div>;
}
```

Note that we still need to wrap the selector in a lambda and pass the state and parameter manually to the selector. This is only needed because we need to pass the ID to the selector, for selectors that don't need the extra variables, we can just write:

```jsx
const allPokemon = useSelector(getPokemon);
```

We can make our paramaterized selector call a little cleaner with a simple custom hook:

```jsx
function useParamSelector(selector, ...params) {
  return useSelector(state => selector(state, ...params));
}
```

```jsx
function PokemonCard({ id }) {
  const pokemon = useParamSelector(getPokemonById, id);
  return <div>{pokemon.name}</div>;
}
```

## Multiple parameters

This works well for a single variable, but what if we need to pass multiple parameters to our selector? Maybe we want to perform a search with several criteria.

Reselect is not limited to a single parameter:

```jsx
const getPokemon = state => state.pokemon;

// Get the first parameter
const getNameParam = (_, name) => name;
// Get the second parameter
const getTypeParam = (_, _, type) => type;

const searchPokemon = createSelector(
  getPokemon,
  getTypeParam,
  getNameParam,
  (pokemon, type, name) => {
    // Filter our Pokemon
    return Object.values(pokemon)
      .filter(p => p.type === type)
      .filter(p => p.name.includes(name));
  }
);
```

This can obviously become qute messy if we use many parameters. Also, as one of Reselect's benefits is the composability of the selectors, this presents an issue because each selector in the tree must rely on the parameters being in the correct position.

Here's an example:

```jsx
// Let's make our state a little more complex
const initialState = {
  moves: {
    solarbeam: { name: "Solar Beam", power: 120 },
    tackle: { name: "Tackle", power: 35 }
    // ...
  },
  pokemon: {
    // Pokemon now have links to moves
    1: { name: "Bulbasaur", type: "Grass", moves: ["tackle"] }
    // ...
  }
};
```

Let's say we want to find all moves where the power is greater than 100, for all grass type Pokemon.

We might have a selector like this to fetch all of the Pokemon of a type.

```jsx
const getTypeParam = (_, type) => type;

const getPokemonOfType = createSelector(
  getPokemon,
  getTypeParam,
  (pokemon, type) => {
    return Object.values(pokemon).filter(p => p.type === type);
  }
);
```

And a selector for selecting high-power moves

```jsx
const getMoves = state => state.moves;
const getPowerParam = (_, power) => power;
const getMovesWithPowerOver = createSelector(
  getMoves,
  getPowerParam,
  (moves, power) => {
    return Object.values(moves).filter(m => m.power > power);
  }
);
```

But now we cannot compose these existing selectors to get our final result, because they both expect their filtering parameter to be passed in the same position.

## Combine the parameters?

What if we pass all of the parameters to the selector as a javascript object, like so:

```jsx
const getParams = (_, params) => params;

const searchPokemon = createSelector(
  getPokemon,
  getParams,
  (pokemon, params) => {
    // Filter our Pokemon using the params
    return Object.values(pokemon)
      .filter(p => p.type === params.type)
      .filter(p => p.name.includes(params.name));
  }
);

// Used like this
const pokemon = useParamSelector(searchPokemon, { name, type });
```

The issue with this is that we are passing a new object to Reselect each render, which means that we will break the memoization as Reselect makes a shallow comparison of each of the arguments. We can kind of get around this by memoizing the parameter object using `useMemo` before passing it to the hook. However, each selector will still be re-ran if we change just a single one of the parameters, even if it did not use the changed parameter, as the selector is passed the whole (new) parameter object.

## A solution

What I found works well is to create "parameter selectors" for each parameter that a selector might use, and have the main selectors define which parameters they need. We still pass all of the parameters in as a javascript object, but extract just the ones we want.

```jsx
// A helper function to create the parameter selectors
function createParameterSelector(selector) {
  return (_, params) => selector(params);
}

const getNameParam = createParameterSelector(params => params.name);
const getTypeParam = createParameterSelector(params => params.type);

const searchPokemon = createSelector(
  getPokemon,
  getTypeParam,
  getNameParam,
  (pokemon, type, name) => {
    // Filter our Pokemon
    return Object.values(pokemon)
      .filter(p => p.type === type)
      .filter(p => p.name.includes(name));
  }
);
```

The memoization works here because although the result of `getParams` changes each render, the parameter selectors return primitive values, which are passed to `createSelector`, and only the parameters that are required are taken into account when deciding to recalculate the resulting value.

This also strikes me as quite a composable pattern, each selector defines the data it needs, by combining selectors and parameters. These parameter selectors are then able to be re-used in other selectors elsewhere in the application.

In many applications your components are within the scope of a single entity and it's identifier, whether it's an order number, user ID etc. and these IDs can be used in many selectors, so being able to just select the ID without worrying about the the position of the parameters is really nice.