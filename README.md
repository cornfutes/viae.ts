# Background

`Viae.ts` was conceived during my time at [Waze](https://www.waze.com) to tame integration tests consisting of a complex set of overlapping graphs of async function calls.
A naive attempt of applying [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) and manually composing utility functions did not work. This was cargo cult.
The [system-under-test](https://developers.google.com/waze/wam-api) was prone to many failure modes but the haphazard addition of error handling, caching, 
sychronization and other pseudo-robust control plane logic in callback hell only exacerbated the complexity. The utility functions were indirections, not
abstractions, and shuffled complexity around. The test suite became intractable to onboard, extend, refactor, inspect, and code review. `Viae.ts` reduced the time for these processes from hours to seconds.

The library is generic. It is meant for any situation
where you have async functions that are composed of other async functions, and this composition can be modeled as a directed-acylic-graph.
`Viae.ts` focuses clients on the problem domain rather than imperatively constructing/orchestrating the call graph.
`Viae.ts` is exponentially useful if you have one complex graph and care about its many subgraphs. In the case of Waze, we had one big 
"knowledge graph" from which tests drew from and we were able to reduce virtually all boilerplate code.

## Usage

The first aspect of `Viae.ts` is a declarative API that forces clients to be explicit about their problem domain.
The second aspect is being an inversion-of-control container that traverses the graph.

```javascript
const graph = new Viae();
graph.value('age', 1);  // Terminal value which has no dependency.
graph.async('birthday', age => Promise.resolve(age+1)); // The resolved value of 'birthday' node is resolved for downstream nodes.
graph.entryPoint(birthday => {
  console.log(birthday);
});
```

Note that Viae's semantics of the actual parameter `birthday` in the `#entryPoint` function is the return value of the function `birthday` and not the function itself.
The magic is that `Viae.ts`  waits for async functions then unboxes the returned Promise before dependency-injecting it into downstream nodes. 

Hopefully, the example should be intuitive and self-explanatory. The test suite encapsulates and concisely address behavior beyond this contrived scenario.
That's it! That's the entire API interface. `Viae.ts` has proven to be a sufficiently powerful abstraction with this small footprint. 

Observe that the client never had to declare the keyword `async` or `await`. `Viae.ts` parallellizes calls where possible but without an oracle 
it's impossible to know ahead-of-time the optimal order to fan-out. `Viae.ts` also memoizes resolved function values in cases where a node is visited more than once
(consider the inefficiency of un-memoized recursive Fibonacci).

## Formalization 

More academic food-for-thought...

###  Declaration of a directed graph

Clients register with `Viae.ts` a set of named nodes that are either a value (data) or a function (computation).
The formal parameters of the function should correspond to named nodes.

```javascript
const v = new Viae();
v.value('age', 1);
v.async('birthday', age => Promise.resolve(age+1)); 
```

Each formal parameter is a dependency relationship. In other words, this is a graph edge
starting from the referenced node and ending in the node for the function under consideration. 
In combination, these nodes and edges form directed graph(s). Cycle detection is not enforced at registration but 
at execution time. 

`Viae.ts` formalizes the declaration of a graph structure by forcing clients to make explicit the relationships 
between data and computation.

### Inversion-of-control Container

The graph declaration by itself is static and unproductive. The action is in
traversing the graph. Clients declare a special ephemeral function that references named nodes. This is the exact 
same pattern as declaring a named function. This function can return a value, but is inconsequential as no other
function node can depend on this function and make use of said value.


```javascript
v.entryPoint((birthday) => {
  console.log(birthday);
});
```

`Viae.ts` acts as an Inversion-of-control container performing dependency-injection. 
`Viae.ts` looks at the formal parameters, resolve its value, pass it as the formal argument and evaluate the function. 

1. If the node referenced in the formal parameter is a value node, `Viae.ts` passes the value as the actual parameter.
2. If the node referenced in the formal parameter is a function node, `Viae.ts` passes the return value of the function.

(1) is the base case and (2) is the recursive case: function nodes depend on other nodes, recursively apply the algorithm.

### Interpretted as Function Composition

The embedded graph structure is not an arbitrary one but the call graph of function composition. 

```javascript
v.value('x', 1); 
v.async('f', x => x + 1);
v.async('g', f => 2 * f);
```

Resolving the value of `g` is equivalent to `g(f(1))` or rather `g(await f(1))`.

The API surface could have been made even more minimal by omitting the `#value` method altogether.
It's really just a special case function, one that takes no argument. 

```javascript
v.async('x', () => x);
```

In the functional programming sense, data *is* computation. This would have been a more minimal
design, but it was clever and arguably less simple.

### Context-aware Auto-unboxing

If a function returns `Promise<T>`, `Viae.ts` passes `T` as the actual parameter. 
If a function returns `T` (and `T` is not a Promise), `Viae.ts` passes `T`.
As corollary, if a function returns `Promise<Promise<T>>`, `Viae.ts` returns `Promise<T>`.
`Viae.ts` acts as a one-level deep `#flatMap` or a Monadic Bind. 

An alternative API design would be to make a distinction between async and synchronous functions.
As such, functions registered with `#async` must always return a Promise. This could easily
be enforced at the TypeScript compiler level. Evaluating synchronous function would be a straightforward
function call where we do not need to block nor be context-aware about the returned value.

```javascript
v.function('a', x => x + 1);
v.function('a', x => Promise(x+ 1)); // Will not attempt to resolve or 
v.async('a', x => Promise(x+1));
v.async('b', x => x + 1); // Will have compiler error
```

This was actually in the original API design. However, the nature of async is that if you have one async function then
you are in the async world and everything is async. The current API confronts clients with this fact.
If we are only dealing with synchronous function calls, the value-add of `Viae.ts` is drastically reduced and you are 
probably better off with functional programming. 

Underneath the hood, synchronous functions registered with `#async` are promoted to asynchronous functions.
This adds a bit of compute time overhead, but its offset by not having to make a decision at every junction.

## Development

```bash
$ npm i # install
$ npm t # test
$ npm run fmt # format
```

## About the name

`Viae.ts` is derived from the latin word for road. Also, a complex network of roads in the Roman empire.

`Viae.ts` also means "argument". The philosopher Thomas Quinas provided five arguments dubbed Quinque `Viae.ts` or "Five Ways". One of the formulations is that things which exist in the universe are either caused by some other prior process, or uncaused. `value` nodes in `Viae.ts` simply exist and `async` nodes must be invoked to produce a value.

Plus, this project was conceived at Waze which is also a play on the word "ways".
