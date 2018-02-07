# Raj by Example

> [Raj](https://github.com/andrejewski/raj) is the best JavaScript framework.

This is the complement to [Why Raj](https://jew.ski/why-raj/), which offers a much more high level explanation for why you should use Raj. Here, we show code samples written with Raj.

Written for reading in order, we gradually build up Raj programs with libraries from the [Raj ecosystem](https://github.com/andrejewski/raj#ecosystem).

## Contents

1. [The Atom](#1-the-atom): Basic program structure
1. [Mixed Signals](#2-mixed-signals): Explaining messages
1. [Real Work](#3-real-work): Performing side-effects
1. [Big Picture](#4-big-picture): Program composition
1. [Prove It](#5-prove-it): Testing programs
1. [Elm Street](#6-elm-street): Comparing with Elm

## 1. The Atom
In Raj, the atomic unit is a program. Every Raj application is a program no matter the complexity. The program consists of three parts: `init`, `update`, and `view`. The smallest valid program is:

```js
export default {
  init: [],
  update: () => [],
  view () {}
}
```

This program does nothing but highlight the required parts. Note:

- `init` must be an array
- `update` must be a function and must return an array
- `view` must be a function

Now we will make the smallest "useful" program, a counter.

```js
export default {
  init: [0], // State is an integer to count
  update (message, state) {
    return [state + 1] // Increment the state
  },
  view (state, dispatch) {
    const keepCounting = window.confirm(`Count is ${state}. Increment?`)
    if (keepCounting) {
      dispatch()
    }
  }
}
```

This program is simple but highlights more than the previous program. Note:

- `init` must be an array whose first index `init[0]` is the initial state of the program. In the previous example, the `init` of `[]` meant the initial state equals undefined. This is valid because there are no restrictions on what state can be. In this new example, `0` is the initial state.

- `update` receives two arguments `message` and `state`. We do not know the purpose of `message` yet, but `state` must be the current state. We also see we return an array with the first index being the new state.

- `view` receives two arguments `state`, which again is the current state, and `dispatch` which is some function we can call. We can have a useful program without have view return anything.

At this point, we have not introduced Raj. These programs defined above are plain objects and certainly cannot call themselves. To get these programs to *run* we need some sort of *run*-time. Raj is a runtime.

```js
import {program} from 'raj/runtime'

program({
  init: [0], // State is an integer to count
  update (message, state) {
    return [state + 1] // Increment the state
  },
  view (state, dispatch) {
    const keepCounting = window.confirm(`Count is ${state}. Increment?`)
    if (keepCounting) {
      dispatch()
    }
  }
})
```

Now we have a working program. For now we can think about the runtime as having one job: store the current program state. The Raj runtime needs to work like this:

```js
export function program ({init, update, view}) {
  let state = init[0]
  const render = () => {
    const message = undefined // still don't know what this is
    state = update(message, state)[0]
    view(state, render)
  }
  view(state, render)
}
```

This is missing some features but the Raj runtime creates this run loop under the hood.

## 2. Mixed Signals

At this point, we have at least two questions:

- What is `message`!?
- When does this start being useful?

To answer these questions, we need more realistic examples. Raj is view layer agnostic meaning (as we have seen) `view` receives the newest state and a `dispatch` function and then can do anything. View layer integrations take advantage of this open-ended contract.

One such integration from the Raj ecosystem [`raj-react`](https://github.com/andrejewski/raj-react) allows us to write Raj programs which become React components. We will use React in the following examples but keep in mind we could be using a different view library. It makes no difference to Raj.

Let's update the counter example to use `raj-react`.

```js
import React from 'react'
import ReactDOM from 'react-dom'
import {program} from 'raj-react'

const Program = program(React.Component, () => ({
  init: [0], // State is an integer to count
  update (message, state) {
    return [state + 1] // Increment the state
  },
  view (state, dispatch) {
    return <div>
      <p>Count is {state}.</p>
      <button onClick={() => dispatch()}>Increment</button>
    </div>
  }
}))

ReactDOM.render(<Program />, document.getElementById('app'))
```

Note the `init` and `update` remain the same. Since we are separating concerns well, it makes sense we change the `view` alone. The `raj-react` `program()` returns a React component we mount inside of our webpage.

Now we can start building more complex programs. We have been able to increment, now we'll decrement too.

```js
import React from 'react'
import ReactDOM from 'react-dom'
import {program} from 'raj-react'

const Program = program(React.Component, () => ({
  init: [0], // State is an integer to count
  update (message, state) {
    return [state + message] // Add to the state
  },
  view (state, dispatch) {
    return <div>
      <p>Count is {state}.</p>
      <button onClick={() => dispatch(1)}>Increment</button>
      <button onClick={() => dispatch(-1)}>Decrement</button>
    </div>
  }
}))

ReactDOM.render(<Program />, document.getElementById('app'))
```

We see that `message` is the first argument passed to `dispatch`. In this case the message is `+1` or `-1` which gets added to the state every time we click a button.

Now let's go crazy and add a reset button that will take the state back to zero when we click it. We *could* do `dispatch(-state)` to do this, but I'd rather keep the view stupid and behavior driven. Let's define the behaviors `increment`, `decrement`, and `reset` and dispatch those.

```js
export default {
  init: [0], // State is an integer to count
  update (message, state) {
    switch (message) {
      case 'increment': return [state + 1]
      case 'decrement': return [state - 1]
      case 'reset': return [0]
    }
  },
  view (state, dispatch) {
    return <div>
      <p>Count is {state}.</p>
      <button onClick={() => dispatch('increment')}>Increment</button>
      <button onClick={() => dispatch('decrement')}>Decrement</button>
      <button onClick={() => dispatch('reset')}>Reset</button>
    </div>
  }
}
```

This is a better contract because now we can change the business logic in `update` without needing to change the `view`. For example, if we decided as requirements change to increment and decrement by 2 the place to make this change is clear.

We can get even crazier, but the key take ways are:

- The `message` is the first argument to `dispatch`.
- Messages can be anything, but best practice is to write them as behaviors and leave the business logic to the `update` function.

This is how programs build up in Raj. Complex programs use libraries to help build their messages. The recommended library for this is [`tagmeme`](https://github.com/andrejewski/tagmeme).

## 3. Real Work
We are continuing to unravel the program pattern, but there is still a lurking question: why do we have to wrap `init` and new states from `update` in an array? The answer is: side effects.

Until now all we have been doing is updating our own state. Side effects let us start interacting with the outside world. While our `init`, `update`, and `view` functions are synchronous and deterministic, effects will allow us to incorporate asynchronous and non-deterministic behavior into our programs, sanely.

The smallest effect you can have is `() => {}`, a no-op. An effect does not have to do anything. Let's make a somewhat useful one:

```js
export default function effect (dispatch) {
  setTimeout(() => dispatch('beep'), 1000)
}
```

Raj calls all effects with the `dispatch` function. This effect waits one second and then dispatches a "beep" message. The provided `dispatch` function is the same one passed to `view`. The dispatched beep message goes into the runtime in the same way, calling the `update` and creating a new state.

The reason we need to wrap every state in an array is because the second index of that array is for an optional effect. Let's put this effect to use in our counter.

```js
export function effect (dispatch) {
  setTimeout(() => dispatch('beep'), 1000)
}

export default {
  init: [0, effect],
  update (message, state) {
    switch (message) {
      case 'increment': return [state + 1]
      case 'decrement': return [state - 1]
      case 'reset': return [0]
      case 'beep': return [-state, effect]
    }
  },
  view (state, dispatch) {
    return <div>
      <p>Count is {state}.</p>
      <button onClick={() => dispatch('increment')}>Increment</button>
      <button onClick={() => dispatch('decrement')}>Decrement</button>
      <button onClick={() => dispatch('reset')}>Reset</button>
    </div>
  }
}
```

Now our counter is idiotic. Every second the counter will switch signs. We can still click the buttons like normal. Note:

- The `init` has an optional effect. When the runtime starts, Raj calls that function with `dispatch`.
- The `update` can return an optional effect. Raj calls it the same way as the `init` effect.
- Since `init` will trigger a "beep" and the "beep" case will trigger a "beep" we have built for ourselves an effect cycle. Do not do this in applications, this is to practice effects.

Instead of an effect loop like above we can write effects which dispatch messages. These require no new syntax because any effect can call `dispatch` any number of times. An effect that dispatches beep every second looks like this:

```js
export default function beepEverySecond (dispatch) {
  setInterval(() => dispatch('beep'), 1000)
}
```

We can use AJAX/`fetch` to make network requests in effects too.
Raj does not care.

### 3.1. Real Life

With effects as open-ended as they are we do need to be aware of the pitfall: death. The above `beepEverySecond` effect will run forever. This may be what you want in a simple program but probably not in a large application. For these effects that we want to stop sometime later, we need to think in *cancellable* subscriptions.

A subscription has no required syntax, but the recommended approach to building subscriptions is like:

```js
export default function subscription () {
  // internal state for the subscription
  return {
    effect (dispatch) {
      // this effect starts the subscription
      // setup a recurring dispatch
    },
    cancel () {
      // this effect ends the subscription
      // teardown the recurring dispatch (if it exists)
      // NOTE: we don't use dispatch here
    }
  }
}
```

Let's rewrite our `beepEverySecond` as a subscription to see something more than a scaffold.

```js
export default function beepEverySecond () {
  let intervalId
  return {
    effect (dispatch) {
      intervalId = setInterval(() => {
        dispatch('beep')
      }, 1000)
    },
    cancel () {
      if (intervalId) {
        clearInterval(intervalId)
      }
    }
  }
}
```

Now we can cancel the beeping sometime later in the program. Refactoring the counter example, we now have:

```js
export function beepEverySecond () {
  let intervalId
  return {
    effect (dispatch) {
      intervalId = setInterval(() => {
        dispatch('beep')
      }, 1000)
    },
    cancel () {
      if (intervalId) {
        clearInterval(intervalId)
      }
    }
  }
}

const {effect, cancel} = beepEverySecond()
export default {
  init: [0, effect], // start beeping
  update (message, state) {
    switch (message) {
      case 'increment': return [state + 1]
      case 'decrement': return [state - 1]
      case 'reset': return [0]
      case 'beep': return [-state, cancel] // end beeping
    }
  },
  view (state, dispatch) {
    return <div>
      <p>Count is {state}.</p>
      <button onClick={() => dispatch('increment')}>Increment</button>
      <button onClick={() => dispatch('decrement')}>Decrement</button>
      <button onClick={() => dispatch('reset')}>Reset</button>
    </div>
  }
}
```

When the program runs, the subscription `effect` gets called, the "beep" message hits `update` and the `cancel` gets called. The result is that the "beep" message happens once.

### 3.2. Die Hard
Since we are on the topic of death, let us talk about the death of runtimes. Like effects that we want to stop, we also want to stop a Raj runtime in some cases. For example, in `raj-react` when our `<Program />` leaves the page we should stop the runtime. Doing so makes us memory safe and garbage collectable.

The `raj-react` component kills the runtime itself, but if you were looking to kill a normal Raj runtime, you would do:

```js
import {program} from 'raj/runtime'

const killProgram = program({
  init: [],
  update: () => [],
  view () {}
})

killProgram() // the runtime has stopped
```

We have a problem here. What if the program had an active subscription and the runtime died? The subscription would run forever and never cancel. To handle this, we have to introduce the purposefully neglected up to this point optional program method `done`.

```js
export default {
  init: [],
  update: () => [],
  view () {},
  done () {} // optional
}
```

The `done` method receives the final state of program when the runtime dies, giving us our last chance to stop all active subscriptions.

We can make the counter example fully safe by leveraging this new `done` method.

```js
export function beepEverySecond () {
  let intervalId
  return {
    effect (dispatch) {
      intervalId = setInterval(() => {
        dispatch('beep')
      }, 1000)
    },
    cancel () {
      if (intervalId) {
        clearInterval(intervalId)
      }
    }
  }
}

const {effect, cancel} = beepEverySecond()
export default {
  init: [0, effect], // start beeping
  update (message, state) {
    switch (message) {
      case 'increment': return [state + 1]
      case 'decrement': return [state - 1]
      case 'reset': return [0]
      case 'beep': return [-state, cancel] // end beeping
    }
  },
  view (state, dispatch) {
    return <div>
      <p>Count is {state}.</p>
      <button onClick={() => dispatch('increment')}>Increment</button>
      <button onClick={() => dispatch('decrement')}>Decrement</button>
      <button onClick={() => dispatch('reset')}>Reset</button>
    </div>
  },
  done (state) {
    // we don't need state in this example
    // but we often store cancel effects in the state
    cancel()
  }
}
```

Now we know everything there is to know about the Raj runtime. Having gone through these examples, we can now comprehend the [source code of the runtime](https://github.com/andrejewski/raj/blob/master/runtime.js).

## 4. Big Picture

We spent a lot of time understanding the runtime, which is crucial. Every Raj program will follow this same structure so we need this foundation. We understand the runtime and the next step is to build real applications.

Per application, there is one runtime. This runtime represents the "root" top-most construct. In fact, most Raj application need Raj for no more than one line of boilerplate. The rest, following this program pattern, is your own application which you have creative liberty to build. This flexibility and strict pattern make Raj applications fun to write, design, and compose together while creating good programs.

Program composition has concerns:

- Nesting programs
- Shared state
- Parent to child communication
- Child to parent communication

Using our counter to exemplify these concerns:

- We have a program to which we want to add our counter.
- We have two counters and we want them to manipulate the same number.
- We have an initial value other than `0` we want a counter to start from.
- We want to let the parent program know when the counter is high `> 100`.

The simple increment counter we made in React is good enough to show the concepts. For reference:

```js
// counter.js
export default {
  init: [0], // State is an integer to count
  update (message, state) {
    return [state + 1] // Increment the state
  },
  view (state, dispatch) {
    return <div>
      <p>Count is {state}.</p>
      <button onClick={() => dispatch()}>Increment</button>
    </div>
  }
}
```

### 4.1. Nesting programs
We have a program which contains a counter program.

```js
import counter from './counter'

const [counterState, counterEffect] = counter.init
let effect
if (counterEffect) {
  effect = dispatch => {
    counterEffect(message => {
      dispatch({
        type: 'counterMessage',
        data: message
      })
    })
  }
}
const init = [{
  counterState
}, effect]

export default {
  init,
  update (message, state) {
    if (message.type === 'counterMessage') {
      const [
        newCounterState,
        counterEffect
      ] = counter.update(message.data, state.counterState)
      const newState = {...state, counterState: newCounterState}
      let effect
      if (counterEffect) {
        effect = dispatch => {
          counterEffect(message => {
            dispatch({
              type: 'counterMessage',
              data: message
            })
          })
        }
      }
      return [newState, effect]
    }
  },
  view (state, dispatch) {
    return <div>
      <p>This is the root program.</p>
      {counter.view(state.counterState, message => {
        dispatch({
          type: 'counterMessage',
          data: message
        })
      })}
    </div>
  }
}
```

Note:

- The parent program's `init` contains the `init` of the counter.
- The parent program wraps the `init` effect's messages in `{type: 'counterMessage', data}` by intercepting the messages from the counter effect and then re-dispatching them as part of the parent's messages.
- The parent program `update` calls the `update` of the counter with its state and that message we wrap.
- The parent program wraps the `init` effect's messages in `{type: 'counterMessage', data}` by intercepting the messages from the counter effect and then re-dispatching them as part of the parent's messages.
- The parent program `view` calls the `view` of the counter with its state and wraps dispatched messages.

We know counter does not yet have effects, but more useful programs will so we need to know how to handle them.

This is a lot of boilerplate to compose programs. We can leverage the Raj ecosystem library [`raj-compose`](https://github.com/andrejewski/raj-compose) to clean up this plumbing. The most helpful utility in this case is `mapEffect` which does the following:

```js
export function mapEffect (effect, callback) {
  if (!effect) {
    return effect
  }
  return function _mapEffect (dispatch) {
    function intercept (message) {
      dispatch(callback(message))
    }

    return effect(intercept)
  }
}
```

We rewrite the previous example as follows, also pulling out that "counterMessage" wrapper:

```js
import {mapEffect} from 'raj-compose'
import counter from './counter'

const counterMessage = message => ({
  type: 'counterMessage',
  data: message
})

const [counterState, counterEffect] = counter.init
const init = [
  {counterState},
  mapEffect(counterEffect, counterMessage)
]

export default {
  init,
  update (message, state) {
    if (message.type === 'counterMessage') {
      const [
        newCounterState,
        counterEffect
      ] = counter.update(message.data, state.counterState)
      const newState = {...state, counterState: newCounterState}
      return [newState, mapEffect(counterEffect, counterMessage)]
    }
  },
  view (state, dispatch) {
    return <div>
      <p>This is the root program.</p>
      {counter.view(
        state.counterState,
        message => dispatch(counterMessage(message))
      )}
    </div>
  }
}
```

We *could* reduce the boilerplate further by leveraging `raj-compose/mapProgram` or something even more prescriptive. Be wary of optimizing for boilerplate: if we write code that is too concise we sacrifice readability and understanding of our programs.

### 4.2. Shared state
We have two counters and we want them to manipulate the same number.

```js
import {mapEffect} from 'raj-compose'
import counter from './counter'

const counterMessage = message => ({
  type: 'counterMessage',
  data: message
})

const [counterState, counterEffect] = counter.init
const init = [
  {counterState},
  mapEffect(counterEffect, counterMessage)
]

export default {
  init,
  update (message, state) {
    if (message.type === 'counterMessage') {
      const [
        newCounterState,
        counterEffect
      ] = counter.update(message.data, state.counterState)
      const newState = {...state, counterState: newCounterState}
      return [newState, mapEffect(counterEffect, counterMessage)]
    }
  },
  view (state, dispatch) {
    return <div>
      <p>This is the root program.</p>
      {counter.view(
        state.counterState,
        message => dispatch(counterMessage(message))
      )}
      {counter.view(
        state.counterState,
        message => dispatch(counterMessage(message))
      )}
    </div>
  }
}
```

In this case shared state is easy: we are changing the view to render two counters that receive the same state. Shared state needs to be at least as high up the program chain to contain the relevant sub-programs.

### 4.3. Parent to child communication
We have an initial value other than `0` we want a counter to start from.

```js
// back in counter.js
export function initWithCount (initialCount) {
  return [initWithCount]
}
```

```js
import {mapEffect} from 'raj-compose'
import counter, {initWithCount} from './counter'

const counterMessage = message => ({
  type: 'counterMessage',
  data: message
})

const [counterState, counterEffect] = initWithCount(42)
const init = [
  {counterState},
  mapEffect(counterEffect, counterMessage)
]

export default {
  init,
  update (message, state) {
    if (message.type === 'counterMessage') {
      const [
        newCounterState,
        counterEffect
      ] = counter.update(message.data, state.counterState)
      const newState = {...state, counterState: newCounterState}
      return [newState, mapEffect(counterEffect, counterMessage)]
    }
  },
  view (state, dispatch) {
    return <div>
      <p>This is the root program.</p>
      {counter.view(
        state.counterState,
        message => dispatch(counterMessage(message))
      )}
    </div>
  }
}
```

The parent decides the initial count with `initWithCount` instead of using the regular `init`. We are not following the program structure precisely. The parent always has access to the full state of its children so we are free to be creative with how we communicate to the child from the parent. Variations on `init`, `update`, and `view` are good places to start.

We created the `initWithCount` which returns the provided number in an array. This may seem like overkill when we *could* do `{counterState: 42}` based on what we know about how the counter's implementation. Work with a child's state through provided methods instead of via direction manipulation. Having these contracts communicated by methods allows the implementation details of the counter to change without breaking the parent. For example, if the counter ever adds an initial effect we will not have to change the parent(s) that use it.

### 4.4. Child to parent communication
We want to let the parent program know when the counter is high `> 100`.

```js
// back in counter.js
export function isCountHigh (state) {
  return state > 100
}
```

```js
import {mapEffect} from 'raj-compose'
import counter, {isCountHigh} from './counter'

const counterMessage = message => ({
  type: 'counterMessage',
  data: message
})

const [counterState, counterEffect] = counter.init
const init = [
  {counterState},
  mapEffect(counterEffect, counterMessage)
]

export default {
  init,
  update (message, state) {
    if (message.type === 'counterMessage') {
      const [
        newCounterState,
        counterEffect
      ] = counter.update(message.data, state.counterState)
      const newState = {...state, counterState: newCounterState}
      if (isCountHigh(newCounterState)) {
        // TODO: do something because the count is too dang high
      }
      return [newState, mapEffect(counterEffect, counterMessage)]
    }
  },
  view (state, dispatch) {
    return <div>
      <p>This is the root program.</p>
      {counter.view(
        state.counterState,
        message => dispatch(counterMessage(message))
      )}
      {counter.view(
        state.counterState,
        message => dispatch(counterMessage(message))
      )}
    </div>
  }
}
```

Again we use the same strategy of having the counter provide a method which the parent can call. Here after every counter `update` the parent can check if the counter is too high and act appropriately. Having the same communication pattern work for both parent-to-child and child-to-parent is nice.

Composition is an art. Raj gives you the freedom to be creative with how you fit these pieces together. When you do put a program into the runtime it does have to follow the program pattern but subprograms can glue together to best fit your application.

The `raj-compose` library has recommended composition utilities worth getting familiar with. If your application goes Single-Page Application (SPA) check out the ecosystem [`raj-spa`](https://github.com/andrejewski/raj-spa) program that uses `raj-compose` and the composition patterns above.

## 5. Prove It
We have touched on a lot. The best for last is here. The main focus of Raj is testability and we should look at the advantages of architecting our programs the way the runtime makes us.

Testing `init` is a deep equal equality check. Testing `update` is constructing a message and input state and doing a deep equality check on the output. Test the `view` and effects by what messages they send `dispatch`. Testing is consistent and simple.

Giving up side-effects to the runtime also means the rest of your code can be synchronous. Most business logic you write will be input-output, testable with unit tests which are easier to reason about and much faster to run than their asynchronous counterparts.

### 5.1. Debugging
Raj has a terrific debugging experience due to how small it is. Application stack traces are almost entirely of code belonging to the application. Coming from frameworks where the stack traces can easily be hundreds of functions deep, all irrelevant to the problem at hand, the signal to noise ratio is amazing stepping through a Raj program.

Leveraging composition patterns we also can build reusable and powerful debugging utilities.

A question newcomers to Raj ask is, "How can I get access to the current state?" This is a fair question because other state-management solutions offer those APIs. The reason Raj does not is because it is an anti-pattern to avoid in everything but development. The reason for this is contract boundaries between programs. It would be too easy for a programmer to mistakenly make assumptions about the running program from the outside, relying on specifics of the app state that may change over time. Thus Raj does not offer that foot-gun, but you can add it at your own risk.

```js
function tapProgram (program, onChange) {
  return {
    ...program,
    view (model, dispatch) {
      onChange(model)
      return program.view(model, dispatch)
    }
  }
}
```

This `tapProgram` is a high-order-program (HOP), a function which accepts a Raj program as input and returns a Raj program. Anytime the program's state changes, we call `onChange()` with the new state. Using this HOP, we could for example have the current program state set on `window.app` via:

```js
import { tapProgram } from './above-snippet'
import { myProgram } from './my-app'
import { program } from 'raj/runtime'

const newProgram = tapProgram(myProgram, state => {
  window.app = state
})

program(newProgram)
```

Another question is, "How can I do error handling?" This is important for production applications which must adapt to and record errors. We can use another high-order-program `errorProgram` to trap program errors:

```js
function errorProgram (program, errorView) {
  const [programModel, programEffect] = program.init
  const init = [
    { hasError: false, error: null, programModel },
    programEffect
  ]

  function update (msg, model) {
    let change
    try {
      change = program.update(msg, model.programModel)
    } catch (error) {
      return [{ ...model, hasError: true, error }]
    }

    const [programModel, programEffect] = change
    return [{ ...model, programModel }), programEffect]
  }

  function view (model, dispatch) {
    return model.hasError
      ? errorView({ error: model.error })
      : program.view(model.programModel, dispatch)
  }

  let done
  if (program.done) {
    done = function (model) {
      return program.done(model.programModel)
    }
  }

  return { init, update, view, done }
}
```

We can catch errors that happen in the `update()` of all programs within `errorProgram` and display an error view based on the thrown error. Error recording is not demonstrated, but follows the same composition pattern. Also note that `errorProgram` is view-library independent.

The pinnacle byproduct of this highly testable architecture is the ecosystem [`raj-web-debugger`](https://github.com/andrejewski/raj-web-debugger). Leveraging the program pattern, we get a time traveling debugger for free. We record every state in our application and pause, play, rewind, and fast-forward them at will while developing. In two line changes, the debugger HOP can wrap any Raj program:

```diff
import { program } from 'raj/runtime'
import { myProgram } from './my-app'
+ import debug from 'raj-web-debugger'

- program(myProgram)
+ program(debug(myProgram))
```

## 6. Elm Street
Raj adapts the Elm Architecture for JavaScript. Trying [Elm](http://elm-lang.org/) is highly recommended and Raj serves to bring its architecture to JavaScript until Elm is ready for JavaScript's much wider community.

Notably, Elm and Raj handle subscriptions and side-effects differently. In Raj any side-effect can be a subscription. In Elm there are commands (single dispatch) and subscriptions (multi-dispatch). In Raj you write a subscription to receive a message every interval of time like this:

```js
export function everyTime (milliseconds, tagger) {
  let intervalId
  return {
    effect (dispatch) {
      intervalId = setInterval(() => {
        dispatch(tagger(Date.now()))
      }, milliseconds)
    },
    cancel () {
      if (intervalId) {
        clearInterval(intervalId)
      }
    }
  }
}
```

In Elm, the [same subscription](http://package.elm-lang.org/packages/elm-lang/core/latest/Time#every) uses [effect managers](https://github.com/elm-lang/core/blob/b06aa4421f9016c820576eb6a38174b6137fe052/src/Time.elm#L164-L243) and requires help from the [low-level Elm runtime](https://github.com/elm-lang/core/blob/b06aa4421f9016c820576eb6a38174b6137fe052/src/Native/Time.js#L10) to work. Elm's solution fits its language. The Raj solution fits JavaScript.

Expect Elm to develop into a language that makes Raj obsolete as it solves the harder problems of client application development. Until then Raj brings Elm's great architectural patterns to you today.

## 7. Game Over
Holy crap! You made it to the end. That is impressive. I hope this has prepared you to build an application with Raj. Sincerely, thank you for reading.

---

A personal note from [Chris](https://twitter.com/compooter):

> Raj is not a product I am trying to sell. I believe it is the right tool for building client applications. I am excited to build applications with Raj and Elm, please join me.
>
>  Please star [Raj on Github](https://github.com/andrejewski/raj). It took months of careful planning and development to get us here and every star is a dopamine spike for me.
>
> If you plan on or are building an application in Raj, please reach out to me or visit [my team’s website](https://www.bitovi.com/?utm_source=raj_by_example) to see if we can help you build your project or train your team.
