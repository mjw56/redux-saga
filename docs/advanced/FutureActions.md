# Pulling future actions

Until now we've used the helper function `takeEvery` in order to spawn a new task on
each incoming action. This mimics somewhat the behavior of redux-thunk: each time a
Component, for example, invokes a `fetchProducts` Action Creator, the Action Creator will
dispatch a thunk to execute the control flow.

In reality, `takeEvery` is just a helper function built on top of the lower level and more
powerful API. In this section we'll see a new Effect, `take`, which makes it possible to build complex
control flow by allowing total control of the action observation process.

## A simple logger

Let's take a simple example of a Saga that watches all actions dispatched to the store and 
logs them to the console.

Using `takeEvery('*')` (with the wildcard `*` pattern) we can catch all dispatched actions regardless
of their types.

```javascript
import { takeEvery } from 'redux-saga'

function* watchAndLog(getState) {
  yield* takeEvery('*', function* logger(action) {
    console.log('action', action)
    console.log('state after', getState())
  })
}
```

Now let's see how to use the `take` Effect to implement the same flow as above

```javascript
import { take } from 'redux-saga/effects'

function* watchAndLog(getState) {
  while(true) {
    const action = yield take('*')
    console.log('action', action)
    console.log('state after', getState())
  })
}
```

The `take` is just like `call` and `put` we saw earlier. It creates another command object
that tells the middleware to wait for a specific action. The resulting behavior of the `call` Effect is the same as when the middleware suspends the Generator until a Promise resolves. In the `take`
case it'll suspend the Generator until a matching action is dispatched. In the above example
`watchAndLog` is suspended until any action is dispatched.

Note how we're running an endless loop `while(true)`. Remember this is a Generator function,
which doesn't have a run-to-completion behavior. Our Generator will block on each iteration
waiting for an action to happen.

Using `take` has a subtle impact on how we write our code. In the case of `takeEvery` the invoked
tasks have no control on when they'll be called. They will be invoked again and again on each matching
action. They also have no control on when to stop the observation.

In the case of `take` the control is inversed. Instead of the actions being *pushed* to the
handler tasks, the Saga is *pulling* the action by itself. It looks as if the Saga is performing
a normal function call `action = getNextAction()` which will resolve when the action is
dispatched.

This inversion of control allows us to implement control flows that are non-trivial to do with
the traditional *push* approach.

As a simple example, suppose that in our Todo application, we want to watch user actions
and show a congratulation message when the User has created his three first Todos.

```javascript
import { take, put } from 'redux-saga/effects'

function* watchFirstThreeTodosCreation() {
  for(let i = 0; i < 3; i++) {
    const action = yield take('TODO_CREATED')
  }
  yield put({type: 'SHOW_CONGRATULATION'})
}
```

Instead of a `while(true)` we're running a `for` loop which will iterate only three times. After
taking the first three `TODO_CREATED` actions, `watchFirstThreeTodosCreation` will cause the
application to display a congratulation message then terminate. This means the Generator will be
garbage collected and no more observation will take place.

Another benefit of the pull approach is that we can describe our control flow using a familiar
synchronous style. For example, suppose we want to implement a login flow with 2 actions `LOGIN`
and `LOGOUT`. Using `takeEvery` (or redux-thunk) we'll have to write 2 separate tasks (or thunks) : one for
`LOGIN` and the other for `LOGOUT`.

The result is that our logic is now spread in 2 places. In order for someone reading our code to
understand what's going on, he has to read the source of the 2 handlers and make the link
between the logic in both. It means he has to rebuild the model of the flow in his head
by rearranging mentally the logic placed in various places of the code in the correct
order.

Using the pull model we can write our flow in the same place instead of handling the same action repeatedly.

```javascript
function* loginFlow() {
  while(true) {
    yield take('LOGIN')
    // ... perform the login logic
    yield take('LOGOUT')
    // ... perform the logout logic
  }
}
```

The `loginFlow` Saga more clearly conveys the expected action sequence. It knows that the `LOGIN` action should always be followed by
a `LOGOUT` action. And that `LOGOUT` is always followed by a `LOGIN` (a good UI should always enforce
a consistent order of the actions, by hiding or disabling unexpected action).
