# redux-logic

Redux middleware for organizing business logic and action side effects.

> "I wrote the rxjs code so you won't have to."

[![Build Status](https://secure.travis-ci.org/jeffbski/redux-logic.png?branch=master)](http://travis-ci.org/jeffbski/redux-logic) [![Codacy Grade Badge](https://img.shields.io/codacy/grade/3687e7267e6d466b9d226c22b24f0061.svg)](https://www.codacy.com/app/jeff-barczewski/redux-logic) [![Codacy Coverage Badge](https://img.shields.io/codacy/coverage/3687e7267e6d466b9d226c22b24f0061.svg)](https://www.codacy.com/app/jeff-barczewski/redux-logic) [![NPM Version Badge](https://img.shields.io/npm/v/redux-logic.svg)](https://www.npmjs.com/package/redux-logic)


You declare some behavior that wraps your code providing things like filtering, cancelation, limiting, etc., then write just the simple business logic code that runs in the center.

Inspired by redux-observable epics, redux-saga, and custom redux middleware.

## tl;dr

One place to keep all of your business logic and side effects with redux

With simple code you can:

 - validate, verify, auth check actions and allow/reject or modify actions
 - transform - augment/enhance/modify actions
 - process - async processing and dispatching, orchestration, I/O (ajax, REST, web sockets, ...)

Built-in declarative functionality

 - filtering, cancellation, takeLatest, throttling, debouncing


## Quick Example

This is an example of logic which will listen for actions of type FETCH_POLLS and it will perform ajax request to fetch data for which it dispatches the results (or error) on completion. It supports cancellation by allowing anything to send an action of type CANCEL_FETCH_POLLS. It also uses `take latest` feature that if additional FETCH_POLLS actions come in before this completes, it will simply cancel the previous requests and just use the latest.

The developer can just declare the type filtering, cancellation, and take latest behavior, no code needs to be written for that. That leaves the developer to focus on the real business requirements which are invoked in the process hook.

```js
const fetchPollsLogic = createLogic({

  // declarative built-in functionality wraps your code
  type: FETCH_POLLS, // only apply this logic to this type
  cancelType: CANCEL_FETCH_POLLS, // cancel on this type
  latest: true, // only take latest

  // your code here, hook into one or more of these execution
  // phases: validate, transform, and/or process
  process({ getState, action }, dispatch) {
    axios.get('https://survey.codewinds.com/polls')
      .then(resp => resp.data.polls)
      .then(polls => dispatch({ type: FETCH_POLLS_SUCESS,
                                payload: polls }))
      .catch(({ statusText }) =>
             dispatch({ type: FETCH_POLLS_FAILED, payload: statusText,
                        error: true })

  }
});
```

## Table of contents

 - <a href="#goals">Goals</a>
 - <a href="#usage">Usage</a>
 - <a href="./docs/api.md">Full API</a>
 - <a href="#examples">Examples</a> - [JSFiddle](#jsfiddle-live-examples) and [full examples](#full-examples)
 - <a href="#comparison-summaries">Comparison summaries</a> to <a href="#compared-to-fat-action-creators">fat action creators</a>, <a href="#compared-to-redux-thunk">thunks</a>, <a href="#compared-to-redux-observable">redux-observable</a>, <a href="#compared-to-redux-saga">redux-saga</a>, <a href="#compared-to-custom-redux-middleware">custom middleware</a>, <a href="#compared-to-sam-or-pal-pattern">SAM/PAL pattern</a>
 - <a href="#other">Other</a> - features under consideration, todo, inspiration, license

## Goals

 - organize business logic keeping action creators and reducers clean
   - action creators are light and just post action objects
   - reducers just focus on updating state
   - intercept and perform validations, verifications, authentication
   - intercept and transform actions
   - perform async processing, orchestration, dispatch actions
 - wrap your core business logic code with declarative behavior
   - filtered - apply to one or many action types or even all actions
   - cancellable - async work can be cancelled
   - limiting (like taking only the latest, throttling, and debouncing)
 - features to support business logic and large apps
   - have access to full state to make decisions
   - easily composable to support large applications
   - inject dependencies into your logic, so you have everything needed in your logic code
   - dynamic loading of logic for splitting bundles in your app
   - your core logic code stays focussed and simple, don't use generators or observables unless you want to.
   - create subscriptions - streaming updates
   - easy testing - since your code is just a function it's easy to isolate and test


## Usage

```bash
npm install redux-logic --save
```

```js
// in configureStore.js
import { createLogicMiddleware } from 'redux-logic';
import rootReducer from './rootReducer';
import arrLogic from './logic';

const deps = { // optional injected dependencies for logic
  // anything you need to have available in your logic
  A_SECRET_KEY: 'dsfjsdkfjsdlfjls',
  firebase: firebaseInstance
};

const logicMiddleware = createLogicMiddleware(arrLogic, deps);

const middleware = applyMiddleware(
  logicMiddleware
);

const enhancer = middleware; // could compose in dev tools too

export default function configureStore() {
  const store = createStore(rootReducer, enhancer);
  return store;
}


// in logic.js - combines logic from across many files, just
// a simple array of logic to be used for this app
export default [
 ...todoLogic,
 ...pollsLogic
];


// in polls/logic.js

const validationLogic = createLogic({
  type: ADD_USER,
  validate({ getState, action }, allow, reject) {
    const user = action.payload;
    if (!getState().users[user.id]) { // can also hit server to check
      allow(action);
    } else {
      reject({ type: USER_EXISTS_ERROR, payload: user, error: true })
    }
  }
});

const addUniqueId = createLogic({
  type: '*',
  transform({ getState, action }, next) {
    // add unique tid to action.meta of every action
    const existingMeta = action.meta || {};
    const meta = {
      ...existingMeta,
      tid: shortid.generate()
    },
    next({
      ...action,
      meta
    });
  }
});

const fetchPollsLogic = createLogic({
  type: FETCH_POLLS, // only apply this logic to this type
  cancelType: CANCEL_FETCH_POLLS, // cancel on this type
  latest: true, // only take latest
  process({ getState, action }, dispatch) {
    axios.get('https://survey.codewinds.com/polls')
      .then(resp => resp.data.polls)
      .then(polls => dispatch({ type: FETCH_POLLS_SUCESS,
                                payload: polls }))
      .catch(({ statusText }) =>
             dispatch({ type: FETCH_POLLS_FAILED, payload: statusText,
                        error: true })

  }
});

// pollsLogic
export default [
  validationLogic,
  addUniqueId,
  fetchPollsLogic
];

```

## Full API

See the [docs for the full api](./docs/api.md)

## Examples

### JSFiddle live examples

 - [async fetch - single page](http://jsfiddle.net/jeffbski/954g5n7h/)

### Full examples

 - [async-fetch-vanilla](./examples/async-fetch-vanilla) - async fetch example using axios
 - [async-fetch](./examples/async-fetch) - async fetch example using axios and redux-actions
 - [countdown](./examples/countdown) - a countdown timer implemented with setInterval
 - [countdown-obs](./examples/countdown-obs) - a countdown timer implemented with Rx.Observable.interval
 - [form-validation](./examples/form-validation) - form validation and async post to server using axios, displays updated user list
 - [single-file](./examples/single-file) - async fetch example with all code in a single file

## Comparison summaries

Following are just short summaries to compare redux-logic to other approaches.

For a more detailed comparison with examples, see by article in docs, [Where do I put my business logic in a React-Redux application?](./docs/where-business-logic.md).


### Compared to fat action creators

 - no easy way to cancel or do limiting like take latest with fat action creators
 - action creators would not have access to the full global state so you might have to pass down lots of extra data that isn't needed for rendering. Every time business logic changes might require new data to be made available
 - no global interception using just action creators - applying logic or transformations across all or many actions
 -  Testing components and fat action creators may require running the code (possibly mocked).

### Compared to redux-thunk

 - With thunks business logic is spread over action creators
 - With thunks there is not an easy way to cancel async work nor to perform limiting (take latest, throttling, debouncing)
 - no global interception with thunks - applying logic or transformations across all or many actions
 - Testing components and thunked action creators may require running the code (possibly mocked).


### Compared to redux-observable

 - redux-logic doesn't require the developer to use rxjs observables. It uses observables under the covers to provide cancellation, throttling, etc. You simply configure these parameters to get this functionality. You can still use rxjs in your code if you want, but not a requirement.
 - redux-logic hooks in before the reducer stack like middleware allowing validation, verification, auth, tranformations. Allow, reject, tranform actions before they hit your reducers to update your state as well as accessing state after reducers have run. redux-observable hooks in after the reducers have updated state so they have no opportuntity to prevent the updates.

### Compared to redux-saga

 - redux-logic doesn't require you to code with generators
 - redux-saga relies on pulling data (usually in a never ending loop) while redux-logic and logic are reactive, responding to data as it is available
 - redux-saga runs after reducers have been run, redux-logic can intercept and allow/reject/modify before reducers run also as well as after


### Compared to custom redux middleware

 - Both are fully featured to do any type of business logic (validations, tranformations, processing)
 - redux-logic already has built-in capabilities for some of the hard stuff like cancellation, limiting, dynamic loading of code. With custom middleware you have to implement all functionality.
 - No safety net, if things break it could stop all of your future actions
 - Testing requires some mocking or setup

### Compared to SAM or PAL Pattern

 - With redux-logic you can implement the SAM / PAL pattern without giving up React and Redux. Namely you can separate out your business logic from your action creators and reducers keeping them thin. redux-logic provides a nice place to accept, reject, and transform actions before your reducers are run. You have access to the full state to make decisions and you can trigger actions based on the updated state as well.
 - Implementing the SAM/PAL pattern on your own requires lots of boilerplate code

<a name="other"></a>

## Other possible feature additions

Some features under consideration.

### Timeout

Additional `createLogic` properties introducing new timeout behavior

 - timeout N ms, default 0 (disables)
 - timeoutType - action type to use in the event of a timeout

### Predetermined type or action creators for dispatch

Introduce new `processOptions` with these properties

 - successType - dispatch this action type using contents of dispatch as the payload (also would work with with promise or observable passed to dispatch)
 - failureType - dispatch this action type using contents of error as the payload, sets error: true (would also work for rejects of promises or error from observable)

The successType and failureType would enable this type of code, where you can simply dispatch a promise or observable that resolves to the payload and rejects on error.

```js
const fetchPollsLogic = createLogic({

  // declarative built-in functionality wraps your code
  type: FETCH_POLLS, // only apply this logic to this type
  cancelType: CANCEL_FETCH_POLLS, // cancel on this type
  latest: true, // only take latest

  processOptions: {
    successType: FETCH_POLLS_SUCCESS, // dispatch this success act type
    failureType: FETCH_POLLS_FAILED // dispatch this failed action type
  },
  process({ getState, action }, dispatch) {
    dispatch( // can just dispatch promise which resolves to payload
      axios.get('https://survey.codewinds.com/polls')
        .then(resp => resp.data.polls);
    )
  }
});
```

### Returning promise or observable with predetermined types/action creators

This allows us to get much of the action details out of the code leaving mostly just business logic. Instead of using dispatch return promise or observable.

Introduce new `processOptions` with these properties

 - useReturn - the returned value of the process function will be dispatched or if it is a promise or observable then the resolve, reject, or observable values will be dispatched applying any successType or failureType logic if defined.
 - successType - dispatch this action type using contents of dispatch as the payload (also would work with with promise or observable)
 - failureType - dispatch this action type using contents of error as the payload, sets error: true (would also work for rejects of promises or error from observable)


The successType and failureType would enable this type of code, where you can simply return a promise or observable that resolves to the payload and rejects on error.

```js
const fetchPollsLogic = createLogic({

  // declarative built-in functionality wraps your code
  type: FETCH_POLLS, // only apply this logic to this type
  cancelType: CANCEL_FETCH_POLLS, // cancel on this type
  latest: true, // only take latest

  processOptions: {
    useReturn: true, // use returned value instead of dispatch function
    // provide action types or action creator functions to be used
    // with the resolved/rejected values from promise/observable returned
    successType: FETCH_POLLS_SUCCESS, // dispatch this success act type
    failureType: FETCH_POLLS_FAILED, // dispatch this failed action type
  },

  // useReturn option allows you to simply return obj, promise, obs
  // not needing to use dispatch directly
  process({ getState, action }) {
    return axios.get('https://survey.codewinds.com/polls')
      .then(resp => resp.data.polls);
    )
  }
});
```

This is pretty nice leaving us with mainly our business logic code that could be easily extracted and called from here.

## Inspiration

redux-logic was inspired from these projects:

 - [redux-observable epics](https://redux-observable.js.org)
 - [redux-saga](http://yelouafi.github.io/redux-saga/)
 - [redux middleware](http://redux.js.org/docs/advanced/Middleware.html)

## Minimized/gzipped size with all deps

(redux-logic only includes the modules of RxJS 5 that it uses)
```
redux-logic.min.js.gz 11KB
```

Note: If you are already including RxJS 5 into your project then the resulting delta will be much smaller.

## TODO

 - more docs
 - more examples
 - evaulate additional features as outlined above

## Get involved

If you have input or ideas or would like to get involved, you may:

 - contact me via twitter @jeffbski  - <http://twitter.com/jeffbski>
 - open an issue on github to begin a discussion - <https://github.com/jeffbski/redux-logic/issues>
 - fork the repo and send a pull request (ideally with tests) - <https://github.com/jeffbski/redux-logic>
 - See the [contributing guide](http://github.com/jeffbski/redux-logic/raw/master/CONTRIBUTING.md)


<a name="license"/>

## License - MIT

 - [MIT license](http://github.com/jeffbski/redux-logic/raw/master/LICENSE.md)
