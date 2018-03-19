# Raj by Chris

The main Raj by Example guide focuses on the Raj framework, building a foundation around the runtime.
It only makes passing reference to tools and packages in the Raj ecosystem, in an attempt to be agnostic.
The ecosystem is big enough that I cannot vet everything, and I only use so many tools in my own projects.
This guide demonstrates my day-to-day use of Raj and the pieces of the ecosystem that solve my problems.

Consider the main guide as a source of truth with advice much more proven and applicable.
Consider this guide to be an articulation of my personal preferences.

## Contents

1. [Ecosystem tools](#1-ecosystem-tools)
2. [File structure](#2-file-structure)
3. [Data types](#3-data-types)
4. [Multiple pages](#4-multiple-pages)

## 1. Ecosystem tools
The tools I use depend on the application I am building.

### Tools for anything
- [`raj-compose`](https://github.com/andrejewski/raj-compose)
- [`raj-react`](https://github.com/andrejewski/raj-react)
- [`tagmeme`](https://github.com/andrejewski/tagmeme)

### Tools for SPAs
- [`raj-spa`](https://github.com/andrejewski/raj-spa)
- [`tagged-routes`](https://github.com/andrejewski/tagged-routes)

I think `raj-compose` is the most important Raj package.
Originally, it existed inside the main `raj` package but was split out to give me more room to experiment with composition.
I cannot imagine being effective without it.

Tagmeme is the most useful general purpose tool I use in Raj.
It has really good error messages and API for tagged unions which I use for messages in my programs.

## 2. File structure
I follow a pattern advocated by Elm creator Evan Czaplicki in [The Life of a File [video]](https://www.youtube.com/watch?v=XpDsk374LDE).
Each program gets its own file.
We build out this program in its single file until we start to see common patterns, sub-programs, and/or data structures form.
Those are put into their own files.
A file can get pretty large, but we are only building what we need.
I don't have to make a few small files per program upfront, which I do not care for in other frameworks.

Inside a program file, it is the same pattern every time:

1. Imports
1. Data layer
1. Messages
1. Business logic
1. View
1. Assemble

More concretely:

```js
// IMPORTS
import React from 'react'
import { union } from 'tagmeme'
import { assembleProgram } from 'raj-compose'

// DATA LAYER
// This is where all effects/subscriptions are located.
const data = dataOptions => ({
  signIn: ({ username, password }) => dispatch => {
    // Use dataOptions to send credentials to server...
  }
})

// MESSAGES
// I defined messages after data because the data layer should not
//   be aware of program specific messages.
// Using raj-compose/mapEffect(data.effect, Msg.Message) in logic
//   is how I encapsulate the data handling.
const Msg = union(['SetUsername', 'SetPassword', 'SignIn'])

// BUSINESS LOGIC
// Every state transition happens here.
// This function returns init, update, and optionally done
function logic (data, logicOptions) {
  const init = [{
    username: '',
    password: ''
  }]

  function update (msg, model) {
    return Msg.match(msg, {
      SetUsername: username => [{ ...model, username }],
      SetUsername: password => [{ ...model, password }],
      SignIn: username => [
        model,
        data.signIn({
          username: model.username,
          password: model.password
        })
      ],
    })
  }

  return { init, update }
}

// VIEW
function view (model, dispatch, viewOptions) {
  return <form onSubmit={e => {
    e.preventDefault()
    dispatch(Msg.SignIn())
  }}>
    <input type='text' onChange={e => dispatch(Msg.SetUsername(e.target.value))} />
    <input type='password' onChange={e => dispatch(Msg.SetPassword(e.target.value))} />
    <input type='submit' value='Sign in' />
  </form>
}

// ASSEMBLE
// This is the main export of the file, putting the program together.
// The dataOptions are used for dependency injection
// The logicOptions are used for configuration and passing child programs
// The viewOptions are for passing child views
export function makeProgram ({ dataOptions, logicOptions, viewOptions }) {
  return assembleProgram({
    data,
    dataOptions,
    logic,
    logicOptions,
    view,
    viewOptions
  })
}
```

I like this pattern because:

- The data layer is the only place that is asynchronous.
- Concerns are well separated. It is clear where code fits.
- All parts of the file can be exports and tested individually.

## 3. Data types
JavaScript doesn't have immutability by default, but I choose to write immutable code.
It is much easier to reason about and the performance differences are negligible for most work.

I do not use any special immutability libraries.
I fight against the mutability of JavaScript with heavy use of object/array rest and spread.
I do not use the newer, mutable JavaScript data structures `Set` and `Map`.
There are exceptions.

Raj does not force any data pattern.
You can use any library or helper you prefer.
I don't have enough experience with any.

The `Result` type is a tagged union of `Ok` (success) and `Err` (failure):

```js
import { union } from 'tagmeme'
export const Result = union(['Ok', 'Err'])
```

This is useful for making sure I handle error conditions in my business logic.
It is very common for my data layer to dispatch `Result` values.

## 4. Multiple pages
For single-page apps I have application structures like this:

```
<root>
  ? <authentication>
    |
    + <sign-in>
    + <sign-up>
  : <application>
    |
    + <header / footer / sidebar >
    + <raj-spa>
      |
      + <resource-show>
      + <resource-hide>
      + ...
      + <settings-page>
```

The root program first shows authentication, which will make sure the user is logged in, showing sign-in and/or sign-up programs.
Once the user is authenticated, the root will switch to showing the application.
The application will combine things that are page-independent (header/footer) with the current page provided by `raj-spa`.
This is often done using `raj-compose/batchPrograms`.

Every page that is dependent on the URL parameters and query is a child of the `raj-spa` program.
Those programs very often have sub-programs for different sections of their page, once they grow large enough.
I do dependency injection from the root down using the `makeProgram` style.
It is a bit tedious, but once we reach `raj-spa` passing dependencies is not too bad considering how easy testing can be.

My setups for `raj-spa` leverage a custom built router: hash-change, push-state, or manual in the case of React Native.
There are tons of history libraries but all `raj-spa` needs is a subscription effect that dispatches a URL string.
In `raj-spa/getRouteProgram` the URL string runs through `tagged-routes/getRouteForURL`, which gives me a tag to pattern match.
I pattern match and pass along the relevant URL parameters and program dependencies to the particular `makeProgram()`.

A lot of routing tools like to do the URL tracking, URL parsing, and URL routing in a single step. I treat them as separate steps to compose, test, and debug how I feel is appropriate.

More concretely:

```js
import spa from 'raj-spa'
import { createRoutes } from 'tagged-routes'
import { makeProgram as makeAppProgram } from './app'
import { makeProgram as makeAppListProgram } from './apps'
import { makeProgram as makeSettingProgram } from './settings'

const { Route, getRouteForURL } = createRoutes(
  {
    AppList: '/apps',
    AppMain: '/apps/:appId',
    Settings: '/settings'
  },
  'NotFound'
)

const blankPage = name => ({
  init: [],
  update: () => [],
  view () {
    return <h1>{name}</h1>
  }
})

export function makeProgram ({ dataOptions, customRouter }) {
  return spa({
    router: customRouter,
    getRouteProgram (urlString) { 
      return Route.match(
        getRouteForURL(urlString),
        {
          AppList: () => makeAppListProgram({ dataOptions }),
          AppMain: ({ routeParams: { appId } }) =>
            makeAppProgram({ dataOptions, appId }),
          Settings: () => makeSettingsProgram({ dataOptions }),
          NotFound: () => blankPage('Page not found')
        }
      )
    },
    initialProgram: blankPage('Hello')
  })
}
```

I usually will pull the routing table into its own file to use `tagged-routes/getURLForRoute` to construct links from any program.

---

Thank you for reading. I hope this was useful.