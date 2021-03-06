# redux-saga-fetch  [![Build Status](https://travis-ci.org/dat2/redux-saga-fetch.svg?branch=master)](https://travis-ci.org/dat2/redux-saga-fetch) [![codecov](https://codecov.io/gh/dat2/redux-saga-fetch/branch/master/graph/badge.svg)](https://codecov.io/gh/dat2/redux-saga-fetch/)
A saga that helps reduce http duplication when writing `redux-saga` code.

### [Full API Documentation](docs/API.md)

# Getting Started

## Install

```
$ npm install --save redux-saga-fetch
```

or

```
$ yarn add redux-saga-fetch
```

## Usage Example
Let's say you read the `redux-saga` readme, and start implementing the 3 kinds
of actions `USER_FETCH_REQUESTED`,`USER_FETCH_SUCCEEDED` and `USER_FETCH_FAILED`.
Then you start to create the `userSaga`. You start copy/pasting this `userSaga`
for all of your objects, and start thinking "This is not very DRY", and maybe
even give up `redux-saga`. `redux-saga-fetch` is here to save the day!

Let's create a simple actions file for now. This will use the same actions from
the `redux-saga` readme.

#### `actions.js`
```javascript
import { get } from 'redux-saga-fetch';

// this is where the magic happens
// the saga will dispatch the actions (either userFetchSucceeded oruserFetchFailed) automatically
// as the fetch completes.
export function userFetchRequested(userId) {
    return get(`/users/${userId}`, {
        success: response => response.json().then(userFetchSucceeded),
        fail: userFetchFailed
    })
}

export function userFetchSucceeeded(user) {
    return { type: 'USER_FETCH_SUCCEEDED', user: user }
}

export function userFetchFailed(e) {
    return { type: 'USER_FETCH_FAILED', message: e.message }
}
```

This is the `main.js` taken from the `redux-saga` readme.

#### `main.js`
```diff
import { createStore, applyMiddleware } from 'redux'
import createSagaMiddleware from 'redux-saga'
+import { saga as fetchSaga } from 'redux-saga-fetch'

import reducer from './reducers'
import mySaga from './sagas'

// create the saga middleware
const sagaMiddleware = createSagaMiddleware()
// mount it on the Store
const store = createStore(
  reducer,
  applyMiddleware(sagaMiddleware)
)

// then run the saga
-sagaMiddleware.run(mySaga)
+sagaMiddleware.run(fetchSaga)

// render the application
```

And there you have it! `redux-saga-fetch` removes the need to write an entire
saga for basic use cases.

### Responding to Status Codes
If you want to dispatch different actions based on your status code, you can easily check the `.status` property of the [Response](https://developer.mozilla.org/en-US/docs/Web/API/Response) object like this:

```diff
import { get } from 'redux-saga-fetch';

// this is where the magic happens
// the `get` is an action creator, that takes a url and 2 important functions
// success: an action creator that can return a Promise, or an action
// fail: an action creator that catches any errors, either in fetch or in the
//       success action creator
export function userFetchRequested(userId) {
    return get(`/users/${userId}`, {
-        success: response => response.json().then(userFetchSucceeded),
+        success: response => response.status == 401 ?
+                     notAuthorized(userId) :
+                     response.json().then(userFetchSucceeded)
        fail: userFetchFailed
    })
}

export function userFetchSucceeeded(user) {
    return { type: 'USER_FETCH_SUCCEEDED', user: user }
}

export function userFetchFailed(e) {
    return { type: 'USER_FETCH_FAILED', message: e.message }
}

+ export function notAuthorized(userId) {
+    return { type: 'NOT_AUTHORIZED', userId: userId }
+ }
```

The `saga` calls `Promise.resolve` on whatever you return from the `success` callback, so you can just return the action directly.

### Starting other actions on completion
If you have a more complex use case where you need a successful action to start
some other asynchronous actions, you can still use `redux-thunk` for that.

Add the thunk middleware to your store.
#### `main.js`
```diff
import { createStore, applyMiddleware } from 'redux'
import createSagaMiddleware from 'redux-saga'
+import thunk from 'redux-thunk'
import { saga as fetchSaga } from 'redux-saga-fetch'

import reducer from './reducers'
import mySaga from './sagas'

// create the saga middleware
const sagaMiddleware = createSagaMiddleware()
// mount it on the Store
const store = createStore(
  reducer,
-  applyMiddleware(sagaMiddleware)
+  applyMiddleware(sagaMiddleware, thunk)
)

// then run the saga
sagaMiddleware.run(fetchSaga)

// render the application
```

Then update your `actions.js` file to return a thunk for the `USER_FETCH_SUCCEEDED`
rather than a single action.

#### `actions.js`
```diff
-export function userFetchSucceeeded(user) {
-    return { type: 'USER_FETCH_SUCCEEDED', user: user }
-}
+export function userFetchSucceeded(uesr) {
+    return function(dispatch) {
+        dispatch({ type: 'USER_FETCH_SUCCEEDED', user: user })
+        // start kicking off some other actions
+        dispatch({ type: 'FETCH_TODOS_FOR_USER', user: user })
+        dispatch({ type: 'FETCH_FRIENDS_FOR_USER', user: user });
+    }
+}
```

This allows you to set up a complex chain of asynchronous actions very easily!
