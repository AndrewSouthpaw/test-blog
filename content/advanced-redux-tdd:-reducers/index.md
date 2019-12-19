---
title: "# Advanced Redux TDD: Reducers"
description: "We do TDD for most development at Transparent Classroom, including our business logic written in Redux. Doing TDD for Redux revealed…"
date: "2019-12-19T01:43:56.851Z"
categories: []
published: false
---

  

\# Advanced Redux TDD: Reducers

  

We do TDD for most development at Transparent Classroom, including our business logic written in Redux. Doing TDD for Redux revealed certain frustrating aspects of structuring and testing Redux. I set out to make it better.

This post offers a slightly different structure for your reducers to make them more testable and easy to statically type (for Flow or TypeScript). Let’s dive in.

(Note: we also use Flow for type checking, which adds additional motivation for what we did. Where appropriate, I’ll be adding Flow typing for discussion purposes, otherwise I’ll strip it.)

\# The Status Quo

\[In the offical docs\]([https://redux.js.org/recipes/writing-tests#reducers](https://redux.js.org/recipes/writing-tests#reducers)), reducers are structured using a \`switch\` statement, and then the branches of the reducer are tested using actions:

```
it(‘should handle ADD_TODO’, () => {
 expect(reducer(initialState, addTodo(‘Run the tests’))) .toEqual([
 {
 text: ‘Run the tests’,
 completed: false,
 id: 0
 }
 ])
})
```

There were a couple issues that didn’t sit right with me:

1\. \*\*Navigation was a pain.\*\* We couldn’t navigate directly to that part of the reducer by clicking into a function definition. At best, we’d have to look for usages of the constant, then find it that way.  
1\. \*\*Tests didn’t read naturally\*\*, with all the extra rigamarole with the reducer, action object, etc. We wanted to write functions, because functions are easy to test.  
1\. We’re also \*\*testing the action creators, which is a separate concern\*\*; not a big deal, but also not my favorite.  
1\. \*\*Type checking with Flow was a miserable experience\*\* based on the \[official Flow docs\](TODO-link).

\# Finding a Better Way: Handlers

An alternative to using a switch statement is to have a “handlers object,” \[mapping action types to handlers\]([https://redux.js.org/recipes/reducing-boilerplate#generating-reducers](https://redux.js.org/recipes/reducing-boilerplate#generating-reducers)), like so:

\`\`\`language-javascript  
export const todos = createReducer(\[\], {  
 \[ActionTypes.ADD\_TODO\]: (state, action) => {  
 const text = action.text.trim()  
 return \[…state, text\]  
 }  
})  
\`\`\`

(Amusingly, I thought of this idea on my own after ruminating on how to improve on the switch statement, then discovered it was already discussed in the Redux docs. Turns out coming up with original ideas is hard.)

It was a nice start, but not enough. It didn’t allow for easy navigation between implementation and test, and the object syntax felt a bit weird and limiting.

This approach, however, opened the path to making and testing exported functions, rather than the reducer object itself, which I call \*\*handlers\*\*.

\`\`\`language-javascript  
export const addTodo = ({ todo }, state) => /\* … \*/  
\`\`\`

It might not look so bad here, but it quickly became unwieldly and ugly with larger more complex handlers. Also, it was so dissatisfying in tests:

\`\`\`language-javascript  
it(‘should do a thing’, () => {  
 const action = {  
 thing1: ‘foo’,  
 thing2: ‘bar’,  
 thing3: ‘baz’,  
 }  
 const state = myHandler(action, defaultState)  
 /\* … \*/  
})  
\`\`\`

Ugh. It was no improvement. Rewriting the keys for the action object each time was cumbersome. I really wanted these functions to look and act like normal functions, not just be an extraction of the internals of a reducer.

I wanted to extract the arguments from the action object and just give them to the handler. Turns out, you can do just that.

\`\`\`language-javascript  
const reducer = (state = defaultState, { type, …args }) => {  
 const handler = handlers\[type\]  
 if (!handler) return state  
 return handler(…Object.values(args), state)  
}  
\`\`\`

I thought using \`Object.values\` in this way would be dangerous: we always hear about how \[the iteration order of an object’s keys is not guaranteed\]([https://stackoverflow.com/a/5525820/2672869](https://stackoverflow.com/a/5525820/2672869)). I tried a couple alternatives, including having the action creators generate \`Map\`s rather than objects (where the order is guaranteed). Using \`Map\`s were unpleasant to debug and it bothered me to step further away from how Redux is normally structured (i.e. using plain objects).

ES6, however, did \[introduce a spec\]([http://2ality.com/2015/10/property-traversal-order-es6.html](http://2ality.com/2015/10/property-traversal-order-es6.html)) for the iteration order of object keys. For objects with string (non-\`Symbol\`) keys that can’t be parsed as integers, the iteration order is simply the insertion order.

\`\`\`language-javascript  
Object.values({ foo: 1, bar: 2 })  
// \[1, 2\]  
\`\`\`

Adding keys that are symbols (\`{ \[Symbol(‘first’)\]: true }\`) or ones that can be parsed as integers (\`{ ‘10’: true }\`) behave differently and make the picture more complex, but that doesn’t matter for our purposes, because we want the string keys on the object to reflect parameter names, which cannot be integers or Symbols.

It turns out that assuming consistent iteration order for object keys is totally safe given the above restrictions. We’ve been doing it in our production code for over six months without issue. (Granted, we don’t have to support super old browsers, so YMMV.)

\`\`\`language-javascript  
// actions.js  
export const addTodo = (title, description) => ({ type: ADD\_TODO, title, description })

// reducer.js  
export const addTodo = (title, description, state) => { /\* … \*/ }

// reducer\_spec.js  
it(‘should do a thing’, () => {  
 const state = addTodo(‘My First Todo’, ‘I feel so proud!’, defaultState)  
 /\* … \*/  
})  
\`\`\`

And now you can navigation quickly between test and implementation. Ah, the sweet smell of victory.

\# Wiring the Handlers to the Reducer

The tricky part then became how to tell the reducer to actually use these handlers. After running through a few different approaches (I’ll omit them here, because, ugh they bad), I settled on the concept of “registering” a handler on the handlers object, then giving the object to the reducer.

\`\`\`language-javascript  
const handlers = {}  
const registerHandler = (actionType, handler) => handlers\[actionType\] = handler

export const addTodo = (text, state) => state.concat(text)  
registerHandler(ADD\_TODO, addTodo)

const reducer = (state = defaultState, { type, …args }) => {  
 const handler = handlers\[type\]  
 if (!handler) return state  
 return handler(…Object.values(args), state)  
}  
\`\`\`

This approach made it clear which handlers were actually employed by the reducer (rather than helpers) and colocated the action type with the handler.

It’s worth noting that I structured the handlers to take the state object \*at the end\* rather than at the beginning (like in the actual reducer). I moved it to the end to make piping state from one handler to the next a breeze, and if you curry your handlers it becomes wonderfully simple and expressive. For example, if I wanted to add a couple todos to the state to set up a test:

\`\`\`language-javascript  
const state = pipe(  
 addTodo(‘First Todo’, ‘This was my first todo!’),  
 addTodo(‘Make TDD better’, ‘Focus first on reducers’),  
)(defaultState)  
\`\`\`

Compare this with the alternative, where state is the first parameter:

\`\`\`language-javascript  
const state = pipe(  
 x => addTodo(x, ‘First Todo’, ‘This was my first todo!’),  
 x => addTodo(x, ‘Make TDD better’, ‘Focus first on reducers’),  
)(defaultState)  
\`\`\`

It’s not terrible, but I vastly prefer the flexibility of the former.

With all this setup, you can at last achieve TDD nirvana and test the various parts of your reducer functionality like they were functions. Functions that are composable and modular. (Yes, you functional programmers out there can be all 🙌 here.)

\`\`\`language-javascript  
it(‘#completeTodo should mark a todo at an index as completed’, () => {  
 const state = pipe(  
 addTodo(‘A’),  
 addTodo(‘B’),  
 addTodo(‘C’),  
 completeTodo(1),  
 )(defaultState)

expect(state.map(x => x.completed)).toEqual(\[false, true, false\])  
})  
\`\`\`

As a side note, we generally don’t curry our handlers, for two reasons:

1\. IDEs generally don’t handle curried versions well, and do not correctly infer the parameters  
1\. Flow typing of curried functions can still be fiddly

Hopefully that will change some day…

\# Static Typing for Reducers

Using this approach of exporting handlers and registering them on the reducer provides the added benefit of dramatically simplifying the type checking for reducers. I’ll discuss it using Flow, but I’m sure TypeScript would be similar.

Setting up type checking based off the Flow docs is a mouthful, requiring a lot of extra typing:

\`\`\`language-javascript  
// types.js  
type FooAction = { type: “FOO”, foo: number }  
type BarAction = { type: “BAR”, bar: boolean }

type Action =  
 | FooAction  
 | BarAction

// actions.js  
const fooAction: FooAction = (foo: number) => ({ type: ‘FOO’, foo })  
const barAction: BarAction = (bar: boolean) => ({ type: ‘BAR’, bar})

// reducer.js  
function reducer(state: State, action: Action): State {  
 switch (action.type) { /\* … \*/ }  
}  
\`\`\`

It works, but not perfectly. Flow sometimes behaves strangely with disjoint unions because it’s not a perfect system. It’s also hard to understand, requiring a deeper knowledge of Flow and typing. Finally, inside the reducer, you don’t actually know your types at the various branches of your switch statement; you have to refer to a different file entirely to figure out the types of the action object for that branch.

Fortunately, using handlers circumvents all these issues, because the handlers are simply functions whose parameters can be easily typed. The inputs to the function are explicitly typed right with the definition, making it a breeze to read.

We don’t bother typing our actions because it hasn’t served any utility. There’s a temptation to “Type All The Things” with Flow, but we’ve decided to be more strategic around what we type to maximize benefit and not waste time.

\# TL;DR

To recap, and structure the code into some helper functions, here’s the code to set it all up:

\*\*reducer\_helpers.js\*\*

\`\`\`language-javascript  
// this mutates the handler object! That’s okay though.  
export const setupRegistration = (handlers) => (actionType, handler) => {  
 handlers\[actionType\] = curry(handler)  
}

export const createReducerWithHandlers = (handlers, defaultState, state, { type, …args }) => {  
 const handler = handlers\[type\]

if (!handler) return state || defaultState

// This is safe for empty args arrays, the spread will still come out to be \[state\], not \[null, state\]  
 return handler(…values(args), state)  
}

\`\`\`

\*\*reducer.js\*\*

\`\`\`  
const handlers = {}  
const registerHandler = setupRegistration(handlers)

export const addTodo = (title, description, state) => { /\* … \*/ }  
registerHandler(ADD\_TODO, addTodo)

export default createReducerWithHandlers(handlers, initialState)  
\`\`\`

  

Writing our reducers in this way has vastly improved the experience of TDD for Redux, simplified our static typing of reducers using Flow, and made our reducer files easier to read and understand. I hope you will give it a try and see if you get similar benefits. I’d love to hear your thoughts and how it works out in your projects.
